---
layout: post
title: "How to Layout Data in C++ Classes"
date: 2025-02-01 12:00:00 +0100
category: "C++"
tags: c++ data
excerpt_separator: <!--more-->
---

The layout of data members within a class is an important consideration in writing C++, it affects readability and understanding of the class, and can impact performance as well. There are a lot of things to consider when organising and ordering data members, and in this article I will go through my thoughts and explain the guidelines I used when writing C++ code.

<!--more-->

Take note that these are just guidelines and one size will not fit all situations, so feel free to mix and match them as required. I've ordered them roughly from most readable and least packed to least readable and most packed. 

### Initializer List Order

The most important thing to remember is that the order of initialisation of the member variables happens in the order they are declared in the class definition, not in the order they are listed in the initializer list in the constructor. There is a [C++ core guideline](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Rc-order) which says to keep the order of members in the class definition and in the initializer list the same.

The main effect of this in regards to the layout of data members is that you will want to initialise some data members before others, especially when they are not just simple assignments. For example computing the `size` from `width` and `height` parameters before allocating an array to store the data.

### Group Related Members

The first *(and probably main)* method of organisation is to group related members together so that they are logically close in the source code. An example of this is putting members related to storage of data in one group, and members related to efficiently finding the data in another group. I'd like to think that people do this naturally, grouping related members together surrounded with whitespace, but it may just be me.

One could argue that you don't need to create these groupings because in principle a class should only do one thing, so all its members should be in one group. However in reality classes can do one thing but still contain many members and syb-systems used to accomplish that, or they are responsible for several things so it makes sense to group the members for those logically to aid readability.

One down side of this method is that you're likely to get plenty of padding inside and in between groups of members, as smaller data members such as `bool`, `int`, etc are placed next to larger data structures which have bigger alignment requirements. Though depending on the class it might not matter as we will discuss below.

### Group Members By Usage

Related to the above method is to group members by usage together, so that they are physically close to each other in memory, and more likely to be on the same cache line in the processor. The main difference between the previous method and this is that you're taking into consideration how the data is used and not just what part of the system it logically belongs to.

This can reduce readability of the class definition but in the same vein it can also increase performance in some specific circumstances.

### Minimise Padding

The next major method of organisation is to arrange the members of a class in a way that minimises padding and wasted space within the class. This is especially useful when you're creating many thousands or millions of instances of these classes, as each byte of wasted space becomes significant in aggregate. This also helps to efficiently pack these classes contiguously in memory, but it can come at a cost to readability.

An important detail to remember here is that a class's size is a multiple of its alignment, and its alignment is the highest alignment of its members. This means that padding will be inserted after the last data member to make the class's size a multiple of its alignment.

The simplest way to minimise padding is to put the elements with the highest alignment requirements at the beginning of the class, followed by members with successively smaller alignment requirements, with byte sized elements like `bool` or `uint8_t` at the end. Of course in doing this there may be gaps created in between the larger members, and in this case fill the gaps by moving any appropriately sized elements in between the larger ones. If all goes well then there should not be any padding, or only a few bytes of padding at the end of the class.

> Think of this as filling a jar with various sized rocks, first put in the biggest rocks (big members), then fill the gaps in with smaller pebbles (smaller members, integers, etc), and finally pour in the sand to fill in the remaining space (bytes and booleans).

This works best with more plain\-old\-data style classes and structures that contain many smaller sized members that themselves have smaller alignment requirements and do not have any internal padding. If larger classes are used be aware that they might have their own internal padding which is created automatically due to their alignment requirements. A simple example of this is:

```c++
struct BufferView	// Size: 16 bytes, Alignment: 8 bytes
{
	void* data;	// Size: 8 bytes, Alignment: 8 bytes
	int size;	// Size: 4 bytes, Alignment: 4 bytes
			// Padding: 4 bytes
};
```

Making this a member of another class will introduce 4 bytes of padding each time it is used, even if it is followed by a 4-byte value which would otherwise fit within the padding.

#### Packing

