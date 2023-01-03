---
layout: post
title: "Three ways to setup Vcpkg in a CMake project for Visual Studio 2022"
date: 2023-01-03 12:00:00 +0100
category: "CMake"
tags: cmake vcpkg visualstudio c++
excerpt_separator: <!--more-->
---

In this article I present three different ways of integrating and using the [vcpkg](https://vcpkg.io/) package manager with a [CMake](https://cmake.org/) project. Each of these ways is something I've used before in my own projects, and each is useful in its own way with its own benefits and drawbacks. This article builds on the previous [CMake Visual Studio project article]({% post_url 2022-11-11-setting_up_basic_project_cmake_visual_studio %}) and it uses that setup in the examples here.

<!--more-->

In all cases you'll still need to use `FindPackage` and `target_link_libraries` within CMake to actually reference and consume the installed dependencies. These methods just determine how and where the dependencies are installed, and how much they can interfere with other projects using vcpkg on your system.

# Contents

* [Installing Vcpkg](#installing-vcpkg)
* [Global Installation](#global-installation)
* [Exporting Dependencies](#exporting-dependencies)
* [Manifest Files](#manifest-files)


# Installing Vcpkg

In order to implement any of the methods described below you need to [install vcpkg](https://vcpkg.io/en/getting-started.html) to somewhere on your system. My recommendation is to put it somewhere that is easy and convenient to access as you'll want to interact with it through the command line and occasionally examine the files in the repository. It is enough to just clone the vcpkg repository and run the bootstrap batch file, as there is no need to explicitly integrate vcpkg with MSBuild or Visual Studio.

I would recommend that you create an environment variable called `VCPKG_ROOT` which points to the top level vcpkg directory, and make sure that this directory is also in your `PATH` environment variable.

# Global Installation

This is the simplest and most space efficient method of vcpkg integration as it just uses the libraries from the global installation and doesn't require any specific integration to Visual Studio or a command line shell. I would use this for small or early stage projects where you want to quickly use and experiment with various libraries and ideas.

To integrate vcpkg with your CMake project you must set the CMake toolchain file to point to the vcpkg toolchain file when running CMake. This can be done in a number of ways, based on your preferences:
* Pass it to CMake on the command line with `-DCMAKE_TOOLCHAIN_FILE=%VCPKG_ROOT%\scripts\buildsystems\vcpkg.cmake` and/or add it to the `GenerateSolution.bat` batch file.
* Set it in the top level `CMakeLists.txt` file before the first `project()`, i.e. `set(CMAKE_TOOLCHAIN_FILE, "$ENV{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake")`
* Set it in the `CMakePresets.json` file in the configuration as `"toolchainFile": "$env{VCPKG_ROOT}\\scripts\\buildsystems\\vcpkg.cmake"`

Another thing that you'll need to do is to set what vcpkg triplet gets used to build the dependencies for your project, as these determine the final output of your dependencies and how your project consumes them. This is done in much the same way as setting the toolchain file, though the variable to set is `VCPKG_TARGET_TRIPLET`.

I consider this a necessary step, as unfortunately the [default triplet on Windows](https://github.com/microsoft/vcpkg/issues/53) is actually `x86-windows`, which builds the dependencies in 32 bit mode to match the default setting for new Visual Studio projects. Given that one of my reasons for using CMake in the first place was to simplify the creation and management of 64 bit Visual Studio projects I have to always explicitly set the triplet. The three that I've found most useful for my projects are:
* `x64-windows` which builds all dependencies as DLLs and uses the DLL based C runtime. Most useful for local projects which you don't plan on distributing.
* `x64-windows-static` which builds all dependencies as static libraries and also uses the static C runtime. Most useful for distributing simple programs as everything gets built into a single and portable executable.
* `x64-windows-static-md` which builds all dependencies as static libraries but uses the DLL based C runtime. Most useful for projects with plugins as the main program and the plugins can share the same low level runtime. *Note that this is a community triplet which means that not everything is guaranteed to build with it.*

This method does come with some drawbacks though.

The first being that you're sharing the global library installation with every other project on the system, meaning you have less control over what versions are actually installed, and if you don't explicitly state a version in `FindPackage` then your project could break because one of its dependencies has been updated. For small and quick projects this is not a big deal, and when the project grows and evolves it can be migrated to one of the other methods described below.

The second is that you need to manually install all of the required libraries that are used by your project. This can be fixed by either creating a second small batch file to automate the process *(I've called mine `InstallDependencies.bat`)*, or by just adding the install commands to the existing `GenerateSolution.bat` batch file. In both cases you'll add something like `vcpkg install <lib> <lib> <lib> --triplet=<triplet>` to the batch file. These aren't perfect solutions though, as in the first case you might forget to run the batch file to install/update dependencies every time you need to, and in the second case you always pay the cost of checking and updating the dependencies. The choice comes down to how you work and what works best for you.


# Exporting Dependencies

This is the second method of vcpkg integration where the dependencies are exported from the global vcpkg directory to a subdirectory within the project structure. It is a half way step between using the libraries from the global vcpkg installation and installing them locally using manifest files. It is also a half way step between having your dependencies submitted inside your project directory in version control *(like in a mono-repo)* and having them handled by a package manager. So it can make a useful transition point in your existing project from storing dependencies locally to using a package manager.

The other benefit of using this method is it makes the project more portable in nature, as all dependencies are referenable with paths relative to the project root, and therefore the `VCPKG_ROOT` environment variable is not required.

To implement this method you need to pick a directory to store these exported dependencies in, set the project to point to the toolchain file which will be installed in the directory, and have a way to install and [export](https://devblogs.microsoft.com/cppblog/vcpkg-introducing-export-command/) the dependencies.

In my case I've chosen a directory called `External\` next to `Source\` in my project structure, and I set the toolchain file in the top level `CMakeLists.txt` file before the `project()` statement like so: 
```
set(CMAKE_TOOLCHAIN_FILE "${CMAKE_SOURCE_DIR}/../External/vcpkg/scripts/buildsystems/vcpkg.cmake")
```
Though you are not limited to setting the toolchain file this way, as it can be done in any of the ways described in the first method above, but make sure you adjust the directory appropriately.

Then I modified the batch file which installs my dependencies *(`InstallDependencies.bat` in my case)* to export them after installation. 
```
@echo off
set TRIPLET=x64-windows-static
set LIBRARIES=gtest fmt
vcpkg install %LIBRARIES% --triplet %TRIPLET%
vcpkg export %LIBRARIES% --triplet %TRIPLET% --raw --output-dir=..\External\ --output=vcpkg
```
This ends up creating an `External\vcpkg\` directory structure with the installed binaries, libraries, headers, and supporting files located within. I would recommend adding either the whole `External\` directory or specifically the `External\vcpkg\` directory to the project's `.gitignore` file *(or equivalent)* so that the dependencies are not submitted to version control.

The main down side of using this method is that it's not as clean and integrated as the other two described here, as we need to run manual steps and are using the export command in an unintended way. 

Other minor down sides are that it uses more disk space as the dependency files are installed/copied over from the global vcpkg directory, and it requires adding configuration data to more than a single file. However these are minimal as the other methods also have similar issues so it's not something worth worrying that much about.


# Manifest Files

The third method of integrating vcpkg is probably the most recommended and one of the cleanest and requires fewest manual steps. It uses a special manifest file named `vcpkg.json` that is placed next to your main `CMakeLists.txt` file. With this manifest file all of your project's dependencies can be listed in one place, and you can specify which version of each dependency is used.

To implement this method just set up your project as in the first method, specifying the toolchain file and triplet in a convenient way. For me this involves setting them in the root `CMakeLists.txt` and/or CMake presets file.

Then create a [manifest file](https://vcpkg.readthedocs.io/en/latest/specifications/manifests/) named `vcpkg.json` that looks something like:
```
{
	"$schema": "https://raw.githubusercontent.com/microsoft/vcpkg/master/scripts/vcpkg.schema.json",
	"name": "windows-project",
	"version": "0.1",
	"dependencies": [
		"fmt",
		"gtest"
	]
}
```
In this example the `windows-project` has two dependencies, on the {fmt} library and on Google Test, both of which it just uses the latest versions. The name and version fields can be pretty much whatever you want although the name field does have some restrictions in what characters are permitted.

The second main benefit of using a manifest file is that it changes how vcpkg integrates with CMake. With it present whenever CMake checks and updates the Visual Studio solution and/or project files it triggers vcpkg to check and update your project's dependencies. Meaning that they are automatically installed and updated without needing to run a separate batch file or manual process.

This of course can be a double edged sword, as automatically updating dependencies can be both time consuming *(for large dependencies like Boost and Qt)*, and could cause issues in your project when incompatibilities are encountered. However these are only triggered when the vcpkg installation is updated, so be aware of this and be prepared to freeze dependencies to [specific versions](https://vcpkg.readthedocs.io/en/latest/users/versioning/#version-1).
