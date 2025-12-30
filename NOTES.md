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

```cmake
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

```cmake
target_sources(MyProgram
  PRIVATE
    main.cxx
)
```

Each collection of files is prefixed by a scope keyword (here: PRIVATE) which describe how a property should be inherited by dependents of our target. This is covered in depth in [Linking Libraries and Executables](#linking-libraries-and-executables).

Note: Paths in CMake are generally either absolute, or relative to the CMAKE_CURRENT_SOURCE_DIR (normally, this means relative to the location of the current CML)

### Building a Library

We just use the command `add_library(MyLibrary)` which is exactly as `add_executables()` but for libraries.

We will also use **header files**. Header files are not a build requirement. They are a usage requirement. We need to know about header files in order to build other parts of a given target.

Header files are described slightly differently than implementation files like tutorial.cxx. They're also going to need different scope keywords than the PRIVATE keyword we have used so far.

To describe a collection of header files, we're going to use what's known as a `FILE_SET`. Therefore, **we add sources/input files to the library with the following:**

```cmake
target_sources(MyLibrary
  PRIVATE
    library_implementation.cxx

  PUBLIC
    FILE_SET myHeaders
    TYPE HEADERS
    BASE_DIRS
      include
    FILES
      include/library_header.h
)
```

First, **we have our implementation file as a PRIVATE source**. However, **we use PUBLIC for our header file**. This allows consumers of our library to "see" the library's header files.

Following the scope keyword is a `FILE_SET`, **a collection of files to be described as a single unit**. A `FILE_SET` consists of the following parts:

- `FILE_SET <name>` is the name of the `FILE_SET`. This is a handle which we can use to describe the collection in other contexts.
- `TYPE <type>` is the kind of files we are describing. Most commonly this will be headers, but newer versions of CMake support other types like C++20 modules.
- `BASE_DIRS` is the "base" locations for the files. This can be most easily understood as the locations that will be described to compilers for header discovery via -I flags. **When compiling C/C++ code, the compiler needs to know where to find header files. The -I flag tells the compiler "look in this directory for header files."** This means, we can use this BASE_DIRS to get the desired `#include` directives.
- `FILES` is the list of files, same as with the implementation sources list earlier.

**In the exercise**, for BASE_DIRS we need to determine the directory which will allow for the desired `#include <MathFunctions.h>` directive. To achieve this, the MathFunctions folder itself will be a base directory. We would make a different choice if the desired include directive were `#include <MathFunctions/MathFunctions.h>`.

### Linking Libraries and Executables

We must introduce a new command, `target_link_libraries()`. It does a great deal more than just invoke linkers. It describes relationships between targets generally.

```cmake
target_link_libraries(MyProgram
  PRIVATE
    MyLibrary
)
```

**scope keywords** describe how properties are made available to targets. There are three of them, **PRIVATE**, **INTERFACE**, and **PUBLIC**.

- A **PRIVATE** property is only available to the target which owns it.
- An **INTERFACE** property is only available to targets which link the owning target. The owning target does not have access to these properties. A header-only library is an example of a collection of **INTERFACE** properties.
- A **PUBLIC** property is the union of the **PRIVATE** and **INTERFACE** properties.

Example:

```cmake
target_sources(MyLibrary
  PRIVATE
    FILE_SET internalOnlyHeaders
    TYPE HEADERS
    FILES
      InternalOnlyHeader.h

  INTERFACE
    FILE_SET consumerOnlyHeaders
    TYPE HEADERS
    FILES
      ConsumerOnlyHeader.h

  PUBLIC
    FILE_SET publicHeaders
    TYPE HEADERS
    FILES
      PublicHeader.h
)
```

This call to `target_sources()` will **modify two properties of the MyLibrary target**. `HEADER_SETS` and `INTERFACE_HEADER_SETS`, which both contain lists of header file sets. The value internalOnlyHeaders will be added to `HEADER_SETS`, consumerOnlyHeaders to `INTERFACE_HEADER_SETS`, and publicHeaders will be added to both.

When a given target is being built, it will use its own non-interface properties (eg, HEADER_SETS), combined with the interface properties of any targets it links to (eg, INTERFACE_HEADER_SETS).

### Subdirectories

We want to make sure we keep commands local to the files they are dealing with. It can be very useful for large projects with many targets and files.

