---
layout: post
title: A Resource Compiler with CMake and Standard C++
comments: true
---

# A Resource Compiler with CMake and Standard C++

I recently worked on a project which required that I serve a static web page
from an embedded C++ HTTP server. I could have gone the fairly simple route of
installing the static page files alongside the executable and serving them
from the filesystem, but there was a catch: One of the requirements was extreme
portability. This program should be a single executable file that can be moved
around and executed all by itself without having a bunch of files lying around.

I could have gone the truly hideous route of writing the entire static page
inline as a string literal. I don't think it requires much explaination on why
I didn't want to do this. I opted to go with the single-page application
approach: The interface itself is fairly simple. Still, I didn't want to have
something ugly and hard to maintain. I wanted to be able to work with all my
favorite web tooling, Bower, Polymer, etc. I also wanted a half-decent editing
experience. It should be abundantly clear as to why an inline string literal
approach would be completely unusable.

Of course, the best alternative is _resource compilation_. For any unfamiliar
with the term: This is when you take arbitrary files and bake them into an
executable (or library), which you can then load and read from at runtime. It's
a fairly common technique, but usually requires quite a bit of external tooling
and dependencies to get it up and running. There are a few common techniques:

- Qt's `qrc`: This is popular among Qt projects, and it works amazingly well. It
  is tightly integrated with `QFile` and `QDir`, so opening and reading files
  that were compiled in as resources with `qrc` is as simple as opening and
  reading real files on disk. I've used it in the past to greate effect. The
  big problem is that this requires dragging in the entire Qt ecosystem. Anyone
  familiar with Qt knows that you can't just pick-and-choose your favorite parts
  from Qt and just use those. It's an all-or-nothing deal. That's why Qt is
  known as a _framework_, rather than a _library_. I didn't want to pull in Qt
  and all the things it entailed. I wanted an executable that was small and
  easy to distribute, and Qt's licensing would make that difficult. Besides
  that, even if I were able to statically link Qt into the program, the binary
  size would probably explode (not to mention all the other libraries that Qt
  depends on).
- Microsoft's `rc`: This is common on Windows platforms, and is actually how
  executables on Windows get their icons. I'm not as familiar with this
  technique, but I was unable to make use of it because I needed the program to
  be cross-platform. This method just wouldn't work.
- [A pretty neat trick using `objcopy`](https://www.linuxjournal.com/content/embedding-file-executable-aka-hello-world-version-5967):
  I've tinkered with this in the past, but it's somewhat impractical because of
  the weird implications with the linker and symbol names. Of course, it it also
  only available when `objcopy` is also available, which again, makes it not
  cross-platform. A no-go.
- Many, many, other toolset-specific techniques.

I ended up inventing a CMake-based script which would compile the arbitrary
files. I can't show the exact script here (nor would I want to, it was a pretty
big hack), but I think I'd like to build something upon the same ideas:
Something that others my find usable and informative.

## A CMake Foundation

CMake has quite a few tricks up its sleeves that most users are completely
unaware of. I'll be going over those in this post as we see what kind of magic
we can pull off. I'll be building this CMake-based resource compiler as I write
this, so it will be learning experience for both of us. My goal is that all of
this code should work with standard CMake and standard C++11, no more, no less.

Let's begin!

## Step 1: A Simple Test Case

While I don't want to do full-scale TDD, I want to write a little example that
I'd like to get working, and then actually implement. Here will be an example
C++ program:

```c++
#include <cmrc/cmrc.hpp>

#include <iostream>

int main() {
  CMRC_INIT(hello);
  auto data = cmrc::open("hello.txt");
  std::cout << std::string(data.begin(), data.end()) << '\n';
}
```

Here's my idea:

1. `cmrc/cmrc.hpp` will be a header file with the resource-file loading library.
2. `CMRC_INIT` will initialize the resource library. This will be required, I
   believe. We'll discuss it a bit more later.
3. `cmrc::open` will open a compiled-in resource by filename. What type does it
   return? I'm not sure yet... But it should have a `begin()` and `end()`
   returning two RandomAccessIterators.

That's it! Fairly simple, and I'm hoping that upon this one could build more
complex and useful abstractions.

Compiling the file would be simple. Here's the `CMakeLists.txt` I'd like to
have:

