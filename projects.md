---
layout: page
title: Projects
---

Things I build and maintain, the order does not correlate with how much of my head they rent.

- [Fuxedo](https://fuxedo.io) - open source implementation of X/Open XATMI and the Oracle Tuxedo extensions applications actually use, compatible at API, command and System V IPC level
- [tuxedo-python](https://aivarsk.com/tuxedo-python/) - Python bindings for Oracle Tuxedo and Fuxedo, write servers and clients as Python classes
- [django-taskq](https://aivarsk.com/django-taskq/) - asynchronous task queue with a Celery-compatible API using the Django database as the one and only backend
- [sysvipc-python](https://aivarsk.com/sysvipc-python/) - Python wrappers for System V IPC: message queues, semaphores and shared memory
- [OTELite](https://otelite.io) - disposable OTEL backend: OTLP/HTTP in, traces and metrics in one SQLite file, logs as JSONL, one page to see it all, built for CI runs and dev clusters
- [Hisstrace](https://hisstrace.io) - syscall tracing scripted as Python coroutines: wait for the syscall, keep state between awaits, rewrite arguments, set breakpoints
- [microSRE](https://microsre.io) - local-first Linux health checks: reads /proc directly, USE-method signals, log pattern alerts straight to Slack, Discord or Telegram, no Prometheus or agent stack

Smaller experiments and things I no longer admit to live on [github.com/aivarsk](https://github.com/aivarsk).
