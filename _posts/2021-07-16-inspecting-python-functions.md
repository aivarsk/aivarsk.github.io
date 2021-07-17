---
layout: post
title: Inspecting Python function
tags: python c++ pybind11
---

I have [a piece of C++ code](https://github.com/aivarsk/tuxedo-python/blob/master/src/tuxedo.cpp) that calls user-defined functions implemented in Python.  Instead of requiring all functions to have the same signature with 6 arguments, the C++ code inspects the function signature and passes only the arguments function accepts - 1, 3, or all 6 of them. I use the `inspect` module and `getargspec` function for that but it feels a bit wrong and bloated. So let's see how we can get the work done without the `inspect` module.

For extra fun, I will look at Python 3.8 that has [PEP 570](https://www.python.org/dev/peps/pep-0570/) about positional-only arguments implemented.

A function definition using all features looks like this:

```python
def f(pos1, pos2, /, pos_or_kwd, *, kwd1, kwd2):
```

Arguments  `pos1` and `pos2` before the `/` are positional-only arguments. You can call the following function like `f(1, 2)` but not like `f(a=1, b=2)` or you will get a `TypeError`:

```python
>>> def f(a, b, /): pass
... 
>>> f(1, 2)
>>> f(a=1, b=2)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: f() got some positional-only arguments passed as keyword arguments: 'a, b'
```

Arguments  `kwd1` and `kwd2` after the `*` are keyword-only arguments. You can call the following function like `f(a=1, b=2)` and `f(b=2, a=1)` but not like `f(1, 2)` or you will get a `TypeError`:

```python
>>> def f(*, a, b): pass
... 
>>> f(a=1, b=2)
>>> f(b=2, a=1)
>>> f(1, 2)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: f() takes 0 positional arguments but 2 were given
```

And of course, without the positional-only and keyword-only arguments, you can call the function however you like. `f(a=1, b=2)` or `f(b=2, a=1)` or `f(1, 2)` will work:

```python
>>> def f(a, b): pass
... 
>>> f(a=1, b=2)
>>> f(b=2, a=1)
>>> f(1, 2)
```

Let's look how to inspect the following function containing most of the function argument features:

```python
>>> def f(pos1, pos2=2, /, pos_or_kwd=3, *, kwd=4, kwd2=5, kwd3=6): pass
... 
```

How to retrieve the number of arguments this function accepts? Python has a lot of dunder attributes containing interesting bits. First, under `__code__` there are  three interesting attributes:

```python
>>> f.__code__.co_argcount
3
>>> f.__code__.co_kwonlyargcount
3
>>> f.__code__.co_posonlyargcount
2
```

- `__code__.co_argcount` will hold the value `3` which is the number of arguments not including keyword-only arguments
- `__code__.co_kwonlyargcount` will hold the value `3` which is the number of keyword-only arguments
- `__code__.co_posonlyargcount`will hold the value `2` which is the number of positional-only arguments.

To get the number of function arguments you have to sum `__code__.co_argcount` and `__code__.co_kwonlyargcount`. However, `__code__.co_posonlyargcount` is already included in `__code__.co_argcount`.

To find the names of arguments, even the positional-only ones (!), we have to look at `__code__.co_varnames`. It contains both the function arguments and local variables and we have to take only the first elements matching the argument count:

```python
>>> f.__code__.co_varnames[:f.__code__.co_argcount+f.__code__.co_kwonlyargcount]
('pos1', 'pos2', 'pos_or_kwd', 'kwd', 'kwd2', 'kwd3')
```

What about the default values of the arguments? They live on the function object itself:

```python
>>> f.__defaults__
(2, 3)
>>> f.__kwdefaults__
{'kwd': 4, 'kwd2': 5, 'kwd3': 6}
```

- `__defaults__` contains a `tuple` with default values or `None` if there are none. It has the values for all except keyword-only arguments.
- `__kwdefaults__` contains a `dict` with the default values of keyword-only arguments.

For our function argument count (`__code__.co_argcount`) is 3 but the  `__defaults__` contains just 2 default values. Since default arguments must follow non-default arguments, to match defaults with the actual argument names we can do something like this:

```python
>>> dict(zip(
...     f.__code__.co_varnames[: f.__code__.co_argcount][-len(f.__defaults__) :],
...     f.__defaults__,
... ))
{'pos2': 2, 'pos_or_kwd': 3}
```

But what about `*args` and `**kwargs` for arbitrary positional and keyword arguments? Those are special:

```python
>>> def f(*args, **kwargs): pass
... 
>>> f.__code__.co_argcount
0
>>> bin(f.__code__.co_flags)
'0b1001111'
```

They do not affect the argument count (`__code__.co_argcount`) and it will be `0`. Instead, the fact that the function accepts arbitrary arguments is reflected as bits inside flags (`__code__.co_flags`). The third bit (or 4) indicates that the function accepts arbitrary positional arguments and the fourth bit (or 8) indicates that the function accepts arbitrary keyword arguments.

These arguments do not have to be named `args` and `kwargs`, it is just a naming convention. The real argument names can be found in `__code__.co_varnames` but after all function arguments:

```python
>>> def f(pos1, pos2=2, /, pos_or_kwd=3, *args, kwd=4, kwd2=5, kwd3=6, **kwargs): pass
... 
>>> f.__code__.co_varnames
('pos1', 'pos2', 'pos_or_kwd', 'kwd', 'kwd2', 'kwd3', 'args', 'kwargs')
```

Long story short, I replaced the use of `inspect` module with a few lines of direct `pybind11` C++ code:

```c++ 
auto &&func = server.attr(svcinfo->name);
auto &&code = func.attr("__code__");
long argcount = (code.attr("co_argcount") + code.attr("co_kwonlyargcount"))
                    .cast<py::int_>();
auto &&args = code.attr("co_varnames")[py::slice(0, argcount, 1)];
```
