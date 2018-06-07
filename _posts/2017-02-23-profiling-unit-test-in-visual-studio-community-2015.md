---
layout: post
title: "Profiling Unit Test in Visual Studio Community 2015"
date: 2017-02-23T10:53:58+01:00
tags: ["Visual Studio", "Profiling", "Unit tests"]
---

When I was running some unit tests and was waiting for completion I saw one that
took more than 3 seconds to complete. There were some tests that I knew took long
but this one was not one of them. So now the question was whether this was due to
testing framework or my code was to blame.

<!--more-->

For those of you who are working with Visual Studio 2015 Premium or Ultimate
editions this is really simple: right click on problematic unit test and choose
_Profile test_.

This option is not available if you are using Visual Studio Community 2015.
However we can still access this functionality - it just requires a little more
effort. Below are required steps:

__1. Set breakpoint at the beginning of unit test and run it__

Set breakpoint inside test method in desired location. Usually this would be at
the first statement inside unit test. Run unit test in _Debug_ configuration and
wait until breakpoint is hit.

__2. Open second instance of Visual Studio Community 2015 and attach profiler to
test runner process__

![Profiler menu]({{ site.url }}/public/images/posts/profiling-unit-test-in-visual-studio-community-2015/profiler-menu.png)
You can attach profiler by going under _Debug_ -> _Profiler_ ->
_Performance Explorer_ -> _Attach/Detach..._.

![Attach process dialog]({{ site.url }}/public/images/posts/profiling-unit-test-in-visual-studio-community-2015/attach-process.png)
You will be presented with a dialog asking you to choose
process to which to attach profiler. Search for _vstest.executionengine_ or
_vstest.executionengine.x86_ and attach to this process.

__3. Continue with test execution and revise results__

Now that profiler is attached to test runner, switch to the first instance of
Visual Studio and click _Continue_. After test runner finishes running test,
profiler will process data and display report.

Above steps can be used to profile multiple unit tests as well by selecting
desired ones and selecting _Debug Selected Tests_ from context menu. Just don't
forget to set breakpoint to the first test in selection.

As for my problematic unit test I found out that the cause for slow test execution
was not in tested code but rather in combination of test code and mocking framework.
While it was more involved than just selecting option from context menu, it was
simple enough once I knew how to do it.

Thanks for reading.
