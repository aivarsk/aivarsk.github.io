---
layout: post
title: Python JSON encoder apples and oranges and making it faster
tags: python cpython json
---

There are several JSON library benchmarks [like this one](https://catnotfoundnear.github.io/finding-the-fastest-python-json-library-on-all-python-versions-8-compared.html) where the built-in Python `json` library is not the worst performer but there are several libraries that have even up to 4x better result. Some of that might no longer be true for the latest versions of Python. But since I know a bit of Python JSON encoder, I decided to find out why there is a difference.

One of the answers is that the Python JSON encoder does more than others and the simplest example is trying to encode the following structure with circular references:

```python
a = {"a": 1}
b = {"b": 2}
a["b"] = b
b["a"] = a
```

Yes, it will fail with every encoder. But the one in Python does a ton of work just to give you a nice error message:

- `json` module gives you `ValueError: Circular reference detected` and have the exception notes with more details
- [ultrajson](https://github.com/ultrajson/ultrajson) gives you `OverflowError: Maximum recursion level reached`
- [orjson](https://github.com/ijl/orjson) gives you a `TypeError: Recursion limit reached`

Is it worth it? You can make the Python JSON encoder around 15% faster by turning off the check for circular references like this:

```python
json.dumps(data, check_circular=False)
```

I don't think so. Catching the `RecursionError` and raising a `ValueError` for backward compatibility would be much easier.  I will try to make a patch that improves the Python and also removes some code but [exception notes](https://docs.python.org/3/library/exceptions.html#BaseException.__notes__) and the Python JSON encoder tests around that do not make that easy.
