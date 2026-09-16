# Implementation

> Status: draft

## HDF5 target

### Build option

Whether HDF5 support is available is decided at compile-time, but 
client code links into the HDF5 target's API unconditionally, everywhere in SKIRT, rather
than being scattered with `#ifdef BUILD_HDF5` guards.

Concretely, this follows the same pattern already used for SKIRT's MPI support (see
`SKIRT/mpi/CMakeLists.txt` and `ProcessManager.cpp`): a new `hdf5` CMake target, always built as a
static library regardless of `BUILD_HDF5`, that contains every line of code in SKIRT that
touches the HDF5 C API directly. No other target ever includes an HDF5 header or links
against the HDF5 library — code elsewhere in SKIRT depends only on this target's own C++
API, never on HDF5 itself. `BUILD_HDF5` controls only how that one target's own sources are
compiled: when enabled, `find_package(HDF5 REQUIRED)` locates the library, the target links
against it, and a `BUILD_HDF5` preprocessor symbol — defined only for this target's own
sources, exactly as `mpi` does for `BUILD_WITH_MPI` — selects the real implementation inside
each `#ifdef` block; when disabled, the target still builds, taking the `#else` branch of
each of those blocks instead, and links against nothing HDF5-related at all, so a
`BUILD_HDF5`-disabled SKIRT binary has no HDF5 footprint whatsoever, exactly as
Prerequisites in Features requires.

The stub implementation throws a fatal error the moment any real work
is attempted. As an exception, the `H5Lib` class (see below) offers a static `available()`
query that simply reports whether this build was compiled with `BUILD_HDF5`.
Wherever a ski file setting or command-line option would require
HDF5 — anywhere `-i`/`-o`/`-c` names an HDF5 file, or a property such as
`CheckpointTreePolicy::filename` targets one — setup-time validation checks
`H5Lib::available()` first and reports a clear, immediate error if HDF5 support is missing,
rather than letting execution reach a stub's fatal error later.

### Thread safety

All calls into the HDF5 C API are protected by a single mutex internal to this target: the
library cannot be assumed to be thread-safe on its own, since that depends on how the
installed HDF5 happens to have been built, which is outside SKIRT's control.

### C++ API

The C++ API offered by the HDF5 target is a small set of classes representing the concepts
defined in the Data model chapter — suite, bundle, checkpoint, dataset — plus the factory
methods that construct them. Each object acquires its underlying HDF5 resource on
construction and releases it in its destructor. Objects can be moved but not copied.

None of the objects represents an open file: every factory method
resolves a full `<filepath>:<suite>/...` path directly, and file-handle management is this
target's own internal concern. Every concept splits into a read-only class (suffixed `R`)
and a write-only class (suffixed `W`), except suite, which has no writable form.
Writing always targets a specific bundle or checkpoint directly, never
"a suite" as a whole.

**`H5Lib`** is not instantiated; it offers only static factory methods, the API's sole
file-opening entry points — nothing else in this API ever names a file path directly. Every
method below that takes a `<name>` argument, rather than a full `<filepath>:<suite>/...`
path, resolves that name relative to the object the method is called on.

- `bool available()` — true if this build was compiled with `BUILD_HDF5`.
- `H5SuiteR openSuite(<filepath>:<suite>)` — opens a suite for reading.
- `H5BundleR openBundle(<filepath>:<suite>/<bundle>)` — opens a bundle for reading directly,
  without going through its suite first.
- `H5CheckpointR openCheckpoint(<filepath>:<suite>/<checkpoint>)` — opens a checkpoint for
  reading directly, without going through its suite first.
- `H5BundleW createBundle(<filepath>:<suite>/<bundle>)` — creates a bundle for writing,
  erasing any existing bundle of that name first, per How output files are written in
  Features.
- `H5CheckpointW createCheckpoint(<filepath>:<suite>/<checkpoint>)` — creates a checkpoint
  for writing, erasing any existing checkpoint of that name first.

