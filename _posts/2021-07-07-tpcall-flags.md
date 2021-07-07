---
layout: post
title: The hitchhiker's guide to the tpcall flags
tags: tuxedo tpcall msgsnd flags
---


Oracle Tuxedo documentation of the [tpcall](https://docs.oracle.com/cd/E53645_01/tuxedo/docs12cr2/rf3c/rf3c.html#1021731) function describes (mostly) the high-level behavior of the function. But in addition to that, I am interested in how the messages will be sent through the IPC queues and what happens to the transactions as that has an impact on the performance and the behavior of the system under load.

### TPNOFLAGS

First, let us see how the`tpcall` behaves without any flags, either `0` or `TPNOFLAGS`.

Sending the request:

- If the message is larger than the message size limit, it is sent using file transfer (see below).
- If the message is below the limit, Tuxedo attempts to send through the queue first using a non-blocking `msgsnd(..., IPC_NOWAIT)`. If that fails with the `EAGAIN` error, a file transfer is attempted.
- File transfer is performed using a non-blocking `msgsnd(..., IPC_NOWAIT)`. If that fails, the process sleeps for one second and tries again and repeats so either until success or a timeout occurs.

If the caller is in a transaction, Tuxedo gives up the transaction using `xa_end` and joins the transaction using `xa_start` before returning. That means two roundtrips to the database.

### TPNOTRAN

> If the caller is in transaction mode and this flag is set, then when svc is invoked, it is not performed on behalf of the caller’s transaction. Note that svc may still be invoked in transaction mode but it will not be the same transaction: a svc may have as a configuration attribute that it is automatically invoked in transaction mode. A caller in transaction mode that sets this flag is still subject to the transaction timeout (and no other). If a service fails that was invoked with this flag, the caller’s transaction is not affected.

Sending the request:

- If the message is larger than the message size limit, it is sent using file transfer (see below).
- If the message is below the limit, Tuxedo attempts to send through the queue first using a non-blocking `msgsnd(..., IPC_NOWAIT)`. If that fails with the `EAGAIN` error, a file transfer is attempted.
- File transfer is performed using a non-blocking `msgsnd(..., IPC_NOWAIT)`. If that fails, the process sleeps for one second and tries again and repeats so either until success or a timeout occurs.

The caller does not give up the transaction.

### TPNOBLOCK

> The request is not sent if a blocking condition exists (for example, the internal buffers into which the message is transferred are full). Note that this flag applies only to the send portion of tpcall(): the function may block waiting for the reply. When TPNOBLOCK is not specified and a blocking condition exists, the caller blocks until the condition subsides or a timeout occurs (either transaction or blocking timeout).

Sending the request:

- If the message is larger than the message size limit, file transfer is performed using a non-blocking `msgsnd(..., IPC_NOWAIT)`.
- If the message is below the limit, Tuxedo attempts to send through the queue first using a non-blocking `msgsnd(..., IPC_NOWAIT)`.
- There is no attempt to send a message below the size limit using the file transfer. This means that fewer messages fit in the request queue.

If the caller is in a transaction, Tuxedo gives up the transaction using `xa_end` and joins the transaction using `xa_start` before returning.

### TPNOTIME

> This flag signifies that the caller is willing to block indefinitely and wants to be immune to blocking timeouts. However, if the caller is in transaction mode, this flag has no effect; it is subject to the transaction timeout limit. Transaction timeouts may still occur.

Sending the request if the caller is not in a transaction:

- If the message is larger than the message size limit, file transfer is performed using a blocking `msgsnd`.
- If the message is below the limit, Tuxedo attempts to send through the queue first using a non-blocking `msgsnd`.
- There is no attempt to send a message below the size limit using the file transfer. This means that fewer messages fit in the request queue.

Sending the request if the caller is in a transaction:

- If the message is larger than the message size limit, it is sent using file transfer (see below).
- If the message is below the limit, Tuxedo attempts to send through the queue first using a non-blocking `msgsnd(..., IPC_NOWAIT)`. If that fails with the `EAGAIN` error, a file transfer is attempted.
- File transfer is performed using a non-blocking `msgsnd(..., IPC_NOWAIT)`. If that fails, the process sleeps for one second and tries again and repeats so either until success or a timeout occurs.

If the caller is in a transaction, Tuxedo gives up the transaction using `xa_end` and joins the transaction using `xa_start` before returning.

### TPSIGRSTRT

> If a signal interrupts any underlying system calls, then the interrupted system call is reissued.

Assuming your application receives a signal and you have set up a handler for the signal (otherwise the application terminates), Tuxedo will retry the interrupted `msgsnd` call.


### TPNOCOPY

> This flag is only available for Exalogic and used when the Use of Shared Memory for Inter Process Communication feature is enabled (see SHMQ option in UBBCONFIG(5)). It indicates Tuxedo not making safe copy for request buffer during message sending process, thus saving cost of copying large buffers. However, in the event that tpcall() fails, causing the caller application unable to access the request buffer anymore, it is recommended you call tpfree() to release the request buffer. If Use of Shared Memory for Inter Process Communication is not enabled, this flag has no effect.

Unfortunately, I don't have access to Exalogic to have any insight into how it works.

### TPNOCHANGE

> By default, if a buffer is received that differs in type from the buffer pointed to by *odata, then *odata’s buffer type changes to the received buffer’s type so long as the receiver recognizes the incoming buffer type. When this flag is set, the type of the buffer pointed to by *odata is not allowed to change. That is, the type and subtype of the received buffer must match the type and subtype of the buffer pointed to by *odata.

This is a less known feature of Tuxedo that exists on the paper but it does not happen in practice. You pick a buffer type (`FML32` typically) and use it everywhere.