The remedy for this situation is to use compiler-specific attributes and pragmas to specify the packing of elements within the classes. For MSVC this involves surrounding the class declaration(s) with a [pragma pack declaration](https://learn.microsoft.com/en-us/cpp/preprocessor/pack) like so:

```c++
#pragma pack(push, 1)	// Could also be 4 instead of 1
struct BufferView	// Size: 12 bytes, Alignment: 1 byte
{
	void* data;	// Size: 8 bytes, Alignment: 8 bytes
	int size;	// Size: 4 bytes, Alignment: 4 bytes
};
#pragma pack(pop)
```

Likewise on GCC and Clang you can use the [packed attribute](https://gcc.gnu.org/onlinedocs/gcc-14.2.0/gcc/Common-Type-Attributes.html#index-packed-type-attribute) to tell the compiler to pack the members tightly. Note that the packed attribute has to be applied to each class declaration separately in order to pack the elements as tightly as possible.

> Visual Studio 2022 version 17.8.0 introduced a neat little feature to show the size of a type or value in a tooltip, it helps as you can quickly and easily see what effect moving a member has on the size of the class. There are also other plugins which help visualise class members and padding, though I do not use them.

### Compressing Members

In some cases just packing the elements is not enough, as the size of the members just exceeds a multiple of the alignment, thereby creating a relatively large amount of padding at the end of the class.

These techniques can be used to combat this, helping reduce the size of the data and making it fit more nicely within a multiple of the alignment. This becomes important when these classes are stored contiguously and processed by performance critical code, as more instances can be packed within the same amount of memory.

#### Combine Booleans

The first technique is to combine `bool` values into a bitfield, as traditionally each `bool` value takes a byte. This can be done by having an integer and manually using masks, or by declaring a C++ bitfield. One trick I learned is to create a bitfield using `bool`, like `bool a : 1;`, `bool b : 1;` etc, which has the advantage of being descriptive but also combining adjacent values together.

So instead of something like this:
```c++
struct Example
{
	bool a;
	bool b;
	bool c;
};
static_assert(sizeof(Example) == 3);
```

It would instead be something like this:
```c++
struct Example
{
	bool a : 1;
	bool b : 1;
	bool c : 1;
};
static_assert(sizeof(Example) == 1);
```

#### Bitpack Values

The next technique is to bitpack smaller integer values *(and enumerations)* together in a larger field. For example when you have 3 integer values that range only from 0 to 1000 you can store them as three 10 bit values inside a single `uint32_t` instead of one for each value. The left-over bits can be used for flags.

```c++
struct Example
{
	uint32_t r : 10;
	uint32_t g : 10;
	uint32_t b : 10;
	uint32_t a : 2;
};
static_assert(sizeof(Example) == sizeof(uint32_t));
```

#### Encode Values

A more advanced version of this would be to use some other encoding scheme to store multiple values in the same integer variable. This can be something simple like encoding in the same way that multidimensional array indexes are calculated. For example storing values `a`, `b`, `c`, as: `v = (a + A * (b + B * c))`, and then decoding it using `/` and `%`. There are other encoding schemes that could be used, but the more complex the encoding is the slower it will be to interact with the values.

An example of the simple multidimensional array index calculation:
```c++
uint64_t encode(uint32_t x, uint32_t y, uint32_t z)
{
	return x + MAX_X * (y + MAX_Y * z);
}

void decode(uint64_t value, uint32_t& x, uint32_t& y, uint32_t & z)
{
	z = value % MAX_Y;
	value /= MAX_Y;
	y = value % MAX_X;
	value /= MAX_X;
	x = value;
}
```

#### Quantise Floating Point Values

If dealing with floating point values then a good technique is to quantise them and store them in smaller integer types. The simplest version is to store normalised values (ranging from `0.0` to `1.0`) in unsigned integer types like `uint8_t` or `uint16_t`, where `0.0` maps to `0` and `1.0` maps to `255` or `65535` respectively. Values ranging from `-1.0` to `1.0` can be similarly stored in signed integer types, and as long as the range is limited the values can be quantised into a smaller type while retaining reasonable accuracy. The tricky part is that the range has to be defined in code and not in data for there to be a reduction in size.

For example to quantise a value between 0 and 1 into a smaller integer:
```c++
uint8_t encode(float value)
{
	return static_cast<uint8_t>(value * 255.0f);
}

float decode(uint8_t value)
{
	return value / 255.0f;
}
```

#### Drop Computable Values

Another technique which is common in graphics programming is to drop elements of compound types when they can easily be recomputed. When dealing with normalised vectors and unit quaternions, in addition to quantising their values, one of them can be dropped entirely. One caveat to that is the sign of the dropped element must be stored somewhere or explicitly known externally, as squaring a number will make it positive.

### Why and When

A lot of this only matters if the class you're writing will have many *(millions of)* instances created of it and therefore you need to organise your data for optimum efficiency. In all other cases you should make it easy to read and easy to understand.

To summarise:

* If your class will only be instantiated a handful of times, then readability is far more important than data layout, so no need to optimise.
* If your class has a container inside of it, then you will probably be better off optimising the class stored inside the container.
* If your class contains another large class inside of it, then optimise that class first.
* If you instantiate millions of instances then pay special attention to the class and optimise it.
* If you are optimising to get a performance improvement then measure, measure, and measure!
