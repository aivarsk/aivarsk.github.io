---
layout: post
title: Oracle Tuxedo queues illustrated
tags: oracle tuxedo ipc msgsnd msgrcv
---

[I did investigate `tpacall()` before](http://aivarsk.com/2019/03/22/blocking-tpacall/) and you can find more details there.

But this time I had to prepare internal presentation so I developed a small Tuxedo app for simulation and scripts for visualizing the results.

Just 2 Tuxedo servers, one calling the other, and a client program injecting the events. The application can process 10 events per second, 20 events per second are sent to the application, 100 events in total. If everything goes well application should be able to handle 100 events in 10 seconds.

So here we can see how Tuxedo application behaves when queues are big enough and application can buffer excess messages:

![Unlimited queues]({{ site.url }}/public/tuxedo-unlimited-queues.gif)


However, it goes south when queues are too small to buffer excess messages:

![Limited queues]({{ site.url }}/public/tuxedo-limited-queues.gif)

Processing time increases significantly. As I wrote before, it is due to how timeout is implemented using system calls that have no concept of timeout. The algorithm is:

- Try to put the message in the queue in non-blocking mode of `msgsnd()`
- If it fails, try to put message header (file-transfer) in the queue in non-blocking mode of `msgsnd()`
- If it fails, sleep for 1 second (!)
- Try to put message header (file-transfer) in the queue in non-blocking mode of `msgsnd()`
- If it fails, sleep for 1 second (!)
- Try to put message header (file-transfer) in the queue in non-blocking mode of `msgsnd()`
- If it fails, sleep for 1 second (!)
- ...
- Continue until either `msgsnd()` succeeds or blocking timeout occurs.

We can verify it by having small queues but disabling the blocking timeout by using `tpacall(..., TPNOTIME)`:

![Limited queues]({{ site.url }}/public/tuxedo-limited-queues-notime.gif)


If you want your application to run smoothly, increase your queues, and consider using `TPNOTIME` where possible.
