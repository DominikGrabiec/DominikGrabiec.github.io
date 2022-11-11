---
layout: post
title: "Setting up a basic Windows C++ project with CMake for Visual Studio 2022"
date: 2022-11-11 12:00:00 +0100
category: "CMake"
tags: cmake visualstudio c++
excerpt_separator: <!--more-->
---
This is the first article in a series which describes how to configure a C++ project using [CMake](https://cmake.org/) for development in [Visual Studio](https://visualstudio.microsoft.com/) in a way that I use in my own projects. Initially I had intended to write this in a single article but it grew and grew so I am splitting it into smaller more self contained articles where I can use a few more words to explain the reasoning behind these decisions. In this article I will describe the basic layout and configuration of the project, and in later articles I will build on this core, integrating the [vcpkg](https://vcpkg.io/en/index.html) package manager, adding testing with [Google Test](https://github.com/google/googletest), and more.

<!--more-->

*While Visual Studio has gained relatively decent support for using CMake projects directly in recent revisions, I still like to generate the solution and the project files myself. I have more control over what files are generated and where they are stored - so they don't go onto my system drive but rather stay with the project.*

The layout is designed to host a single main product, containing its key libraries, supporting tools, and all manner of tests. It's based on what worked for me developing software and games for the Windows platform, so it follows my personal style and naming scheme. Of course the specific names that are used can be customised to your preferred style.

*While this project structure will work equally well with centralised version control systems like Subversion or Perforce and with distributed version control systems like Git. In this article I will be using Git as it the version control system I use locally, though it should be easy enough to convert to other version control systems.*

# Contents

* [Top Level Structure](#top-level-structure)
* [The .gitignore File](#the-gitignore-file)
* The [Source Directory](#source-directory) Layout
* [Generate Solution Batch File](#generate-solution-batch-file)
* The [Top Level CMakeLists File](#top-level-cmakelists-file)
* The [Project Structure](#project-structure)
* [Project CMakeLists File](#project-cmakelists-file)


# Top Level Structure

In its simplest form the project consists of a single top level root directory which contains only a `.gitignore` file (or equivalent) and a `Source` directory, which are the only essentials to keep in version control. As it grows the root directory can be used to store high level readme files and scripts, and more subdirectories can be added to store various resources, files to build documentation, and so on.

```
(root)\
	Source\
	.gitignore
```

There are two other directories that commonly get created in the root directory. 

The first is the `Build` directory which is used to store the Visual Studio solution, the temporary build files, and any other automatically generated files for our project. We keep these temporary files in a single separate directory so they are easier to locate, manage, and (most importantly) delete. So when we need to clean the build we can just delete this single directory and are effectively back to a clean checkout.

The second is an output directory which is where I store the compiled binaries created by my project in a structure that resembles the final installation directory. It is optional but I do this for a better Visual Studio workflow since it allows me to start debugging without needing to run an explicit install step. This directory can be named `Output` but I am using `Binary` instead so that it appears at/near the top of the directory listing. 

Within the output directory I keep a subdirectory per build configuration (`Release`, `Debug`, etc) which contains the executables. If building for multiple platforms then either an extra layer of subdirectories can be added using the platform name (e.g. `x64\Debug\`), or the platform name can be combined with the build configuration for a single layer of directories (e.g. `x64_Debug\`).

So the structure should look something like this:

```
(root)\
	Binary\
	Build\
	Source\
	.gitignore
```

# The .gitignore File

The `.gitignore` file should be very simple to start with and only contain entries for the top level directories that should never be committed to version control. So at the beginning it should be something like:

```
Binary/
Build/
```

In general we should only be ignoring either top level directories like `Build` and `Binary`, but sometimes we will need to add specific file types that get created in the `Source` directory tree by various tools. One example of this for Windows is if you have an `.rc` file storing Windows resources, Visual Studio's resource editor will create an `.aps` file next to it.

# Source Directory

The `Source` directory contains the top level `CMakeLists.txt` file, a `GenerateSolution.bat` file, and other supporting CMake files.

Now if this is just a small test project with a single executable then the actual program source code can go directly into the `Source` directory. So in the simplest case you would end up with only two files, `CMakeLists.txt` and something like `Main.cpp`. 

*One could argue that for such a simple project these files could be put into the root directory, and while I have done that myself for trivial programs, I would recommend against it for anything that would grow larger than a handful of source files.*

For larger projects where you expect to be developing multiple artefacts like executables and libraries, the `Source` directory is the root of the source code tree. The exact structure of the subdirectories doesn't much matter since things are connected with simple `CMakeLists.txt` files, so it comes down mainly to your personal preference and the logical structure of the project you are making.

One thing to consider is that each directory in the tree should have its own `CMakeLists.txt` file, so this allows you to group like components together and reduce duplication of build settings.

As an example for one of my projects I have `Applications`, `Build`, `Libraries`, and `Plugins` subdirectories, and in these I have subdirectories which hold components that can be built separately - an executable, a library, etc. In this case there's also a `Build` directory which is used to hold various CMake and other build scripts. The `CMakeLists.txt` for these directories are very simple as they contain only `add_subdirectory` directives, with the goal being that each directory should only reference its immediate subdirectories.

```
(root)\
	Source\
		Applications\
			MainApp\
		Build\
		Libraries\
			Library_A\
			Library_B\
		Plugins\
			Plugin_A\
			Plugin_B\
```

In general everything that can be built with CMake and you want integrated into the Visual Studio solution should be located within this directory tree. 

If desired the batch file can be put into the root directory instead to make it easier to discover for new people. However moving the top level `CMakeLists.txt` file there is more involved and only makes sense if there are other top level directories which also use CMake.

# Generate Solution Batch File

The batch file is a convenient script for calling CMake with the correct command line parameters to generate the Visual Studio solution in the appropriate directory. It's quite handy as you can run it without needing to open a command line terminal, and you can encode all of the parameters needed to call CMake so you don't need to remember them yourself. 

In my case the batch file looks like this snippet below.

```
@echo off
mkdir ..\Build
cd ..\Build
cmake ..\Source -G "Visual Studio 17 2022"
set CMAKE_RESULT=%ERRORLEVEL%
cd ..\Source
if not %CMAKE_RESULT% == 0 pause
```

The lines involving `CMAKE_RESULT` are there to store the result of the CMake command, and if it failed to then keep the command prompt window open on the screen so that you can examine what happened. This is very useful when running from Explorer as if everything goes well the window will disappear, but if it failed it will stay up showing you the errors from CMake. 

This can then be expanded upon to do more things to setup your project and work environment, which I will write about in future articles.

# Top Level CMakeLists File

The top level `CMakeLists.txt` file is used to store the overall project configuration, including any global compiler, linker, and preprocessor flags, and it includes the project specific subdirectories. It is also the file from which the Visual Studio Solution is built.

For now the number of build flags set is minimal so they are explicitly set here, but when this grows they could be moved into a separate `Build/CompilerSettings.cmake` file so that this top level file remains relatively simple. I've used both methods and it really depends on your personal preference as to where the compiler flags should go. Keeping them only inside `CMakeLists.txt` files means there's one standard place to look for compiler flags, but there's nothing wrong in having a dedicated file to store them instead.

For this simple case the top level `CMakeLists.txt` file looks like:

```
cmake_minimum_required(VERSION 3.22)

project(Application)

# Allow for using folders in Visual Studio
set_property(GLOBAL PROPERTY USE_FOLDERS ON)

# Set C++20 as the standard
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Make Windows headers use wide character set
add_definitions(/D UNICODE /D _UNICODE)

# Provide a custom way to declare precompiled headers
include(Build/Precompiled.cmake)

add_subdirectory(Libraries)
add_subdirectory(Applications)
add_subdirectory(Plugins)

```

Some things to note:
* Setting the `USE_FOLDERS` property will enable you to organise your source files into folders within Visual Studio named whatever you want, otherwise the files just get split into "Header Files" and "Source Files" which is the Visual Studio default. The [Project Structure](#project-structure) section below will show how I organise files into folders in the IDE.
* Setting the `UNICODE` definitions will toggle the project and the Windows headers into using wide character versions. This is optional but until utf8 works better in Windows it is the better option. *I will be investigating this and write a future article on the current state of UTF8 in Windows development*
* Including the `precompiled.cmake` here to add a CMake function for custom [precompiled headers]({% post_url 2022-10-18-precompiled_header_snippet %}) that we can use to speed up the build.
* Setting the C++ standard to C++20, while you don't need to do the same and can pick a different standard, if you don't pick one then you'll be left to whatever the default is for the compiler, or what other dependencies in CMake set it to.

# Project Structure

This brings me to the structure and configuration of the leaf projects which actually build individual libraries and executables. 

The directory structure is split into a couple of main directories:
* An `Include` directory which contains the header files and the public interface for the project. The header files are put into a subdirectory with the same name as the library, which makes it very clear what library you are including a file from. The include directory is usually not present for executable projects as they do not have a public interface.
* A `Source` directory which contains the private code for your project. This is usually filled with source implementation files (`.cpp` etc) but it can also contain header files for any code which should be visible to the project only.
* A `Test` directory for unit test code, which I will describe in a future article.

Which for a library end up looking like so:

```
Library_A\
	Include\
		Library_A\
	Source\
	Test\
	CMakeLists.txt
```

In addition to these directories you an also add in other project specific executables if it makes sense to do so. The main idea being that these executables are more tightly coupled to your library rather than the overall project, so they would be distributed with the library and not the main product. For example in a math library it might make sense to create an executable that runs performance benchmarks, or in an image loading library you would include a simple executable to load the image for fuzzing.

Support files like `.natvis` can be added to the library's root directory, although depending on the circumstances it could make sense to add them either to the `Source` or `Include` directories. *(I'll probably write a more detailed article on this in the future)*

# Project CMakeLists File

The final part is to describe the prototypical `CMakeLists.txt` file that I use for these leaf projects. A key fact that I realised is that you can define more than one compiled artefact in a single file, which means that in a single file can define the library, its unit test executable, and any other supporting tools and scripts.

For me one key feature is the ability to organise source code into different folders in the Visual Studio interface. To do this the `USE_FOLDERS` property needs to be enabled ([see above](#the-top-level-cmakelists-file)), and this pattern of setting a `source_group` needs to be put into the file, with one for each folder you want to create in the interface. 

*Note that if you don't do this then your files will be put into generic `Header Files` and `Source Files` folders in the Visual Studio project.*

```
set(FILE_SRCS
	Include/File/File.hpp
	Include/File/FileSystem.hpp
	Include/File/Directories.hpp
	Source/File.cpp
	Source/FileSystem.cpp
	Source/Directories.cpp
)
source_group("File" FILES ${FILE_SRCS})
```

Then later once all of these source groups have been defined they need to be added to the actual library or executable like the snippet below. I tend to use a separate `target_sources` statement in CMake because it decouples the semantics of the project from just listing the source files used to build it.

```
add_library(CoreLib STATIC)
target_sources(CoreLib PRIVATE
	${FILE_SRCS}
)
```

Then I add the project dependencies and setup other build flags as necessary, and repeat the whole process once more for the unit test executable, leaving me with a `CMakeLists.txt` file which looks something like this:

```
#
# Copyright Notice
#

#
# File Library
#

set(BLOB_SRCS
	Include/File/Blob.hpp
	Source/Blob.cpp
)
source_group("Blob" FILES ${BLOB_SRCS})

set(FILE_SRCS
	Include/File/File.hpp
	Include/File/FileSystem.hpp
	Include/File/Directories.hpp
	Source/File.cpp
	Source/FileSystem.cpp
	Source/Directories.cpp
)
source_group("File" FILES ${FILE_SRCS})

set(MAIN_SRCS
	Source/Precompiled.hpp
	Source/Precompiled.cpp
	File.natvis
)
source_group("" FILES ${MAIN_SRCS})

add_library(FileLib STATIC)
target_sources(FileLib PRIVATE
	${BLOB_SRCS}
	${FILE_SRCS}
	${MAIN_SRCS}
)

find_package(fmt CONFIG REQUIRED)
target_link_libraries(FileLib PUBLIC fmt::fmt)

target_include_directories(FileLib PUBLIC Include)

target_precompiled_header(FileLib Precompiled.hpp Source/Precompiled.cpp)

set_target_properties(FileLib PROPERTIES FOLDER "Libraries")

#
# File Library Unit Tests
#

set(TEST_TESTS_SRCS
	Test/Blob_Test.cpp
	Test/File_Test.cpp
)
source_group("Tests" FILES ${TEST_TESTS_SRCS})

add_executable(FileLibTest)
target_sources(FileLibTest PRIVATE
	${TEST_TESTS_SRCS}
)

target_link_libraries(FileLibTest PRIVATE FileLib)

set_target_properties(FileLibTest PROPERTIES FOLDER "Tests")
```

Some things to note here:
* In the Visual Studio solution the `FileLib` project will go into a top level `Libraries` folder where as the `FileLibTest` project will go into a `Tests` folder. These can be customised by changing the `FOLDER` property for each target. 
* The `target_include_directories` statement in the library section is necessary so that our include paths point to the `Include` directory for this library and other projects that depend on it. It means we can include files like `#include "File/File.hpp"` rather than `#include "Include/File/File.hpp"` which makes the code much cleaner.
* The `target_precompiled_header` directive calls a function to set up custom [precompiled headers]({% post_url 2022-10-18-precompiled_header_snippet %}) in the project. 

# Closing Remarks

So there you go, this is a basic outline of how I organise my CMake projects for use within Visual Studio. In the coming months I will write about enhancing this basic structure with actual unit testing, integrating a package manager for automatic 3rd party dependency installation, and other things that I've discovered in getting this all to work. I'll also try to upload a sample project to Github because just seeing it on disk can be easier to understand than just reading about it.

If you have any comments, questions, or suggestions then please contact me with the methods below.
