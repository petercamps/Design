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

This section describes how SKIRT locates its input files once HDF5 support is enabled.

### File types

SKIRT currently supports several input file types, each suited to a different kind of
source data:

- **Column text** — a plain text table with one row per item (wavelength, particle,
  cell, …) and one column per property (luminosity, position, density, …).
  It is widely used for importing particle- or cell-based spatial distributions, and for
  specifying spectral energy distributions, mesh definitions, material properties and more.
- **AMR text** — describes a hierarchical, adaptively refined Cartesian grid together with
  its per-cell data. It is used for importing adaptive-mesh-refinement simulation output
  (`AdaptiveMeshSnapshot`).
- **Stored tables (.stab)** — a SKIRT-specific binary multi-axis table format heavily used
  for storing SKIRT's built-in resources. On the input side, it is sometimes used for defining
  multi-axis tables such as SED family templates (`FileSSPSEDFamily`) or
  polarized Stokes-vector tables (`FilePolarizedPointSource`).
- **FITS** — a binary image format widely used in astronomy. It is used to import two- or
  three-dimensional geometries (`ReadFitsGeometry` and `ReadFits3DGeometry`).

The [Data model](03-data-model.md) chapter explains how each of these maps to HDF5
storage mechanisms and how to generate those from Python.

**Incompatibility notes**:

- Support for the `scol` format, a less-frequently-used SKIRT-specific binary alternative
  to regular text column files, is dropped because the HDF5 alternative accomplishes
  the same goals in an industry-standard way.
- Support for saving and re-loading spatial grid tree topology to and from text files is
  dropped and replaced by a more general HDF5 storage mechanism.

### HDF5 concepts

An HDF5 file is organized like a small filesystem. It has a root group, which can contain
further groups as well as datasets — the actual blocks of stored data. A dataset is addressed
by a path within the file, built from the names of the groups leading up to it, similar to a
file path.

### Command-line syntax

The SKIRT command-line option that specifies the input location, `-i`, now accepts up to
three components combined into a single argument:

```
-i <dir>/<hdf>:<anchor>
```

- `<dir>` — the input directory, exactly as before.
- `<hdf>` — optional: the name of an HDF5 file located inside `<dir>`. Both the `.hdf5` and
  `.hf` extensions are recognized.
- `<anchor>` — optional, and only meaningful together with `<hdf>`: a path within the HDF5
  file, used as described below.

Only `<dir>` is required.

### How input files are located

If only `<dir>` is given, SKIRT behaves exactly as before: every input file it needs is
expected to be a plain file inside `<dir>`. (The ski file itself is not an input file in this
sense — its full path is given directly as a separate command-line argument, independent of
`-i`.)

If `<hdf>` is also given, SKIRT looks for each input file in two steps: first as a plain file
in `<dir>`, exactly as before; if that file is not found, SKIRT looks inside `<hdf>` instead,
for a dataset whose name matches the input file name.

If `<anchor>` is given as well, it is prefixed to the input file name before that lookup, so
SKIRT looks for the dataset `<anchor>/<name>` rather than `<name>`. This makes it possible to
store the input for several simulations inside a single HDF5 file, each under its own anchor,
and select the right one per run through the `-i` option.

### Examples

Assuming a ski file `mysim.ski`, an input directory `in`, and an HDF5 file `data.hdf5`:

- `skirt -i in mysim.ski` — unchanged behavior: all input files are read from plain files in
  `in`.
- `skirt -i in/data.hdf5 mysim.ski` — input files are sought as plain files in `in` first,
  then as datasets in `in/data.hdf5`.
- `skirt -i in/data.hdf5:run01 mysim.ski` — as above, but datasets are looked up under the
  `run01` group, i.e. as `run01/<name>`.
- `skirt -i in/data.hdf5:campaign7/galaxy042 mysim.ski` — the anchor can itself be a
  multi-level path, here selecting the input for one galaxy out of many stored in the same
  file.
- `skirt -i ./data.hdf5 mysim.ski` — edge case: the HDF5 file sits directly in the current
  directory (`<dir>` is `.`).
- `skirt -i in/data.hf mysim.ski` — edge case: the alternative `.hf` extension.

## Output

This section describes how SKIRT writes its output files once HDF5 support is enabled.

### File types

SKIRT currently writes the following kinds of output files:

- **Text column files** — employed for all single-axis tables: SEDs, per-cell and
  per-position probe output, and instrument statistics tables (`FluxRecorder`).
  Uses the same header-comment convention as the input side (`# column N: ... (unit)`).
- **FITS files** — employed for instrument frames and data cubes,
  and for planar cuts or projections produced by probes.
- **Spatial grid plot files** — a polyline format where
  each line holds 2 or 3 coordinates meaning "draw a line to this point", and a blank line
  starts a new, disconnected segment. Every spatial grid writes one of these to plot its own
  cell geometry.
- **Unstructured text files** - the `convergence.dat` plain text file intended for human
  consumption, written by `ConvergenceInfoProbe`.
- **XML file** - the `parameters.xml` file, a reformatted version of the ski file
  governing the simulation.
- **Log file** - The `log.txt` plain text file with progress, warning and error messages.
  Because this file grows as the simulation runs, and users should be able to view the
  simulation's progress in real-time, the log file is always written as a regular file
  and then stored in the HDF file after the simulation has ended.

The [Data model](03-data-model.md) chapter explains how each of these maps to datasets in
HDF5, and how to read them from Python.

**Incompatibility note**:

- Support for saving and re-loading spatial grid tree topology to and from text files is
  dropped and replaced by a more general HDF5 storage mechanism.

### Command-line syntax

The command-line option that specifies the output location, `-o`, follows the same
`<dir>/<hdf>:<anchor>` format as `-i`, with the same meaning for each component (see
Command-line syntax under Input, above).

### How output files are written

If only `<dir>` is given, SKIRT behaves exactly as before: every output file is written as a
plain file inside `<dir>`.

If `<hdf>` is also given, SKIRT writes into that HDF5 file instead. The file as a whole is
never replaced: if it already exists, SKIRT opens it and adds to it. Each individual output
is written as a new dataset, or replaces an existing one if a dataset with the same name is
already present; every other dataset in the file is left untouched. The dataset name matches
the plain output file that would otherwise have been produced, prefixed with `<anchor>` if
given, exactly as on the input side.

This makes it possible to collect the output of several simulations in a single HDF5 file,
each under its own anchor, or to store both the input and the output of a single simulation
together in one file.

## Checkpointing

TODO: describe the checkpointing feature.
