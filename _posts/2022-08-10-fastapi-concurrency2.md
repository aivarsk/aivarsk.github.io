---
layout: post
title: FastAPI and cooperative multi-threading pt. 2
tags: python fastapi concurrency
---

After finding a good enough solution for [FastAPI and cooperative multi-threading](https://aivarsk.com/2022/01/21/fastapi-concurrency/) issues, a part of me was still not happy with the results. There was a significant drop in the number of concurrent requests:

```
1643620388 309
1643620389 5
1643620390 3
1643620391 6
1643620392 5
1643620393 322
```

It's not this bad in practice or for any realistic response size. The numbers above were obtained by running the whole process under `strace` which made it slower but also revealed that threads were waiting on a mutex (`futex(...)` functions) and even timing out while trying to acquire it. The other frequent activity was memory allocation. So I started to follow the JSON encoder of the response model as that was taking the most time.

It starts by encoding the root Pydantic model as a dictionary and then calling the [`json.dumps` function](https://github.com/samuelcolvin/pydantic/blob/36c53ceaa3e72876d4b438e124fc90a2cbc4ecef/pydantic/main.py#L490) and waiting for resulting JSON string. The `json` library has a mixed Python and C implementation. If possible, it will try to use [the C implementation for the speed](https://github.com/python/cpython/blob/0fc3517cf46ec79b4681c31916d4081055a7ed09/Lib/json/encoder.py#L14) unless you request a nice indented output.

So the entire JSON is encoded with a single C function call. Having written several Python modules in C, I know that the C code is called with the notorious Global Interpreter Lock held and no Python code can execute in the meanwhile. It is the responsibility of the C extension to release that lock if possible. And we are back to the topic of cooperative multi-threading but this time it's not coroutines and threads but Python threads and native threads.

CPython is like a single CPU computer that gives an impression of parallel processing by switching between all threads and letting each execute for a while. At any point in time, only a single thread of Python code is being executed. The interpreter switches between Python threads when a thread has executed a certain amount of instructions. However once Python calls a C extension the interpreter can no longer count instructions and interrupt the native thread. The C extension has to either finish or explicitly release the GIL before long-running system calls and acquire GIL once the system call returns. All other Python threads will starve while the C extension holds the GIL.

But if JSON is encoded by the C extension, why did I still observe some concurrent requests instead of zero? After going through the code once again I realized that the `json` library knows how to encode only the built-in Python types. But the response I am trying to serialize is a tree of Pydantic models. How does this not fail with "Account is not JSON serializable" error?

For unrecognized types `json.dumps` uses the optional parameter named `default`. [See the documentation for more details about it.](https://docs.python.org/3/library/json.html#json.dumps) The Pydantic library provides an implementation for this function that [converts the current model to a dictionary](https://github.com/samuelcolvin/pydantic/blob/36c53ceaa3e72876d4b438e124fc90a2cbc4ecef/pydantic/json.py#L72). `json.dumps` then encodes the resulting dictionary and may call `pydantic_encoder` again for Pydantic models it finds.

What happens is that Python is jumping back and forth between the `json` module C extension and the Pydantic encoder written in Python. And while Python instructions are executed, a threshold is reached for switching to a different Python thread running concurrent requests. And this is where I got stuck thinking of a way how to improve the concurrency.

A) I could convert all Pydantic models to dictionaries ahead of time so the C code completes faster without jumping back and forth between C extension and Python code. But that means no other code will be executed concurrently and I will be back to where I started.

B) I could use a pure Python implementation of `json.dumps` for better multithreading. But that will be slower and lead to longer response times. I think it puts more pressure on the memory because it collects all parts of the JSON string in a list and [builds a single string](https://github.com/python/cpython/blob/0fc3517cf46ec79b4681c31916d4081055a7ed09/Lib/json/encoder.py#L202) from that even when generators are used.

I did not like any of those choices so this is where I ended my quest for better concurrency and did not make any changes to my code.

Spoiler alert! Following the boy scout principle and leaving the world better than I found it I did a couple of patches to CPython and improved the performance of `json.dumps`.

![json.dumps]({{ site.url }}/public/jsondumps.png)
