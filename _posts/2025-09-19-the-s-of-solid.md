---
layout: post
title: The S of SOLID
tags: SOLID SRP cohesion coupling
---

So many people throw SOLID around as the unquestionable truth that I decided to dig a bit deeper and look into its origins. I will start with S.

[Wikipedia says](https://en.wikipedia.org/wiki/SOLID) that S stands for:

| Single-responsibility principle (SRP): "There should never be more than one reason for a class to change." In other words, every class should have only one responsibility.

From there I went to the linked [SRP article](https://web.archive.org/web/20150202200348/http://www.objectmentor.com/resources/articles/srp.pdf) written by Robert C. Martin. And this is where the rabbit hole started.

## Single Responsibility Principle is just another name for Cohesion 

Here's what the SRP article says:

| This principle was described in the work of Tom DeMarco [DeMarco79] and Meilir Page-Jones [PageJones88]. They called it **cohesion**.

So is the Single-responsibility principle just a new name for cohesion? For me (a non-native speaker) the word "cohesion" was always about putting related things together, not separating every single thing. It was part of [low coupling and high cohesion](https://wiki.c2.com/?CouplingAndCohesion) forces that pull the code in opposite directions and have a sweet spot somewhere in between. 

Maybe I misunderstood something? So I went through the source materials of the "cohesion" property mentioned in the SRP article.

### DeMarco79

Here is a book you can read online for free: [Structured Analysis and System Specification, Tom DeMarco, Yourdon Press Computing Series, 1979](https://archive.org/details/structuredanalys0000dema) The book has an index in the end and you can find all pages that mention cohesion without reading the whole book.

Cohesion as described in the book is the same idea of cohesion I have in my mind, I did not find any surprises. But there are a couple of quotes I would like to highlight. See what it says about attempts to divide things even further:

| A highly cohesive module is a collection of statements and data items that should be treated as a whole because they are so closely related. *Any attempt to divide them up would only result in increased coupling and decreased readability.*

So it's a collection of closely related things. Splitting them further into even more "single responsibilities" would increase the coupling which we do not want. And it's about modules doing one *or more* related tasks. Although you can argue that a module has a broader scope than a class, I think the limitation to a *single* task is too restrictive:

| Modules are of acceptable cohesion when they do one and only one allocated task, or they do several related tasks that are grouped together because they are strongly bound by use of the same data items. 

### PageJones88

Once again, you can read it yourself online for free: [The Practical Guide to Structured Systems Design, 2d. ed., Meilir PageJones, Yourdon Press Computing Series, 1988](https://archive.org/details/practicalguideto02edpage)

This book had a lot to say about cohesion, these are my favourite bits:

| Designers should create strong, highly cohesive modules, modules whose  elements are strongly and genuinely related to one another. *On the other hand, the elements of one module should not be strongly related to the elements of another module, because that would lead to tight coupling between the modules.*

| [...] making sure all modules have good cohesion is the best way to minimize coupling between modules

I just love how it always mentions both cohesion and coupling in the same paragraph or even sentence, as if those properties are strongly coupled :D

Among other things, this book talks about 7 levels of cohesion (best to worst): functional cohesion, sequential cohesion, communicational cohesion, procedural cohesion, temporal cohesion, logical cohesion, and
coincidental cohesion. I think [Locality of Behavior](https://htmx.org/essays/locality-of-behaviour/), which gets mentioned as an alternative approach in my bubble, falls closer to communicational cohesion: elements contribute to activities that use the same input or output data.

## Responsibility means "a reason for change"

Back to the Single-responsibility principle article, this made me scratch my head:

| In the context of the Single Responsibility Principle (SRP) we define a responsibility to be “a reason for change.”

Why do we have to redefine something at all? Why not call this the Single Reason (for change) Principle? Is "responsibility" just "a reason for change"? There might be some correlation between the two, but I can't equate both. I believe the dictionary is on my side here.

## SRP is ISP sometimes

Then there is a part where Single-responsibility principle (SRP) is applied to the interfaces a class implements not the implementation itself. So it looks like the interface segregation principle (ISP) to me:

| It separates the two responsibilities into two separate interfaces. This, at least, keeps the client applications from coupling the two responsibilities. However, notice that I have recoupled the two responsibilities into a single `ModemImplementation` class. This is not desirable, but it may be necessary.

This is not a bad thing and a nice tool to have, I just wish more SOLID advocates realized this is an option and you don't have to split implementation.

## Final thoughts

It will not surprise you that I prefer the original "cohesion" and the [low coupling, high cohesion](https://wiki.c2.com/?CouplingAndCohesion) principle for finding the right balance between both. I think the Single-responsibility principle is just a redefinition of redefinition (another name for cohesion and "responsibility" is another name for "a reason for change"). Even [Locality of Behavior](https://htmx.org/essays/locality-of-behaviour/) is closer to the spirit of coupling than the SRP is. We would do better with the original and avoid misinterpretations and would keep the coupling close to it. I know some people say that D of SOLID speaks about the coupling aspect but (spoiler alert!) I suspect D is to "coupling" what S is to "cohesion": lost in translation.
