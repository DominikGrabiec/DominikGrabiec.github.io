---
layout: post
title: "Ever Useful Blob Classes"
date: 2023-04-28 12:00:00 +0100
category: "C++"
tags: c++ code
excerpt_separator: <!--more-->
---

In this article I describe (and make available) a set of simple but useful classes for use in file and other IO. I don't claim that these are completely original or unique, but I have found them useful in various circumstances and therefore want to share them with others.

<!--more-->

The core feature of these classes is to be able to take some memory and easily interact with it using types but in a clean C++ kind of way, without needing to do complicated pointer arithmetic or having to write explicit casts. They were also intended to be helpful in nature rather than explicitly preventing you from doing anything unsafe, though they still contain simple safeguards to catch misuse and memory bugs.

The main class of the set is the `Blob` class, which can be thought of as a unique buffer with some helper functions included. To go along with that there are two companion classes, a `BlobView` which is a read-only view over a piece of memory, and a `BlobSpan` which is a writeable span over a piece of memory.

These end up being very useful when writing binary file parsers since you can load the whole file _(or part thereof)_ into a buffer and then use the classes to parse the file format. This also allows for portions of the loaded file to be passed to other functions for processing.

These classes were originally designed and implemented many years ago, after C++11 introduced the `string_view` class but before C++20 introduced its own generic `span` class. They have been iterated on over the years as new useful features have been added to the C++ standards, but they largely still adhere to the original simple design with minimal templates used only where necessary.

One more design decision of note is that these classes *do not use exceptions*, but rather [assert]({% post_url 2023-02-28-making_a_flexible_assert %}) if there's been some sort of error. This was because originally they were developed as part of a home game engine where exceptions are not generally used. They can be adapted to use exceptions but it will require changing `ASSERT` code to `throw` and removing `noexcept` from the affected functions.

# BlobView Class

This is the simplest class to describe as it is a read-only view over a piece of memory. It stores a `const void*` pointer and a `size_t` size, and because it doesn't own the data it is pointing to it can be freely and cheaply copied, moved, created, and destroyed.

In addition to a set of simple constructors it has a number of convenience constructors that take existing container classes and construct a view over the top of them. 
```c++
template <typename T, typename A>
[[nodiscard]] constexpr BlobView(const std::vector<T, A>& vector) noexcept;

template <typename T, size_t N>
[[nodiscard]] constexpr BlobView(const std::array<T, N>& array) noexcept;

template <typename T, size_t N>
[[nodiscard]] constexpr BlobView(const T (&array)[N]) noexcept;
```
These constructors allow creating a view over containers of arbitrary types, though in general use it is intended that the type `T` be a trivial or primitive type, most likely some sort of byte like `uint8_t` or `unsigned char`.

In addition to these there is an implicit constructor taking a `BlobSpan` object, which will be used to convert from the writeable view of the memory to a read-only one in this class.

Like any other view class it has simple ways to access the underlying data.
```c++
[[nodiscard]] constexpr bool Empty() const noexcept;
[[nodiscard]] constexpr size_t Size() const noexcept;
[[nodiscard]] constexpr const void* Data() const noexcept;

[[nodiscard]] inline const void* Data(const size_t offset) const;
```
In this case the first `Data()` function just returns the pointer stored in the object, much like the `Size()` function returns the size member, where as the second `Data(offset)` function returns the pointer offset by the specified number of bytes.

The real magic though is in the other data access functions, where they return the underlying memory at a specified byte offset, cast to a specific type.
```c++
template <typename T>
[[nodiscard]] inline const T* Pointer(const size_t offset = 0) const;

template <typename T>
[[nodiscard]] inline const T& As(const size_t offset = 0) const;
```

The `As<>()` function returns a reference to the data as a specific type making it useful to access the contents of the memory. Either as single values like a `uint32_t` to quickly get the size or count of something, or as a plain old data struct to interpret things like a file header.
```c++
uint32_t count = view.As<uint32_t>();
// or
const auto& file_header = view.As<FileHeader>();
```

Likewise the `Pointer<>()` function returns a typed pointer, making it useful to access arrays of things within the memory. It can also be used to expand the interface and add support for functions returning other view like classes like a typed array view.

