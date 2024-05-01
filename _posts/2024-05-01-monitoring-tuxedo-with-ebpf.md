---
layout: post
title: Monitoring Oracle Tuxedo with Linux eBPF
tags: tuxedo tpcall msgsnd msgrcv ebpf
---

I have built several scripts for monitoring Oracle Tuxedo over the years. Most relied on the `tmtrace` and `tputrace(3c)` features provided by Tuxedo itself. I also tried to collect statistics on the IPC queues using standard system calls and tools. But all these years I have been thinking about how to measure the time a message spends in a queue. Messages in IPC queues are just an array of bytes, there is no field I could use to store that information. IPC queues were not created with observability in mind.

Over the last weeks, I was strongly incentivized to dive into Linux enhanced Berkeley Packet Filter (eBPF) and I checked probes for `msgsnd` and `msgrcv` again. I went through examples of bpftrace scripts. And I looked at the Linux kernel source code again. And there I found the solution.

There are 2 probes for copying IPC messages from the user space to the kernel space and back. I can store all information needed in `bpftrace` maps (associative arrays) by using the message address in kernel space as the key. The rest was just extra hops to make it all work: not all probes receive the pointer to the message so the information has to be passed through another map indexed by the thread id. Which is a standard practice for making parameters available in the return probes.

So here is a proof of concept that works [msg_latency.bt](https://github.com/fuxedo/tuxtrace/blob/master/msg_latency.bt) and I can work on making it smarter to recognize the service names and service requests and respones.
