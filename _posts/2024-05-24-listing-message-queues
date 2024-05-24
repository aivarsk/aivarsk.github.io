---
layout: post
title: Listing SystemV IPC queues
tags: unix systemv sysvipc
---

The correct way to list the queues on the Linux machine is by using the `ipcs -q` command. However, I have not seen anything in the system calls that would provide such a capability. But Today I Learned...

The [ipcs](https://github.com/util-linux/util-linux/blob/master/sys-utils/ipcutils.c#L369) utility has two ways of retrieving the list of queues. The first and main one is to read and parse `/proc/sysvipc/msg`.
If the file is not present, it does a fallback by using a strange Linux-specific `msgctl` call with `MSG_INFO` command. Which among other things has this property:

[successful IPC_INFO or MSG_INFO operation returns the index of the highest used entry in the kernel's internal array recording information about all message queues](https://man7.org/linux/man-pages/man2/msgctl.2.html)

And then it's just a loop from 0 to that max ID and checking all possible queue identifiers.
