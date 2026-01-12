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

### Conditionals and Loops

When given a string, `if()` will first check if it is one of the known constant values previously discussed. If the string isn't one of those values the command assumes it is a variable, and checks the brace-expanded contents of that variable to determine the result of the conditional.

Strings containing whitespace require double quotes, else they are treated like lists; CMake will concatenate the elements together with semicolons. The reverse is also true, when brace-expanding lists it is necessary to do so inside quotes if we want to preserve the semicolons. Otherwise CMake will expand the list items into space-separate strings.

`if()`, recognize the difference between quoted and unquoted strings. `if()` will only check that the given string represents a variable when the string is unquoted.

Finally, `if()` provides several useful comparison modes such as `STREQUAL` for string matching, `DEFINED` for checking the existence of a variable, and `MATCHES` for regular expression checks. It also supports the typical logical operators, `NOT`, `AND`, and `OR`.

**CMake provides two loop structures, `while()`, which follows the same rules as `if()` for checking a loop variable, and `foreach()`, which iterates over lists of strings.**

### Organizing & the Include Command

For small CMake functions and utilities, it is often beneficial for them to live in their own `.cmake` files outside the project CMLs and separate from the rest of the build system. This allows for separation of concerns, removing the project-specific elements from the utilities we are using to describe them.

To incorporate these separate `.cmake` files into our project, we use the `include()` command. This command immediately begins interpreting the contents of the `include()`'d file in the scope of the parent CML. It is as if the entire file were being called as a macro.

**Traditionally, these kinds of `.cmake` files live in a folder named "cmake" inside the project root.**

## Step 3: Configuration and Cache Variables

CMake has project-specific configuration variables. CMake has many ways that an invoking user or process can communicate these, but the most fundamental of them are `-D` flags.

We'll explore how to provide project configuration options from within a CML, and how to invoke CMake to take advantage of configuration options provided by both CMake and individual projects.

We will want to provide reasonable defaults for these configuration choices, and a way to communicate the purpose of a given option. This function is provided by the `option()` command.

```cmake
option(COMPRESSION_SOFTWARE_USE_ZLIB "Support Zlib compression" ON)
option(COMPRESSION_SOFTWARE_USE_ZSTD "Support Zstd compression" ON)

if(COMPRESSION_SOFTWARE_USE_ZLIB)
  message("I will use Zlib!")
  # ...
endif()

if(COMPRESSION_SOFTWARE_USE_ZSTD)
  message("I will use Zstd!")
  # ...
endif()
```

```bash
$ cmake -B build \
    -DCOMPRESSION_SOFTWARE_USE_ZLIB=OFF
...
I will use Zstd!
```

**The names created by `-D` flags and `option()` are cache variables.** Cache variables are **globally visible** variables which are *sticky*, their value is difficult to change after it is initially set. In fact they are so sticky that, in project mode, **CMake will save and restore cache variables across multiple configurations**. If a cache variable is set once, **it will remain until another `-D` flag preempts the saved variable**.

CMake has dozens of normal and cache variables used for configuration, documented at `cmake-variables(7)`.

`set()` can also be used to manipulate cache variables, but will not change a variable which has already been created.

```cmake
set(StickyCacheVariable "I will not change" CACHE STRING "")
set(StickyCacheVariable "Overwrite StickyCache" CACHE STRING "")

message("StickyCacheVariable: ${StickyCacheVariable}")
```

```bash
$ cmake -P StickyCacheVariable.cmake
StickyCacheVariable: I will not change
```

Because `-D` flags are processed before any other commands, they take precedence for setting the value of a cache variable. Also, cache variables can be shadowed by normal variables. We can observe this by `set()`'ing a variable to have the same name as a cache variable, and then using `unset()` to remove the normal variable.

```cmake
set(ShadowVariable "In the shadows" CACHE STRING "")
set(ShadowVariable "Hiding the cache variable")
message("ShadowVariable: ${ShadowVariable}")

unset(ShadowVariable)
message("ShadowVariable: ${ShadowVariable}")
```

```bash
$ cmake -P ShadowVariable.cmake
ShadowVariable: Hiding the cache variable
ShadowVariable: In the shadows
```

### Using Options

We can imagine a scenario where consumers really want our library, and the Tutorial utility is a "take it or leave it" add-on. In that case, we might want to add an option to allow consumers to disable building our Tutorial binary, building only the MathFunctions library.

With our knowledge of options, conditionals, and cache variables we have all the pieces we need to make this configuration available.

### CMAKE Variables

CMake has several important normal and cache variables provided to allow packagers to control the build. Decisions such as compilers, default flags, search locations for packages, and much more are all controlled by **CMake's own configuration variables**.

Among the most important are language standards. As the language standard can have significant impact on the ABI presented by a given package. For example, it's quite common for libraries to use standard C++ templates on later standards, and provide polyfills on earlier standards.

Ensuring all of our targets are built under the same language standard is achieved with the `CMAKE_<LANG>_STANDARD` cache variables. For C++, this is `CMAKE_CXX_STANDARD`.

