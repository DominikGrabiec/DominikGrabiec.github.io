---
layout: post
title: "Investigations into Classifying Characters"
date: 2023-12-04 12:00:00 +0100
category: "C++"
tags: c++ compilers lexers
excerpt_separator: <!--more-->
---

For an application that I'm building I am developing a compiler, and the first part of that is writing a Lexer to convert characters into tokens. A fundamental part of this is to classify characters (ASCII, UTF-8, etc) into character classes in an efficient manner so that they can be lexed appropriately. Doing some research and thinking about this provided me with three methods of classifying characters, but this made me curious and I had questions I wanted the answers to. 

<!--more-->

This was originally meant to be a single article describing my investigation but as I was writing it I started to have more questions and found more things to investigate. At this point I realised that there was too much to say about each method and so I split it up into a series of articles. This article introduces the project, the next few articles describe the the methods, and a final article discusses the results and hopefully answers the questions.

1. Introduction _(this article)_
2. [Classifying Characters with Simple Functions]({% post_url 2023-12-08-classifying_characters_with_functions %})
3. Classifying Characters with Bit Masks
4. Classifying Characters with Table Lookup
5. Analysis and Results

I also want to give a mention to the [article by Daniel Lemire](https://lemire.me/blog/2023/11/07/generating-arrays-at-compile-time-in-c-with-lambdas/) which made me want to revisit this topic and write my findings in article form, as I was thinking that this would be too simple and basic a topic.

## Methods

During my research I identified three basic methods of classification:
1. Simple functions using logical operators.
2. Bitmask lookup table for each classification.
3. Combined lookup table for all classifications.

All of these have been implemented in places in various forms, from writing the simple functions yourself, to using table based lookup in Clang, and various forms of bit mask lookups. 

## Questions

Given these options I was curious about how well they performed and had several questions I wanted answers to.
* First and most importantly, which method was the fastest and most efficient in classifying characters?
* What were the costs and trade-offs in each method?
* What kind of optimisations would different compilers apply to each method?

While researching the answers to these questions I also thought of more questions:
* What is the optimal trade-off between memory used for lookup tables versus code complexity?
* Would it be possible to write simple compile time code to generate lookup tables?
* Is it worth the extra code complexity to implement specific micro-optimisations in code?

## Testing

The first and simplest way to evaluate these was to use Compiler Explorer to see what assembly instructions were generated for each method by each compiler. While this wouldn't tell us which method is fastest it would be able to show which methods end up equivalent and which optimisations in C++ actually matter and which don't. We would also be able to see how each compiler generates code and what the differences (if any) between them are.

The next way to evaluate each method would be with a micro-benchmark. This should tell us something about the relative performance of each method, in isolation from the rest of the system. The trick here is to write the test well enough that you're testing the actual code and not just training the branch predictor of the processor.

The last way to evaluate each method would be to integrate it with a simple lexer so we can see how well each method performs within a larger system and how it gets integrated at the assembly level. With this we can also feed it a larger data set than a micro-benchmark so any differences can be more easily observed. 

### Environments

* Windows 10 Pro 22H2
* Ubuntu 22.04 LTS _(WSL2)_
* [Compiler Explorer](https://godbolt.org/)

### Compilers

* Microsoft C/C++ Compiler 19.37
* Clang 17
* GCC 13.2

