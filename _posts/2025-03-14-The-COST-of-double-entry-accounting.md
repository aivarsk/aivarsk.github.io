---
layout: post
title: The COST of double-entry accounting
tags: PostgreSQL, ledger, Django, Modern Treasury
---

Sometimes I think the most important paper of our times is [Scalability! But at what COST?](https://www.usenix.org/system/files/conference/hotos15/hotos15-paper-mcsherry.pdf)
We build "web-scale" systems and get excited about how scalable they are and how we solve the problems of scale, but we do not measure the COST: Configuration that Outperforms a Single Thread. We lack reasonable, well-done single-threaded implementations for different domains. One of which is ledger and accounting.

Because of that, many get the impression that [you have to go "web-scale" to reach more than 10 ledger entries per second](https://aivarsk.com/2023/11/04/re-double-entry-accounting-at-scale/). You do not. All you need is a well-written implementation. Better late than never, I tried to implement a simple Proof of Concept for a ledger similar to what Modern Treasury provides: [https://github.com/aivarsk/django-modern-treasury-poc](https://github.com/aivarsk/django-modern-treasury-poc).

So here is a reasonable baseline: 200 ledger entries per second. You should be able to reach at least 300 entries per second by running this in parallel even for "hot accounts". And for normal accounts, you should hit the limits of your database and network.

There's no magic, just an understanding of how [UPDATEs work in read-committed isolation level](https://www.postgresql.org/docs/current/transaction-iso.html#XACT-READ-COMMITTED) which also explains how and why optimistic locking works with databases. And [here is me](https://2023.djangoday.dk/talks/aivars/) discovering it in FAFO approach.


P.S. This started with a challenge [to implement it with a raw SQL](https://x.com/jamesacowling/status/1899913337829548257) and the errors I found there and [a simpler implementation](https://github.com/aivarsk/django-debitcredit).
