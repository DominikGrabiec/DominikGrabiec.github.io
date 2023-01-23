---
layout: post
title: "C++ Anti-Patterns: Passing a result variable in by reference badly"
date: 2022-02-20 12:00:00 +0100
category: "C++"
tags: c++ anti-pattern opinion
excerpt_separator: <!--more-->
---
This C++ anti-pattern might be a bit controversial to some people, as it's a common design holdover from the early days of game development and from C or early C++ code. The pattern is to pass a container into a function by reference (or pointer) and use that to return the result from the function. While in general this is not strictly an anti-pattern, this particular example shown below does deserve that title, and I've seen it written frequently in code, even by experienced senior people.

```
void FooHolder::GetFooElements( vector<Foo>& out )
{
	out.reserve( m_elements.size() );
	for ( const auto& foo : m_elements )
	{
		out.push_back( foo );
	}
}
```

<!--more-->

This example snippet of C++ code has several problems:
* Because you pass in the result container it does not use or receive any benefit from Return Value Optimisation (RVO) or Named Return Value Optimisation (NRVO). Meaning that the object needs to be separately constructed outside the scope of the function and then passed in, rather than being handled by the compiler and constructed in place at the destination.
* It reserves only the number of elements that are to be added to the array but does not take into account the existing size of the array. This means that there will be at least one more allocation, and possibly many more depending on how many elements will be added and how often the container needs to resize itself.
* It is not clear from the code if this function is a simple getter function, or if it is intended to be used to append data to an existing array. Specifically we don't know if the incoming array will be empty or not.
* If passing the result array by pointer it implies that the parameter is optional, as a pointer can be null where as a reference cannot.

In general though there are two cases where the pattern of passing the result container as a parameter is applicable:
1. *To reuse the container in order to reduce the number of memory allocations.* So that memory does not need to be allocated every time this function is called. This is most useful in a loop where we are gathering things for processing from many different sources that have a different number of items. The result container will help us reduce the number of allocations, but we must remember to clear it every time before passing into this function.
2. *To append many elements to the same container from many different sources.* This will help a bit because in some cases we would be able to push the elements onto the array without needing to allocate memory. However unless we allocate the maximum memory we need up front then some allocation will still need to occur.

> Side Note: In both of these cases it is most efficient to allocate the container only once before even calling the function, therefore the reserve call should not be needed.

The example snippet of code fails on both of these counts and should be rewritten to clearly state what the function is intended to do.

If it is meant to be a getter function that returns the container of values it should be implemented as one of:

```
vector<Foo> FooHolder::GetFooElements() const
{
	return m_elements; // explicit copy
}
```

Or:

```
const vector<Foo>& FooHolder::GetFooElements() const
{
	return m_elements; // return reference, let caller copy if needed
}
```

Or if explicit processing is required for each element:

```
vector<Foo> FooHolder::GetFooElements() const
{
	vector<Foo> result;
	result.reserve( m_elements.size() );
	for ( const auto& foo : m_elements )
	{
		result.push_back( foo.element );
	}
	return result;
}
```

This will either return by const reference and let the caller determine if the values will be copied into another container, or it will make the compiler use NRVO and construct the new array at the location specified by the caller. Even if this is the same speed as the original example it is more idiomatic C++ and clear in what the function does.

Instead if the function is intended to append to an existing container, it should have its name changed to indicate it will "append" to the given parameter, and the call to reserve needs to take into account the current container size. So it should look like the following:

```
void FooHolder::AppendFooElements( vector<Foo>& out ) const
{
	out.reserve( out.size() + m_elements.size() );
	for ( const auto& foo : m_elements )
	{
		out.push_back( foo );
	}
}
```

However if none of this is to your liking there is one last way to fix the function and make it less unpredictable, and that is to add a precondition check to make sure that the container being passed in is empty. This will at least catch the cases in code where this assumption is being violated so that you don't get unexpected results or inefficient memory allocations.

```
void FooHolder::GetFooElements( vector<Foo>& out ) const
{
	ASSERT( out.empty(), "Result vector must be empty" );
	out.reserve( m_elements.size() );
	for ( const auto& foo : m_elements )
	{
		out.push_back( foo );
	}
}
```

