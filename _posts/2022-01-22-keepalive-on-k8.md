---
layout: post
title: Idle HTTP connections in Scala on Kubernetes
tags: kubernetes, k8, java, scala, keep-alive
---

![Kubernetes]({{ site.url }}/public/simplyk8.png)

Both the HTTP client (Scala) and HTTP server (Python) are running in Kubernetes. Depending on the input, the service may take even more than 10 minutes to produce a response. Message queues would be a better solution instead of long HTTP connections but that is not an option at this point.

Everything works for short requests of a couple of minutes. Once a request takes 5-6 or more minutes, strange things start to happen:

- The server returns a response after 6 minutes
- The client does not receive it and keeps waiting until read timeout occurs even when it is 15 minutes, 60 minutes, or 180 minutes.

So it looks like a timeout issue but where? And why the client does not detect the broken connection? Let's investigate:

- The service can be called from my laptop so it's not a server issue
- The service can be called even from the client's container using `curl` so it is not a problem with connectivity between containers
- There should be something that the Scala code and `curl` do differently

Running `curl` using my old friend `strace` reveals some socket configuration `curl` does:

```c
socket(AF_INET, SOCK_STREAM, IPPROTO_TCP) = 3
setsockopt(3, SOL_TCP, TCP_NODELAY, [1], 4) = 0
setsockopt(3, SOL_SOCKET, SO_KEEPALIVE, [1], 4) = 0
setsockopt(3, SOL_TCP, TCP_KEEPIDLE, [60], 4) = 0
setsockopt(3, SOL_TCP, TCP_KEEPINTVL, [60], 4) = 0
```

What it does is set up TCP keep-alive to exchange packets at least once every 60 seconds. Why doesn't Scala code do the same? Because that was added [only to Java 11](https://docs.oracle.com/en/java/javase/11/docs/api/jdk.net/jdk/net/ExtendedSocketOptions.html) and backported to Java 8 later. I haven't seen much code using them and hours of Googling did not find a way how to set these values for the Scala HTTP library being used. So the client is using the operating system defaults. How bad are the default settings?

```bash
# more /proc/sys/net/ipv4/tcp_keepalive_* | cat
::::::::::::::
/proc/sys/net/ipv4/tcp_keepalive_intvl
::::::::::::::
75
::::::::::::::
/proc/sys/net/ipv4/tcp_keepalive_probes
::::::::::::::
9
::::::::::::::
/proc/sys/net/ipv4/tcp_keepalive_time
::::::::::::::
7200
```

`tcp_keepalive_time` value means that the first keep-alive packet will be sent after 2 hours. But can I change those Linux default values? I could do that for a VM or Linux running on bare metal easily. Kubernetes allows changing some kernel parameters but unfortunately the [tcp_* keep-alive settings are considered unsafe](https://kubernetes.io/docs/tasks/administer-cluster/sysctl-cluster/). I would need some administrator superpowers for writing YAML and configuring `kubelet` on AWS EKS.

And then I found out about [libkeepalive.so](https://tldp.org/HOWTO/html_single/TCP-Keepalive-HOWTO/#libkeepalive). I don't have to dig through Java classes trying to inject Socket options after it is created but before it is used. Or to perform Kubernetes black mass and pray to AWS gods. Nobody can stop me from adding new files to the container and setting additional environment variables.