```cmake
cmake_minimum_required(VERSION 3.6)
project(CMRCExample)

include(CMakeRC.cmake)

cmrc_add_resource_library(hello hello.txt)
add_executable(main main.cpp)
target_link_libraries(main PRIVATE hello)

include(CTest)
enable_testing()
add_test(main main)
set_property(TEST main PROPERTY PASS_REGULAR_EXPRESSION "Hello, world!")
```

Here's the idea here:

1. `CMakeRC.cmake` is the script that we'll be going over in this post. It will
   be doing most of the heavy lifting.
2. `cmrc_add_resource_library` does a bunch of magic and eventually calls
   `add_library` to generate a new target with the given resource files compiled
   in.
3. `target_link_libraries` called with the resource library will make the
   resources defined in the linked library available to the linkee.
4. The simple test case should output `Hello, world!`, so we set that as the
   expected output for CTest to check for us.

Of course, we need an example resource file to compile. That will be
`hello.txt`, which contains the text `Hello, world!`. We should expect that to
be the output of the program.

## Step 2: Getting the Test Case to Build

Of course, trying to build the test case won't even configure. We can get it to
at least configure by adding a `CMakeRC.cmake` file like this:

```cmake
function(cmrc_add_resource_library name)
    add_library(${name} INTERFACE)
endfunction()
```

We'll see it fail to build immediately as it cannot find `cmrc.hpp`. We must
generate this file! We can do that when `CMakeRC.cmake` is first included. Let's
add some basic content at the top of the module:

```cmake
if(COMMAND cmrc_add_resource_library)
    # CMakeRC has already been included! Don't do anything
    return()
endif()

set(this_script "${CMAKE_CURRENT_LIST_FILE}")

set(CMRC_INCLUDE_DIR "${CMAKE_BINARY_DIR}/_cmrc/include" CACHE INTERNAL "Directory for CMakeRC include files")
# Let's generate the primary include file
file(MAKE_DIRECTORY "${CMRC_INCLUDE_DIR}/cmrc")
file(WRITE "${CMRC_INCLUDE_DIR}/cmrc/cmrc.hpp" [==[
#ifndef CMRC_CMRC_HPP_INCLUDED
#define CMRC_CMRC_HPP_INCLUDED

#include <string>

namespace cmrc {

class resource {
public:
    char* begin() { return nullptr; }
    char* end() { return nullptr; }
};

extern resource open(const char* fname) { return {}; }
inline resource open(const std::string& fname) {
    return open(fname.data());
}

}

#endif // CMRC_CMRC_HPP_INCLUDED
]==])

add_library(cmrc-base INTERFACE)
target_include_directories(cmrc-base INTERFACE "${CMRC_INCLUDE_DIR}")
target_compile_features(cmrc-base INTERFACE cxx_nullptr)
set_property(TARGET cmrc-base PROPERTY INTERFACE_CXX_EXTENSIONS OFF)
add_library(cmrc::base ALIAS cmrc-base)
```

Now, as soon as `CMakeRC.cmake` is included, there will be a target `cmrc::base`
that can be linked to so that anyone can get the proper include directories. We
can propagate that further by modifying `cmrc_add_resource_library`:

```cmake
function(cmrc_add_resource_library name)
    add_library(${name} INTERFACE)
    target_link_libraries(${name} INTERFACE cmrc::base)
endfunction()
```

Now anyone who links to a resource library will also link to `cmrc::base`, and
will be able to include `cmrc/cmrc.hpp`. They'll also get the required compiler
flags for `nullptr` to be available when they compile. Great!

Of course, the test case doesn't yet pass. We've got a few things to do before
we can get there...

## Step 3: Compiling a Resource

Now, how does one actually _compile an arbitrary file_? Well, it's not actually
hard _in theory_. What we need is to convert the file to a sequence of
character literals that we will put into a `char[]` array that can then be read
by the program at runtime.

So, how do we do that?

Well, we need to generate a source file that can be read by the compiler. CMake
has great support for this via `add_custom_command`! Let's define a function,
called `_cmrc_generate_intermediate_cpp`, which will generate an intermediate
C++ file containing such a character array:

```cmake
function(_cmrc_generate_intermediate_cpp outfile infile)
    add_custom_command(
        # This is the file we will generate
        OUTPUT "${outfile}"
        # These are the primary files that affect the output
        DEPENDS "${infile}" "${this_script}"
        COMMAND
            "${CMAKE_COMMAND}"
                -D_CMRC_GENERATE_MODE=TRUE
                "-DINPUT_FILE=${infile}"
                "-DOUTPUT_FILE=${outfile}"
                -P "${this_script}"
        COMMENT "Generating intermediate file for ${infile}"
    )
endfunction()
```

