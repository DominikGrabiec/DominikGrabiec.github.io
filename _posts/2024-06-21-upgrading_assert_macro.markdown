---
layout: post
title: "Upgrading Assert Macro in C++"
date: 2024-06-21 12:00:00 +0100
category: "C++"
tags: c++ preprocessor assert
excerpt_separator: <!--more-->
---

An article detailing investigations and upgrades to the [Flexible Assert Macro]({% post_url 2023-02-28-making_a_flexible_assert %}) to fix some oversights and add some C++20 features which improve the generated code. These updates are now available on [Github](https://github.com/DominikGrabiec/Assert).

<!--more-->

# Making Conditions Unlikely

The first update was to add the `[[unlikely]]` [attribute](https://en.cppreference.com/w/cpp/language/attributes/likely) to the assert condition. This will tell the compiler to generate the assembly code under the assumption that the condition will not be true at runtime _(but not with the assumption it will never be true)_.

This actually changes the generated assembly quite a bit in places, moving the handling of the assert to the end of the function and out of the immediate code to execute. While I haven't measured any performance impact of this change, the assembly code looks tidier and because the regular function code doesn't need a jump to reach it should be more efficient.

**Before Unlikely**

```assembly
; 9    : 	ASSERT(name.length() <= 255);

	cmp	QWORD PTR [rcx+16], 255			; 000000ffH
	mov	rbx, rcx
	jbe	SHORT $LN2@Example
	lea	rax, OFFSET FLAT:??_C@_08MPNMAILL@Test?4cpp@
	mov	DWORD PTR $T1[rsp], 9
	mov	QWORD PTR $T1[rsp+8], rax
	lea	rdx, OFFSET FLAT:??_C@_0BF@CFNDKGCM@name?4length?$CI?$CJ?5?$DM?$DN?5255@
	lea	rax, OFFSET FLAT:??_C@_0HF@HDFDIDJI@int?5__cdecl?5Example?$CIconst?5class@
	mov	DWORD PTR $T1[rsp+4], 2
	lea	rcx, QWORD PTR $T1[rsp]
	mov	QWORD PTR $T1[rsp+16], rax
	call	?handle_assert@error@@YAXUsource_location@std@@PEBD@Z ; error::handle_assert
	int	3
	call	?terminate@std@@YAXXZ			; std::terminate
	int	3
$LN2@Example:

; ... Normal function code here ...
```			

**After Unlikely**

```assembly
; 9    : 	ASSERT(name.length() <= 255);

	cmp	QWORD PTR [rcx+16], 255			; 000000ffH
	mov	rbx, rcx
	ja	SHORT $LN20@Example

; ... Normal function code here ...

$LN20@Example:

; 9    : 	ASSERT(name.length() <= 255);

	lea	rax, OFFSET FLAT:??_C@_08MPNMAILL@Test?4cpp@
	mov	DWORD PTR $T1[rsp], 9
	mov	QWORD PTR $T1[rsp+8], rax
	lea	rdx, OFFSET FLAT:??_C@_0BF@CFNDKGCM@name?4length?$CI?$CJ?5?$DM?$DN?5255@
	lea	rax, OFFSET FLAT:??_C@_0HF@HDFDIDJI@int?5__cdecl?5Example?$CIconst?5class@
	mov	DWORD PTR $T1[rsp+4], 2
	lea	rcx, QWORD PTR $T1[rsp]
	mov	QWORD PTR $T1[rsp+16], rax
	call	?handle_assert@error@@YAXUsource_location@std@@PEBD@Z ; error::handle_assert
	int	3
	call	?terminate@std@@YAXXZ			; std::terminate
	int	3
```

# Checking if Debugger is Attached

The next investigation was trying various ways to integrate a check to see if a debugger was attached before triggering the debug break. The main goal behind this was that when running the program with a debugger attached it would trigger the breakpoint and allow the programmer to see the assert that was triggered, and when running outside of a debugger it would just terminate without triggering a breakpoint.

The simplest way of doing this was to wrap the `__debugbreak()` (or `DebugBreak()`) with a check like `if (IsDebuggerPresent()) { ... }`. Doing this added a function call, a test, and a jump to the assert code, which in most cases made the code significantly larger. It also required forward declaring or including `debugapi.h`, into what otherwise is a fairly low level header.

Another way of doing this was to move the `IsDebuggerPresent()` call to be inside the `handle_assert` function, and have that return a boolean indicating if the breakpoint should be triggered or not. This eliminated a function call instruction from the assert macro but it didn't clean up the assembly all that much.

Overall I wasn't happy with either of these solutions so I ended up looking for alternatives, but not ones which would require me to implement magical assembly or weird intrinsics. _(For reference most alternatives involved manually implementing the `IsDebuggerPresent()` function by looking up the debugger present flag in the thread information block in Windows. As such I didn't want the support burden to keep this up to date with newer versions of Windows.)_

It was when I was investigating how to handle other program faults that I realised that asserts (and error handling in general) need to be handled differently in developer and retail versions of the program. During development you want to use breakpoints to catch problems early, either by running in a debugger or by being able to attach one as easily as possible. However in retail mode you cannot do that so you want to create a detailed error report with plenty of supporting information, and send that to yourself as a package in order to try and figure out what went wrong.

This means that a separate retail version of the assert macro and assert handler function will need to be created, though that can be done at a later time together with a more thorough error reporting system.

# Actually Making it Fatal

The last thing to add was a call to `std::terminate()` inside the macros to actually make the asserts fatal and exit the program.

One interesting thing discovered by doing this was that in some cases adding the terminate function to the macro caused the compiler to move the implementation of the assert contents to the end of the function, in a similar way as when adding the unlikely attribute. But it did not do this in every situation, therefore using the unlikely attribute is still a good idea.
