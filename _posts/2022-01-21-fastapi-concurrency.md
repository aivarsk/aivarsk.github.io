---
layout: post
title: FastAPI and cooperative multi-threading
tags: python fastapi concurrency
---

[Cal Paterson wrote a great article](https://calpaterson.com/async-python-is-not-faster.html) comparing and describing synchronous and asynchronous Python frameworks and explaining why
asynchronous frameworks go a bit wobbly under load. This is a story of how we experienced wobbliness in a recent project.

We are using FastAPI, Pydantic, and Kubernetes to build microservices. One of them is a query service that returns a paginated result containing a list of entities implemented as Pydantic models. During tests, we tried to retrieve thousands of entities from the API endpoint. It took several seconds to produce results as we expected but some requests failed. As we started to investigate, it turned out that the liveness and readiness probes of the Kubernetes container failed and containers were restarted by Kubernetes leading to failing requests. Why didn't the FastAPI service respond to probes? It was alive and working and FastAPI should be able to handle concurrent requests.

Let's start with a simplified service code for testing this behavior in isolation. The response model still contains a lot of fields because it is the key to triggering the issue we faced. The real models have even more fields.

```python
from datetime import date, datetime
from typing import List

from fastapi import FastAPI
from pydantic import BaseModel

  
class Address(BaseModel):
    id: int
    str1: str = None
    str2: str = None
    str3: str = None
    str4: str = None
    str5: str = None
    str6: str = None
    str7: str = None
    str8: str = None


class Account(BaseModel):
    id: int
    address: Address = None
    str1: str = None
    str2: str = None
    str3: str = None
    str4: str = None
    str5: str = None
    str6: str = None
    str7: str = None
    str8: str = None
    str9: str = None
    str10: str = None
    str12: str = None
    str13: str = None
    str14: str = None
    str15: str = None
    str16: str = None


class Client(BaseModel):
    id: int
    address: Address = None
    bank_accounts: List[Account]
    str1: str = None
    str2: str = None
    str3: str = None
    str4: str = None
    str5: str = None
    str6: str = None
    str7: str = None
    str8: str = None

class ClientsResponse(BaseModel):
    items: List[Client]


app = FastAPI()


@app.get("/.well-known/live")
def live():
    return "OK"


@app.get("/clients", response_model=ClientsResponse)
def clients():
    return ClientsResponse(
        items=[
            Client(id=i, address=Address(id=i), bank_accounts=[Account(id=i)])
            for i in range(40000)
        ]
    )
```

This service provides two endpoints: `/.well-known/live` for liveness checks and `/clients` for returning a list of clients.

The second piece of code will test the concurrency of the service by calling the liveness probe endpoint and counting how many requests per second it can process:

```python
import time
  
import requests

count = 0
second = int(time.time())
while True:
    try:
        r = requests.get("http://localhost:8000/.well-known/live", timeout=1)
        count += 1
    except requests.exceptions.ReadTimeout as ex:
        pass
    now = int(time.time())
    if now != second:
        print(second, count)
        second = now
        count = 0
```

Once both scripts are running I see that the current setup can process 600 liveness probe requests per second. As soon as I request the real endpoint `curl localhost:8000/clients` these numbers drop and stay at 0 for several seconds:

```
1642154590 673
1642154591 649
1642154592 384
1642154593 0
1642154594 0
1642154595 0
1642154596 0
1642154597 0
1642154598 0
1642154599 0
1642154600 0
1642154601 0
1642154602 1
1642154603 608
1642154604 664
```

What is happening? FastAPI is an asynchronous framework.
Unlike traditional multi-threading where the kernel tries to enforce fairness by brutal force, FastAPI relies on cooperative multi-threading where threads voluntarily yield their execution time to others.  Services can be implemented both as coroutines (`async def`) or regular functions. Synchronous functions which are not yielding their execution time are called through a thread pool to ensure they do not block the main execution thread.

Despite doing their best to run concurrently, FastAPI still has synchronous code that is executed from the main thread. Some of those functions do a lot of work and may clog the main thread when processing many large response objects. These functions are:

- [`_prepare_response_content`](https://github.com/tiangolo/fastapi/blob/master/fastapi/routing.py#L120) converts Pydantic models to Python dictionaries.
- [`jsonable_encoder`](https://github.com/tiangolo/fastapi/blob/master/fastapi/encoders.py#L29) ensures that the whole object tree can be converted to JSON. It does the most work for our test case.

So what is the solution to improve the concurrency of FastAPI services? One of the solutions is to run several Uvicorn workers and hope that all of them are not clogged at the same time.

The other solution is to off-load the encoding of the response to another thread and unblock the main thread. FastAPI even has a special response type `Response` that skips the `_prepare_resonse_content` and `jsonable_encoder` functions and returns response data as-is. Since our service function is already executed through a thread pool, we can convert the response to JSON there. And it requires minimal changes to the code:

```python
    from fastapi.responses import Response
    return Response(
        content=ClientsResponse(
            items=[
                Client(id=i, address=Address(id=i), bank_accounts=[Account(id=i)])
                for i in range(40000)
            ]
        ).json(),
        media_type="application/json",
    )
```

With those changes applied, the FastAPI service behaves much better:

```
1642158924 551
1642158925 666
1642158926 578
1642158927 13
1642158928 9
1642158929 2
1642158930 423
1642158931 690
1642158932 661
1642158933 692
```

There still is a drop in the number of concurrent requests but the service experiences wobbliness for a shorter period and can respond to liveness probes.