Well... It doesn't actually generate the array. It adds a build command which
will generate the file _at build time_. We _do not_ want to generate the
intermediate file immediately: It may be costly (the input file may be very
large), or the input file may be the output of a previous build command. This
also allows us to make use of any inherent parallelism afforded us by a build
tool like Make or Ninja. We invoke the `CMakeRC.cmake` script as the build
command using `cmake -P`, which is quite a useful trick for building more
complex commands. We pass in three primary arguments: the input file, the output
file, and a boolean `_CMRC_GENERATE_MODE`. I'll get back to that one later.

So, how do we use `_cmrc_generate_intermediate_cpp`?
We modify `cmrc_add_resource_library` to call `_cmrc_generate_intermediate_cpp`
for each resource file pass in, like so:

```cmake
function(cmrc_add_resource_library name)
    # Generate a static library with the compiled in character arrays.
    set(sources)
    foreach(input IN LISTS ARGN)
        get_filename_component(abs_input "${input}" ABSOLUTE)
        # Generate a filename based on the input filename that we can put in
        # the intermediate directory.
        file(RELATIVE_PATH relpath "${CMAKE_CURRENT_SOURCE_DIR}" "${abs_input}")
        if(relpath MATCHES "^\\.\\.")
            # For now we just error on files that exist outside of the soure dir.
            message(SEND_ERROR "Cannot add file '${input}': File must be in a subdirectory of the source directory")
            continue()
        endif()
        get_filename_component(abspath "${CMAKE_CURRENT_BINARY_DIR}/${name}-intermediate/${relpath}.cpp" ABSOLUTE)
        # Generate the rule for the intermediate source file
        _cmrc_generate_intermediate_cpp("${abspath}" "${abs_input}")
        list(APPEND sources "${abspath}")
    endforeach()
    # Generate the actual static library. Each source file is just a single file
    # with a character array compiled in containing the contents of the
    # corresponding resource file.
    add_library(${name} STATIC ${sources})
    target_link_libraries(${name} PUBLIC cmrc::base)
endfunction()
```

Currently, we error if the resource it outside of the source directory. That's
pretty bad, but we'll get to that later, as it requires a bit of trickery to get
right.

