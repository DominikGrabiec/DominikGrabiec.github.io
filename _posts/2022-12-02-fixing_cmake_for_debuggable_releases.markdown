---
layout: post
title: "Fixing CMake projects for debuggable releases on Windows"
date: 2022-12-02 12:00:00 +0100
category: "CMake"
tags: cmake visualstudio c++ debugging
excerpt_separator: <!--more-->
---

This short article provides a few snippets that work to clean up the default CMake build configurations and make them more useful to developing and releasing software on Windows. Namely making sure that debugging symbols are available in the `Release` configuration, and removing two of the configurations that don't make much sense.

<!--more-->

# Release with Debug Symbols

The main issue is that the default `Release` configuration does not emit symbol files which you will need to keep once you release the software. These symbol files are important to keep so you can debug and process crash dumps that you get from users of your software. 

There is a `RelWithDebInfo` configuration which is meant to be used as "release with debugging", but it actually has different build parameters to the `Release` configuration so the binaries built are not identical. While this may help with debugging, having symbols for the released software is still the best way to go.

To enable generating symbol files in a `Release` build just add this to your top level `CMakeLists.txt` file before any executables or libraries are defined.

```
# Make Release mode emit PDB for compiled artifacts
add_compile_options("$<$<CONFIG:RELEASE>:/Zi>")
add_link_options("$<$<CONFIG:RELEASE>:/DEBUG>")
```

This adds the appropriate [compiler](https://learn.microsoft.com/en-us/cpp/build/reference/z7-zi-zi-debug-information-format?view=msvc-170) and [linker](https://learn.microsoft.com/en-us/cpp/build/reference/debug-generate-debug-info?view=msvc-170) options to the `Release` configuration. Of course this only works on `MSVC` so if you are working on a multi-platform project then you need to wrap it up with an `if (MSVC) ... endif ()`. (And of course figure the appropriate flags out for the other platforms.)

# Removing RelWithDebInfo and MinSizeRel

The other issue I have is that in most Windows projects the `RelWithDebInfo` and `MinSizeRel` configurations don't make much sense and just clutter up the Visual Studio UI. The `RelWithDebInfo` config doesn't make sense once the `Release` config has symbols being generated for it, and the `MinSizeRel` isn't really needed unless you're working in constrained environments and want a smaller executable size. 

If you want to experiment with a minimum size release then you can either change the build flags on the actual release config, or temporarily bring back `MinSizeRel`, experiment, and then copy over the build settings to the actual `Release` config. It doesn't make much sense to have multiple release type configurations in the same project long term.

This short snippet can be used to disable these two configurations, or rather enable only `Debug` and `Release` configurations. The main tricky point is that this has to be put into your top level `CMakeLists.txt` file **before** the first `project()` directive, otherwise it will not have any effect.

```
# Only have Release and Debug configurations
set(CMAKE_CONFIGURATION_TYPES "Debug;Release" CACHE STRING "" FORCE)
```
