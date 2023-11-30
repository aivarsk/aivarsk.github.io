---
layout: post
title: Optimizing Python application's Docker image with strip
tags: python docker pip
---

I was going through some Docker images of large applications and looking at the layer sizes with `dive`. A lot of space was used by installed dependencies under `/usr/local/lib/python3.10/dist-packages/` and virtual environments. Yes, there were a lot of `.py` and `.pyc` files as you would expect but the largest files were the `.so` files. In one of the cases, there were 269M of compiled C and C++ code in shared library files (`find /usr/local/lib/python3.10/dist-packages/ -name "*.so" | xargs du -hsc). I suspected that it was too much and most of them might contain the debug information. And I don't think Python developers would use the `gdb` and look at backtraces of C++ code. So I tried to remove the debug information with `strip`.  And behold - the size was reduced to 119M, and more than 100M were saved.

So now I added striping of libraries right after the `pip install` command in `Dockerfile`:

```bash
find /usr/local/lib/python3.10/dist-packages/ -name "*.so" | xargs strip
```

Even for simple projects with SQLAchemy, it reduces shared library size from 4.9M to 980K. 
