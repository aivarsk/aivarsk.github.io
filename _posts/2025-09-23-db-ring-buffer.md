---
layout: post
title: Ring buffer in the database
tags: django, ORM, database
---

We had a requirement to display the last N transactions on the ATM screen (”mini-statement”). The simplest solution is to keep the list of transactions, order them by date, and take the newest N transactions. But it gets tricky once you realize there are active customers making several transactions per day and inactive ones who use the card occasionally and might make a transaction every couple of months. Which means you have to preserve transaction records for a long period, and queries run more slowly. For a larger customer base, even half a year of data might be challenging to query in real time.

Now we start thinking of keeping only the last N transactions per payment card and doing an `INSERT` followed by a clever `DELETE` that discards all extra transactions. In a regular programming language, a better solution would be to have a [ring buffer](https://en.wikipedia.org/wiki/Circular_buffer) with a fixed capacity and a “write pointer” that points to where the newest record should be stored, overwriting the oldest one. To implement something similar in the database, we would need locking to prevent concurrent updates between retrieving the “write pointer”, storing the record, and advancing the “write pointer”.

Years ago, I came up with a solution that I still have mixed feelings about. It is nice because it can be done with a single SQL statement, requires no explicit locking, and it works (in production as well). On the other hand, it is a bit fugly, my internal DBA demon complains about the size of the WAL records, and it’s weird. Old colleagues called that “shifter” because it works like the bit-shifting operations `<<` and `>>`.

First, you create a model with as many message/transaction fields as you want to keep:

```python
class History(models.Model):
    ...
    message1 = models.TextField(blank=True, null=True)
    message2 = models.TextField(blank=True, null=True)
    message3 = models.TextField(blank=True, null=True)
    message4 = models.TextField(blank=True, null=True)
```

Every time you want to store a new message, you assign it to the field `message1`. Whatever value was there in `message1` you store it in `message2`. Whatever value was there in `message2`
you store it in `message3` , etc. And the value of the last field `message4` in this case is forgotten.

And here's how it works. We start with an empty ring buffer:

```python
>>> History.objects.get(pk=1).__dict__
{'_state': <django.db.models.base.ModelState object at 0x721f90e8bf50>, 'id': 1, 'message1': None, 'message2': None, 'message3': None, 'message4': None}
```

We add the first message and verify it is stored correctly:

```python
>>> History.objects.filter(pk=1).update(message4=F("message3"), message3=F("message2"), message2=F("message1"), message1="first message")
>>> History.objects.get(pk=1).__dict__
{'_state': <django.db.models.base.ModelState object at 0x721f90e8bfe0>, 'id': 1, 'message1': 'first message', 'message2': None, 'message3': None, 'message4': None}
```

We add the second message and verify it is stored correctly:

```python
>>> History.objects.filter(pk=1).update(message4=F("message3"), message3=F("message2"), message2=F("message1"), message1="second message")
>>> History.objects.get(pk=1).__dict__
{'_state': <django.db.models.base.ModelState object at 0x721f90e8bfb0>, 'id': 1, 'message1': 'second message', 'message2': 'first message', 'message3': None, 'message4': None}
```

We add the third message and verify it is stored correctly:

```python
>>> History.objects.filter(pk=1).update(message4=F("message3"), message3=F("message2"), message2=F("message1"), message1="third message")
>>> History.objects.get(pk=1).__dict__
{'_state': <django.db.models.base.ModelState object at 0x721f90ea8110>, 'id': 1, 'message1': 'third message', 'message2': 'second message', 'message3': 'first message', 'message4': None}
```
We add the fourth message and verify it is stored correctly:

```python
>>> History.objects.filter(pk=1).update(message4=F("message3"), message3=F("message2"), message2=F("message1"), message1="fourth message")
>>> History.objects.get(pk=1).__dict__
{'_state': <django.db.models.base.ModelState object at 0x721f90ea81a0>, 'id': 1, 'message1': 'fourth message', 'message2': 'third message', 'message3': 'second message', 'message4': 'first message'}
```

We add the fifth message and verify it is stored correctly and the first message has been discarded:

```python
>>> History.objects.filter(pk=1).update(message4=F("message3"), message3=F("message2"), message2=F("message1"), message1="fifth message")
>>> History.objects.get(pk=1).__dict__
{'_state': <django.db.models.base.ModelState object at 0x721f90ea82f0>, 'id': 1, 'message1': 'fifth message', 'message2': 'fourth message', 'message3': 'third message', 'message4': 'second message'}
```

So? Is this stupid or smart? After all these years I still have mixed feelings about it.
