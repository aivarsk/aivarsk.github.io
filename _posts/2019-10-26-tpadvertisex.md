---
layout: post
title: tpadvertisex(3c)&#58; a new cool Oracle Tuxedo 12.2 feature
---

A service call in Oracle Tuxedo ends up in some System V message queue. What happens next depends on the configuration in `UBBCONFIG`: 
 
In a Single-Server-Single-Queue setup exactly one instance of the server will process the message. But it might also crash and lose all requests currently in the queue. Or requests may queue up because one request takes a longer time to process than others. Requests will stay in the queue although there are idle instances of servers that could process them. 
 
In a Multiple-Servers-Single-Queue set up multiple instances of server process messages: it provides a better load balancing because any idle instance will pick up the next message. It also provides a better availability because a single instance can crash and be restarted while others continue working and no messages are lost. MSSQ is what you need most of the time. The only downside is that there is no possibility to call a specific instance of the server because instance can't provide a unique service (limitation of MSSQ). 

I have a couple of use cases for that: 

- "adapter" servers that have a long-running service polling network, database or filesystem
- notifications to reload caches 
- writing buffers to disk, closing files, committing database transactions 
 
Oracle Tuxedo 12.2 adds a new [`tpadvertisex()` function call](https://docs.oracle.com/cd/E72452_01/tuxedo/docs1222/rf3c/rf3c.html#2548645) that allows advertising unique services. It requires a new [`SECONDARYRQ` parameter](https://docs.oracle.com/cd/E72452_01/tuxedo/docs1222/rf5/rf5.html#1532198) in `UBBCONFIG` that creates a new private message queue for each server. Each server can listen on two queues now. That should have the best of both worlds: load balancing, availability and unique service for each server that I can call when needed. 
 
But it does not work. For now. 

At first, I was worried when MIB's `T_SVCGRP` showed too many duplicate services but I hoped it was a cosmetic bug. But `tpcall()` to the services that should be unique for each server got received by server instances that did not advertise them. So for now there is no difference between `tpadvertise()` and `tpadvertisex()`. 

I created a new service request and hope to see a fix soon.

[Here is a simple code with `tpadvertisex()` that does not work](https://github.com/fuxedo/tuxedo-examples/tree/master/tpadvertisex)
