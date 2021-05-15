---
layout: post
title: tpacall(3c) and XA transactions
tags: oracle tuxedo xa tpcall tpacall tpgetrply xid
---

Oracle Tuxedo allows us to develop transactional service-oriented (or microservice) applications easily and performance is quite good. So easy and performant that I would not care about implementing sagas or compensating transactions most of the time.

And then there is `tpacall()` function call that allows to parallelize execution by calling some service in asynchronous mode, doing work in parallel in caller and callee and then waiting for the service to complete at some point. Just like launching processing in some thread.

A long time ago I wanted to improve the latency of application by doing work in parallel in multiple services. Good for me that I did some measurements: the parallel execution took a longer time than the single-process implementation. I assumed that was because of service call overhead and gave up on the idea. But recently I discovered the real reason. And that is a combination of `tpacall()` and XA transactions in Oracle database.

Oracle Tuxedo uses tightly-coupled transactions for the whole group. That means all services in the group will use the same transaction identifier (XID) and have a single database transaction. [Latest Tuxedo performance improvements use a single XID everywhere](https://docs.oracle.com/cd/E72452_01/tuxedo/docs1222/xpp/xpp.html#1108766). Guess how many sessions with the same XID Oracle Database can handle at once?

There can be only one session using the XID at a time. For multiple processes to work on the same transaction one must give up the transaction so second process can work. That means that Oracle Tuxedo suspends the XID on the caller side, joins XID on the callee side, does work, finishes XID on the callee side, resumes XID on the caller side. Something like this:


| time | process 1 | process 2 |
|------|-----------|-----------|
| t1 | begin tpcall | |
| t2 | xa_end | |
| t3 | msgsnd | |
| t4 | begin msgrcv | |
| t5 | wait | msgrcv |
| t6 | wait | xa_start |
| t7 | wait | tpservice |
| t8 | wait | ... |
| t9 | wait | treturn |
| t11 | wait | xa_end |
| t12 | wait | msgsnd |
| t13 | end msgrcv | |
| t14 | xa_start | |
| t15 | end tpcall | |

The proof for that can be seen in ULOG when you turn the trace on using `tmadmin` command like `changetrace -m all "*:ulog:dye"`. But what happens when an asynchronous call is made instead of synchronous? When does the caller give up the transaction? Using the same trace and adding a delay between `tpacall()` and `tpgetrply()` we can observe something like this:

| time | process 1 | process 2 |
|------|-----------|-----------|
| t1 | begin tpacall | |
| t2 | msgsnd | |
| t3 | end tpacall | |
| t4 | ... | msgrcv |
| t5 | ... | begin xa_start |
| t6 | ... | wait | 
| t7 | sleep | wait | 
| t8 | begin tpgetrply | wait | 
| t9 | xa_end | wait |
| t10 | begin msgrcv | wait |
| t11 | wait | end xa_start |
| t12 | wait | tpservice |
| t13 | wait | ... |
| t14 | wait | treturn |
| t15 | wait | xa_end |
| t16 | wait | msgsnd |
| t17 | end msgrcv | |
| t18 | end tpgetrply | |

Turns out that `tpgetrply()` gives up the transaction, not `tpacall()` which makes a perfect sense for the caller. Caller keeps the transaction and can continue to do something useful. But callee will have to wait until the caller starts waiting for a reply. And that means the callee will not be able to make any progress in parallel unless it uses non-transactional resources or at least a different database.

So here is a rule of thumb:

- Use `tpacall()` only with `TPNOTRAN` flag unless you have a good reason not to.
- Double-check your `tpcall()` flags for performance-critical paths and add `TPNOTRAN` flags where possible. Giving up the transaction (`xa_end` + `xa_start`) costs two roundtrips to the database.

[Here is a simple code to investigate `tpacall()` and XA transactions](https://github.com/fuxedo/tuxedo-examples/tree/master/xa)
