---
layout: post
title: The O of SOLID
tags: SOLID OCP open closed
---

[Wikipedia says](https://en.wikipedia.org/wiki/SOLID) that O stands for:

| Open–closed principle: "Software entities ... should be open for extension, but closed for modification."

Once again, let's go to the actual article written by Robert C. Martin 
[The Open-Closed Principle](https://web.archive.org/web/20150905081105/http://www.objectmentor.com/resources/articles/ocp.pdf). This time the article does not have any redefinitions and cites a principle described in another book: [Object Oriented Software Construction, Bertrand Meyer, Prentice Hall, 1988, p 23](https://archive.org/details/objectorientedso00unse)

This book first defines the open/closed principle and then proposes in another chapter to use inheritance to solve it. However, what was the real problem OCP is attempting to solve?

Here are some quotes from the book to think about.

| The openness requirement is inescapable, because we seldom grasp all the implications of subproblems when we start solving it. This means we inevitably overlook important aspects when we write a module, and must be prepared to add new items later on. But closing modules is just as indispensable: **we cannot wait until all the information is in to make a module available for others. If we did, no multi-module software would ever be produced, as work on every module would be hanging on the completion of work on the suppliers.**

| In classical design approaches, the solution is to close modules when it appears that they have reached a reasonable level of stability, and then, when a modification is needed, to reopen them. **But if you reopen a module, you must also reopen all its clients to update them, since they rely on the old version.**

| For example, if you realize later in the project that the library also needs to handle a type of publication not previously considered, like technical reports, with its specific fields, the above declaration must be changed. **All client modules must be recompiled; besides, they must probably be changed...**

Do you see where it's going? It talks mostly about compile-time dependencies and scaling development by not blocking other teams waiting on shared software artifacts. It speaks of specific lower-level languages like C and C++, where the number of fields, field order, and virtual functions affect the physical bits in the memory and may require recompilation. And I sense the waterfall approach here.

## Final thoughts

This is the most followed principle **in libraries** because every time a Django developer inherits from a `Model`, `View`, and other classes - it is OCP or at least can be put behind the OCP banner. But it's different in "our code".

A team working on its services does not have a problem that OCP tries to solve. You compile/build the whole project anyway, you run all test cases anyway, etc. If other teams are not blocked by waiting on your implementation and changes do not require **massive adjustments and recompilation for other teams**, there is no problem. Instead, most SOLID tutorials invent the problem when there is none and add needless abstractions and extension points. You should not be afraid to modify your own code.
