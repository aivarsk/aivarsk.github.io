---
layout: post
title: Oracle Tuxedo internals - blocking `tpgetrply()`
---

This post is a follow-up for [blocking tpacall()](http://aivarsk.github.io/2019/03/22/blocking-tpacall/)

The `msgrcv()` function is used to implement `tpgetrply()` in Oracle Tuxedo. It has the same problems with implementing timeouts as I described in the previous post. But Oracle Tuxedo has a different strategy for the timeout of `tpgetrply()/msgrcv()`: it always performs a *blocking* `msgrcv()`. So what happens next?

It is one of the responsibilities Tuxedo's `BBL` process has: after timeout expires `BBL` process puts a message in the reply queue and `msgrcv()` call finishes. Although I don't have direct evidence I think each Tuxedo client writes information to shared memory before performing `msgrcv()` call. There are two types of timeouts in Tuxedo: blocking timeout that is configured as `SCANUNIT` times `BLOCKTIME` parameter and a transaction timeout that is specified either in configuration or programmatically. Both of them work in the same way:

- Write information to shared memory for `BBL` process
- Perform a blocking `msgrcv()` call
- Every `SCANUNIT` seconds `BBL` stops accepting service requests for the `.TMIB` service and performs sanity checks
- If either `SCANUNIT`x`BLOCKTIME` or transaction timeout has expired, puts a message in the queue
- `msgrcv()` call returns

However, there is no way to interrupt the service call that takes too long, it will still perform the work and the response will be put in reply queue of the calling process. That means the caller must take care of such late replies so that reply queue does not fill up. And indeed it does that before every n-th `tpacall()` or before shutting down (as in the example below): `msgrcv(..., IPC_NOWAIT)` is performed for `mtype` of requests that timed out. It also introduces a race condition: the real reply may come right before the timeout reply is added to the queue. So the housekeeping process must take care of both cases.

There are 2 ways to change the behavior of `tpgetrply()`:

- TPNOBLOCK flag causes it to return immediately if the reply is not available.
- TPNOTIME flag makes the call immune to `SCANUNIT`x`BLOCKTIME` timeout but transaction timeout may still occur.

One more issue I have to mention is: if transaction timeout is not used there may be cases when the caller has received the blocking timeout but the request is still in the request queue of the callee. No one cancels the request and the callee will process it although no one is waiting for the response anymore. I think Oracle TSAM has a feature that allows dropping such requests but it's an additional product for an additional price. So if your services take several seconds to complete it might be a good idea to add "request expiry time" field to the message and drop expired requests in your program code.


[Here is a simple code to investigate blocking `tpgetrply()`](https://github.com/fuxedo/fuxedo-examples/tree/master/blocking-tpgetrply)