**Reading.**

**`H5SuiteR`**

- `vector<string> getBundleNames()` — the names of every bundle directly in this suite,
  excluding checkpoints.
- `vector<string> getCheckpointNames()` — the names of every checkpoint directly in this
  suite.
- `H5BundleR openBundle(<name>)` — opens one of this suite's bundles for reading.
- `H5CheckpointR openCheckpoint(<name>)` — opens one of this suite's checkpoints for reading.

**`H5BundleR`**

- `bool hasAttribute(<name>)`
- `getAttribute(<name>)` — one overload per supported attribute type (boolean, 32-bit
  integer, 64-bit float, string).
- `bool hasDataset(<name>)`
- `H5DatasetR getDataset(<name>)` — opens one of this bundle's datasets for reading.

**`H5CheckpointR`**

- `bool hasAttribute(<name>)`
- `getAttribute(<name>)` — as `H5BundleR::getAttribute()`, above.
- `bool hasBundle<Name>()` — `<Name>` is one of the fixed sub-bundle names (Spatial grid,
  Medium state, Radiation field, Recorded fluxes), never an arbitrary one.
- `H5BundleR getBundle<Name>()` — opens one of the checkpoint's fixed-name sub-bundles for
  reading.

**`H5DatasetR`**

- `bool hasAttribute(<name>)`
- `getAttribute(<name>)` — as `H5BundleR::getAttribute()`, above.
- `read()` — one overload per supported dataset content type (boolean, 32-bit integer,
  64-bit float), returning the dataset's full contents at whatever shape it was created
  with.

**Writing.**

**`H5BundleW`**

- `setAttribute(<name>, value)` — one overload per supported attribute type, as for
  `H5BundleR::getAttribute()`, above.
- `H5DatasetW createDataset(<name>)` — creates one of this bundle's datasets for writing,
  with the shape and on-disk type the Data model chapter specifies for it.

**`H5CheckpointW`**

- `setAttribute(<name>, value)` — as `H5BundleW::setAttribute()`, above.
- `H5BundleW createBundle<Name>()` — creates one of the checkpoint's fixed-name sub-bundles
  for writing.
- `linkBundle<Name>ToCheckpoint(<suite>/<checkpoint>)` — hard-links this checkpoint's `<Name>`
  sub-bundle to the same-named, already-written sub-bundle of the checkpoint at
  `<suite>/<checkpoint>`, instead of writing a new copy of it — what Checkpoint probe
  behavior in Features relies on to avoid re-storing unchanged data across checkpoints.

**`H5DatasetW`**

- `setAttribute(<name>, value)` — as `H5BundleW::setAttribute()`, above.
- `write()` — as `H5DatasetR::read()`, above, plus one further overload for `text`, since
  that type is output-only.

A bundle created for writing attaches the three standard attributes (`producer`, `format`,
`created`) automatically; one opened for reading exposes whatever attributes it actually has.
A checkpoint handles the checkpoint-specific attributes (`when`, `iteration`, the ski
parameters from Iterating across simulations) and hands out the up to four sub-bundles
(Spatial grid, Medium state, Radiation field, Recorded fluxes) by name.
The `hasAttribute()` function is needed
because in some places attribute absence is meaningful and distinct from an empty value.

### Object lifetime

Any object this API hands out remains fully valid after the object it came from has been
destroyed. For example, an `H5Dataset` does not depend on its `H5Bundle` still being alive.
The HDF5 C library reference-counts its own resources internally, so a file
stays genuinely open, at the library level, for as long as any handle into it remains open,
and only actually closes once the last handle does. It is thus allowed to let an `H5Bundle`
go out of scope as soon as the single `H5Dataset` actually needed has been obtained from it.

This is not, however, a license to lose track of things: every object this API hands out must
still eventually be destroyed by the client, and until that has happened for everything ever
opened against a given file, that file remains genuinely open. This target places that
responsibility on the client rather than tracking open descendants itself. The concrete case
is MPI: when input and output are pointed at the same HDF5 file, every rank must have
destroyed all objects obtained while reading it before the root process opens that same file
for writing, since reading and writing must not overlap in time (see Concurrency under
Output in Features).

