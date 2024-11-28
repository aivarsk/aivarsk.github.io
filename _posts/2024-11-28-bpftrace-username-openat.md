---
layout: post
title: bpftrace username and the openat system call
tags: bpftrace username openat
---

Once again I am playing with eBPF and bpftrace. This time I am trying to trace all file access. Whenever a file is open, created, or deleted I want to print the filename, the process ID, and the user who did it. bpftrace has the `username` built in to get the username. However, I noticed I was missing a lot of file creations and by trial and error, I discovered many applications use the `openat` system call for that. Once I started tracing its invocations the tracing got stuck in an infinite loop.

Turns out that every time the bpftrace scripts try to print the user (`username`), it opens the `/etc/passwd` to get the user name for the given user ID. And it does that using the `openat` system call. That triggers the `openat` probe which tries to retrieve the user name again with `openat` system call that triggers the `openat` probe and ...
