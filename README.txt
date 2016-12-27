//===----------------------------------------------------------------------===//
// C Language Family Front-end
//===----------------------------------------------------------------------===//

Welcome to Clang.  This is a compiler front-end for the C family of languages
(C, C++, Objective-C, and Objective-C++) which is built as part of the LLVM
compiler infrastructure project.

Unlike many other compiler frontends, Clang is useful for a number of things
beyond just compiling code: we intend for Clang to be host to a number of
different source-level tools.  One example of this is the Clang Static Analyzer.

If you're interested in more (including how to build Clang) it is best to read
the relevant web sites.  Here are some pointers:

Information on Clang:              http://clang.llvm.org/
Building and using Clang:          http://clang.llvm.org/get_started.html
Clang Static Analyzer:             http://clang-analyzer.llvm.org/
Information on the LLVM project:   http://llvm.org/

If you have questions or comments about Clang, a great place to discuss them is
on the Clang development mailing list:
  http://lists.llvm.org/mailman/listinfo/cfe-dev

If you find a bug in Clang, please file it in the LLVM bug tracker:
  http://llvm.org/bugs/


# Reflection

This fork of the Clang compiler provides an experimental implementation of 
static reflection for C++. This is a work in progress; the design and 
implementation are known to be incomplete.

Note that this fork tracks the GitHub mirrors of LLVM and Clang. I try to sync 
with those repositories about once per week. This means that this repository
will have the same features (and bugs) as trunk.


## Getting started

The getting started instructions for this fork of the compiler are similar to
those of the main trunk. The only difference is that, when you clone this
repository, you need to make sure it is named `clang` and not `clang-reflect`.

Here are the set of commands I use to set up the build environment.

```bash
git clone https://github.com/llvm-mirror/llvm
cd llvm/tools
git clone https://github.com/asutton/clang-reflect clang
cd ../projects
git clone https://github.com/llvm-mirror/libcxx
```

If you forget to provide the name `clang` in the second step, you can simply
rename the directory later:

```
cd llvm/tools
mv clang-reflect clang
```

## Building LLVM + Clang

Follow the usual instructions for building LLVM + Clang. 

However, I would make the following recommendations. First, unless you plan to
hack on the implementation, make sure you're building a release compiler.
Otherwise, compilation will be *very slow*.

Second, make sure you're installation path is configured before you start
compiling the program. If you decide to change the installation path after
building LLVM + Clang, you're going to have to rebuild the entire compiler.
That can take a while.

You can configure these from the command line when you run `cmake`. Here's
how I do that:

```bash
cd llvm
mkdir build
cd build
cmake -DCMAKE_BUILD_TYPE=Release -DCMAKE_INSTALL_PREFIX=$HOME/opt
```

This will configure a native release-mode compiler installed in `$HOME/opt`.

Building and installing is straightforward.

```bash
make
make install
```

Note that `make` may take a while. It goes faster using `make -j4` (or some
other number relative to your system).

## Library support

Reflection requires library support. This is provided by a separate, header
only library `libcpp3k`, which can be found in the `clang/tools` directory.
The header(s) are installed when you install LLVM + Clang in the configured
location. For example, with the configuration above, these files are installed
at `$HOME/opt/include`.

We've opted to make library support separate from the C++ standard library for
the time being. This allows the library to be used by other compiler 
implementations experimenting with the same approach to reflection.


## Using the compiler

The compiler will be installed in the given location. If this is not on path
(which I strongly recommend against), then you will need to invoke the compiler
using its full path. A better solution is to define the `CXX` environment
variable to refer to that compiler.

```bash
export CXX=$HOME/opt/bin/clang++
$CXX -version  # print version info
```

This has the added benefit of working with CMake for your own projects. That is,
CMake will detect the C++ compiler as the one specified by the `$CXX`.

To enable reflection, you will need to compile with the following flags:

```bash
$CXX -std=c++1z -Xclang -freflection
```

The first enables the latest C++ language features. The next two flags
`-Xclang -freflection` enable C++ reflection.

Depending on your installation path, you may also need to add include 
directories to the search path for libcpp3k.

```bash
$CXX -std=c++1z -Xclang -freflection -I$HOME/opt/include
```

Remember that libcpp3k is installed with (and alongside) this version of the
compiler, so that wherever Clang is installed, that's where libcpp3k is also
installed.

## Reflection examples

A number of reflection examples can be found in the `clang/tools/libcpp3k`
directory under `examples`. These can be built separately from the compiler
using CMake.

If everything has been compiled and installed correctly, then you can build
the examples like this:

```bash
cd llvm/tools/clang/tools/libcpp3k
mkdir build
cd build
export CXX=$HOME/opt/bin/clang++
cmake ..
make
```

These examples are largely taken from the WG21 proposal p0385r1
(http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2016/p0385r1.pdf).

## Errata and bugs

If you find compiler bugs, or things don't work the way you expect, please file
an issue for the issue tracker in this repository.
