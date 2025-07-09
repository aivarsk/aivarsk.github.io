---
layout: page
title: About
---

My name is Aivars Kalvans. I wrote software (C, C++ and Python) that runs on Oracle Database and Tuxedo for more than 18 years. I tinker with internals of Oracle and various other software and develop [an Open Source alternative to Oracle Tuxedo](https://github.com/fuxedo/fuxedo) called [Fuxedo](http://fuxedo.io).

I have written a book about developing modern Oracle Tuxedo applications using Python. I also conduct Oracle Tuxedo training for developers and administrators, contact me for more information.

<a href="https://amzn.to/3ljktiH"><img src="https://m.media-amazon.com/images/I/61yT5MMhxDL._SY466_.jpg"></a>

Some other code I have written both bad and "good enough":

- [Django Task Queue](https://github.com/aivarsk/django-taskq) - asynchronous task queue with a Celery-compatible API using a database backend
- [Scruffy UML](https://github.com/aivarsk/scruffy) - creates UML diagrams from textual description.
- [libvmod-rewrite](https://github.com/aivarsk/libvmod-rewrite) - a plugin for Varnish (proxy server) that allows to modify cached pages.
- [pattern-matcher](https://github.com/aivarsk/pattern-matcher) and [cloud-instances](https://github.com/aivarsk/cloud-instances) are winning entries of Latvia Java User Group programming contest. The first one was written while I was still drunk after a party.
- [Scrapy proxies](https://github.com/aivarsk/scrapy-proxies) - Scrapy (web scraping framework) extension for using random proxies. Some people find it useful.
- ... and other Open Source on [github.com/aivarsk](https://github.com/aivarsk)
- ... and info about Closed Source on [stackoverflow.com/story/aivarsk](https://stackoverflow.com/cv/aivarsk)

Some contributions to OSS:

- Python 3.12
  - [Replace PyAccu with PyUnicodeWriter](https://github.com/python/cpython/issues/95005) to speed up `json.dumps` and to remove some C code
  - [Fastpath for encoding unsorted dict to JSON](https://github.com/python/cpython/issues/95385) to speed up `json.dumps`
- Django 5.1
  - [Allow customizing queryset in Model.refresh_from_db](https://github.com/django/django/commit/f92641a636a8cb75fc9851396cef4345510a4b52) for locking rows and fetching related models

I have given public talks on databases, payments and payment cards, Python, Django, Kafka and other topics. [I keep a list here.](/speaking)

[I have Opinions about how software should be done](/mindset)

You can reach me via e-mail: [aivars.kalvans@gmail.com](mailto:aivars.kalvans@gmail.com)

<small>
I write to think, I write to record, I write to put my thoughts together. Since I had some issues before:<br/>
The information in this weblog is provided “AS IS” with no warranties, and confers no rights.
This weblog does not represent the thoughts, intentions, plans or strategies of my employer. It is solely my opinion.</small>
