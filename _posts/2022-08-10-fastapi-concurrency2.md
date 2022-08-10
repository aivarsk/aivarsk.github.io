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

The numbers above were obtained by running the whole process under `strace` which made it slower but also revealed that threads were waiting on a mutex (`futex(...)` functions) and even timing out while trying to acquire it. So I started to follow the JSON encoder of the response model as that was taking the most time.

It starts by encoding the root Pydantic model as a dictionary and then calling the [`json.dumps` function](https://github.com/samuelcolvin/pydantic/blob/36c53ceaa3e72876d4b438e124fc90a2cbc4ecef/pydantic/main.py#L490). The `json` library has a mixed Python and C implementation. If possible, it will try to use [the C implementation for the speed](https://github.com/python/cpython/blob/0fc3517cf46ec79b4681c31916d4081055a7ed09/Lib/json/encoder.py#L14).

So the entire JSON is encoded with a single C function call. Having written several Python modules in C, I know that the C code is called with the notorious Global Interpreter Lock held and no Python code can execute in the meanwhile. It is the responsibility of the C extension to release that lock if possible. And we are back to the topic of cooperative multi-threading but this time it's not coroutines and threads but Python threads and native threads.

In CPython, only a single thread of Python code is executed at once. The interpreter switches between Python threads once a thread has executed a certain amount of instructions. But the interpreter is helpless if Python code calls into C extension. C extension has to either finish or explicitly release the GIL around long-running system calls.

But if JSON is encoded by the C extension, why did I still observe some concurrent requests instead of zero? After going through the code I realized that the `json` library knows how to encode only built-in Python types. But the response I am trying to serialize is a tree of Pydantic models. How does this not fail with "Foo is not JSON serializable" error?

For that `json.dumps` accepts an optional parameter named `default` which is used to encode unrecognized types. [Check the documentation for more details.](https://docs.python.org/3/library/json.html#json.dumps) And Pydantic library provides an implementation for the function that [converts the current model to a dictionary](https://github.com/samuelcolvin/pydantic/blob/36c53ceaa3e72876d4b438e124fc90a2cbc4ecef/pydantic/json.py#L72).

What then happens is that Python is jumping back and forth between the `json` module C extension and the Pydantic encoder written in Python. And while Python instructions are executed, a threshold is reached for switching to a different Python thread running concurrent requests. And this is where I got stuck.

A) I could convert all Pydantic models to dictionaries so the C code completes faster without jumping back and forth between C extension and Python code. But that means no other code will be executed concurrently and I will be back where I started.

B) I could use a pure Python implementation of `json.dumps` for better multithreading. But that will be slower and it puts more pressure on the memory because it collects all parts of the JSON string in a list and [builds a single string](https://github.com/python/cpython/blob/0fc3517cf46ec79b4681c31916d4081055a7ed09/Lib/json/encoder.py#L202) from that even when generators are used.

I do not like any of those choices because they have too many cons so I did a couple of patches to CPython and improved the performance of `json.dumps`. I could not improve concurrency but at least there is a positive outcome from this investigation.

![json.dumps]({{ site.url }}/public/jsondumps.png)
