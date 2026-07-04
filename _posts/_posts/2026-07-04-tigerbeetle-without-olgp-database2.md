---
layout: post
title: Running TigerBeetle without a control plane database. Part two.
tags: tigerbeetle, accounting, gl, transactions, transfers
---

[Part one](https://aivarsk.com/2025/12/06/tigerbeetle-without-olgp-database1/) and some articles in between.

Transactions and transaction lifecycles are complicated. Not to go into the craziness of payment card systems, we can say that all transactions consist of multiple transfers for fees, increasing and checking various limits, and may have reversals.

Here, a transaction is a collection of individual [Transfers](https://docs.tigerbeetle.com/reference/transfer/). Similar to accounts, transfers can be linked together by having the same UUID value in `user_data_128`. All transactions' transfers will have `tb.AccountFlags.LINKED` in `flags` (except the last transfer) to achieve atomic behavior. I will also put the UUID as the `id` of the first transfer to enforce uniqueness and idempotency.


I want to have my transfers arbitrary: that means no clever ID generation schema, and I might choose when to store some transfers of 0 and when to skip to avoid noise. Whenever I need to know the exact transfers and their IDs, I will use the [`lookup_transfers`](https://docs.tigerbeetle.com/reference/requests/lookup_transfers/) and find transfers by `user_data_128`. I will recognize the kind of transfer by the value of `code`. Now, about some more interesting features.


### Usage limits

We want to have daily/weekly/monthly limits for certain transaction types. These are easily implemented using pending transfers with [timeouts](https://docs.tigerbeetle.com/reference/transfer/#timeout): you can decrease the available weekly limit with a pending transfer that times out after 7x24x60x60 seconds. Or if you want calendar limits instead of rolling limits, you can calculate seconds until the end of the current week. TigerBeetle will do it’s best to void the transfer once the timeout expires, but it might do it a second or two later. But it is not a problem: existing payment systems are not precise as well and a drift even of several minutes is acceptable.
It gets a bit tricky when combined with two-phase transactions out of [two-phase transfers](https://docs.tigerbeetle.com/coding/two-phase-transfers/). You have to read all the transfers that were created within the transaction and post all except transfers used for limits.


### Reversals

Some of the transactions might be reversed, and we must enforce idempotency to avoid reversing them twice. (I am ignoring partial reversals at this point). One way is to generate a unique reversal ID outside TigerBeetle and use it. But that leaves the enforcement of “transaction can be reversed once” to the caller.
A better way is to create a dummy “reversed” transfer of 0 in the original transaction in the pending state (here with the code 42):

```python
transfers = [
    tb.Transfer(
        id=tb.id(),
        debit_account_id=sink.id,
        credit_account_id=account.id,
        amount=1,
        ledger=1,
        code=1,
        flags=tb.TransferFlags.LINKED,
    ),
    tb.Transfer(
        id=tb.id(),
        debit_account_id=sink.id,
        credit_account_id=account.id,
        amount=0,
        ledger=1,
        code=42,
        flags=tb.TransferFlags.PENDING,
    ),
]
errors = client.create_transfers(transfers)
```

Once reversal arrives, the first thing is to post (or void) the “reversed” transfer and then proceed with the rest of transfers.

```python
transfers = [
    tb.Transfer(
        id=tb.id(),
        pending_id=*ID of transfer with code 42*,
        flags=tb.TransferFlags.VOID_PENDING_TRANSFER | tb.TransferFlags.LINKED,
    ),
    tb.Transfer(
        id=tb.id(),
        debit_account_id=account.id,
        credit_account_id=sink.id,
        amount=1,
        ledger=1,
        code=1,
    ),
]
errors = client.create_transfers(transfers)
```

TigerBeetle ensures that each transfer can be posted or voided exactly once, and we get our idempotency out of that. Repeated reversals will result with error:

```python
[CreateTransfersResult(index=0, result=<CreateTransferResult.PENDING_TRANSFER_NOT_PENDING: 26>), CreateTransfersResult(index=1, result=<CreateTransferResult.LINKED_EVENT_FAILED: 1>)]
```


### Linked two-phase transfers

There are two features that combine in a bit of a surprising way: (linked events)[https://docs.tigerbeetle.com/coding/linked-events/] and (two-phase transfers)[https://docs.tigerbeetle.com/coding/two-phase-transfers/].
We can create several linked transfers for them to behave atomically: either all succeed, or all fail. And we can utilize two-phase transfers to implement card transaction-like authorization requests and financial advice.
We successfully created some linked transfers in `pending` state. Now to post or void them, each transfer has to be posted or voided individually: the linked transfers no longer behave as linked, although they still have linked flags stored. You can have:
* both posted and pending transfers in the same linked batch initially
* some of the pending ones can be voided
* some of the pending ones can be posted
* some of the pending ons can stay pending

It both helps in some cases (calendar limits) and requires extra work in others.


### Retries

Let’s say we have a transaction that might fail with insufficient balance.  And we follow the principles listed above and use the transaction UUID as the ID of the first transfer to enforce uniqueness and idempotency.

>>> Transfer(id=2155649320254879050885200131040001544, debit_account_id=42, credit_account_id=43, amount=1, pending_id=0, user_data_128=0, user_data_64=0, user_data_32=0, timeout=6, ledger=42, code=42, flags=<TransferFlags.PENDING: 2>, timestamp=0)

<<< CreateTransfersResult(index=0, result=<CreateTransferResult.EXCEEDS_CREDITS: 54>)



In some cases (subscriptions, fees, etc.), we consider an insufficient balance a transient error and want to retry later, when the customer might have a sufficient balance. But here’s the catch: repeating the same transfer with the same ID will complain about a previous failed(!) transfer already existing - [it’s a part of the official documentation](https://docs.tigerbeetle.com/reference/requests/create_transfers/#id_already_failed). This is not what I expected based on previous systems, but OK.


>>>  Transfer(id=2155649320254879050885200131040001544, debit_account_id=42, credit_account_id=43, amount=1, pending_id=0, user_data_128=0, user_data_64=0, user_data_32=0, timeout=6, ledger=42, code=42, flags=<TransferFlags.PENDING: 2>, timestamp=0)
<<< CreateTransfersResult(index=0, result=<CreateTransferResult.ID_ALREADY_FAILED: 68>)

Now doing a lookup for that transfer ID with [lookup_transfers](https://docs.tigerbeetle.com/reference/requests/lookup_transfers/) will return nothing - a transfer was not created, but it exists somehow anyway, like a ghost.



### Recipes vs real life

TigerBeetle has several recipes that utilize pending transfers for doing conditional transfers like [balance invariant transfers](https://docs.tigerbeetle.com/coding/recipes/balance-invariant-transfers/). In real life, there are cases where you have to move money through some accounts for traceability or because the accountants said so. I think we first hit it when doing lower and upper bound checks at the same time.
Here’s a simplified example: we want to transfer 1 from account A to B and then back to A. Works fine with the regular transfers:

```python
transfers = [
    tb.Transfer(
        id=tb.id(),
        debit_account_id=sink.id,
        credit_account_id=account.id,
        amount=1,
        ledger=1,
        code=1,
        flags=tb.TransferFlags.LINKED,
    ),
    tb.Transfer(
        id=tb.id(),
        debit_account_id=account.id,
        credit_account_id=sink.id,
        amount=1,
        ledger=1,
        code=1,
    ),
]
client.create_transfers(transfers)
```

Now we want to do the same with pending (because recipes use pending), but the second transfer fails and cancels the whole batch.

```python
transfers = [
    tb.Transfer(
        id=tb.id(),
        debit_account_id=sink.id,
        credit_account_id=account.id,
        amount=1,
        ledger=1,
        code=1,
        flags=tb.TransferFlags.LINKED | tb.TransferFlags.PENDING,
    ),
    tb.Transfer(
        id=tb.id(),
        debit_account_id=account.id,
        credit_account_id=sink.id,
        amount=1,
        ledger=1,
        code=1,
        flags=tb.TransferFlags.PENDING,
    ),
]
client.create_transfers(transfers)
```

```python
[CreateTransfersResult(index=0, result=<CreateTransferResult.LINKED_EVENT_FAILED: 1>), CreateTransfersResult(index=1, result=<CreateTransferResult.EXCEEDS_CREDITS: 54>)]
```

Turns out balance checks do not use pending credits or debits, even for linked transfers. This breaks my brain, which is used to database transactions. But it kinda makes sense, knowing TigerBeetle implementation, that it’s possible to post or void individual transfers, and there is no way to express that some transfers will *always* be either voided or posted atomically.
Not a huge issue once you wrap your brain around it, but the recipes have to be implemented with regular transfers with manual cleanup, and all “technical” transfers stay in the posting history.

### Error reporting

TigerBeetle reports just the error code and transfer index to the caller. To make a generic and future-proof error handling that does not rely on the ordering of transfers, we have a function that takes errors and transfers and figures out the correct error code by looking at the error code, transfers `code` field and maybe account IDs. Sometimes “CreateTransferResult.EXCEEDS_CREDITS” means there are insufficient funds, sometimes it means exceeded daily limits, sometimes the requested transaction amount is lower than the transaction fees.

_To be continued._
