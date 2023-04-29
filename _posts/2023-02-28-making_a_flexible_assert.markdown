---
layout: post
title: "Making a flexible assert in C++"
date: 2023-02-28 12:00:00 +0100
category: "C++"
tags: c++ preprocessor
excerpt_separator: <!--more-->
---

In this article I tell the tale of development of a flexible assert macro for my base/common/core C++ library which I use in a number of projects.

<!--more-->

# Requirements

My initial goal was to have a single assert macro which could take an expression to test and an optional formatted message. I wanted it to be as simple to use as possible and therefore use only have a single macro name (`ASSERT`), and handle any number of parameters in the formatted message.

I also wanted to use the [{fmt}](https://fmt.dev) library to do the formatting, as historically one would use either `sprintf` or `iostreams` to craft the formatted message, but they each have serious deficiencies. With the `sprintf` method it is easy to get inconsistencies between the formatting string and the types being formatted, which could lead to corrupt/useless information when the assert actually triggers, or even a crash. On the other hand `iostreams` are just notoriously slow, more verbose than necessary, and require putting in `<<` operators to format parameters.

The other benefit that {fmt} has is that format string and parameters can be checked at compile time, giving you feedback about errors in your message rather than finding that out at runtime when the assert actually got triggered.

In short something like this:
```c++
ASSERT(pointer != nullptr);
ASSERT(!str.empty(), "Must provide message in string");
ASSERT(person.height >= min_height, "Person '{}' must be at least {} tall to ride, but are only {} tall.", person.name, min_height, person.height);
```

# Initial Design

The tricky part is to be able to handle three different cases within the same macro:
1. Assert with just the expression to test, no message, and no message parameters.
2. Assert with the expression to test and a simple message, no message parameters.
3. Assert with the expression to test, the formatting message, and messaage parameters.

Using normal functions you can rely on function overloading to get most of the way there, but this is not possible with the preprocessor as it's just a token replacement engine. So this meant that the macro would only have a single required parameter, the expression to check, and the rest would need to be handled with variadic arguments.

Another requirement was to support fmt's compile time error detection and message compilation, which required passing through the message and message parameters to the `FMT_COMPILE` macro. *(Though with the latest versions recommending `FMT_STRING` instead I might switch it over in future)*

<!-- TODO: Check if I should be using FMT_COMPILE or FMT_STRING these days -->

This led me to implementing the following initial code: *(Note: lots of code omitted)*
```c++
#include <fmt/format.h>
#include <fmt/compile.h>
#include <source_location>

namespace error
{
void handle_assert(std::source_location location, const char* expression);
void handle_assert_impl(std::source_location location, const char* expression, fmt::string_view message, fmt::format_args args);

template <typename S>
void handle_assert(std::source_location location, const char* expression, const S& message)
{
	handle_assert_impl(location, expression, fmt::string_view(message), fmt::format_args());
}

template <typename S, typename... Args>
void handle_assert(std::source_location location, const char* expression, const S& message, Args&&... args)
{
	auto format_string = fmt::string_view(message);
	handle_assert_impl(location, expression, format_string, fmt::make_format_args(args...));
}
} // namespace error

// <snip>

#define ASSERT(expression, ...) \
	do \
	{ \
		if (!(expression)) \
		{ \
			error::handle_assert(std::source_location::current(), #expression ASSERT_MESSAGE(__VA_ARGS__)); \
			DebugBreak(); \
		} \
	} \
	while (false)
```

Most of it is fairly standard though I am using `std::source_location` instead of the traditional `__FILE__` and `__LINE__`, primarily because it is new and wanted to see how well it worked, but also because its never certain what function signature macros are available on what platform.

The key implementation detail is the `ASSERT_MESSAGE` macro and how that gets used to deal with a similar set of three three cases, where it can be called with:
* no arguments
* one argument - the message string
* multiple arguments - the message string and message parameters

# Implementation

Looking through the [C++ documentation](https://en.cppreference.com/w/cpp/preprocessor/replace) for `__VA_ARGS__` I found that C++20 added a new `__VA_OPT__` macro which can be used to greatly simplify writing variadic macros such as this. Its key function is to include the text within its parenthesis if any variadic macro arguments are present, and to exclude the text if they are not.

With this I managed to implement the `ASSERT_MESSAGE` macro like so:
```c++
#define ASSERT_FORMAT_STRING(format, ...) FMT_COMPILE(format)

#define ASSERT_FORMAT_CHECKED_ARGS(format, ...) __VA_OPT__(, __VA_ARGS__)

#define ASSERT_MESSAGE(...) __VA_OPT__(, ASSERT_FORMAT_STRING(__VA_ARGS__) ASSERT_FORMAT_CHECKED_ARGS(__VA_ARGS__))
```

* The `__VA_OPT__` in the `ASSERT_MESSAGE` macro handles the distinction between 0 or N arguments, so if any arguments are present it inserts a comma and sends the arguments to the `ASSERT_FORMAT_STRING` and `ASSERT_FORMAT_CHECKED_ARGS` macros.

* The `ASSERT_FORMAT_STRING` macro takes the first argument and forwards it to `FMT_COMPILE` for compile time processing. 

* The `ASSERT_FORMAT_CHECKED_ARGS` macro handles the distinction between 1 or N arguments. It always ignores the first argument passed in (the message), but if there are any more arguments then it uses `__VA_OPT__` to add a comma followed by the arguments.

This code works for recent versions of both Clang and GCC, however it does not work by default for MSVC, hence leading me on a journey of discovery *and also why this article doesn't end here*.

# Microsoft Implementation Issues

This is because MSVC's preprocessor has historically worked in a [different way to other compilers](https://learn.microsoft.com/en-us/cpp/preprocessor/preprocessor-experimental-overview?view=msvc-170). In Visual Studio 2022 (and in 2019 starting from version 16.5) there is a [switch](https://learn.microsoft.com/en-us/cpp/build/reference/zc-preprocessor) (`/Zc:preprocessor`) to enable a new conformant preprocessor which can handle this code. However I'm not confident that all libraries which I would want to use have been updated with this preprocessor in mind, therefore I decided to take the hard road and implement asserts to work with the legacy or traditional preprocessor.

To help with the implementation there's a [predefined macro](https://learn.microsoft.com/en-us/cpp/preprocessor/preprocessor-experimental-overview?view=msvc-170#new-predefined-macro) `_MSVC_TRADITIONAL` which is `1` or undefined when the traditional, legacy, and non-conformant preprocessor is being used, and 0 if the new conformant preprocessor is being used. So the way to use it is:
```c++
#if !defined(_MSVC_TRADITIONAL) || _MSVC_TRADITIONAL
// Code using the traditional/legacy/non-conformant preprocessor ... (1)
#else
// Code using the new conformant preprocessor ... (2)
#endif
```

With that I wrapped up the previous `ASSERT_MESSAGE` macro code into the conformat preprocessor section *(2)*, and could then implement the legacy or traditional preprocessor code into the first section *(1)*.

In the end this took a lot of research and experimentation but I managed to get it working. What really helped was being able to get near instant feedback by using [Compiler Explorer](https://godbolt.org), allowing for much faster iteration time. 

<!-- TODO: Create a template for experimenting with assert macros and provide a link for it here -->

# Traditional Microsoft Implementation

The challenges were similar to what I faced in the initial implementation:
* to figure out if there was an optional message and wrap it with `FMT_COMPILE`
* to detect if there were any additional arguments and pass them through verbatim
* to correctly place commas between the elements

With that the most helpful resources I found were these:
* [Detect Empty Macro Arguments](https://gustedt.wordpress.com/2010/06/08/detect-empty-macro-arguments/) for the way to detect the presence or absence of arguments.
* [Default Arguments for C99](https://gustedt.wordpress.com/2010/06/03/default-arguments-for-c99/) for the way to detect 0, 1, or 2 arguments.
* [Variadic Macro Without Arguments](https://stackoverflow.com/questions/32047685/variadic-macro-without-arguments), specifically the answer by Richard Smith.
* [Overloading Macro on Number of Arguments](https://stackoverflow.com/questions/11761703/overloading-macro-on-number-of-arguments) for a way to combine the techniques to get something close to what I wanted.

One thing that I didn't want to have to implement was a generic way to count the number of arguments, as this requires hard coding an upper limit and manually implementing long and tedious macros. Luckily I didn't actually need to do this as all I needed to detect was to if there were 0, 1 or more than 1 arguments, and switch logic based on that.

This is what I ended up with *(to be put inside the (1) section above)*:
```c++
#define ASSERT_FORMAT_NOTHING
#define ASSERT_FORMAT_EXPAND(...) __VA_ARGS__
#define ASSERT_FORMAT_HEAD(head, ...) head
#define ASSERT_FORMAT_TAIL(head, ...) __VA_ARGS__

#define ASSERT_FORMAT_COMMA_IF_PARENS(...) ,
#define ASSERT_FORMAT_LPAREN (

#define ASSERT_FORMAT_STRING_EMPTY()
#define ASSERT_FORMAT_STRING(x) , FMT_COMPILE(x)

#define ASSERT_FORMAT_CHOICE(a1, a2, selected, ...) selected
#define ASSERT_FORMAT_CHOOSE_IMPL(...) \
	ASSERT_FORMAT_EXPAND(ASSERT_FORMAT_CHOICE ASSERT_FORMAT_LPAREN \
	ASSERT_FORMAT_COMMA_IF_PARENS __VA_ARGS__ (), \
	ASSERT_FORMAT_STRING_EMPTY, ASSERT_FORMAT_STRING, ))

#define ASSERT_FORMAT_CHOOSE(format) ASSERT_FORMAT_CHOOSE_IMPL(format)(format)
#define ASSERT_MESSAGE(...) \
	ASSERT_FORMAT_CHOOSE(ASSERT_FORMAT_EXPAND(ASSERT_FORMAT_HEAD(__VA_ARGS__ ASSERT_FORMAT_NOTHING))), \
	ASSERT_FORMAT_EXPAND(ASSERT_FORMAT_TAIL(__VA_ARGS__ ASSERT_FORMAT_NOTHING))
```

A few quick technical notes about the traditional MSVC preprocessor:
* Trailing commas within variadic arguments lists are automatically removed if the `__VA_ARGS__` [evaluates to empty](https://learn.microsoft.com/en-us/cpp/preprocessor/variadic-macros?view=msvc-170).
* `ASSERT_FORMAT_EXPAND` is used because `__VA_ARGS__` gets treated as a single token so it needs to be [expanded](https://stackoverflow.com/questions/5134523/msvc-doesnt-expand-va-args-correctly) to work correctly.

The `ASSERT_FORMAT_HEAD` and `ASSERT_FORMAT_TAIL` macros are used to select either the first parameter (`HEAD`) or the rest of the parameters (`TAIL`), much like similar functions in functional programming and to the macros in the conformant implementation.

In this case they're called with `__VA_ARGS__ ASSERT_FORMAT_NOTHING` to work around the fact that the preprocessor expects at least one argument to be passed in. The `ASSERT_FORMAT_NOTHING` macro is a placeholder which acts like a token to the preprocessor macros so that something is always passed in even if `__VA_ARGS__` is empty, but at the end evaluates to nothing so no C++ code is generated. 

Therefore the `ASSERT_MESSAGE` macro ends up with two parts, which I've separated onto two lines and are located at the end of the code above.
* The first part uses `ASSERT_FORMAT_HEAD` to extract the first argument (or nothing) and passes it to `ASSERT_FORMAT_CHOOSE` to compile the assert message. This was the trickiest part and hardest to figure out and understand and I will explain it below.
* The second part is `ASSERT_FORMAT_EXPAND(ASSERT_FORMAT_TAIL(__VA_ARGS__ ASSERT_FORMAT_NOTHING))` which takes all except the first argument and passes it verbatim to the output. This is relatively simple as it doesn't need to do any processing of the arguments.

The `ASSERT_FORMAT_CHOOSE` macro evaluates to calling `ASSERT_FORMAT_STRING_EMPTY()` if there are no parameters, or `ASSERT_FORMAT_STRING(format)` is there is a parameter. It does this by evaluating `ASSERT_FORMAT_CHOOSE_IMPL(format)` first to select which macro to call, and then calling the result with `(format)`. Hence why you see `(format)` twice in the macro body.

The `ASSERT_FORMAT_CHOOSE_IMPL` macro works by preparing the right number of parameters so that when the `ASSERT_FORMAT_CHOICE` macro is evaluated it ends up with the desired identifier in the `selected` parameter. It accomplishes this by using a few clever preprocessor tricks. 

Firstly `ASSERT_FORMAT_LPAREN` is used to delay evaluation of the `ASSERT_FORMAT_CHOICE` macro until other macros inside have been evaluated. Note that there is an extra closing parenthesis at the end of the macro to account for this.

Then the `ASSERT_FORMAT_COMMA_IF_PARENS()` macro is used to fill the parameter list with either 1 or 2 parameters.

* If `__VA_ARGS__` is empty then the `ASSERT_FORMAT_COMMA_IF_PARENS` identifier will be next to the empty parenthesis and result in the macro being evaluated and replaed with a comma, resulting in two empty parameters.

* Otherwise `__VA_ARGS__` will be a single parameter which will be inserted between the `ASSERT_FORMAT_COMMA_IF_PARENS` identifier and the parenthesis, thereby preventing it from being considered a macro and getting evaluated, resulting in only a single parameter.

Lastly there are the two identifiers which we are selecting between followed by a trailing comma.

This will then create a parameter list for `ASSERT_FORMAT_CHOICE` which will have either `ASSERT_FORMAT_STRING_EMPTY` or `ASSERT_FORMAT_STRING` in the third parameter, thereby selecting it as the identifier that `ASSERT_FORMAT_CHOOSE_IMPL(format)` evaluates to. Those identifiers then get evaluated with the extra `(format)` and result in either nothing or `, FMT_COMPILE(message)`.

# Intellisense

Unfortunately we are not quite finished yet, as the Visual Studio Intellisense system is broken by these macros and ends up putting those red squiggles underneath the asserts. This can easily be fixed by adding an `#if __INTELLISENSE__` block where we redefine the `ASSERT_FORMAT_STRING` and `ASSERT_MESSAGE` macros in a form that it can understand. 

```c++
#if __INTELLISENSE__

// Note: Simplifying the format string here for Intellisense so it makes it easier to read and understand in the tooltip
#undef ASSERT_FORMAT_STRING()
#define ASSERT_FORMAT_STRING(x) , x

// Note: Adding an extra "" parameter here so Intellisense doesn't complain about "expected an expression" when it expands the macro incorrectly
#undef ASSERT_MESSAGE()
#define ASSERT_MESSAGE(...) \
	ASSERT_FORMAT_CHOOSE(ASSERT_FORMAT_EXPAND(ASSERT_FORMAT_HEAD(__VA_ARGS__ ASSERT_FORMAT_NOTHING))), \
	ASSERT_FORMAT_EXPAND(ASSERT_FORMAT_TAIL(__VA_ARGS__, ""))

#endif // __INTELLISENSE__
```

The good news though is that this specific problem has been fixed in [Visual Studio version 17.4](https://developercommunity.visualstudio.com/t/C-VS2019-Intellisense-errors-for-n/1491519), so these measures might not be required any more. I have not updated my code yet but when I do I will update this article with the latest information.

Also I have not tested this code with tools like Visual Assist or ReSharper, though if they exhibit similar behaviour then the same fix can be applied to them.

# Source Code

To finish off this article I have created a simple project on [Github](https://github.com/DominikGrabiec/Assert) that hosts this assert for everyone to use and learn from.

---

# Update 2023-03-21

There was a bug in the `handle_assert` code above which would print the format string twice instead of the values in the assert message. I suspect this was caused by an upgrade from version 8 to version 9 of the fmt library but I'm not certain. The article now contains the updated code to work with the latest version of the fmt library.