### Supported data types

Attributes (on `H5Bundle`, `H5Checkpoint`, and `H5Dataset`) and dataset contents share three
data types: boolean, 32-bit integer, and 64-bit float. Attributes additionally support
`string`, passed as a `std::string` in UTF-8 and stored as a fixed 128-byte field —
comfortably larger than any attribute value in the Data model chapter, including the free-form
`description` attributes. Datasets additionally support `text`, for output only, passed in as a
`std::istream` in UTF-8 and stored as a variable-length string on disk — Unstructured text
file's `text` dataset is the only case that needs it, and unlike an attribute, its content
can genuinely be arbitrarily long.

No conversion happens when writing attributes or data, so client code is responsible for writing
exactly the data type each table in the Data model chapter specifies. Reading returns
the data type the caller requests and leans on HDF5's own conversion machinery for however the
value is actually stored on disk, as described under Bundles in the Data model chapter.

## Name resolution

Parsing and combining the various segments of file and bundle names (directory, prefix,
filename, suite, bundle) and the "try a plain file first, then fall back to a bundle" logic
are deliberately outside the API of the HDF5 target described in the previous section. To
handle this, the `FilePaths` class's API is adjusted as follows.

**Setup**

- `setInputPath(string value)` — the value of the `-i` option: a plain directory path, or a
  directory plus an HDF5 file and suite.
- `setOutputPath(string value)` — the value of the `-o` option, in the same form.
- `setCheckpointPath(string value)` — the value of the `-c` option: always a directory plus
  an HDF5 file and suite, since a checkpoint has no plain-file form.
- `setOutputPrefix(string value)` — the ski filename without its extension; unchanged.

**Trivial queries**

- `inputPath()` and `outputPath()` are removed; nothing needs these two queries.
- `outputPrefix()` is unchanged, still returning whatever `setOutputPrefix()` set.

**Name-constructing queries**

- `input(string name)` and `output(string name)` each return a pair of strings: a plain-file
  form and an HDF5 form.
- `checkpoint(string name)` returns a single string, since a checkpoint has no plain-file
  form.

Given the value of a ski file's `filename` attribute, or a programmatically fabricated
filename or bundle name, these three functions combine that name with the setup values taken
from the command line to construct fully resolved locations. For `input()`/`output()`, the
first string of the pair is the absolute canonical path for a plain file corresponding to
`name`, and the second is an HDF5 file path plus a resolved bundle path corresponding to
`name`. Either string is empty if there is no such correspondence, independent of whether
that file or bundle actually exists — these functions do no I/O. The caller is
expected to try the plain file first, and fall back to the HDF5 bundle only if that fails.

For example, with `-i /mydisk/mydata.hdf5:mysim` on the command line, `input("import.txt")`
returns `/mydisk/import.txt` as the first string and `/mydisk/mydata.hdf5:mysim/import.txt`
as the second.

## Stored table bundle

The data in a stored table input bundle cannot be memory mapped (as for a regular .stab file)
for several reasons:

- The quantity values are ordered differently (interleaved in .stab, per dataset in bundle).
- The bundle datasets may be compressed and/or chunked.
- If the same HDF file is used for input and output, memory mapping a portion of the file
  is unsafe because writing to the file might relocate its contents.

The StoredTable constructor for a bundle must therefore copy the data into newly allocated
memory, and the destructor must deallocate that memory.

## Notes to revisit

- `CheckpointTreePolicy` must match each live node to its counterpart in the recorded
  topology — parsed into a real tree at setup via its `parent_id` links — by following the
  same parent/child-slot path from the root, rather than assuming live construction visits
  nodes in the same order the checkpoint was originally written in; once a live node falls
  past what was recorded, it correctly answers no from there down.
