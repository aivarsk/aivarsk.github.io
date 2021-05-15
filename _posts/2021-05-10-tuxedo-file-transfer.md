---
layout: post
title: Tracking Oracle Tuxedo file transfer
tags: oracle tuxedo msgsnd file transfer
---

Oracle Tuxedo uses System V IPC message queues for sending messages between processes. These queues live inside the OS kernel and are limited in size:

```bash
sudo sysctl -a | grep kernel.msgm
```

Unless you have modified these parameters, you should see the following result on Linux:


```bash
kernel.msgmax = 8192
kernel.msgmnb = 16384
kernel.msgmni = 32000
```

`kernel.msgmax` is the maximum size of an individual message and `kernel.msgmnb` is the maximum number of bytes in a single queue. However, Tuxedo can send megabytes of data between processes. This technique is called file transfer. Instead of storing the message in the queue, Tuxedo will store the message in a temporary file and put a tiny message in the queue containing the filename as shown below:

![IPC queue]({{ site.url }}/public/ipcq.png)


The file transfer is slower than sending the whole message through the queue. It is acceptable for some use cases but if most of your messages are sent through files, you should think about resizing Tuxedo queues.

But how can you know if Tuxedo used file transfer to send the message? Let's investigate!

We will use a simple `toupper.py` server that converts a string to upper-case:


```python
#!/usr/bin/env python3
import sys
import tuxedo as t


class Server:
    def TOUPPER(self, req):
        return t.tpreturn(t.TPSUCCESS, 0, req.upper())


t.run(Server(), sys.argv)
```

We will also need a UBBCONFIG with configuration for our server:


```bash
*RESOURCES
MASTER tuxapp
MODEL SHM
IPCKEY 32769

*MACHINES
"15c365dcb562" LMID=tuxapp
    TUXCONFIG="/home/oracle/code/tuxconfig"
    TUXDIR="/home/oracle/tuxhome/tuxedo12.2.2.0.0"
    APPDIR="/home/oracle/code"

*GROUPS
GROUP1 LMID=tuxapp GRPNO=1

*SERVERS
"toupper.py" SRVGRP=GROUP1 SRVID=1
    REPLYQ=Y MAXGEN=2 RESTART=Y GRACE=0
    CLOPT="-s TOUPPER:PY"
```

File transfer messages are stored inside the temporary folder (`/tmp`) and start with "TUX" followed by a unique random string. We could scan this folder for new files but there is a better mechanism: [inotify](https://man7.org/linux/man-pages/man7/inotify.7.html). We will subscribe to notifications about new files created inside the temporary folder.

Filesystem monitoring will be running in a background thread and we will send messages with a size close to the system limits to observe how first the request message is transferred through files and later both the request and response send through files.


```python
import tuxedo as t
import os
import inotify.adapters
import inotify.constants
import threading


def monitor():
    i = inotify.adapters.Inotify()
    i.add_watch("/tmp", mask=inotify.constants.IN_CREATE)

    for event in i.event_gen(yield_nones=False):
        (_, type_names, path, filename) = event
        if filename.startswith("TUX"):
            print("\t{} {}".format(",".join(type_names), os.path.join(path, filename)))


threading.Thread(target=monitor, daemon=True).start()


for i in range(7400, 7800):
    print("tpacall({} bytes)".format(i))
    cd = t.tpacall("TOUPPER", "a" * i)
    print("tpgetrply()")
    t.tpgetrply(cd)
```

By running the program, you will get output like this:

```bash
tpacall(7798 bytes)
        IN_CREATE /tmp/TUXRUwjPT317790016
tpgetrply()
        IN_CREATE /tmp/TUXZP86iD844330816
tpacall(7799 bytes)
        IN_CREATE /tmp/TUX1llyD1317790016
tpgetrply()
        IN_CREATE /tmp/TUXNuGq7K844330816
```

As I mentioned before, file transfer is not bad but you could monitor how many messages are sent through files compared to the number of service calls or transactions per second. And this simple Python program using the `inotify` library could be part of your Oracle Tuxedo application monitoring.


Read about this and other things about Oracle Tuxedo in [my book about Modernizing Oracle Tuxedo Applications with Python](https://amzn.to/3ljktiH).
