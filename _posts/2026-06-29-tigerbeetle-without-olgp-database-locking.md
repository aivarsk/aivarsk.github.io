---
layout: post
title: Running TigerBeetle without a control plane database. Locking.
tags: tigerbeetle, accounting, gl, locking
---

Of course, TigerBeetle does not provide anything for locking, but I don’t want to introduce a control-plane database or add Redis to the mix, so let’s build some new nails for the TigerBeetle hammer.

First, we will need a dummy account to use as the second leg for the double-entry accounting.

```python
account = tb.Account(
    id=43,
    ledger=42,
    code=42,
)
errors = client.create_accounts([account])
```

Then we will need a new account for each lock (mutex) with `flags` that prevent a negative balance:

```python
mutex = tb.Account(
    id=42,
    ledger=42,
    code=42,
    flags=tb.AccountFlags.DEBITS_MUST_NOT_EXCEED_CREDITS,
)
errors = client.create_accounts([mutex])
```

Each mutex must be initialized with an initial balance of 1. To make it atomic, we can use the account identifier as the transfer identifier to enforce idempotency. We will get `CreateTransferResult.EXISTS` when the mutex account has been initialized before.

```python
init = tb.Transfer(
    id=42,
    debit_account_id=43,
    credit_account_id=42,
    amount=1,
    ledger=42,
    code=42,
)

errors = client.create_transfers([init])
if errors and errors[0].result != tb.CreateTransferResult.EXISTS:
    raise ValueError(errors)
```

Now we can implement the actual locking. We will implement lock expiration (or Time-To-Live) to avoid crashed or stalled processes from halting the system. TigerBeetle has a [timeout feature](https://docs.tigerbeetle.com/reference/transfer/#timeout) that will come in handy.

Locking will transfer an amount of 1 from the mutex account back to the dummy account with a timeout that will automatically cancel the transfer:

```python
lock = tb.Transfer(
    id=tb.id(),
    debit_account_id=42,
    credit_account_id=43,
    amount=1,
    ledger=42,
    code=42,
    flags=tb.TransferFlags.PENDING,
    timeout=ttl,
)
```

TigerBeetle does not provide a tool to wait for the account balance, so we need a busy-loop that will try to acquire the lock for some time. We will retry the locking transfer when it fails with `CreateTransferResult.EXCEEDS_CREDITS`. The curious detail is that the failing transfer is still persisted (?!) - we need to assign a new transfer ID for each try, or it will fail with an error saying that the transfer already exists. And to avoid polluting TigerBeetle with failed locking attempts, I added a small delay after each failure. But just imagine all our locking attempts stored durably for audit purposes :D

```python
deadline = time.monotonic() + timeout
while time.monotonic() < deadline:
    lock.id = tb.id()
    errors = client.create_transfers([lock])
    if errors:
        if errors[0].result == tb.CreateTransferResult.EXCEEDS_CREDITS:
            time.sleep(0.1)
        else:
            raise ValueError(errors)
    else:
        break
else:
    raise TimeoutError()
```

And now the easy part - releasing the lock is just voiding the pending lock transfer. The only interesting part here is handling the expired transfers - when you have held the lock for a longer time than the Time-To-Live. We might want to report these cases to better adjust the timeout values.

```python
unlock = tb.Transfer(
    id=tb.id(),
    pending_id=lock.id,
    flags=tb.TransferFlags.VOID_PENDING_TRANSFER,
)
errors = client.create_transfers([unlock])
if errors:
    if errors[0].result == tb.CreateTransferResult.PENDING_TRANSFER_EXPIRED:
        pass
    else:
        raise ValueError(errors)
```

Is it crazy? Yes. But does it work? Yes.
