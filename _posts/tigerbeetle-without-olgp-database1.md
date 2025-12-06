---
layout: post
title: Running TigerBeetle without a control plane database. Part one.
tags: tigerbeetle, accounting, gl
---

TigerBeetle is a database built for financial accounting, and the only record types available are [Accounts](https://docs.tigerbeetle.com/reference/account/) and [Transfers](https://docs.tigerbeetle.com/reference/transfer/). That might be enough for the simplest accounting setup, but not for any realistic financial product.

The way [TigerBeetle solves that](https://docs.tigerbeetle.com/coding/system-architecture/) is by requiring an Online General Purpose (OLGP) database in the control plane that stores metadata and mapping between TigerBeetle’s identifiers and identifiers used by the rest of the systems. This can be done, and the documentation is really nice on guiding you, but... what about the dual write problem?

Here’s an idea: what about running TigerBeetle without the control plane database? I am not saying you should do it, but I wanted to try out if that is possible and the best ways to do it. This is a work in progress.

### Challenge

It depends on the banking and payment card system, but often your account or card is not just a single physical account record. It is a “product” and “product agreement” that links together multiple accounts, conditions, metadata, and other types of records, just to tell you what the current balance is. Years ago, I worked with something we called “analytical accounting.” We had separate accounts for purchases, cashout, refunds, credit/debit transfers, interest, and different kinds of fees. Something similar that you can achieve by analytical systems, but it was running inside the accounting system with the same consistency guarantees.

Accounts and Transfers in TigerBeetle are immutable. You can change credit and debit amounts by posting transactions. You can also update Account `flags` and set or unset `AccountFlags.CLOSED` by (posting a pending transfer or voiding it)[https://docs.tigerbeetle.com/coding/recipes/close-account/]. In practice, you often have more statuses for accounts to accept credits while rejecting debit operations, etc.

### Tools

TigerBeetle gives us 3 fields where we can store arbitrary information for both Accounts and Transfers: `user_data_128`, `user_data_64`, `user_data_32`, containing 16, 8, and 4 bytes of information. There are other fields like `ledger` and `code` that can be used for some things, but generally they should represent the ledger (and the currency) and `code` to distinguish different account and transfer types. And there is one more field: the `id` field itself (128 bits or 16 bytes), which gives us uniqueness checks out of the box.

When your systems use integer IDs, it is a straightforward task. Most likely, you will have UUIDs, and it is easy to convert them to integers and back using Python:

```python
>>> user_data_128 = uuid.uuid4().int
>>> uuid.UUID(int=user_data_128)
UUID('a1833b6a-a185-47aa-90ac-2f78979df3be')
```

If you have textual codes and identifiers, you can even store those with some encoding scheme:

```python
>>> user_data_32 = int.from_bytes("EUR".encode())
>>> user_data_32.to_bytes(3).decode()
'EUR'
```

Further, TigerBeetle provides [`lookup_accounts`](https://docs.tigerbeetle.com/reference/requests/lookup_accounts/) and [`lookup_transfers`](https://docs.tigerbeetle.com/reference/requests/lookup_transfers/) to retrieve Accounts and Transfers by the `id` field. And there are [`query_accounts`](https://docs.tigerbeetle.com/reference/requests/query_accounts/) and [`query_transfers`](https://docs.tigerbeetle.com/reference/requests/query_transfers/) to query Accounts and Transfers by a combination of `user_data_128`, `user_data_64`, `user_data_32`.

### Solution

Dealing with 1:1 relations is easy, just put our UUID in the `id` field.

1:n relations are harder. First, let’s use `user_data_128` to store our UUID and link together multiple accounts by having the same value in that field. Second, there might be multiple cards and accounts for the same client UUID. For that, you can store the counter per client in the `user_data_32` field.

You can find all cards and accounts of a client by running a query on the `user_data_128` field. After choosing the desired card/account, you can retrieve the whole TigerBeetle account set for the card/account product agreement by running a query on `user_data_128` and `user_data_32`.

Something like this:

```python
import os
import uuid
from enum import IntEnum

import tigerbeetle as tb

class Code(IntEnum):
    MAIN_ACCOUNT = 1
    LIMIT_ACCOUNT = 2
    EUR = 978
    STATUS = 9000

def register_new_account_agreement(client_id: uuid.UUID):
    with tb.ClientSync(cluster_id=0, replica_addresses=os.getenv("TB_ADDRESS", "3000")) as client:
        existing = client.query_accounts(
            tb.QueryFilter(user_data_128=client_id.int, ledger=Code.EUR, code=Code.MAIN_ACCOUNT, limit=100)
        )

        seq = (max(account.user_data_32 for account in existing) + 1) if existing else 1

        accounts = [
            tb.Account(
                id=tb.id(),
                user_data_128=client_id.int,
                user_data_32=seq,
                ledger=Code.EUR,
                code=Code.MAIN_ACCOUNT,
                flags=tb.AccountFlags.LINKED,
            ),
            tb.Account(
                id=tb.id(),
                user_data_128=client_id.int,
                user_data_32=seq,
                ledger=Code.EUR,
                code=Code.LIMIT_ACCOUNT,
            ),
        ]
        account_errors = client.create_accounts(accounts)
```

There is a race condition between finding the highest value of the sequence number and assigning it to an account. But that can be solved by running account creation from a single thread or by serialisation through locks in Redis or somewhere else.

This solves the creation of agreements/account sets. But what about updates?

### If all you have is a hammer, everything looks like a nail

You can’t modify any of the Account fields, but you can post a Transfer of `0` amount with no financial impact and store information in any of the Transaction’s user data fields. And to read back the current value, you can ask for the last Transfer of a specific type (`code` field) by using the `limit=1` and `tb.AccountFilterFlags.REVERSED` flag. Like this:

```python
        main, limit = accounts[0], accounts[1]
        transfer_errors = client.create_transfers(
            [
                tb.Transfer(
                    id=tb.id(),
                    debit_account_id=limit.id,
                    credit_account_id=main.id,
                    amount=0,
                    ledger=main.ledger,
                    user_data_128=0b1001001,
                    code=Code.STATUS,
                )
            ]
        )
        ...

        transfers = client.get_account_transfers(
            tb.AccountFilter(
                account_id=main.id,
                limit=1,
                code=Code.STATUS,
                flags=tb.AccountFilterFlags.CREDITS | tb.AccountFilterFlags.REVERSED,
            ),
        )
        print(transfers[0].user_data_128)

```

When one Transfer has too little fields, you can always post two, three or more and retrieve the same amount to reconstruct the information. Not that you should do it, but if two extra fields are the only reason to introduce an OLGP database, you might choose to abuse TigerBeetle to achieve the same.

_To be continued about building a transaction out of multiple Transfers._
