---
layout: post
title: Ledgers are simple, stop spreading FUD!
tags: ledger, accounting, double-entry
---

When you do need thousands and tens of thousands of transactions per second, [TigerBeetle](https://tigerbeetle.com/) is there for you. A few hundred per second [you can do it yourself in ~25 lines of code](https://aivarsk.com/2025/03/14/The-COST-of-double-entry-accounting/). And you can reach thousands but it requires an understanding of databases and sometimes creative approaches.

A double-entry entry transaction can be done with 2 SQL statements in a single database transaction:

1. You insert an entry record into the database with the credit and debit leg. If any accounts do not exist in the database, you will get a foreign key constraint violation, the transaction will roll back and you will report an error to the caller. If it succeeds - you have not taken any locks on the accounts and this can scale until it hits hardware limits.

2. You do a relative update of both account balances and check that the debit side has sufficient balance. This works because the database will first find the credit account and debit account with a sufficient balance. Then it locks both rows. There is no race condition present, the database ensures conditions are still true after holding locks in one way or another depending on the implementation of MVCC. Then it performs the relative update against the current balance while holding the locks. Account records will be there and foreign key constraints have enforced that on step #1. When the debit account has insufficient balance, the database will not find it and will not update it. Therefore you check that exactly 2 rows were updated and rollback the database transaction and report an error to the caller. The database will remove the entry created on step #1 for you.

3. You are done and all you have to do is to commit the successful transaction as soon as possible because locks on both accounts were taken on step #2 and you release them by doing a commit to enable as many concurrent updates as possible.

All of that is a piece of cake with Django:

```python
def transfer(debit: Account, credit: Account, amount: int):
    with transaction.atomic():
        Journal.objects.create(debit=debit, credit=credit, amount=amount)
        count = Account.objects.filter(
            Q(pk=debit.pk, balance__gte=amount) | Q(pk=credit.pk)
        ).update(
            balance=Case(
                When(pk=debit.pk, then=F("balance") - amount),
                When(pk=credit.pk, then=F("balance") + amount),
            )
        )
        if count != 2:
            raise InsufficientBalance()
```

It does get a tiny bit more complicated if you want to track credits and debits separately and implement active/passive, assets/liabilities in different ways, [but it's doable and nothing to be scared about](https://github.com/aivarsk/django-modern-treasury-poc/blob/main/debitcredit/models.py#L38).

Now, the really complicated part is the structure of ledger and deciding how many accounts to keep and through which accounts to move the money to satisfy local laws and your accountants. But that is a different story.
