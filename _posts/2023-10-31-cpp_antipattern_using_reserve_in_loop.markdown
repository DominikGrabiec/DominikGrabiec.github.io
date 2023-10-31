---
layout: post
title: "C++ Anti-Patterns: Calling reserve in a loop"
date: 2023-10-31 12:00:00 +0100
category: "C++"
tags: c++ anti-pattern opinion
excerpt_separator: <!--more-->
---

This C++ anti-pattern is found in code where the programmer wanted to do the right thing for performance reasons, but for one reason or another it didn't end up that way. That is when there's a call to `reserve` inside of a loop. This may be the result of a refactor where a loop was added surrounding existing code, or just written with the best intentions. 

<!--more-->

This often ends up looking like:
```c++
Container container;
for (const auto& entry : entries)
{
	container.reserve(container.size() + entry.ItemCount());
	entry.AppendItems(container);
}
```

The main problem with the code is that it becomes a pessimisation, and in general use it will cause an allocation each time through the loop, defeating the whole purpose of reserving memory ahead of time.

The simplest solution would be to just allocate a huge amount of memory for the container so that it should never reallocate. While in certain circumstances - such as in a scratch or very simple program - this is a valid solution, in general use it is not, as it makes it easy to run out of memory especially if the code gets run on multiple threads.

```c++
Container container;
container.reserve(1000000000); // Reserve a billion units
for (const auto& entry : entries)
{
	entry.AppendItems(container);
}
```

The best solution is to pre-allocate the container to the exact size required before the loop so it never has to grow but without over allocating. With this there are two general strategies to figure out the number of total elements:

* Loop through the data twice, the first time accumulating the total number of elements, then reserving, and then looping through again to perform the required actions. This is most commonly used with strings or small data elements that are quick to iterate through.

```c++
size_t total_items = 0;
for (const auto& entry : entries)
{
	total_items += entry.ItemCount();
}
Container container;
container.reserve(total_items);
for (const auto& entry : entries)
{
	entry.AppendItems(container);
}
```

* Instrument the code to find the maximum number of elements processed, or figure out how many elements will handle most cases and let the rare outliers reallocate.

> For example if in most cases the loop has 100 or fewer items to process but on rare occasions it has over 10,000. Then it might make sense to always reserve 100 elements, and then on those rare occasions where you have more than 10,000 elements, to just let the container reallocate as needed.

The next best solution would be to just let the container grown organically, performing reallocations within the loop as required but not forcing them to reallocate. In most cases this should result in fewer allocations than in the pessimistic original case. 

```c++
Container container;
for (const auto& entry : entries)
{
	entry.AppendItems(container);
}
```

> Of course there will be a degenerate case which will cause a reallocation on each iteration of the loop, but this should be exceedingly rare, and in that case one of the other strategies should be employed.

