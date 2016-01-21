---
layout: post
title: DIY Tuxedo Monitoring - Collecting information
---

At one point I started to build a monitoring solution for Oracle Tuxedo so these are my notes along the way. The best way to start is reading [Monitoring Your Tuxedo Application](https://docs.oracle.com/cd/E13161_01/tuxedo/docs10gr3/ada/admon.html).

What I want from a monitoring tool is to find bottleneck services that take the longest time or are called too often. Maybe even some profiler-like output for Tuxedo services. There are 3 main ways to get information about Tuxedo system and application.

## MIB

MIB is a piece of shared memory that contains both static (UBBCONFIG) and dynamic information about the system. A common way to use it by tmadmin commands like

- *bbs* shows number of servers, services and queues
- *pt* shows information about outstanding transaction
- *psc* gives number of service executions
- *psr* gives number of requests done and load done

This is what [AppDynamics Tuxedo monitoring extension](https://github.com/Appdynamics/tuxedo-monitoring-extension) uses to collect information.

A less common way is to write a program that calls .TMIB service or uses tmadmcall() function to retrieve information. This approach has many benefits: you don't have to parse output of tmadmin and work with FML32 APIs instead. It also returns more information than tmadmin including number of tpenqueue(), tpcall(), tpcommit() and other ATMI calls.  One issue I've run into using tmadmcall() instead of calling .TMIB service is that it returns a limited number of entries and does not support cursors for retrieving the rest of entries. But tmadmcall() is great because it does not make a service call.

You can find more information about this in Tuxedo reference manuals TM_MIB(5) and tmadmcall(3c).

## tmtrace

tmtrace allows you to trace the execution of Tuxedo application by writing records about ATMI calls to user log. You can write a program that parses user log and extracts useful information. Compared to information provided by MIB you can find out how much time each service execution took, which services it called, how much time it waited on other services to return, etc.

<pre>
111401.mordor!IN.6769.3053347840.0: TRACE:at:  { tpservice({"IN", 0x0, 0x0x1bca548, 4096, 0, 0, {1453367641, 0, 5}})
111401.mordor!IN.6769.3053347840.0: TRACE:at:    { tpforward("HUB", 0x0x1bca548, 0, 0x0)
111401.mordor!IN.6769.3053347840.0: TRACE:ia:      { tpacall("HUB", 0x0x1bca548, 0, 0x1000002)
111401.mordor!IN.6769.3053347840.0: TRACE:ia:      } tpacall = 0 [CLIENTID {1453367641, 0, 5}]
111401.mordor!IN.6769.3053347840.0: TRACE:at:    } tpforward [long jump]
111401.mordor!IN.6769.3053347840.0: TRACE:at:  } tpservice
111401.mordor!HUB.6768.1951171584.0: TRACE:tr:  trace("*:ulog:dye")
111401.mordor!HUB.6768.1951171584.0: TRACE:tr:  dye
111401.mordor!HUB.6768.1951171584.0: TRACE:at:  { tpservice({"HUB", 0x0, 0x0x19d3548, 4096, 0, 0, {1453367641, 0, 5}})
111401.mordor!HUB.6768.1951171584.0: TRACE:at:    { tpacall("SVCA", 0x0x19d3548, 0, 0xc)
111401.mordor!HUB.6768.1951171584.0: TRACE:at:    } tpacall = 0 [CLIENTID {1453367641, 0, 5}]
</pre>

Fortunately since Tuxedo 9 you can use tputrace(3c) to implement your own function that is called on each trace point. So instead of parsing user log you can reduce the performance penalty by deciding what to write and skip trace points like tpalloc()/tpfree(), when to write it (aggregate) and where to write it (shared memory, network, etc.). You can also use tputrace() to store requests or responses of some services to logfiles or database.

## TSAM+

TSAM+ is the official solution for monitoring Tuxedo system and application. It collects all the data and displays it in a GUI. One interesting feature is that it collects full call chains and you can trace each call from WebLogic to Tuxedo to Oracle database and back.

TSAM+ collects information using another approach: turns out that Tuxedo can load plug-ins and has a registry to manage and configure them. It also allows you to write custom plug-ins implementing "engine/performance/monitoring" interface and store some information for your own needs and tools although I don't see much sense in this since you already have everything TSAM+ offers.

But a great power comes with a great price.

So what if you want the great power but without the price and build everything yourself? While this is not documented, it is possible to implement a plug-in for another "engine/tsam/agent" interface. You will have to find out API yourself which should not be a problem if you have some debugging skills and experience with ATMI calls. I know that it works but I'm not a lawyer so think twice before taking this path.


