---
layout: post
title: "Thoughts on Ignorable (or Skippable) Assertions"
date: 2023-01-23 12:00:00 +0100
category: "Opinion"
tags: anti-pattern opinion
excerpt_separator: <!--more-->
---

> For those that are unfamiliar, an ignorable or skippable assert is like a normal assert - which halts the program and displays an error dialog to the user if a checks fails - with the added option of being able to ignore or skip the assert instead, and continue execution of the program.

These are my thoughts about so called ignorable or skippable assertions in code, most often found in game development. In short I believe that they are a terrible idea, where there is minimal (if any) short term benefit compared to many problems that will develop and only become apparent in the long term.

<!--more-->

# Why are they terrible?

The main reason why ignorable asserts are a terrible idea is that they lead to and are a symptom of *sloppy development practices*. In time these lead to a *culture of complacency* within the team, and bring about unintended side effects which negatively impact the quality of the software. Constant vigilance can be used to mitigate these effects but there are much better solutions available which will be more beneficial in the long term.

The most often used justification for introducing ignorable asserts is so that errors in the data or code that have been submitted do not block people from running the software. The intention is to allow people to continue working even though the software is in a broken state. Of course the key assumption here is that the software is safe to use in this state and the resulting data will also be correct.

This also comes with a couple of implications, firstly that having broken software is something normal and expected[^1], and secondly that adding features is prioritised over fixing existing problems. Over time this will cause the bugs present to accumulate and the software will degrade to a situation where dozens of assert dialogs need to be skipped in order to start it.

This will train people on the team to just click ignore on any errors that get presented, making it harder to see new problems when they come up. In time I can see feature requests coming in to show dialogs only once for each assert, and then later taking this further to never show asserts, making them just as useful as a log message.

# Programming Perspective

At the code level this breaks one of the fundamental features of assertions, that the code following the assertion can rely on the condition in the assertion being true. This means that all code has to be written with excessive error handling to deal with the situation where the user chose to ignore the error and continue. Failure to do so will result in the assertion dialog popping up and then the application crashing anyway.

For example you'll see code like this where you have both assertions and error checking:
```
ASSERT(pointer != nullptr);
if (pointer != nullptr) { /* do something */ }
```

This kind of code not only makes it harder to reason about correctness but it also ends up being slower because of all the duplicated checks and alternate code paths that much be implemented.

Another fundamental feature of asserts is that they should be used to check against programmer errors when code is being written. If asserts can be ignored then it may be tempting to use them for more general error handling as well. As the saying goes: *"if all you have is a hammer then every problem looks like a nail"*. So programmers will put asserts in to report errors for non-fatal circumstances like in loading or processing data, scripts, and configuration.

# What to do instead!

In most cases when people want to make asserts ignorable they are already sick and tired of the software crashing, be it due to bad code, bad data, or most commonly both. While both bad code and bad data will be present during software development *(and especially in game development)* there are a number of steps that can be taken instead that will lead to a much more positive outcome long term. It can be described as a *defence in depth* against errors.

As a rule bad data shouldn't cause the software to trigger an assert or crash, as it should be an expected event, and therefore handled gracefully[^2]. The easiest way to make this happen is to build an alternative error reporting mechanism for data. The checks should be as easy to insert into the code as asserts, and should report errors to an external bug tracker so they can be handled appropriately by the right people. 

For code there should be a whole variety of tests implemented to verify it works as expected. Unit tests for the low level systems and where else appropriate, but also functional and integration tests in other places to test higher level features of the software. Techniques such as fuzzing should also be used to catch weird edge cases that the hand written tests might not be able to catch.

In all cases the files submitted should go through a Continuous Integration system with both pre and post commit checks to try and guard against broken code and data getting in. The system should also be used to run longer randomised tests[^3] to catch errors that aren't immediately apparent and weirder edge cases. 

* The pre-commit tests should be quick checks to make sure that a minimum standard of quality is adhered to. Any failures at this stage should prevent the files from being committed to version control.
  * For code this means that the code compiles, the unit tests pass, and simple smoke tests verify that the software can be started without errors.
  * For data this means that simple checks and metrics pass criteria and the data can be loaded without errors into the program.
* The post-commit tests should be longer and exercise both the code and data in unison. Any failures at this stage should trigger an automatic back out of the broken files from version control.

While implementing all of these systems won't make a buggy program good overnight, or fix a sloppy development culture, it will make it easier to make positive incremental changes and increase the software quality over time. It will also remove the desire to make asserts skippable because they won't be a common enough occurrence for people to be frustrated by them. In my opinion this is a much better long term outcome than just unblocking people by making asserts ignorable.

<!-- Footnotes -->
---

[^1]: While it is generally the case that software in development is buggy and crash prone, there are many steps that can be taken to ensure at least a minimum level of quality and stability.

[^2]: Not handling bad data well is also a security issue as it can lead to exploits in the software.

[^3]: These should include (but not be limited to): random monkey tests, integration tests, performance tests, fuzzing tests, etc.