If you try to build now, you'll see an error about `add_library` being
un-scriptable. This is because `_cmrc_generate_intermediate_cpp` uses the
existing `CMakeRC.cmake` as the script that will generate the intermediate
C++ file. When using `-P` mode to CMake, we cannot use some commands, including
`add_library` (after all, such a thing doesn't make sense). This is the
purpose of the `_CMRC_GENERATE_MODE` flag I mentioned earlier. At the top of
the `CMakeRC.cmake` script, we'll add a short-circuit to do something different
if this flag is set:


```cmake
if(_CMRC_GENERATE_MODE)
    message(STATUS "Generating file ${OUTPUT_FILE}")
    return()
endif()
```

If I run the build now, we'll see `Generating file <some-path>/hello.txt.cpp`,
followed by an immediate compiler error because it tries to compile that C++
file, but we never really generated it. So, how do we generate it?

When I was initially thinking about how to solve this resource-compiling
problem, the one thing that made me immediately think that CMake was up to the
task was one very specific and rarely used CMake feature: `file(READ HEX)`. Yep,
CMake's `file(READ)` command lets you read in the bytes of a file as a sequence
of hexadecimal digits, just by using the `HEX` keyword on the end. And
what can we do with those digits? Well, we can use them to generate character
literals, of course! Here's a simple example:

```cmake
# Read in the digits
file(READ "${INPUT_FILE}" bytes HEX)
# Forat each pair into a character literal
string(REGEX REPLACE "(..)" "'\\\\x\\1', " chars "${bytes}")
# Just write it to stdout for this example
message("${chars}")
```

This example stands entirely on its own, none of the other code is needed.
Execute it like so:

```sh
cmake -DINPUT_FILE=hello.txt -P CMakeRC.cmake
```

You'll see the contents of the file printed to standard out, but as a
comma-separated list of character literals:

```
'\x48', '\x65', '\x6c', '\x6c', '\x6f', '\x2c', '\x20', '\x77', '\x6f', '\x72', '\x6c', '\x64', '\x21',
```

Integrating this into our existing `CMakeRC.cmake` is very simple:

```cmake
if(_CMRC_GENERATE_MODE)
    # Read in the digits
    file(READ "${INPUT_FILE}" bytes HEX)
    # Format each pair into a character literal
    string(REGEX REPLACE "(..)" "'\\\\x\\1', " chars "${bytes}")
    file(WRITE "${OUTPUT_FILE}" "
namespace { const char file_array[] = { ${chars} }; }
extern const char* const file_begin = file_array;
extern const char* const file_end = file_array + sizeof(file_array);
")
    return()
endif()
```

Everything should now compile and run, but the test will still fail. If you take
a peek into the build directory, you can see the intermediate generated file:

```c++
namespace { const char file_array[] = { '\x48', '\x65', '\x6c', '\x6c', '\x6f', '\x2c', '\x20', '\x77', '\x6f', '\x72', '\x6c', '\x64', '\x21' }; }
extern const char* const file_begin = file_array;
extern const char* const file_end = file_array + sizeof(file_array);
```

The `file_array` will contain the contents of our `hello.txt` file encoded as a
C++ character array. Pretty neat, and it only took 8 SLOC in `CMakeRC.cmake`!

There's a problem, though: We've named the two pointers `file_begin` and
`file_end`, and we never vary them when compiling files. This means that we can
only compile one resource into our binary, ever! That's pretty useless...

In addition, how do we go about accessing the contents from `cmrc::open`? We
could just return a pointer to `file_begin` and `file_end`, but we want to get
rid of names those and somehow access based on filename. Hm...

## Step 4: Encoding the Filesystem Structure

We want to be able to access things by the path from which they were compiled,
so we must somehow encode the file paths into both the symbol names as well as
some kind of lookup-table that is searched at runtime based on filenames.

### Step 4a: Generating Identifiers/Symbols

This is actually a fairly simple task. We want to make a symbol based on the
name of the file that is passed in to `cmrc_add_resource_library`. Let's make
a function to do that:

```cmake
function(_cm_encode_fpath var fpath)
    string(MAKE_C_IDENTIFIER "${fpath}" ident)
    string(MD5 hash "${fpath}")
    string(SUBSTRING "${hash}" 0 4 hash)
    set(${var} f_${hash}_${ident} PARENT_SCOPE)
endfunction()
```

We do not manipulation of the path: We assume that the caller of
`_cm_encode_fpath` will pass us in the path that they want to be encoded,
exactly as they want it. To make us of this, we need to change up
`_cmrc_generate_intermediate_cpp` a little bit:

```cmake
function(_cmrc_generate_intermediate_cpp libname symbol outfile infile)
    add_custom_command(
        # This is the file we will generate
        OUTPUT "${outfile}"
        # These are the primary files that affect the output
        DEPENDS "${infile}" "${this_script}"
        COMMAND
            "${CMAKE_COMMAND}"
                -D_CMRC_GENERATE_MODE=TRUE
                -DLIBRARY=${libname}
                -DSYMBOL=${symbol}
                "-DINPUT_FILE=${infile}"
                "-DOUTPUT_FILE=${outfile}"
                -P "${this_script}"
        COMMENT "Generating intermediate file for ${infile}"
    )
endfunction()
```

We add two new parameters to the front: a `libname` and a `symbol`. `symbol` is
where we'll pass the name of the symbol for the file, whereas `libname` will
pass the name of the resource library we are generating. That way, to resource
libraries which compile in a file with the same path will not collide with
eachother. We pass `libname` and `symbol` to our generation script as `LIBRARY`
and `SYMBOL`, respectively.

We'll also need to change the generating block at the top of `CMakeRC.cmake` to
make use of these new options:

```cmake
if(_CMRC_GENERATE_MODE)
    # Read in the digits
    file(READ "${INPUT_FILE}" bytes HEX)
    # Format each pair into a character literal
    string(REGEX REPLACE "(..)" "'\\\\x\\1', " chars "${bytes}")
    file(WRITE "${OUTPUT_FILE}" "
namespace { const char file_array[] = { ${chars} }; }
namespace cmrc { namespace ${LIBRARY} { namespace res_chars {
extern const char* const ${SYMBOL}_begin = file_array;
extern const char* const ${SYMBOL}_end = file_array + sizeof(file_array);
}}}
")
    return()
endif()
```

The only thing we needed to change was the generated C++ code template. Great!

The last thing we need to do is change our call to
`_cmrc_generate_intermediate_cpp` in `cmrc_add_resource_library`:

```cmake
# Generate a symbol name for the file's character array
_cm_encode_fpath(sym "${input}")
# Generate the identifier for the resource library's namespace
string(MAKE_C_IDENTIFIER "${name}" ident)
# Generate the rule for the intermediate source file
_cmrc_generate_intermediate_cpp(${ident} ${sym} "${abspath}" "${abs_input}")
```

And that's that! We can now add as many source files as we wish, and each one
will be compiled to a different object file using a different symbol.

### Step 4b: Generating a Lookup Table

The final step will be to generate some code to actually access the resource
files at runtime. We can do this by generating a simple C++ file for the
library which references each file by name and the corresponding symbol names.
This will require that we slightly modify just about everything we've done at
least a little bit. For this simple use case, we can maintain a single global
lookup table for all the resources that are loaded at runtime. Here's what
the string for `cmrc.hpp` should contain:

```c++
#ifndef CMRC_CMRC_HPP_INCLUDED
#define CMRC_CMRC_HPP_INCLUDED

#include <string>
#include <map>
#include <mutex>

#define CMRC_INIT(libname) \
    do { \
        extern void cmrc_init_resources_##libname(); \
        cmrc_init_resources_##libname(); \
    } while (0)

namespace cmrc {

class resource {
    const char* _begin = nullptr;
    const char* _end = nullptr;
public:
    const char* begin() const { return _begin; }
    const char* end() const { return _end; }

    resource() = default;
    resource(const char* beg, const char* end) : _begin(beg), _end(end) {}
};

using resource_table = std::map<std::string, resource>;

namespace detail {

inline resource_table& table_instance() {
    static resource_table table;
    return table;
}

inline std::mutex& table_instance_mutex() {
    static std::mutex mut;
    return mut;
}

// We restrict access to the resource table through a mutex so that multiple
// threads can access it safely.
template <typename Func>
inline auto with_table(Func fn) -> decltype(fn(std::declval<resource_table&>())) {
    std::lock_guard<std::mutex> lk{ table_instance_mutex() };
    return fn(table_instance());
}

}

inline resource open(const char* fname) {
    return detail::with_table([fname](const resource_table& table) {
        auto iter = table.find(fname);
        if (iter == table.end()) {
            return resource {};
        }
        return iter->second;
    });
}

inline resource open(const std::string& fname) {
    return open(fname.data());
}

}

#endif // CMRC_CMRC_HPP_INCLUDED
```

It's pretty compact: There's not much to this file.

`with_table` protects against races on accessing the lookup table.
We use a regular mutex in this case, but using a `shared_mutex` would
definitely lead to better performance under load on `open`, since multiple
threads should be able to safely read from the table at any time. In my
example, I'm sticking to what is available to C++11, just because that's
what I had available when implementing my previous version.

We can now see the definition of that `CMRC_INIT` macro as well. We'll get to
that shortly. For now, we can leave it aside.

The `resource` class is beginning to be filled out. It honestly doesn't need
much. We can create it from two pointers, and the `begin()` and `end()` member
functions will return those pointers.

There are two considerable changes to `cmrc_add_resource_library`:

```cmake
function(cmrc_add_resource_library name)
    # Generate the identifier for the resource library's namespace
    string(MAKE_C_IDENTIFIER "${name}" libident)
    # Generate a library with the compiled in character arrays.
    set(cpp_content [=[
        #include <cmrc/cmrc.hpp>
        #include <map>

        namespace cmrc { namespace %{libident} {

        namespace res_chars {
        // These are the files which are available in this resource library
        $<JOIN:$<TARGET_PROPERTY:%{libname},CMRC_EXTERN_DECLS>,
        >
        }

        inline void load_resources() {
            // This initializes the list of resources and pointers to their data
            static std::once_flag flag;
            std::call_once(flag, [] {
                cmrc::detail::with_table([](resource_table& table) {
                    $<JOIN:$<TARGET_PROPERTY:%{libname},CMRC_TABLE_POPULATE>,
                    >
                });
            });
        }

        namespace {
            extern struct resource_initializer {
                resource_initializer() {
                    load_resources();
                }
            } dummy;
        }

        }}

        // The resource library initialization function. Intended to be called
        // before anyone intends to use any of the resource defined by this
        // resource library
        extern void cmrc_init_resources_%{libident}() {
            cmrc::%{libident}::load_resources();
        }
    ]=])
    get_filename_component(libdir "${CMAKE_CURRENT_BINARY_DIR}/${name}" ABSOLUTE)
    get_filename_component(libcpp "${libdir}/lib.cpp" ABSOLUTE)
    string(REPLACE "%{libname}" "${name}" cpp_content "${cpp_content}")
    string(REPLACE "%{libident}" "${libident}" cpp_content "${cpp_content}")
    string(REPLACE "\n        " "\n" cpp_content "${cpp_content}")
    file(GENERATE OUTPUT "${libcpp}" CONTENT "${cpp_content}")
    # Generate the actual static library. Each source file is just a single file
    # with a character array compiled in containing the contents of the
    # corresponding resource file.
    add_library(${name} STATIC ${libcpp})
    target_link_libraries(${name} PUBLIC cmrc::base)
    foreach(input IN LISTS ARGN)
        get_filename_component(abs_input "${input}" ABSOLUTE)
        # Generate a filename based on the input filename that we can put in
        # the intermediate directory.
        file(RELATIVE_PATH relpath "${CMAKE_CURRENT_SOURCE_DIR}" "${abs_input}")
        if(relpath MATCHES "^\\.\\.")
            # For now we just error on files that exist outside of the soure dir.
            message(SEND_ERROR "Cannot add file '${input}': File must be in a subdirectory of the source directory")
            continue()
        endif()
        get_filename_component(abspath "${libdir}/intermediate/${relpath}.cpp" ABSOLUTE)
        # Generate a symbol name for the file's character array
        _cm_encode_fpath(sym "${input}")
        # Generate the rule for the intermediate source file
        _cmrc_generate_intermediate_cpp(${libident} ${sym} "${abspath}" "${abs_input}")
        target_sources(${name} PRIVATE ${abspath})
        set_property(TARGET ${name} APPEND PROPERTY CMRC_EXTERN_DECLS
            "// Pointers to ${input}"
            "extern const char* const ${sym}_begin\;"
            "extern const char* const ${sym}_end\;"
            )
        set_property(TARGET ${name} APPEND PROPERTY CMRC_TABLE_POPULATE
            "table.emplace(\"${input}\", resource{res_chars::${sym}_begin, res_chars::${sym}_end})\;"
            )
    endforeach()
endfunction()
```

There are two important new things going on here:

1. The library has one main C++ file contained in the `cpp_content` string. We
   use CMake's bracketed argument syntax, so we must do the variable
   replacement ourselves in the lines following. This file is written to the
   path defined by the `libcpp` variable. The content of the file is written
   using `file(GENERATE)` rather than `file(WRITE)`. I'm planning on
   immediately doing some refactoring of this function which will make this
   decision much more clear soon. We use two target properties to generate the
   file: `CMRC_EXTERN_DECLS` and `CMRC_TABLE_POPULATE`, both of which are
   properties containing lines of C++ code which will declare the resource
   pointers and populate the lookup table, respectively.
2. As we loop through and generate each resource file's object file, we append
   to the aforementioned target properties the relevant lines of C++ code that
   will both declare the resource pointers and populate the lookup table for
   the file.

You'll also notice that the resource library's C++ file defines a function
beginning with `cmrc_init_resources_` and ending with the library's name. This
is the function that must be called to initialize the resource library at
runtime. This leads back to the `CMRC_INIT` macro. This macro must be called
with the name of the resource library: It calls this function to initialize the
resources therein. If this macro is not called, the resources will not be
available when we ask for them with `cmrc::open`. There are a few things I've
tried to get aroud this requirement:

1. Define a static object inside the library's C++ file whose constructor will
   initialize the resources
   - This is a good idea, and it works pretty well. But alas! The linker is not
     so keen on letting us succeed so easily: Since we reference no symbols
     from the resource library from `main()` neither directly nor indirectly,
     the linker never considers the object file for inclusion, and the object
     file is simply discarded: It assumes (Perhaps incorrectly?) that since
     nothing in the translation unit is referenced from another TU, that TU can
     have no effect on the rest of the program. This is an important linker
     optimization. There _are_ ways to get around it, such as using specific
     linker flags to tell it to never discard any symbols or to specifically
     link in a given object file. I myself was unable to find a way to get these
     to work in a cross-platform manner using CMake.
2. Define a global constructor using `__attribute__ ((constructor))`
   - This works as well, but only on GCC/Clang, and it still suffers all the
     same problems listed above for static object constructors.

If anyone can find a way to initialize the resource tables without the
`CMRC_INIT` macro, _please_ don't hesitate to contact me or leave an issue/PR on
the GitHub project.

## Step 5: Celebrate?

Well, the test is passing now: We see "Hello, world!" on the output, and CTest
recognizes it as a passing test too. Great! But I think we can make some
improvements...

## Step 6: Another Test

Let's check that we can read larger, non-text files. Here's a new test case that
reads in an image of a flower and tests it against a compiled in `flower.jpg`.

```c++
#include <cmrc/cmrc.hpp>

#include <algorithm>
#include <fstream>
#include <iterator>
#include <iostream>


int main(int argc, char** argv) {
    if (argc != 2) {
        std::cerr << "Invalid arguments passed to flower\n";
        return 2;
    }
    std::cout << "Reading flower from " << argv[1] << '\n';
    std::ifstream flower_fs{ argv[1], std::ios_base::binary };
    if (!flower_fs) {
        std::cerr << "Invalid filename passed to flower: " << argv[1] << '\n';
        return 2;
    }

    using iter = std::istreambuf_iterator<char>;
    const auto fs_size = std::distance(iter(flower_fs), iter());
    flower_fs.seekg(0);

    CMRC_INIT(hello);
    auto flower_rc = cmrc::open("flower.jpg");
    const auto rc_size = std::distance(flower_rc.begin(), flower_rc.end());
    if (rc_size != fs_size) {
        std::cerr << "Flower file sizes do not match: FS == " << fs_size << ", RC == " << rc_size << "\n";
        return 1;
    }
    if (!std::equal(flower_rc.begin(), flower_rc.end(), iter(flower_fs))) {
        std::cerr << "Flower file contents do not match\n";
        return 1;
    }
}
```

It's fairly simple as well: It reads in the path to the source image as a
command line argument, checks that the size of the file on disk and file in
memory are the same, then checks that they are byte-for-byte identical. Here's
the definition for this executable and test case in the `CMakeLists.txt`:

```cmake
add_executable(flower flower.cpp)
target_link_libraries(flower PRIVATE hello)
add_test(flower flower ${CMAKE_CURRENT_SOURCE_DIR}/flower.jpg)
```

We just need to add a `flower.jpg` file into the source directory. The
particular image is unimportant, but I've chosen a 2.5 MB image from WikiMedia
Commons in the public domain, [found here](https://upload.wikimedia.org/wikipedia/commons/0/0c/White_and_yellow_flower.JPG).

Running CTest again, the `flower` test will fail because the resource size is
zero! That's because we haven't added it to the `hello` resource library.

I _could_ just add it to the list of resources to our original
`cmrc_add_resource_library` above, _or_ I could take the opportunity to refactor
out some of the ugly from `cmrc_add_resource_library`. I did the latter:

```cmake
function(cmrc_add_resources name)
    get_target_property(is_reslib ${name} CMRC_IS_RESOURCE_LIBRARY)
    if(NOT TARGET ${name} OR NOT is_reslib)
        message(SEND_ERROR "cmrc_add_resources called on target '${name}' which is not an existing resource library")
        return()
    endif()

    # Generate the identifier for the resource library's namespace
    string(MAKE_C_IDENTIFIER "${name}" libident)

    foreach(input IN LISTS ARGN)
        get_filename_component(abs_input "${input}" ABSOLUTE)
        # Generate a filename based on the input filename that we can put in
        # the intermediate directory.
        file(RELATIVE_PATH relpath "${CMAKE_CURRENT_SOURCE_DIR}" "${abs_input}")
        if(relpath MATCHES "^\\.\\.")
            # For now we just error on files that exist outside of the soure dir.
            message(SEND_ERROR "Cannot add file '${input}': File must be in a subdirectory of the source directory")
            continue()
        endif()
        get_filename_component(abspath "${libdir}/intermediate/${relpath}.cpp" ABSOLUTE)
        # Generate a symbol name for the file's character array
        _cm_encode_fpath(sym "${relpath}")
        # Generate the rule for the intermediate source file
        _cmrc_generate_intermediate_cpp(${libident} ${sym} "${abspath}" "${abs_input}")
        target_sources(${name} PRIVATE ${abspath})
        set_property(TARGET ${name} APPEND PROPERTY CMRC_EXTERN_DECLS
            "// Pointers to ${input}"
            "extern const char* const ${sym}_begin\;"
            "extern const char* const ${sym}_end\;"
            )
        set_property(TARGET ${name} APPEND PROPERTY CMRC_TABLE_POPULATE
            "table.emplace(\"${relpath}\", resource{res_chars::${sym}_begin, res_chars::${sym}_end})\;"
            )
    endforeach()
endfunction()
```

Now, any time we wish to add additional resources to a resource library, we can
call `cmrc_add_resources` instead of providing them all upfront. I have further
plans for this function later.

Additionally, our `cmrc_add_resource_library` function now is much simpler, with
only four lines at the tail end:

```cmake
    add_library(${name} STATIC ${libcpp})
    target_link_libraries(${name} PUBLIC cmrc::base)
    set_property(TARGET ${name} PROPERTY CMRC_IS_RESOURCE_LIBRARY TRUE)
    cmrc_add_resources(${name} ${ARGN})
endfunction()
```

We simply make use of our factored out functionality. Convenient and easy. Now,
to add our `flower.jpg` to our resource library we simply add one line to our
`CMakeLists.txt`:

```cmake
cmrc_add_resources(hello flower.jpg)
```

And now our `hello` resource library now contains the `flower.jpg`, and our
tests are all green!

# Step 7: Moar Features

Qt's resource system has some pretty snazzy features, like the ability to
rewrite file path prefixes. Wouldn't that be great? I'd like that too.

```cmake

function(cmrc_add_resources name)
    get_target_property(is_reslib ${name} CMRC_IS_RESOURCE_LIBRARY)
    if(NOT TARGET ${name} OR NOT is_reslib)
        message(SEND_ERROR "cmrc_add_resources called on target '${name}' which is not an existing resource library")
        return()
    endif()

    set(options)
    set(args WHENCE PREFIX)
    set(list_args)
    cmake_parse_arguments(PARSE_ARGV 1 ARG "${options}" "${args}" "${list_args}")

    if(NOT ARG_WHENCE)
        set(ARG_WHENCE ${CMAKE_CURRENT_SOURCE_DIR})
    endif()

    # Generate the identifier for the resource library's namespace
    string(MAKE_C_IDENTIFIER "${name}" libident)

    foreach(input IN LISTS ARG_UNPARSED_ARGUMENTS)
        get_filename_component(abs_input "${input}" ABSOLUTE)
        # Generate a filename based on the input filename that we can put in
        # the intermediate directory.
        file(RELATIVE_PATH relpath "${ARG_WHENCE}" "${abs_input}")
        if(relpath MATCHES "^\\.\\.")
            # For now we just error on files that exist outside of the soure dir.
            message(SEND_ERROR "Cannot add file '${input}': File must be in a subdirectory of ${ARG_WHENCE}")
            continue()
        endif()
        get_filename_component(abspath "${libdir}/intermediate/${relpath}.cpp" ABSOLUTE)
        # Generate a symbol name relpath the file's character array
        _cm_encode_fpath(sym "${relpath}")
        # Generate the rule for the intermediate source file
        _cmrc_generate_intermediate_cpp(${libident} ${sym} "${abspath}" "${abs_input}")
        target_sources(${name} PRIVATE ${abspath})
        set_property(TARGET ${name} APPEND PROPERTY CMRC_EXTERN_DECLS
            "// Pointers to ${input}"
            "extern const char* const ${sym}_begin\;"
            "extern const char* const ${sym}_end\;"
            )
        if(ARG_PREFIX AND NOT ARG_PREFIX MATCHES "/$")
            set(ARG_PREFIX "${ARG_PREFIX}/")
        endif()
        set_property(TARGET ${name} APPEND PROPERTY CMRC_TABLE_POPULATE
            "// Table entry for ${input}"
            "table.emplace(\"${ARG_PREFIX}${relpath}\", resource{res_chars::${sym}_begin, res_chars::${sym}_end})\;"
            )
    endforeach()
endfunction()
```

The primary change is that we now have two keyword arguments to
`cmrc_add_resources`: `WHENCE` and `PREFIX`. `WHENCE` determines the root
directory where each resource is read from. This handles the case where we want
to add resource files which exist outside of the source directory: We simply
need to tell `CMakeRC` how to rewrite the paths into the resource library.
`PREFIX` lets us add a directory-like prefix to the path at runtime.