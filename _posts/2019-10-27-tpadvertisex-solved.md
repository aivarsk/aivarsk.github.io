---
layout: post
title: tpadvertisex(3c)&#58; a new cool Oracle Tuxedo 12.2 feature that works
---

[I wrote before about `tpadvertisex()` and how it did not work for me](/2019/10/26/tpadvertisex/)

It does! The documentation of `tpadvertisex()` [mentions "flags" argument](https://docs.oracle.com/cd/E72452_01/tuxedo/docs1222/rf3c/rf3c.html#2548645) but does not explain it's usage. I assumed that it's just a placeholder like some other Tuxedo functions have.

So I was working on some other project creating Python bindings for Oracle Tuxedo and needed a list of all error codes. And there it was in `atmi.h`:

```c
/*Flags to tpadvertisex*/
#define TPSINGLETON        0x00000001  /* advertise the singleton service */
#define TPSECONDARYRQ       0x00000002 /* advertise the service on the secondary queue for the MSSQ server*/
```

Turns out this is just a documentation bug. I tried calling `tpadvertisex()` with both flags set and it worked as expected: I got a unique service for servers in MSSQ configuration. But since 2 flag values can be combined I also explored each of them. Until Oracle updates the documentation here are my observations:

- `flags=0` - `tpadvertisex()` works exactly like `tpadvertise()` does and advertises services on the main queue.

- `flags=TPSINGLETON` - `tpadvertisex()` works like `tpadvertise()` does and advertises service on the main queue *but* it also checks that the service is unique and no other server advertises it. I get `TPENOSINGLETON` otherwise.

- `flags=TPSECONDARYRQ` - `tpadvertisex()` advertises the service on the secondary request queue or returns `TPENOSECONDARYRQ` if it is not configured. It allows for multiple servers to advertise the same service- strange, but maybe there is a use for that. I also observed that MIB returned `TA_RQADDR=UNKNOWN` for these services, probably an indication it's the secondary request queue.

- `flags=TPSINGLETON+TPSECONDARYRQ` - `tpadvertisex()` advertises the service on the secondary request queue or returns `TPENOSECONDARYRQ` if it is not configured. It also checks that the service is unique and returns `TPENOSINGLETON` otherwise. Again, MIB returned `TA_RQADDR=UNKNOWN` for these services.

[Here is a demo of `tpadvertisex()`](https://github.com/fuxedo/tuxedo-examples/tree/master/tpadvertisex)
