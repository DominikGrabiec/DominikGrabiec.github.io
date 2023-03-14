---
layout: post
title: "Configuring Vcpkg with a Custom Triplet"
date: 2023-03-14 12:00:00 +0100
category: "CMake"
tags: cmake vcpkg visualstudio c++
excerpt_separator: <!--more-->
---

In this article I extend the existing [Visual Studio CMake project]({% post_url 2022-11-11-setting_up_basic_project_cmake_visual_studio %}) [with vcpkg]({% post_url 2023-01-03-three_ways_vcpkg_cmake_visual_studio %}) to be able to customise packages that we use in [vcpkg](https://vcpkg.io). The main reasons to do this is to select alternative configurations of packages like [spdlog](https://github.com/gabime/spdlog), or to switch the linking method for a particular package, for example linking dynamically with LGPL libraries but statically with everything else.

<!--more-->

While most vcpkg package configuration options can be specified by selecting the appropriate features in the manifest file there are some which need to be specified earlier before the package gets built. 

One of these is the `SPDLOG_WCHAR_FILENAMES` option for the spdlog library, as it needs to be [specified in the triplet file](https://github.com/microsoft/vcpkg/blob/d51b969a7db84d56d2083bb22b2f95254bdc4c3f/ports/spdlog/portfile.cmake) because it is an alternative way of using the library and not an additional feature which can be added to it.

Another configuration option that you'd want to specify at the triplet level is to change the way a specific library gets compiled and linked into your project. The example here is you're building a closed source application and by default statically linking all libraries into your executable but you need to dynamically link certain LGPL libraries like Qt so you can distribute them separately.

# Custom Triplet File

The first step is to customise your top level `CMakeLists.txt` file and add/modify two statements before the first `project()` declaration.

* First specify the custom triplet directory so that vcpkg knows where to look for it. It's done by setting the `VCPKG_OVERLAY_TRIPLETS` variable to the appropriate directory, which in this case is a `Triplets` directory within the `Source/Build/` directory.
In my case this line looks like: `set(VCPKG_OVERLAY_TRIPLETS "${CMAKE_CURRENT_SOURCE_DIR}/Build/Triplets")`

* Next change the triplet that you use to refer to the new name of your triplet file. You might need to add this line if it doesn't exist, and it should look something like: `set(VCPKG_TARGET_TRIPLET "x64-windows-custom")`.
Of course if you used a different way of specifying the triplet file then you'll need to make the change there instead.

The next step is to create the `Triplets` directory inside `Source/Build/` and then to create your triplet file within. Since I'm calling my triplet `x64-windows-custom` then the name of the file needs to be `x64-windows-custom.cmake`. To create this file either copy and rename the previous triplet file you were using from the `triplets` directory in the vcpkg installation or create a blank text file and fill in the contents yourself.

Since I based my custom triplet on the `x64-windows-static` community triplet the `x64-windows-custom.cmake` file initially looks like this:
```
set(VCPKG_TARGET_ARCHITECTURE x64)
set(VCPKG_CRT_LINKAGE dynamic)
set(VCPKG_LIBRARY_LINKAGE static)
```

At this stage the project should work exactly like it did before as we've just copied the settings from the previous triplet file. However when you next run CMake for your project expect vcpkg to take a while as it will have to build and install all your dependencies again for your new triplet.

# Customising the Triplet

Once the custom triplet file is up and working we can then add the customisations to it.

To handle the first spdlog case is relatively simple as you just need to add `set(SPDLOG_WCHAR_FILENAMES ON)` to the file and that's it. You can then clean up the temporary files of our project and re-run CMake to run vcpkg to build and install the new versions of our libraries. 

To handle building and linking libraries in a different way to the default requires adding a bit more code, as the triplet file gets processed to build every dependency that you use. In the example described above where everything should be linked statically except the Qt libraries we need to add code to override the value of `VCPKG_LIBRARY_LINKAGE` but only for Qt libraries.

Handily there's a `PORT` variable available while processing the triplet file which gets set to the name of the package currently being processed. Using this we can check if the `PORT` is anything related to `qt` and change the linkage based on that, with the resulting code looking like this:
```
if(PORT MATCHES "qt")
    set(VCPKG_LIBRARY_LINKAGE dynamic)
endif()
```

This is actually near identical to the example used in the [Microsoft vcpkg documentation](https://learn.microsoft.com/en-us/vcpkg/users/triplets#per-port-customization) on per-port customisation.

# Final Notes

I would limit myself to only making these two types of changes to the custom triplet file. Adding configuration items which will be required for the entire platform like with spdlog using wide characters for filenames, and changing the linker options for certain libraries. Putting in feature selection or other non-platform related settings seems like the wrong thing to do and there are better and more standard places to put those.