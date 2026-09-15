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
is attempted. As an exception, the `H5File` class (see below) offers a static `available()`
query that simply reports whether this build was compiled with `BUILD_HDF5`.
Wherever a ski file setting or command-line option would require
HDF5 — anywhere `-i`/`-o`/`-c` names an HDF5 file, or a property such as
`CheckpointTreePolicy::filename` targets one — setup-time validation checks
`H5File::available()` first and reports a clear, immediate error if HDF5 support is missing,
rather than letting execution reach a stub's fatal error later.

### Thread safety

All calls into the HDF5 C API are protected by a single mutex internal to this target: the
library cannot be assumed to be thread-safe on its own, since that depends on how the
installed HDF5 happens to have been built, which is outside SKIRT's control.

### C++ API

The C++ API offered by this target is a small set of RAII classes — each one acquires its underlying HDF5
resource on construction and releases it in its destructor. Together these classes offer everything
a client needs to implement the bundles described in the Data model chapter, without that
client ever seeing an HDF5 handle.

**`H5File`** wraps one open HDF5 file — opened for reading, or for writing (create if
missing, otherwise open and add to, per How output files are written in Features). Its only
job is handing out `H5Bundle` and `H5Checkpoint` objects by anchored path.

**`H5Bundle`** wraps one HDF5 group. A bundle created for writing stamps the three standard
attributes (`producer`, `format`, `created`) automatically; one opened for reading exposes
whatever attributes and datasets it actually has. Beyond that, `H5Bundle` offers
`setAttribute`/`getAttribute`, `createDataset`/`openDataset` (returning an `H5Dataset`),
`hasDataset()` and `datasetNames()` (returning a `vector<string>`) to discover what a
bundle with a data-dependent schema actually
contains, and a `linkTo()` operation that hard-links this bundle's group to an
already-written one instead of copying it — what Checkpoint probe behavior in Features
relies on to avoid re-storing unchanged data across checkpoints.

**`H5Checkpoint`** holds an `H5Bundle` for a checkpoint's own group — composition, not
inheritance, since its API differs in kind from a plain bundle's: instead of arbitrary
datasets and attributes, it sets the checkpoint-specific ones (`when`, `iteration`, the ski
parameters from Iterating across simulations) and hands out the up to four sub-bundles
(Spatial grid, Medium state, Radiation field, Recorded fluxes) by name, plus a
`hasBundle()` check for whether a given one is actually present. Unlike `H5Bundle`'s
datasets, the set of possible sub-bundles is fixed, so no enumeration method is needed.

**`H5Dataset`** wraps one HDF5 dataset; its dataspace and datatype are internal details a
client never sees. Created with a shape — of any rank, including the scalar case
Unstructured text file's `text` dataset needs — and a type, which becomes the dataset's
on-disk type. `write()` fills the whole dataset in one shot from a buffer already in
that same type. No conversion happens on write, so client code
is responsible for writing exactly the type each table in the Data model
chapter specifies. `read()` takes
whatever type the caller wants and leans on HDF5's own conversion machinery for however the
value is actually stored on disk, exactly as described under Bundles in the Data model
chapter. `H5Dataset` also offers `setAttribute`/`getAttribute`, for the many
per-dataset attributes documented throughout that chapter, plus `hasAttribute()` — needed
because in some places attribute absence is meaningful and distinct from an empty value.

### Object lifetime

Any object this API hands out remains fully valid after the object it came from has been
destroyed — an `H5Bundle` or `H5Dataset` does not depend on its `H5File`, or any intermediate
`H5Bundle`, still being alive. HDF5 reference-counts its own resources internally, so a file
stays genuinely open, at the library level, for as long as any handle into it remains open,
and only actually closes once the last handle does. It can be handy to let an `H5File` go
out of scope as soon as the single `H5Bundle` or `H5Dataset` actually needed has been
obtained from it, instead of keeping the whole chain alive for no reason.

It is not, however, a license to lose track of things: every object this API hands out must
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

## Name resolution

Parsing and combining the various segments of file and bundle names (directory, prefix,
filename, anchor, bundle) and the "try a plain file first, then fall back to a bundle" logic
are deliberately outside the API of the HDF5 target described in the previous section. To
handle this, the `FilePaths` class's API is adjusted as follows.

**Setup**

- `setInputPath(string value)` — the value of the `-i` option: a plain directory path, or a
  directory plus an HDF5 file and anchor.
- `setOutputPath(string value)` — the value of the `-o` option, in the same form.
- `setCheckpointPath(string value)` — the value of the `-c` option: always a directory plus
  an HDF5 file and anchor, since a checkpoint has no plain-file form.
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
`name`; either string is empty if there is no such correspondence, independent of whether
that file or bundle actually exists — this function does no I/O of its own. The caller is
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
