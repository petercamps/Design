# System requirements

## Compiler

### C++17

SKIRT 10 is proposed to move from C++14 to **C++17**. Every compiler already used to build
SKIRT — GCC, Clang, MSVC, Intel, and MinGW — has implemented C++17 in full for years, so
this is a change of language standard, not a change of minimum compiler version. A few
C++17 features are worth calling out because they let existing code say the same thing
more concisely.

**`if` with an initializer.** A pattern like this one, from `AtomUtils.cpp`:

```cpp
auto it = atomMap.find(element);
if (it == atomMap.end())
    throw FATALERROR("No element found with name " + element + " for ion: " + ion);
Z = it->second;
```

becomes:

```cpp
if (auto it = atomMap.find(element); it != atomMap.end())
    Z = it->second;
else
    throw FATALERROR("No element found with name " + element + " for ion: " + ion);
```

`it` no longer leaks into the enclosing scope once it is only needed to decide between
the two branches.

**Structured bindings.** A pattern like this one, from `FilePaths.cpp`:

```cpp
for (const auto& pair : _resourcePaths)
{
    const string& resource = pair.first;
    // ...
```

becomes:

```cpp
for (const auto& [resource, path] : _resourcePaths)
{
    // ...
```

removing the manual `.first`/`.second` aliasing entirely.

**`std::byte`.** A type that represents nothing but a chunk of raw memory, with only
bitwise operators (`|`, `&`, `^`, `<<`, `>>`) defined on it. Unlike `char` or `unsigned
char`, it supports no arithmetic and no implicit conversion to or from an integer, so
code handling raw bytes cannot accidentally be misread as handling small numbers or text.

### Why not C++20

C++20 is often described as close to a new language: concepts, ranges, coroutines, and
modules each bring a genuinely different way of writing code, on top of smaller
additions like designated initializers and the `<=>` comparison operator. None of this
is needed for SKIRT's own purposes, and concepts and ranges in particular would be a
real adjustment for the project's largely occasional C++ programmers — physicists and
astronomers writing simulation code, not full-time software engineers keeping up with
each new standard. C++17 gets the concrete, everyday improvements above without that
cost.

## Build tools

**CMake.** The required version is proposed to increase from the current
`3.2.2...3.5` to `3.18...3.30`, following the same two-number range syntax already used
today: the first number is the actual minimum, the second caps which version's policies
get applied even when a newer CMake is actually installed, so that adopting a future
CMake release cannot silently change build behavior. `3.18` is a reasonably modern
floor, comfortably available across current package managers and HPC module systems by
the time SKIRT 10 is released.

**Link-time optimization.** SKIRT gains a `BUILD_WITH_LTO` option (proposed default: on),
letting the compiler optimize across `.cpp` file boundaries at link time. Measured with
Apple Clang 17, run time is roughly 2-4% lower, and the link step takes about 5 s instead
of 1 s. Not yet tested with other compilers/operating systems. CMake's
`INTERPROCEDURAL_OPTIMIZATION` target property drives this feature, applied only to
`Release` builds, and only if the compiler and linker actually support it.

## HDF5

HDF5 support is optional: the SKIRT build procedure offers a CMake option called
`BUILD_HDF5`. If enabled, the build procedure expects the official C HDF5 library
— not the C++ wrapper layer that HDF5 also provides — to be installed on the system.

### Obtaining and installing HDF5

The library, its source code, and prebuilt binaries for Windows, macOS, and Linux are
distributed by [The HDF Group](https://github.com/HDFGroup/hdf5/releases), the organization
that develops and maintains HDF5. Full installation documentation is available from the
[HDF5 documentation](https://support.hdfgroup.org/documentation/) site; the summary below
should be enough for a first-time installation.

**macOS.** The simplest route is [Homebrew](https://brew.sh):

```bash
brew install hdf5
```

**Ubuntu and other Debian-based Linux.** Install the development package, which includes the
headers needed to compile against the library:

```bash
sudo apt install libhdf5-dev
```

**Other Unix-like systems (Fedora, RHEL, and derivatives).** Use the equivalent `dnf`/`yum`
package:

```bash
sudo dnf install hdf5-devel
```

For other Unix flavors, check the distribution's package manager first; if no package is
available, HDF5 can be built from source with CMake, following the instructions linked above.

**Windows.** The most convenient option for a CMake-based project such as SKIRT is
[vcpkg](https://vcpkg.io):

```bash
vcpkg install hdf5
```

Prebuilt Windows installers are also available from the releases page linked above.

### HPC systems

On shared HPC clusters, HDF5 is usually already installed as an environment module rather
than needing to be built by hand. A typical example (exact module names and versions vary by
site — check `module avail hdf5` on the cluster in question):

```bash
module load hdf5
```

