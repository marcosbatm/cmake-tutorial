# CMake Tutorial: Notes

In order to learn how to set up scalable, cross-platform, complex projects I decided to learn how to use CMake as a building tool.

This file has all the notes taken while completing the CMake Tutorial.

## Step 0: Before You Begin

### Getting CMake

CMake is often available as part of the base image of most CI/CD runners targeting C/C++. You should consult the documentation for your software build environment to see if CMake is already available.

To see cmake's version: `cmake --version`.

### Generators

CMake is not ultimately responsible for running the commands which produce the software build. Instead, CMake generates a build system based on project, environment, and user-provided configuration information.

Using CMake thus requires one of the build programs which consumes this generator output be available. The default generator on Windows is typically the newest available Visual Studio version on the machine running CMake, everywhere else it is Unix Makefiles. **Which generator is used can be controlled via the CMAKE_GENERATOR environment variable.**

### Single vs Multi-Config Generators

It is possible to treat the underlying build system as an implementation detail and not differentiate between, for example, `ninja` and `make`. However, **there is one property of the generator which we need to be aware of: if the generator supports single configuration builds, or if it supports multi-configuration builds.** Software builds often have several variants which we might be interested in (like Debug, Release, RelWithDebInfo, and MinSizeRel). A single-configuration build system always builds the software the same way, (it will always produce a Debug build). A multi-configuration build system can produce different outputs depending on the configuration specified at build time.

Examples of single-configuration generators are `Ninja` and `Unix Makefiles`.When using a single-configuration generator, the build type is selected based on the CMAKE_BUILD_TYPE environment variable, or can be specified directly when invoking CMake via `cmake -DCMAKE_BUILD_TYPE=<config>`. This means that we still can have multiple types of builds in single-configuration generators, but we can only build one type at a time.

When using a multi-configuration generator, the build configuration is specified at build time using either a build-system specific mechanism, or via the `cmake --build --config` option.

### Basic Commands

- The main documentation of the `cmake` command is [available here.](https://cmake.org/cmake/help/latest/manual/cmake.1.html)
- `cmake -S <dir>`: the project root directory, where CMake will find the project to be built. This contains the root CMakeLists.txt file, defaults to the current working directory.
- `cmake -B <dir>`: the build directory, where CMake will output the files for the generated build system and artifacts of the build itself when the build system is run, defaults to the current working directory.
- `cmake --build <dir>`: Runs the build system in the specified build directory. For multi-configuration generators, the desired configuration can be requested via: `cmake --build <dir> --config <cfg>`.

### Try It Out

[`Step0/`](Step0/) directory contains a simple "Hello World" C++ project. If we navigate there and run `cmake -B build`, then the build/ folder is created with the default generator for the platform we're at. If we run `cmake -G Ninja -B build`, `Ninja` will be used instead.

**Note: It is necessary to delete the build directory between CMake runs if you want to switch to a different generator using the same build directory.**

How we build and run the project after generating the build system depends on the kind of generator we're using. If it is a single-configuration generator on a non-Windows platform, we can simply do:

```.sh
cmake --build build
./build/hello # On Windows we might need to specify the file extension, ./build/hello.exe
```

If we're using a multi-configuration generator, we will specify the build configuration and the result of the build will be stored in a configuration-specific subdirectory of the build folder (ie. build/debug). Then we just run the following.

```.sh
cmake --build build --config Debug
./build/Debug/hello
```

## Step 1: Getting Started

In this step we will be able to describe executables, libraries, source and header files, and the linkage relationships between them using CMake.

CMake revolves around one or more files named CMakeLists.txt (or CML). Within a given software project, a CMakeLists.txt will exist within any (but not all) directory where we want to provide instructions to CMake on how to handle files and operations local to that directory or subdirectories.

The project root should contain one as the entry point for CMake and always contain the same two commands at or near the top the file.

```.txt
cmake_minimum_required(VERSION 3.X)

project(MyProjectName)
```

The `cmake_minimum_required()` is a compatibility guarantee and ensures that CMake will adopt the behavior of the listed version.

The `project()` informs CMake that what follows is the description of a distinct software project of a given name and CMake performs various checks to ensure the environment is suitable for building software.

Also, there are four backbone commands of most CMake usage, which will be introduced and learned during the tutorial:

- `add_executable()` and `add_library()` commands for describing output artifacts the software project wants to produce.
- `target_sources()` command for associating input files with their respective output artifacts.
- `target_link_libraries()` command for associating output artifacts with one another.

In other words, we use the commands to tell CMake to create **output artifacts**, which can be libraries or executables, and we associate those with **input files** or other libraries (so our output artifact can use and access them).

### Building an Executable

We need `add_executable()`. This command creates a target. In CMake lingo, a target is a name the developer gives to a collection of properties. Targets are simply names.

Some examples of properties are:

- The artifact kind (executable, library, header collection, etc)
- Source files
- Include directories
- Output name of an executable or library
- Dependencies
- Compiler and linker flags
- and many more...

So, we write `add_executable(MyProgram)` to create the "MyProgram" target, which is a name we can use from now on to add properties like source files we want to build and link. The primary command for this is `target_sources()`, which takes as arguments a target name followed by one or more collections of files. For example:

```.txt
target_sources(MyProgram
  PRIVATE
    main.cxx
)
```

Each collection of files is prefixed by a scope keyword (here: PRIVATE) which describe how a property should be inherited by dependents of our target. This is covered in depth in [Linking Libraries and Executables](#linking-libraries-and-executables).

Note: Paths in CMake are generally either absolute, or relative to the CMAKE_CURRENT_SOURCE_DIR (normally, this means relative to the location of the current CML)

### Building a Library

### Linking Libraries and Executables

### Subdirectories