The final lot of functions included are ones which return a sub-view of the current view. These are most useful when you want to work with a smaller piece of memory, such as when handing it off to a more specific parsing function.
```c++
[[nodiscard]] inline BlobView SubView(size_t offset = 0) const;
[[nodiscard]] inline BlobView SubView(size_t offset, size_t bytes) const;
```

At various points in time and in different implementations of this class I have also included a `BlobView Slice(size_t begin, size_t end)` function. This extracts out a sub-view by explicitly specifying the beginning (inclusive) and ending (exclusive) of the memory. While this function is not strictly necessary it can be a nice convenience if the things you are parsing specify memory ranges in terms of begin & end rather than start & size.

# BlobSpan Class

This class is like the `BlobView` class above except it returns non-const values of the data so that the memory can be changed. 

One difference between the `BlobSpan` and the `BlobView` class is that construction of a span from a view is prohibited by marking the constructor as deleted. This prevents accidental creation of writeable spans from read-only views of the memory.

In previous versions I had implemented both the const and non-const member functions to access the data but I found it to be unnecessary as changing the underlying data does not need to change the `BlobSpan` object at all. Therefore it was sufficient to have the const member functions return non-const references to the data.

# Blob Class

This class can be best thought of as a unique buffer with functions from both the `BlobView` and `BlobSpan` classes combined. It is designed to own the buffer and provide access functions to interact with the memory, which also include functions to get spans and views of the data.

One big difference between this and the `BlobSpan` class is that the non-const member functions return non-const pointers and references to the data, where as the const member classes only return const pointers and references. This was done to avoid having to use different names for the const and non-const ways to access the data, and it allows for a `const Blob` to act like a view of the memory.

Another difference is that the `Blob` class owns the memory rather than being just a span or a view onto it. It can allocate the memory itself, be given memory to manage from an external source, or move ownership of memory from another `Blob`. It is also a move-only class because copying potentially huge buffers should not be done in a copy constructor that could be called accidentally.
```c++
[[nodiscard]] explicit Blob(size_t size_in_bytes); // Allocates memory
[[nodiscard]] inline Blob(void* data, size_t size); // Takes ownership of memory
template <typename T>
[[nodiscard]] Blob(std::unique_ptr<T[]>&& buffer, size_t elements);
```

To help with this there are several functions to explicitly deal with the buffer. These are `Reset`, `Release`, `Copy`, and `Clear`.
* `Reset` is simple as it just frees the memory and clears the internal state.
* `Release` is a function which releases ownership of the memory from the blob, returning a pair of pointer and size to the caller. This is potentially dangerous but as in `std::unique_ptr` it can serve a purpose.
* `Clear` is just a helper function for zeroing out the memory before writing data to it.
* `Copy` is an explicit function to create a copy of the memory and return it in a new `Blob` object. It is implemented this way so it will be clear in code when an explicit copy of the memory is desired.

In cases where there's an actual unique buffer class in the codebase then the blob class can be implemented as a wrapper around it. This can actually be safer in practice as ownership of the memory will remain within the existing developed system.

# Limitations

Now because these classes do not prevent you from doing certain things there is one important limitation that you should be aware of, which is not to cast to complex classes or specifically aligned types. This applies to the `As<T>()`, `Pointer<T>()`, and any other functions where there's a template type parameter involved.

The reason not to cast to aligned types is that there's no guarantee that a particular `BlobView`, `BlobSpan`, or an offset thereof, will be aligned to the required amount. Instead the cast should be to an underlying type which can then be loaded into the aligned class. 

The simple example of this would be a Matrix class like `class alignas(16) Matrix`, which is aligned to 16 bytes as part of performance optimisations for SIMD execution. Instead of casting directly into the Matrix class using `As<Matrix>`, get a  pointer to the data as floats using `Pointer<float>` and use that to fill the Matrix object.

This situation can be detected and fixed by forbidding casting to aligned types, which can be checked at compile time in the code, but I have not implemented that yet.

The reason not to cast to complex classes (ones with a non-trivial constructor) is that they would maintain internal state and have various pre and post conditions which the memory would probably not adhere to. Though if you need to create such a class the solution is similar to the aligned case, cast the memory to compatible types/structs and then use them to create the complex class.

# Source Code

The source code for these libraries is available on [Github](https://github.com/DominikGrabiec/Blob) for use in most sorts of projects.
