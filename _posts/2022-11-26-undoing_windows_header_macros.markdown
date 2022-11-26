---
layout: post
title: "Undoing Windows Header macro damage"
date: 2022-11-26 12:00:00 +0100
category: "C++"
tags: c++ windows
excerpt_separator: <!--more-->
---

Here is a simple header file ([Github link](https://github.com/DominikGrabiec/WindowsHeader/)) which I use in my own projects instead of including `<windows.h>` directly. It sets some simple flags, includes `<windows.h>`, and then goes and undefines as many character encoding related Windows API macros as possible. This has the effect of removing all of the macros which select either the `A` or `W` set of Windows API functions, and I do this for two reasons.

<!--more-->

The first reason is that while this sort of macro interface might have worked in the past, these days we are much more knowledgeable about character encodings and have far more string processing libraries at our disposal, from new features in the [C++ standard library](https://en.cppreference.com/w/cpp/string) to excellent libraries like [fmt](https://fmt.dev/). This means you need to know what encoding your strings are and explicitly call the appropriate `A` or `W` versions of the Windows API functions, otherwise you'll get things breaking in weird ways when non-latin characters are used.

The second reason is more of an aesthetic and pedantic one, in that these macros tend to override common names like `LoadImage`, `ReadFile`, `WriteFile`, and others. So if you happen to use these as function names in your own classes and interfaces they will actually be renamed as well. Macros do not discriminate where the function is! This could have effects later on where some code was compiled without including the windows header and some with, and you'll get compiler errors at best, or linker errors at worst.

One important thing to note is that this file must be included before any other dependencies that could potentially `#include <windows.h>` themselves, as you could end up with a situation where half your code uses the macros and half doesn't.

The header file is on [Github](https://github.com/DominikGrabiec/WindowsHeader/) and I hope to add more to it soon. Some ideas that I have for expansion are to more closely align with the actual Windows headers and make the file more flexible so it can cover more situations. Right now it is just the simplest version that I use locally in my projects.