Do not `set()` `CMAKE_` globals without very strong reasons for doing so. We'll discuss better methods for targets to communicate requirements like definitions and minimum standards in later steps.

Configuration variables are, by convention, prefixed with the provider of the variable. CMake configuration variables are prefixed with `CMAKE_`, while projects should prefix their variables with `<PROJECT>_`. The tutorial configuration variables follow this convention, and are prefixed with `TUTORIAL_`.

### CMakePresets.json

Managing these configuration values can quickly become overwhelming. In CI systems it is appropriate to record these as part of a given CI step. When developing code locally, typing all these options even once might be error prone. If a fresh configuration is needed for any reason, doing so multiple times could be exhausting.

There are many and varied solutions to this problem, and your choice is ultimately up to your preferences as a developer. It would be impossible to fully enumerate every possible configuration workflow here. Instead we will explore CMake's built-in solution: **CMake Presets**, which give us a format to name and express collections of CMake configuration options.

Presets are capable of expressing entire CMake workflows, from configuration, through building, all the way to installing the software package.

**CMake Presets come in two standard files, `CMakePresets.json`, which is intended to be a part of the project and tracked in source control; and `CMakeUserPresets.json`, which is intended for local user configuration and should not be tracked in source control.**

If we define a `CMakePresets.json` file like this:

```json
{
  "version": 4,
  "configurePresets": [
    {
      "name": "example-preset",
      "cacheVariables": {
        "EXAMPLE_FOO": "Bar",
        "EXAMPLE_QUX": "Baz"
      }
    }
  ]
}
```

Then we can use the preset simply by calling:

```bash
cmake -B build --preset example-preset
```

CMake will search for files named CMakePresets.json and CMakeUserPresets.json, and load the named configuration from them if available. **Command line flags can be mixed with presets. Command line flags have precedence over values found in a preset.**

Presets also support limited macros, variables that can be brace-expanded inside the preset. The only one of interest to us is the `${sourceDir}` macro, which expands to the root directory of the project. We can use this to set our build directory, skipping the `-B` flag when configuring the project.

```json
{
  "name": "example-preset",
  "binaryDir": "${sourceDir}/build"
}
```

## Step 4: In-Depth CMake Target Commands

In this step we will go over all the available target commands in CMake. Not all target commands are created equal. We have already discussed the two most important target commands, `target_sources()` and `target_link_libraries()`. Of the remaining commands, some are almost as common as these two, others have more advanced applications, and a couple should only be used as a last resort when other options are not available.

We'll split these into three groups: the recommended and generally useful commands, the advanced and cautionary commands, and the "footgun" commands which should be avoided unless necessary.

1. **Common/Recommended**

    - `target_compile_definitions()`
    - `target_compile_features()`
    - `target_link_libraries()`
    - `target_sources()`

2. **Advanced/Caution**

    - `get_target_property()`
    - `set_target_properties()`
    - `target_compile_options()`
    - `target_link_options()`
    - `target_precompile_headers()`

3. **Esoteric/Footgun**

    - `target_include_directories()`
    - `target_link_directories()`

This categorization is provided to give newcomers a simple intuition about which commands they should consider first when tackling a problem.

The `get_target_property()` and `set_target_properties()` commands give direct access to a target's properties by name. They can even be used to attach arbitrary property names to a target.

```cmake
add_library(Example)
set_target_properties(Example
  PROPERTIES
    Key Value
    Hello World
)

get_target_property(KeyVar Example Key)
get_target_property(HelloVar Example Hello)

message("Key: ${KeyVar}")
message("Hello: ${HelloVar}")
```

```bash
$ cmake -B build
...
Key: Value
Hello: World
```

The full list of target properties which are semantically meaningful to CMake are documented at cmake-properties(7), however most of these should be modified with their dedicated commands. Conversely, some lesser-used properties are only accessible via these commands.

The `target_precompile_headers()` command takes a list of header files, similar to target_sources(), and creates a precompiled header from them. This precompiled header is then force included into all translation units in the target. This can be useful for build performance.

### Features and Definitions

In earlier steps we cautioned against globally setting `CMAKE_<LANG>_STANDARD` and overriding packagers' decision concerning which language standard to use. On the other hand, many libraries have a minimum required feature set they need in order to build, and for these it is appropriate to use the `target_compile_features()` command to communicate those requirements.

```cmake
target_compile_features(MyApp PRIVATE cxx_std_20)
```

The `target_compile_features()` command describes a minimum language standard as a target property. If the `CMAKE_<LANG>_STANDARD` is above this version, or the compiler default already provides this language standard, no action is taken. If additional flags are necessary to enable the standard, these will be added by CMake.

For C++, the compile features are of the form `cxx_std_YY` where `YY` is the standardization year, e.g. `14`, `17`, `20`, etc.

**Note:** This recommendation makes sense because if you link a library that only supports C++14, you might force it to compile with C++20. It's better to avoid "compiling everything with C++20", instead we want to "declare that a given target needs C++20 features".

