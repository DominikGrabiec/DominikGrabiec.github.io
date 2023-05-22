---
layout: post
title: "Enhancements to the Blob Classes"
date: 2023-05-22 12:00:00 +0100
category: "C++"
tags: c++ code
excerpt_separator: <!--more-->
---

This is a short post to describe an update that I made to the Blob classes from the [previous article]({% post_url 2023-04-28-ever_useful_blob_class %}).

<!--more-->

# Slice!

The first addition is a simple `Slice` function to the `BlobView` and `BlobSpan` classes. This is an alternative way to get a sub-view or sub-span of the data using two indexes (an inclusive begin index and an exclusive end index) rather than an offset and a count. It is most useful when you're already accessing the data with iterator like indexes.

# Typed Array

The next change is adding an `ArrayView` function to `BlobView`, a corresponding `ArraySpan` function to `BlobSpan`, and both functions to the `Blob` class. These return a `std::span` class which represents a typed array over the data. This is a more convenient way to access a contiguous array of items than using pointers and pointer arithmetic explicitly. It also provides iterators to access the data and pass it into other algorithms.

Originally when I implemented these functions there was a dedicated `ArrayView` class in the codebase, so these functions were intended to return that. However since C++20 the `std::span` class is available and provides nearly identical functionality, even if it overlaps somewhat with the Blob classes themselves.

# Other Changes

The final changes include adding simple unit tests for the new functions and also adding a very simple [natvis](https://learn.microsoft.com/en-us/visualstudio/debugger/create-custom-views-of-native-objects) file.

I know that the unit tests for these classes aren't exactly the best but they've helped me verify the code works as intended.

# Source Code

As always these changes are live and available on [Github](https://github.com/DominikGrabiec/Blob).