The `add_subdirectory(SubdirectoryName)` command allows us to incorporate CMLs located in subdirectories of the project. When a CMakeLists.txt in a subdirectory is being processed by CMake all relative paths described in the subdirectory CML are relative to that subdirectory

**Note:** Be sure to pay attention to the path changes necessary when moving the `target_sources()` commands into subdirectories.

## Step 2: CMakeLang Fundamentals

In the wild, we will encounter a great deal more complexity than simply describing lists of source and header files.

To deal with this, CMake provides a "CMake Languague" (a.k.a. *CMakeLang*), a Turing-complete domain-specific language for describing the process of building software. Understanding the fundamentals of this language will be necessary to write more complex CMLs and other CMake files.

However, CMakeLang is not well suited to describing things which are not related to building software. Oftentimes the correct answer is to write a tool in a general purpose programming language which solves the problem, and teach CMake how to invoke that tool as part of the build process.

This step is an exception to the tutorial sequencing, it'll be a sandbox to explore languague features without building software. It neither builds on Step1, nor is the starting point for Step3.

The only fundamental types in CMakeLang are strings and lists. Every object in CMake is a string, and lists are themselves strings which contain semicolons as separators. Booleans, numbers, JSON objects, or otherwise are processed by consuming a string, doing some internal conversion logic (in a language other than CMakeLang), and then converting back to a string for any potential output.

We can create a variable, which is to say a name for a string, using the `set()` command. And its value can be accessed using brace expansion for example inside the `message()` command to print.

```cmake
set(var "World!")
message("Hello ${var}")
```

And then, we use `cmake -P <file>` to execute. `cmake -P` is called "script mode", it informs CMake this file is not intended to have a `project()` command.

Conditionals are entirely by convention of which strings are considered true and which are considered false. "True", "On", "Yes", and (strings representing) non-zero numbers are truthy, while "False" "Off", "No", "0", "Ignore", "NotFound", and the empty string are all considered false.

**Taking some time to consult the `if()` documentation on expressions is worthwhile.** It's recommended to stick to a single pair for a given context, such as "True"/"False" or "On"/"Off".

The `list()` command is useful for manipulating lists, which are strings with semicolons. Many structures within CMake expect to operate with this convention. As an example, we can use the `foreach()` command to iterate over a list.

CMakeLang Code:

```cmake
set(stooges "Moe;Larry")
list(APPEND stooges "Curly")

message("Stooges contains: ${stooges}")

foreach(stooge IN LISTS stooges)
  message("Hello, ${stooge}")
endforeach()
```

CMakeLang Output:

```bash
$ cmake -P CMakeLists.txt
Stooges contains: Moe;Larry;Curly
Hello, Moe
Hello, Larry
Hello, Curly
```

### Macros, Functions and Lists

These can be very helpful when constructing lots of similar targets, like tests, for which we will want to call similar sets of commands over and over again. We do so with `function()` and `macro()`.

Like with many languages, the difference between functions and macros is one of scope. In CMakeLang, both `function()` and `macro()` can "see" all the variables created in all the frames above them. However, a `macro()` acts semantically like a text replacement, so any side effects the macro creates are visible in their calling context. If we create or change a variable in a macro, the caller will see the change.

On the other hand, `function()` creates its own variable scope. To propagate changes to the parent which called the function, we must use `set(<var> <value> PARENT_SCOPE)`.

In CMake 3.25, the `return(PROPAGATE)` option was added, which works the same as `set(PARENT_SCOPE)` but provides slightly better ergonomics.

Lastly, `macro()` and `function()` both support variadic arguments via the ARGV variable, a list containing all arguments passed to the command, and the ARGN variable, containing all arguments past the last expected argument.

`.cmake` is the standard extension for CMakeLang files when not contained in a CMakeLists.txt

CMake variables are names for strings; or put another way, a CMake variable is itself a string which can brace expand into a different string.

**This leads to a common pattern in CMake code where functions and macros aren't passed values, but rather, they are passed the names of variables which contain those values. Thus ListVar does not contain the value of the list we need to append to, it contains the name of a list, which contains the value we need to append to.**

**When expanding the variable with `${ListVar}`, we will get the name of the list. If we expand that name with `${${ListVar}}`, we will get the values the list contains.**
