---
layout: post
title: A broker-less distributed messaging system from the previous century
tags: xatmi, rpc, distributed, tuxedo
---

When examining the [On-Line Transaction Processing Benchmark](https://www.tpc.org/tpcc/results/tpcc_results5.asp), most people focus on the performance numbers and the database software. But there is another column named "TP Monitor" that lists the transaction monitor software. Before cloud-scale systems took over, the best performance numbers were achieved with Oracle Tuxedo (or BEA Tuxedo, before Oracle acquired it). The good results of Oracle database and Tuxedo led my previous company to choose them as the basis for payment card software in the late 1990s and early 2000s.

While Oracle Tuxedo is proprietary software, [The XATMI specification](https://pubs.opengroup.org/onlinepubs/009649399/toc.pdf) is public. The main building block is an RPC call: you call a service by its name (`svc`), pass some data to it (`idata`, `ilen`), and receive data back (`odata`, `olen`):

```c
int tpcall(char *svc, char *idata, long ilen, char **odata, long *olen, long flags);
```

Under the hood, it's all just a wrapper around two API calls: one for sending a request to the service (`tpacall`) and the other one for waiting for a response (`tpgetrply`):

```c
int tpacall(char *svc, char *data, long len, long flags);
int tpgetrply(int *cd, char **data, long *len, long flags);
```

And yes - the creators decided to skip the letter 'e' in `tpgetrply` while still having longer API names like `tpadvertise`, `tpconnect` and `tpunadvertise`. But unlike later specifications like [CORBA](https://en.wikipedia.org/wiki/Common_Object_Request_Broker_Architecture), it made very clear which calls are RPC and you were forced to handle failures and timeouts.

**But**, the API itself is not that interesting. You can implement the API but still have a shitty XATMI implementation. Or you can do it like Tuxedo did and have something simple and elegant. So let's look into that and maybe you can take some design lessons out of it.

Tuxedo was developed by AT&T along with UNIX so it used [System V inter-process message queues, semaphores, and shared memory](https://en.wikipedia.org/wiki/UNIX_System_V#SVR1). Message queues are the foundation of Tuxedo and the most important function calls are `msgsnd`, which just copies data into kernel space, and `msgrcv`, which copies it back. [It's really that simple](https://www.tuhs.org/cgi-bin/utree.pl?file=pdp11v/usr/src/uts/pdp11/os/msg.c). But because the message queue lives in the kernel, messages live there as long as the kernel is running. Senders and receives can come and go, and the messages will stay there. There is no "broker" process as we expect nowadays that has to keep running or do persistence of messages to the storage. Kernel is the broker.

A [request-reply pattern](https://docs.nats.io/nats-concepts/core-nats/reqreply) is built by the caller having its own response queue and the callee having a request queue. Each request message includes the response queue identifier where the response should be stored. Tuxedo implements timeouts waiting for the response, handles (ignores) late responses, and does other housekeeping.

Now queues are identified by a number however the Tuxedo works with nice service names. To map between service names and the queue identifiers, Tuxedo uses shared memory. Again - the shared memory is kept alive by the kernel and outlives all processes. However, all processes can access the memory to do the lookup of the service name to the queue identifier. Like a serverless name server.

![Local Tuxedo]({{ site.url }}/public/tuxedo-local.png)

To put it all together: the caller process#1 looks up the name of the service "FOO". When "FOO" is not found or the queue does not exist, you get an error. When the queue is full, you can either wait or fail based on call mode. Once the request message is added to the queue, the caller process#1 proceeds to wait for a response. When the response is not received within the timeout, you get an error. On the callee side process#2 polls requests from queue#3. Once a request is received, it does the work and puts the response back into the reply queue mentioned in the request (queue#4).

Now what about the "distributed" part? Instead of adding a new transport for the service call, Tuxedo introduces gateways that connect multiple machines. On the caller machine, the gateway says it provides the "FOO" service. Once it receives the request, it forwards it using whatever transport protocol to the gateway on the other machine. On the other machine, the gateway acts as a caller and calls process#2.

![Distributed Tuxedo]({{ site.url }}/public/tuxedo-distributed.png)

Simple and nice, isn't it?
