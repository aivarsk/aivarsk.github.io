---
layout: post
title: tmshutdown and MSSQ
---

A server waits for a new incoming service call by using `msgrcv()` system call on IPC queue. The call blocks until a message is available or a signal interrupts the system call.

So how can Oracle Tuxedo command `tmshutdown` stop and shut down the server? Sending a signal might do more harm to the server code so it's not an option. `tmshutdown` could delete the IPC queue and that should make `msgrcv()` to return. The last and most reliable way is to put a special message in the queue - `msgrcv()` will return and the server can shut down cleanly.

This is fine for a "single queue single server" setup but how about MSSQ (multiple servers single queue) setup when multiple servers do `msgrcv()` on the same queue? How to guarantee that only the requested server receives the shutdown request. Let's investigate!

I start 20 server instances all using the same `RQADDR` / IPC queue and the all start waiting for requests:

	66062 1587730885.032534 msgrcv(35815438,  <unfinished ...>
	66063 1587730885.069161 msgrcv(35815438,  <unfinished ...>
	66064 1587730885.103754 msgrcv(35815438,  <unfinished ...>
	66065 1587730885.138347 msgrcv(35815438,  <unfinished ...>
	66066 1587730885.173324 msgrcv(35815438,  <unfinished ...>
	66067 1587730885.206270 msgrcv(35815438,  <unfinished ...>
	66068 1587730885.239844 msgrcv(35815438,  <unfinished ...>
	66069 1587730885.273854 msgrcv(35815438,  <unfinished ...>
	66070 1587730885.308535 msgrcv(35815438,  <unfinished ...>
	66071 1587730885.342175 msgrcv(35815438,  <unfinished ...>
	66072 1587730885.375979 msgrcv(35815438,  <unfinished ...>
	66073 1587730885.411762 msgrcv(35815438,  <unfinished ...>
	66074 1587730885.445510 msgrcv(35815438,  <unfinished ...>
	66075 1587730885.479610 msgrcv(35815438,  <unfinished ...>
	66076 1587730885.514347 msgrcv(35815438,  <unfinished ...>
	66077 1587730885.546221 msgrcv(35815438,  <unfinished ...>
	66078 1587730885.577737 msgrcv(35815438,  <unfinished ...>
	66079 1587730885.623727 msgrcv(35815438,  <unfinished ...>
	66089 1587730885.658156 msgrcv(35815438,  <unfinished ...>
	66090 1587730885.690927 msgrcv(35815438,  <unfinished ...>

Now let's shut down one of those 20 server instances by server id `tmshutdown -i 3`. `tmshutdown` indeed puts a message in the queue. The other thing you'll notice is `tmshutdown` waits for a response message just like regular Oracle Tuxedo clients do:

	66093 1587730900.264234 msgsnd(35815438, {536870912, "{\0\0\0\16\200\"\2\0\0\0\0\300\2\0\0\0\0\0\200\
	66093 1587730900.269244 msgrcv(36470794,  <unfinished ...>

A server with PID 66062 receives message first. But it's obviously not meant for it. So server puts the message back to the same queue and starts receving messages again:

	66062 1587730900.264318 <... msgrcv resumed> {536870912, "{\0\0\0\16\200\"\2\0\0\0\0\300\2\0\0\0\0\0\
	66062 1587730900.264885 msgsnd(35815438, {1073741824, "{\0\0\0\16\200\"\2\0\0\0\0\300\2\0\0\0\0\0\200
	66062 1587730900.264988 <... msgsnd resumed> ) = 0 <0.000077>
	66062 1587730900.265207 msgrcv(35815438,  <unfinished ...>

Then server with PID 66063 receives the message and again - message is put back into the same queue and servers waits for other messages:

	66063 1587730900.264957 <... msgrcv resumed> {1073741824, "{\0\0\0\16\200\"\2\0\0\0\0\300\2\0\0\0\0\0
	66063 1587730900.265401 msgsnd(35815438, {1073741824, "{\0\0\0\16\200\"\2\0\0\0\0\300\2\0\0\0\0\0\200
	66063 1587730900.265492 <... msgsnd resumed> ) = 0 <0.000070>
	66063 1587730900.265852 msgrcv(35815438,  <unfinished ...>

Until finally the right server with PID 66064 and `SRVID` equal to 3 receives the message, sends a reply to `tmshutdown` and exits the process

	66064 1587730900.265462 <... msgrcv resumed> {1073741824, "{\0\0\0\16\200\"\2\0\0\0\0\300\2\0\0\0\0\0
	66064 1587730900.270236 msgsnd(36470794, {1073807917, "{\0\0\0\n\200,\2\0\0\0\0\324\2\0\0\0\0\0\200\0
	66064 1587730900.275417 exit_group(0)   = ?

And `tmshutdown` gets its response and exits:

	66093 1587730900.271627 msgrcv(36470794, {1073807917, "{\0\0\0\n\200,\2\0\0\0\0\324\2\0\0\0\0\0\200\0
	66093 1587730900.276311 msgrcv(36470794, 0x72ccc8, 4712, 1073807917, IPC_NOWAIT) = -1 ENOMSG (No mess
	66093 1587730903.280395 exit_group(0)   = ?
	

So the algorithm for shutting down a specific server in MSSQ is simply sending the request and returning it back to the queue if the wrong server received it. But how to ensure a single process is not receiving the message, putting it back and receiving it from the queue right away? The answer to that lies in "message type" of IPC queue messages.

`msgrcv(int msqid, void *msgp, size_t msgsz, long msgtyp, int msgflg)` - If msgtyp is less than 0, then the first message in the queue with the lowest type less than or  equal to the absolute value of msgtyp will be read.

How does it work in practice? All instances of server call `msgrcv` with `msgtype` of -1073741824. They are accepting messages with message type 1 up to 1073741824. PID 66062 receives a message with message type 536870912. It puts back the same message into the queue but with a different message type 1073741824 this time. I don't know how Oracle Tuxedo chooses that, but it's the maximal message type any of the servers is accepting. When PID 66062 calls `msgrcv()` with `msgtyp` value of -1073741823 and it will not see the same message anymore because the new message type is greater than the absolute value of -1073741823. Next PID 66063 receives the message and puts it back with the same message type but calls `msgtyp` -1073741823 to not receive it again. And so forth until the right instance receives the request.


[Here is a simple code to investigate `tmshutdown` for MSSQ setup](https://github.com/fuxedo/tuxedo-examples/tree/master/tmshutdown)
