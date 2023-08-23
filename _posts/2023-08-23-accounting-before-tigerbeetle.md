---
layout: post
title: Accounting systems before TigerBeetle
tags: accounting database tigerbeetle
---

TigerBeetle is an interesting project to follow.  It links to interesting papers, it challenges assumptions, algorithms, and architecture of past systems. It satisfies my cravings for low-level programming.

In [one of the presentations](https://youtu.be/LHsjviZJ7PQ), it says "Accounting is the Language of Business" and "SQL is the Language of OLTP" so we get "OLTP Impedance Mismatch". As an example of impedance mismatch, one of the accounting systems had "1 Financial Transaction = 20 SQL Queries". Because double-entry bookkeeping has entries like "Debit Bank Account for $0.02, Credit Customer Account for $0.02" it leads to contention on the bank account which limits the accounting system to "76 Transactions per Second".

But we had systems that handled more than 76 transactions per second on single/dual CPU servers with slow-spinning disks more than 20 years ago. I worked on some more than 10 years ago. How could they do it without relying on TigerBeetle?

## Batch processing by default.

Instead of having the API that accepts transactions one by one, all APIs accept a batch of transactions to process. Processing a single transaction means processing a batch of size 1. But it does not stop there:

- All financial transactions can be processed in a single database transaction.
- Caching: If multiple transactions access the same account, read it from the database just once for the batch.
- Aggregating updates: When multiple transactions update the same account, sum all updates in the application and update the database just once.
- Access the database in batches. In Django ORM terms use joins, `id__in=all_ids`, `.bulk_create`, `.bulk_update`. The number of queries should be constant regardless if we process a batch with 1 or 100 transactions.

## Number of queries

20 queries per transaction is just too much, it can be solved with a better database model or less naive code. The last accounting system I worked on had:

- 2 queries for reading agreements and accounts instead of 1 to better fit the database model and simplify optimistic locking
- 1 query for account-specific conditions (interest, fees, etc.)
- 1 query for generating transaction IDs from the sequence; could use ULIDs (Universally unique Lexicographically sortable IDentifier) instead
- 1 query for updating all accounts
- 1 query for writing a journal
- 3 extra queries: update daily turnovers, update batch statistics, and store some transaction metadata (merchant, terminal, country, etc.)

I think the lower bound is 2 queries: one for the select or update of the account, and one for the journal. Everything else is to support additional functionality. But 20 queries is at least 2x too much.

## Stored procedures

Every time we execute a query, we pay with network roundtrip. A common wisdom years ago was to implement everything as stored procedures executing inside the database. It's still something to consider in extreme cases. I would not do it by default because the developer experience for stored procedures is far from modern IDE-based development.

In most cases, we can get away without using stored procedures as long as we keep the number of queries low and network fast. When network roundtrips are an issue, we can try to run the service on the database server itself. 

## Scaling vertically

In order to scale horizontally we must have a problem domain that can be partitioned well.

In double-entry accounting, many transactions will keep updating the same bank accounts and there is a certain number of times we can do it per second. To exceed that number we have to either tune the database, optimize the code, or scale vertically to run the code faster by using SSDs, faster CPUs, and network.

## Keeping account balance and contention

The main reason for storing an up-to-date balance is to ensure account balance does not become negative. Or "credits do not exceed debits", "debits do not exceed credits" in TigerBeetle terms. But do we care if the bank account has sufficient balance for the debit entry? Most of the time we care only about the direct actions of the client when he pays for something. When a bank wants to collect mortgage payments, it will happily run our accounts into a negative balance.

Since bank accounts lead to contention, we can handle them differently.

We can partition bank accounts using a hash of a client account or a random number and update only a single partition. That allows several parallel updates (the same as the number of partitions) because most databases have row-level locks. We can obtain the balance by summing all partitions. We can even lock all partitions to check the balance assuming not every operation requires that.

A more radical approach is to skip updating the balance and writing deltas to a new table or use the existing journal table. We might have a periodic task that sums all updates, applies to accounts, and marks deltas as processed. Or we can always do a query to calculate the balance of the bank account if it's needed only for reporting.

After all, this is about double-entry bookkeeping where each entry is recorded in at least two accounts as credit and debit. Keep the entries, lose the up-to-date balance. As [Martin Fowler's accounting patterns](https://martinfowler.com/apsupp/accounting.pdf) describe, keeping an up-to-date balance is just an optimization. If it makes the system faster - great, if that limits concurrency - skip it.

## TL;DR

I use this as a rule of thumb: we can achieve more than 100 transactions per second by keeping the number of queries low and database transactions short. Several hundreds of transactions per second can be achieved through batching. And that may be sufficient for our business already. Less than 100 transactions per second indicate an issue with the code, not the limits of the server or database. To push further into 1000s of transactions per second we might need to get rid of up-to-date account balances.

## More?

Accounting systems have more features than just accounts tables and transaction logs. And most of that can be implemented outside the accounting database, it's just convenient to have them there. But there are things that can't. Here's one of them: We often have to decide the amount of the entry based on the current balance of another account. 

When the accounting setup has main and overdraft accounts we have to query the account balance, decide how to split the amount between accounts, and post it. There is no locking of accounts in TigerBeetle so we either have to rely on external locking or come up with an accounting setup where checks will be done through `balancing` or `must_not_exceed` flags and we will retry posting until we succeed.

There might be different fees for the first transaction of a certain type, different fees for a certain amount spent, account turnover during a specific period, etc. That requires preventing concurrent updates, performing the calculation, and posting that to accounts.

