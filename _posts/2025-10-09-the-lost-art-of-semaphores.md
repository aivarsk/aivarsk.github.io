---
layout: post
title: The lost art of semaphores
tags: ipc, semaphore, sysvipc, unix, python
---

I am a huge fan of [System V Inter Process Communication primitives](https://man7.org/linux/man-pages/man7/svipc.7.html). There is some rawness and UNIX spirit to them. There is a newer and kinda “improved” version of those primitives named POSIX IPC. While there are a few things in POSIX IPC that can’t be done with System V IPC, most of the time it’s the other way around. Primarily due to the rawness of System V IPC. Let’s check the [POSIX semaphors](https://man7.org/linux/man-pages/man7/sem_overview.7.html):

* `sem_post` can be used to release a semaphore (increment by 1)
* `sem_wait` can be used to acquire a semaphore (and decrement by 1)

System V IPC has a single call for that: [`semop`](https://man7.org/linux/man-pages/man2/semop.2.html). It can increment or decrement a semaphore by an arbitrary value. It also has the operation flags for each operation. And there, within flags, you can find one of the pearls of System V IPC - the `SEM_UNDO` flag.

What the `SEM_UNDO` flag does is add the operation to an [“undo list” within the kernel](https://github.com/torvalds/linux/blob/50c19e20ed2ef359cf155a39c8462b0a6351b9fa/ipc/sem.c#L2415). Whenever the process terminates because of natural causes or is brutally killed by `SIGTERM`, `SIGKILL`, out-of-memory killer, or other reasons, the kernel will revert the semaphore operation. Think about it - your process acquires a semaphore and gets killed while holding it, and it will prevent other processes from acquiring the semaphore again. With `SEM_UNDO`, you can choose what happens: if you used the semaphore as a counting semaphore, you can ask the kernel to release it automatically. When you acquire the semaphore to modify some shared resources, you can keep the semaphore stuck. It’s all up to you.

Which brings me back to a previous topic of [tracking Gunicorn's busy worker count](https://aivarsk.com/2025/08/26/gunicorn-busy-workers/). I used a semaphore there as a “reverse counting semaphore”: I released the semaphore (increment by 1) every time a process starts and acquired the semaphore (decrement by 1) every time a process stopped. But Python’s `multiprocessing.Semaphore` is a POSIX semaphore. When a worker gets killed by the OOM killer or dies, the semaphore is not decremented, and the worker count is incorrect.

So I decided to build my own Python wrappers around the [System V IPC ](https://github.com/aivarsk/sysvipc-python) to fix this issue and also make my other System V IPC projects more enjoyable. It’s more fun to use Python for quick tests than C++ code. With the library, here’s how you count the Gunicorn’s busy workers.

First, we have to create a new semaphore. It’s just a number that can be shared with others. The downside of that - you have to perform a manual cleanup by scheduling the removal of the semaphore when the main process terminates:

```python

import atexit
import sysvipc

sem = sysvipc.semget(sysvipc.IPC_PRIVATE, 1, 0o600)
atexit.register(lambda: sysvipc.semctl(sem, 0, sysvipc.IPC_RMID))

```

We can read the current value of the semaphore at any time with:

```python

curval = sysvipc.semctl(sem, 0, sysvipc.GETVAL)

```

And then each process can increment the semaphore without confusing the reader with strange names like “unlock” or “post”. I also specify the `SEM_UNDO` flag, and the kernel will apply `-1` to the semaphore when the process terminates for any reason:

```python

sysvipc.semop(sem, [(0, 1, sysvipc.SEM_UNDO)])

```

Once the process is done with the work, I decrement the semaphore. Again - no confusing names like “lock” or “wait”. The `SEM_UNDO` will add `+1` to the kernel’s semaphore adjustments and make the total adjustment `0`. Past this point, when a process terminates, nothing will be subtracted from the semaphore value, and it will correctly represent the number of active workers.

```python

sysvipc.semop(sem, [(0, -1, sysvipc.SEM_UNDO)])

```

And this is just the beginning, I need to write more [Pybind11](http://pybind11.com/) wrappers for System V IPC to unlock more goodies in Python.
