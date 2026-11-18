---
layout: post
title: Running Oracle Tuxedo on modern Linux
tags: tuxedo, tuxedo-python, fuxedo
---

I resumed working on several of my side projects. One of them is [tuxedo-python](https://aivarsk.com/tuxedo-python/) library for writing Oracle Tuxedo servers and clients in Python.

While I did manage to install Tuxedo after several attempts, the modern `glibc>=2.41` (I have 2.43, check with `ldd --version`) no longer lets you `dlopen()` shared objects that are requesting an executable stack (RWE). This is the error message I got: an `ImportError: libengine.so: cannot enable executable stack`. That's OK, my Ubuntu is not on the supported OS list. But I still want to run Tuxedo on it. And this is how I solved it and have not run into any issues. Yet.

First you need to install the `patchelf` utility:

```bash
sudo apt install patchelf
```


After that you can clear the executable-stack flag on the `PT_GNU_STACK` header: it is changed from `RWE` to `RW`. I suspect it's because Tuxedo 12.2 was built with an old toolchain. Fine for Tuxedo 12.2, but a bit surprising that the latest Oracle Tuxedo 22.1 needed the same treatment. Among other things it reduces attack surface.

```bash
patchelf --clear-execstack $TUXDIR/lib/libengine.so $TUXDIR/lib/libsecssl.so
```

With those changes applied [tuxedo-python-1.0.12](https://aivarsk.com/tuxedo-python/) is now cross-tested with Oracle Tuxedo 12.2.2, 22.1 and my own [Fuxedo](https://fuxedo.io).
