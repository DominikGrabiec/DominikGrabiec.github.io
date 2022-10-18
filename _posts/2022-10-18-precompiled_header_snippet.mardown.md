---
layout: post
title: "Custom Precompiled Headers in CMake for Visual Studio"
date: 2022-10-18 12:00:00 +0100
category: "CMake"
tags: cmake visualstudio c++
excerpt_separator: <!--more-->
---

This is a short snippet on how to implement custom precompiled headers in CMake for Visual Studio. I know that CMake has its own way of specifying includes for automatically generating precompiled header files but this option allows for more control and flexibility allowing you to add in defines and forward declarations, and doesn't require rebuilding CMake for every precompiled header change. 

<!--more-->

At its core this is the same configuration you would do manually for setting up precompiled headers within Visual Studio. The `/Yu` setting is applied for the entire target project, and then the `/Yc` setting is added to the source file which will trigger generation of the precompiled header file.

```
function(target_precompiled_header _target _header _source)
	if(MSVC)
		set_target_properties(${_target} PROPERTIES COMPILE_FLAGS "/Yu${_header}")
		set_source_files_properties(${_source} PROPERTIES COMPILE_FLAGS "/Yc${_header}")
	endif()
endfunction()
```

The parameters to the function are:
* `_target` being the CMake target these precompiled headers are for
* `_header` refers to the precompiled header file you need to include from other source files
* `_source` refers to the source file which will build the precompiled header

The easiest way to make this work is to put it into a file called `Precompiled.cmake` and include it from your top level `CMakeLists.txt` file before recursing into subdirectories. Then for each target call the function like so:

```
add_executable(foo WIN32 src/main.cpp inc/precompiled.h src/precompiled.cpp)

target_precompiled_header(foo precompiled.h src/precompiled.cpp)
```

To note is that the path to the source file needs to match the path used to refer the file in other sources declarations.

Using this function will allow you to customise the precompiled header however you like, adding in `#define`s, forward declarations, and changing the includes, all without needing to regenerate the project.
