---
layout: post
title: Oracle Tuxedo internals - blocking `tpacall()`
---

A service call in Oracle Tuxedo is performed by the `tpcall()` function that actually is a wrapper around two other functions:

- `tpacall()` - sending a request to the service
- `tpgetrply()` - waiting for a response from the service

This post will focus on some implementation aspects of the `tpacall()` function.

It is no secret that Oracle Tuxedo uses System V IPC messages queues for interprocess communication. But unlike POSIX message queue API, System V message queue API has no concept of timeouts: POSIX has `mq_send()` and `mq_timedsend()` for sending a message while System V has only `msgsnd()` function. So how does Oracle Tuxedo implement a timeout (`BLOCKTIME` in `UBBCONFIG`) for sending a request to the service?

A standard UNIX way of adding a timeout to blocking calls is by using `alarm()` function that delivers `SIGALRM` signal to the process after a number of seconds. Signal interrupts the execution of the system call and sets `errno` to `EINTR`. That was the way to write the code in the previous century when software was rather simple. Programming with signals was never easy and always error-prone. Add to that cases where software might already be using `alarm()` for other purposes or it might have multiple threads (signal will be delivered to a "random" thread) and it becomes clear that `alarm()` is not the way to go.

So how does Oracle Tuxedo implement the timeout for `tpacall()/msgsnd()` calls? Well, it *always* calls `msgsnd()` with `IPC_NOWAIT` flag that causes the function to return with `errno` equal to `EAGAIN` when there is insufficient space available in the queue. If that happens, Oracle Tuxedo sleeps for one second(!!!) and repeats the call again and again until it either succeeds or blocking timeout occurs. And it does the first retry without sleeping - probably for cases when the queue is full for a tiny fraction of a second.

Clever, right? But this implementation has serious consequences for systems that handle several calls per second (not to mention thousands of calls per second):

- More than 1 request per second is sent to the queue.
- Due to fluctuations in processing time requests queue up.
- The queue becomes full, caller sleeps for one second.
- Caller no longer accepts requests itself so 1 second worth of requests queue up at the caller.
- Since no new requests are added to the first queue, all of the requests might be processed and the process goes idle and looks completely innocent.

A single hiccup that causes one queue to fill up will propagate up the caller stack and bring your Oracle Tuxedo application down on its knees.

I have tried to avoid that sleeping by calling `tpacall` with `TPNOBLOCK` and retrying failed calls until they succeed. But either spinning consumed too much CPU preventing other processes from getting CPU time or `tpacall()` overhead of load balancing and serializing request produced worse results than just letting Oracle Tuxedo to sleep and retry.

The second option is to call `tpacall()` with `TPNOTIME` flag - possibly blocking until the queue has enough space. It sounds bad on paper but the more I think about the actual algorithm under the non-blocking `msgsnd()` the more I like this approach.

But the best solution, of course, is to use big queues and throttling to prevent queues from filling up and having to deal with this problem in the first place.

[Here is a simple code to investigate blocking `tpacall()`](https://github.com/fuxedo/tuxedo-examples/tree/master/blocking-tpacall)