The `target_compile_definitions()` command describes compile definitions as target properties. It is the most common mechanism for communicating build configuration information to the source code itself.

```cmake
target_compile_definitions(MyLibrary
  PRIVATE
    MYLIBRARY_USE_EXPERIMENTAL_IMPLEMENTATION

  PUBLIC
    MYLIBRARY_EXCLUDE_DEPRECATED_FUNCTIONS
)
```

It is neither required nor desired that we attach -D prefixes to compile definitions described with `target_compile_definitions()`. CMake will determine the correct flag for the current compiler.

Essentially, **modern CMake is focused on being target-oriented**. Therefore **we use `target_compile_features()` and `target_compile_definitions()` to communicate language standard and compile definition requirements.**

### Compile and Link Options

We use `target_compile_options()` and `target_link_options()` to exercise specific control over the exact options being passed on the compile and link line.

```cmake
target_compile_options(MyApp PRIVATE -Wall -Werror)
target_link_options(MyApp PRIVATE -T LinksScript.ld)
```

There are several problems with unconditionally calling `target_compile_options()` or `target_link_options()`. **The primary problem is compiler flags are specific to the compiler frontend being used**. In order to ensure that our project supports multiple compiler frontends, we must only pass compatible flags to the compiler.

We can achieve this by checking the `CMAKE_<LANG>_COMPILER_FRONTEND_VARIANT` variable which tells us the style of flags supported by the compiler frontend. In versions after CMake 3.26 checking this variable alone is sufficient. Prior to CMake 3.26, it was only set for compilers with multiple frontend variants. **This tutorial step already includes correct logic for checking the compiler variant for MSVC, GCC, Clang, and AppleClang on CMake 3.23.**

Even if a compiler accepts the flags we pass, the semantics of compiler flags change over time (especially with regards to warnings). **Projects should not turn warnings-as-error flags by default**, as this can break their build on otherwise innocuous compiler warnings included in later releases.

**Note:** For errors and warnings, consider placing flags in `CMAKE_<LANG>_FLAGS` for local development builds and during CI runs (via **preset** or `-D` flags). We know exactly which compiler and toolchain are being used in these contexts, so we can customize the behavior precisely without risking build breakages on other platforms.

### Include and Link Directories

**It is generally unnecessary to directly describe include and link directories, as these requirements are inherited when linking together targets generated within CMake, or from external dependencies imported into CMake with commands we will cover in later steps.**

If we happen to have some libraries or header files which are not described by a CMake target which we need to bring into the build, perhaps pre-compiled binaries provided by a vendor, we can incorporate with the target_link_directories() and target_include_directories() commands.

```cmake
target_link_directories(MyApp PRIVATE Vendor/lib)
target_include_directories(MyApp PRIVATE Vendor/include)
```

These commands use properties which map to the -L and -I compiler flags (or whatever flags the compiler uses for link and include directories).

Of course, passing a link directory doesn't tell the compiler to link anything into the build. For that we need `target_link_libraries()`. When `target_link_libraries()` is given an argument which does not map to a target name, it will add the string directly to the link line as a library to be linked into the build (prepending any appropriate flags, such a `-l`).

## Step 5: In-Depth CMake Library Concepts

In this step you will learn about some of the most common kinds of libraries that CMake can describe. This will cover most of the in-project uses of `add_library()`.

The `add_library()` command accepts the name of the library target to be created as its first argument. The second argument is an optional `<type>` for which the following values are valid:

- `STATIC` Library: an archive of object files for use when linking other targets.
- `SHARED` Library: a dynamic library that may be linked by other targets and loaded at runtime.
- `MODULE` Library: a plugin that may not be linked by other targets, but may be dynamically loaded at runtime using dlopen-like functionality.
- `OBJECT` Library: a collection of object files which have not been archived or linked into a library.
- `INTERFACE` Library: a library target which specifies usage requirements for dependents but does not compile sources and does not produce a library artifact on disk.

All of this definitions are explained thoroughly in `cmake-buildsystem(7)`'s [Binary Targets section](https://cmake.org/cmake/help/latest/manual/cmake-buildsystem.7.html#binary-targets).

### Static and Shared

While the `add_library()` command supports explicitly setting `STATIC` or `SHARED`, and this is sometimes necessary, it is best to leave the second argument empty for most "normal" libraries which can operate as either.

When not given a type, `add_library()` will create either a `STATIC` or `SHARED` library depending on the value of `BUILD_SHARED_LIBS`. If `BUILD_SHARED_LIBS` is true, a `SHARED` library will be created, otherwise it will be `STATIC`. CMake does not define the `BUILD_SHARED_LIBS` variable by default, meaning without project or user intervention `add_library()` will produce `STATIC` libraries.

```cmake
add_library(MyLib-static STATIC)
add_library(MyLib-shared SHARED)

# Depends on BUILD_SHARED_LIBS
add_library(MyLib)
```

This is desirable behavior, as it allows packagers to determine what kind of library will be produced, and ensure dependents link to that version of the library without needing to modify their source code. In some contexts, fully static builds are appropriate, and in others shared libraries are desirable.

