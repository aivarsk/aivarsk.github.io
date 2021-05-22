---
layout: post
title: Oracle Tuxedo MSSQ vs. SSSQ
tags: oracle tuxedo sssq mssq
---

Servers in Oracle Tuxedo can be configured either in Multiple Servers - Single Queue (MSSQ) setup or Single Server - Single Queue (SSSQ) mode. More in-detail information about SSSQ and MSSQ setup can be found in [my book "Modernizing Oracle Tuxedo Applications with Python"](https://amzn.to/3ljktiH).

When it comes to the performance aspect of these setups, Tuxedo documentation recommends using the MSSQ setup when:

- You have a "reasonable" number of servers (2..12)
- Message sizes are less than 75% or the queue size

The MSSQ setup is discouraged when:

- Message sizes are large and exhaust the queue
- You have a large number of servers

Oracle itself uses SSSQ setup for the [TPC-C](http://www.tpc.org/tpcc/) performance tests where it demonstrated more than 500,000 business transactions/second.

However, this list of recommendations does not take the Operating System into account. One of the Pros for MSSQ can be found in the implementation of the Linux kernel. Calling a service in Oracle Tuxedo results in `msgsnd` system call. And [`msgsnd` has a shortcut](https://github.com/torvalds/linux/blob/master/ipc/msg.c#L931) called [`pipelined_send`](https://github.com/torvalds/linux/blob/master/ipc/msg.c#L808) for sending a message to a queue when there is a process waiting to receive a message.
Message can be sent directly to the receiver process instead of being put in the queue. Since the MSSQ setup has multiple servers receiving messages, the fast path will be taken more often and will result in less overhead. It also means that the first message in an empty queue will be delivered faster. As soon as the queue fills up, all new request messages will be stored in the kernel queue and the shortcut will not be taken.
