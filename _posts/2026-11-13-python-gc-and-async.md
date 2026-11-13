---
layout: post
title: Python GC and async
tags: python, gc, async
---

Python's GC is quite simple and it's good (easy to understand) and bad (continue reading) at the same time. [I did a 5-minute lightning talk about it before](https://www.youtube.com/watch?v=acQdzpt68Fk). Let's look at Python 3.12.

There are two kinds of objects: primitive ones with just the reference counter and GC-collectable containers that may contain other objects. Allocation and deallocation of GC-collectable objects keep a gauge ticking. Once it goes over 700 (new object allocations minus deallocations), the 0th generation is collected. 10 0th generation collections trigger the 1st generation collection and 10 1st generation collections trigger the full GC (2nd generation). It says so in the default thresholds:

```python
>>> import gc
>>> gc.get_threshold()
(700, 10, 10)
>>>
```

700 is a very small number for a modern JSON API with Pydantic models. Now add concurrent requests with `asyncio` (hello, FastAPI, my old friend) and you reach 700 quickly and start collecting next generations. Add an OTEL SDK to the mix that collects spans, metrics and logs in memory... you burn through 100 (10x10) rounds of garbage collection quickly and get a full 2nd generation collection. And to make things worse - `asyncio` depends on cooperative multitasking and Python's `gc` is a stop-the-world collector.

Long story short, that's how I got 180+ ms-long GC pauses collecting the 2nd generation. To be fair, K8s pods are a bit shy with CPU requests. But still. All the nice concurrent `async def` and `await` code gets stuck for 1/5 of a second, a couple of times per second. Which leads to more in-flight requests with more allocations and more GC. Now I am adding code to track long GC pauses (more than 1ms) to all my projects. Something like this.

```python
from time import perf_counter
import gc

_gc_started_at = None
def _record_gc_pause(phase: str, info: dict[str, int]) -> None:
    global _gc_started_at
    if phase == "start":
        _gc_started_at = perf_counter()
        return
    if _gc_started_at is None:
        return
    pause_seconds = perf_counter() - _gc_started_at
    _gc_started_at = None
    if pause_seconds > 0.001:
        print(f"GC {info['generation']} paused for {pause_seconds}s")

gc.callbacks.append(_record_gc_pause)
```


To make things a bit better, I use [`gc.freeze`](https://docs.python.org/3/library/gc.html#gc.freeze). After startup but right before starting to handle incoming requests, I force a full garbage collection and move all objects to permanent generation so the following 2nd generation collection cycles do not touch them. That takes care of all imported modules, configuration, etc. Reference cycles involving frozen objects won't get collected but that is an acceptable risk.

```python
gc.collect()
gc.freeze()
```


GC pauses are not the only way an `asyncio` service freezes. I am working on an "asyncio's FIFO vs. six decades of scheduler research" talk for the 2027 conference season. I think GC will get a few slides there along with other war stories.
