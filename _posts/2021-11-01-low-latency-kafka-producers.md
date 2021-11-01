---
layout: post
title: Optimizing Kafka producers for latency
tags: python c++ kafka latency
---

TL;DR Don't forget to set `socket.nagle.disable=True` to disable Nagle's algorithm

The code I am working with uses [Confluent Kafka Python library](https://github.com/confluentinc/confluent-kafka-python) that calls [librdkafka C++ library](https://github.com/edenhill/librdkafka) underneath. The code uses synchronous messaging: produces events one-by-one and ensures it is persisted in Kafka topic by waiting for acknowledgments from all replicas. This code is sensitive to latency and not so much to throughput.

Several articles talk about tuning Kafka for latency. Even `librdkafka` mentions the two most important configuration properties for performance tuning:

`batch.num.messages` - the minimum number of messages to wait for to accumulate in the local queue before sending off a message set.
`queue.buffering.max.ms` - how long to wait for batch.num.messages to fill up in the local queue.

`queue.buffering.max.ms` is the same `linger.ms` mentioned by other sources according to [librdkafka documentation](https://raw.githubusercontent.com/edenhill/librdkafka/master/CONFIGURATION.md). But that's about the only *real* advice those articles provide. Some go into replication factor and number of acknowledgments and sacrificing durability for performance.

I wrote short python code that produces messages sequentally in a loop and calculates the time it takes:
```python
for _ in range(100):
    producer.produce(topic='test', value=value, key=key, on_delivery=delivery_callback)
    producer.flush(timeout=10.0)
```

To my surprise, it resulted in just 20 messages per second which are far below performance numbers advertised for Kafka. I am using Python without message batching, a single producer, waiting for all ISR brokers, etc. But should the numbers be that low? The first step was to turn on additional debug by setting the parameter `debug=protocol`. That provided network round trip times from the C++ library level, not Python level:

```
%7|1635461062.321|SEND|rdkafka#producer-4| [thrd:ssl://xxx.yyy.zzz.amazonaws.com:9]: ssl://xxx.yyy.zzz.amazonaws.com:9094/3: Sent ProduceRequest (v7, 2219 bytes @ 0, CorrId 4)
%7|1635461062.364|RECV|rdkafka#producer-4| [thrd:ssl://xxx.yyy.zzz.amazonaws.com:9]: ssl://xxx.yyy.zzz.amazonaws.com:9094/3: Received ProduceResponse (v7, 89 bytes, CorrId 4, rtt 43.15ms)
```
The `rtt 43.15ms` means the round-trip took 43.15 milliseconds which gives 23 messages per second. At first, I suspected network latency, so I used `tcptraceroute` to measure it. Regular `ping` and `traceroute` were not useful because of firewalls between the application and Kafka broker.

```bash
tcptraceroute xxx.yyy.zzz.amazonaws.com 9094 
Selected device eth0, address 10.13.93.124, port 38695 for outgoing packets
Tracing the path to xxx.yyy.zzz.amazonaws.com (10.19.42.242) on TCP port 9094, 30 hops max
 1  10.13.86.235  0.055 ms  0.041 ms  0.040 ms
 2  * * *
 3  10.19.42.242 [open]  0.840 ms  1.116 ms  1.000 ms
```

The network latency was not to blame, it was around 1ms. For a short time I wanted to blame Python's GIL, threads, callbacks from C++ code into Python, etc. So I used `strace` to check what is happening on the system call level:

```
1635773046.071423 write(12, ..., 30) = 30
1635773046.071522 write(2, "%7|1635773046.071|SEND|rdkafka#producer-1| [thrd:ssl://xxx.yyy.zzz.amazonaws.com:9]: ssl://xxx.yyy.zzz.amazonaws.com:9094/3: Sent ProduceRequest (v7, 2219 bytes @ 0, CorrId 5)\n", 236) = 236
1635773046.072581 poll([{fd=12, events=POLLIN}, {fd=7, events=POLLIN}], 2, 538 <unfinished ...>
1635773046.120417 <... poll resumed>) = 1 ([{fd=12, revents=POLLIN}])
1635773046.120485 read(12, ..., 5) = 5
1635773046.120568 read(12, ..., 28) = 28
1635773046.120677 poll([{fd=12, events=POLLIN}, {fd=7, events=POLLIN}], 2, 490) = 1 ([{fd=12, revents=POLLIN}])
1635773046.120763 read(12, ..., 5) = 5
1635773046.120909 read(12, ..., 117) = 117
1635773046.120999 write(2, "%7|1635773046.120|RECV|rdkafka#producer-1| [thrd:ssl://xxx.yyy.zzz.amazonaws.com:9]: ssl://xxx.yyy.zzz.amazonaws.com:9094/3: Received ProduceResponse (v7, 89 bytes, CorrId 5, rtt 49.50ms)\n", 248) = 248
```

Turns out the library is really waiting more than 40ms for a response from the broker using the `poll` system call.

Hopelessly going through [librdkafka documentation](https://raw.githubusercontent.com/edenhill/librdkafka/master/CONFIGURATION.md), I found a parameter named `socket.nagle.disable` with the default value of `False`. This setting keeps [Nagle's algorithm](https://en.wikipedia.org/wiki/Nagle%27s_algorithm) enabled to trade latency for throughput at the kernel level. I grepped through the source code and saw that `socket.nagle.disable` sets  `TCP_NODELAY` parameter on the socket which is something I do when writing C code.

So the mystery was solved. I set the `socket.nagle.disable` to `True` and my code finally started to perform by producing more than 250 messages per second:

```%7|1635781927.658|SEND|rdkafka#producer-3| [thrd:ssl://xxx.yyy.zzz.amazonaws.com:9]: ssl://xxx.yyy.zzz.amazonaws.com:9094/3: Sent ProduceRequest (v7, 2219 bytes @ 0, CorrId 37)
%7|1635781927.661|RECV|rdkafka#producer-3| [thrd:ssl://xxx.yyy.zzz.amazonaws.com:9]: ssl://xxx.yyy.zzz.amazonaws.com:9094/3: Received ProduceResponse (v7, 89 bytes, CorrId 37, rtt 2.88ms)
```

What still makes me wonder is why `librdkafka` does not set `TCP_NODELAY` by default. Because they are doing something similar to Nagle's algorithm in the library code by using `linger.ms`/ `queue.buffering.max.ms` settings and buffering messages. 

Long story short, to optimize producers for latency, you should set both:

`socket.nagle.disable` = True
`queue.buffering.max.ms` = 0

I thinkg you should also set `socket.nagle.disable` for low latency consumers to deliver acknowledgments as soon as possible.
