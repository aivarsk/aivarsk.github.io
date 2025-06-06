---
layout: post
title: My Friday's "old man yells at cloud" moment
tags: software, development
---

[Casey Muratori had this to say](https://www.youtube.com/watch?v=0WYgKc00J8s&t=2096s):

> You should never take library design advice from anyone who
> hasn't had to make a living selling a library in a competitive arena.

I will rephrase it slightly and put it on the wall:

**"You should never take software design advice from anyone who hasn't had to make a living selling software in a competitive arena."**

"Never take" might be too strong but at least be skeptical. Too much advice and ideas come from people who get paid hourly or the project takes as long as it takes and costs as much as it does. Or they move on to the next shiny project as soon as it's done and never look back. In a way, their assumptions about development, maintenance, costs, and extensibility never get challenged or measured. But but but the "developer performance"... compared to what?

Selling software or running it for years does the reality check. Does your methodology and "tech stack" leave a space for a profit margin? Do your state machines, event logs, and databases allow you to recover from bugs and incidents? Why extensive testing did miss those bugs? How often do you change the database, XML/JSON parser, or the (G)UI? Or things you put into configuration files because you might want to modify those later? How easy is it to modify and implement new features in the clean unit-tested OCP micro-service multi-az code?
