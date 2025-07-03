---
layout: post
title: Transactional task outbox in Django with django-taskq
tags: django, celery, outbox, django-taskq
---

We have given up on distributed transactions (2PC) but have not given up working with multiple resources like the database, message brokers, and queues. Instead, everybody tries to build their own atomic operations over multiple resources. Some do code&pray, and others try to have a database as a source of truth with [the transactional outbox pattern](https://microservices.io/patterns/data/transactional-outbox.html).

[django-taskq](https://pypi.org/project/django-taskq/) uses the Django database as the one and only backend for storing tasks (function calls with parameters). Because it uses the Django ORM under the hood, it also obeys Django transactions:

```python
from django_taskq.celery import shared_task

@shared_task(queue="kafka-events", autoretry_for=(Exception,))
def something_happened(*, key: str, value: str):
    ...

with transaction.atomic():
    model1.save()
    model2.save()
    something_happened.delay(key=str(uuid.uud4()), value=payload.model_dump_json())
```

Either all models will be updated and a new task scheduled or none of it will happen. And when the task runs, the model changes will be there in the database. I guess all of us have a Celery story about having a similar code and tasks executed before changes are committed and visible in the database. At which point everybody starts using `transaction.on_commit` that works most of the time while the broker keeps running, the network to broker is reliable and Redis or application is not being killed by OOM killer.

Django taskq still does not prevent failures while the task is being executed or the task not being idempotent but at least both the model changes and the task is recorded atomically and task failure or success will be recorded in the database as well.
