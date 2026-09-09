# Features

> Status: draft

## Prerequisites

SKIRT uses the official C HDF5 library — not the C++ wrapper layer that HDF5 also
provides. HDF5 support is optional: the SKIRT build procedure offers a CMake option called
`BUILD_HDF5`, following the same pattern already used for SKIRT's MPI support. When enabled,
this option locates and links the HDF5 library and compiles in the corresponding
input/output/checkpoint code; when disabled (the default), SKIRT builds exactly as it does
today, with no dependency on HDF5.

Before enabling `BUILD_HDF5`, the HDF5 C library must be installed on the build machine.

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

## Input

TODO: describe the HDF5-based input feature.

## Output

TODO: describe the HDF5-based output feature.

## Checkpointing

TODO: describe the checkpointing feature.
