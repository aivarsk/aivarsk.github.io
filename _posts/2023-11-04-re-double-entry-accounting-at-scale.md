---
layout: post
title: Re&colon; Double-entry accounting at scale
tags: PostgreSQL, ledger, Django
---

![only 10s of updates]({{ site.url }}/public/updates_per_second.png)

I have been looking at some FinTech talks about Ledgers and Accounting and then I found one about [Ledger implementation at Modern Treasury](https://youtu.be/knnSIKCsX34?si=6Zxv-B8c5TabSuGT). It had some good ideas like delaying balance updates until they are needed. I scanned the documentation and at least from the API side, it had the distinction between a relative update by checking the remaining balance and optimistic locking by checking the row's version number which is what I talk about in [my database concurrency presentation](https://2023.djangoday.dk/talks/aivars/). Some implementation details were strange but I will save it for the next post.

But what triggered me the most was the slide I shared above. Only 10s of updates per second for a single row? That is so wrong... Accounting systems were doing 100s and 1000s of transactions per second on the spinning disks 10 and more years ago. As I wrote before - [less than 100 transactions per second indicate an issue with the code, not the limits of the server or database](https://aivarsk.com/2023/08/23/accounting-before-tigerbeetle/).

And that is so easy to demonstrate. I quickly started up the same environment I use for my database concurrency talk and wrote a quick benchmark using Python and Django:

```python
account = Account.objects.get(id=1)

start = time.time()
for _ in range(n):
    with transaction.atomic():
        account.balance = F("balance") + 1
        account.save()
end = time.time()

print(n / (end - start), "updates per second")
```

It increments the account balance N times by 1 and does each increment in a separate transaction to avoid auto-commit tricks. The result I had was 288 updates per second on a laptop (with SSD but still), with PostgreSQL running in a docker container with no tuning, data files in docker overlay, and "slow" Python ORM code.

```bash
288.93953161942886 updates per second
```
And when I started multiple processes each of them ran slower but the total number of updates for a single row was similar.

```bash
76.18239300396263 updates per second
74.92634295378109 updates per second
74.35262507610665 updates per second
74.0825835532335 updates per second
```

You **can** update the same row hundreds of times per second before row locks become the bottleneck. What is making the code slow is everything else you do. When we use mutexes in application code, we try to keep the critical section as small as possible. The same principle must be used with databases: keep critical sections as short as possible, and release locks as soon as possible.

I have a new side project now: to create a simple ledger service with multiple ledgers, accounts, entries, and transactions and show how it can easily achieve more than 100 business transactions per second with some simple code changes, database knowledge, and mechanical sympathy. I will eat my own dogfood of [Pessimism, optimism, realism and Django database concurrency](https://pycon.ie/pycon-2023/schedule/)
