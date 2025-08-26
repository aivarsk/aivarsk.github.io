---
layout: post
title: Tracking Gunicorn's busy worker count
tags: django, gunicorn
---

I was investigating performance issues of a Django application running with Gunicorn behind a Nginx server. First, I added more timing information to Nginx access.log:

```
log_format timing '$remote_addr - $remote_user [$time_local] '
                  '"$request" $status $body_bytes_sent '
                  '"$http_referer" "$http_user_agent" rt="$request_time" uct="$upstream_connect_time" uht="$upstream_header_time" urt="$upstream_response_time"';

access_log /var/log/nginx/access.log timing;
```

After the Nginx reload, it started to report the total request time and the time waiting for a response from Gunicorn. I also checked the timing in Chrome developer tools. All the times matched, which means the network latency or Nginx was not to blame.

However, for a specific URL, response times were in the range of 3 to 22 seconds. The Gunicorn access log already contained the time spent processing the request by using [a custom access log format string](https://docs.gunicorn.org/en/stable/settings.html#access-log-format).  And within the Gunicorn, those URL requests took less than a second. It was clear that requests get buffered between Nginx and Gunicorn in the [connection backlog](https://docs.gunicorn.org/en/stable/settings.html#backlog), and Gunicorn needs pmore workers to process the requests](https://docs.gunicorn.org/en/stable/settings.html#worker-processes).

But how to find out how many Gunicorn workers are being used, how many are idle, and how to monitor that? I did not find a good answer. However,  Gunicorn has functions that are called [before a request is processed by the worker](https://docs.gunicorn.org/en/stable/settings.html#pre-request) and [after it has been processed](https://docs.gunicorn.org/en/stable/settings.html#post-request). I could use that to maintain the total number of active requests. But how to do that across multiple processes and avoid the lost updates?

Meet the [multiprocess Semaphore](https://docs.gunicorn.org/en/stable/settings.html#access-log-format)! It is a number living in the OS kernel memory: acquiring a semaphore decrements its value, releasing a semaphore increments the value, and a semaphore can’t become negative (acquiring will block). Normally, it is used for synchronization, but I will use it as an atomic gauge: release to increase it and acquire to decrease it.

Another trick I discovered: the Gunicorn access log can log the request headers. So instead of adding custom logs, I added a new HTTP header to the request and stored the counter value in it. And here is the complete solution for it:


```python
from multiprocessing import Semaphore

accesslog = "-"
access_log_format = "%(t)s [%({HTTP_X_REAL_IP}e)s] '%(m)s' %(s)s %(b)s '%(U)s' '%(q)s' '%(a)s' '%(D)s' busy=%({x-busy}i)s"


busy = Semaphore(0)


def pre_request(worker, req):
    busy.release()
    req.headers.append(("x-busy", str(busy.get_value())))


def post_request(worker, req, environ, resp):
    busy.acquire()
```
