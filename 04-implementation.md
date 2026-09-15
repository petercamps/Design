# Implementation

> Status: draft

## HDF5 target

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
is attempted. As an exception, the trivial H5 class offers a static `available()`
query that simply reports whether this build was compiled with `BUILD_HDF5`.
Wherever a ski file setting or command-line option would require
HDF5 — anywhere `-i`/`-o`/`-c` names an HDF5 file, or a property such as
`CheckpointTreePolicy::filename` targets one — setup-time validation checks
`H5File::available()` first and reports a clear, immediate error if HDF5 support is missing,
rather than letting execution reach a stub's fatal error later.

All calls into the HDF5 C API are protected by a single mutex internal to this target: the
library cannot be assumed to be thread-safe on its own, since that depends on how the
installed HDF5 happens to have been built, which is outside SKIRT's control.

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

Attributes (on `H5Bundle`, `H5Checkpoint`and `H5Dataset`) and dataset contents share three types:
boolean, 32-bit integer, and 64-bit float. Attributes additionally support `string`,
passed as a `std::string` in
UTF-8 and stored as a fixed 128-byte field — comfortably larger than any attribute value in
the Data model chapter, including the free-form `description` attributes, without the
extra indirection and reclaim bookkeeping a variable-length string would cost for values
this short. Datasets additionally support `text`, for output only, passed in as a
`std::istream` in UTF-8 and stored as a variable-length string on disk — Unstructured text
file's `text` dataset is the only case that needs it, and unlike an attribute, its content
can genuinely be arbitrarily long.

Bundle *name* construction — the `<prefix>_...` conventions throughout the Data model
chapter — and the "try a plain file first, then fall back to a bundle" resolution logic from
How input files are located in Features are deliberately outside this API: both are SKIRT
domain logic rather than HDF5 mechanics, so they stay in the calling code, which only ever
hands this API an already-resolved bundle path.

## Notes to revisit

- When running under MPI with input and output pointed at the same HDF5 file, all ranks
  must close their read access to that file before the root process opens it for writing —
  reading and writing must not overlap in time (see Concurrency under Output in Features).

- `CheckpointTreePolicy` must match each live node to its counterpart in the recorded
  topology — parsed into a real tree at setup via its `parent_id` links — by following the
  same parent/child-slot path from the root, rather than assuming live construction visits
  nodes in the same order the checkpoint was originally written in; once a live node falls
  past what was recorded, it correctly answers no from there down.

## Stored table bundle

The data in a stored table input bundle cannot be memory mapped (as for a regular .stab file)
for several reasons:

- The quantity values are ordered differently (interleaved in .stab, per dataset in bundle).
- The bundle datasets may be compressed and/or chunked.
- If the same HDF file is used for input and output, memory mapping a portion of the file
  is unsafe because writing to the file might relocate its contents.

The StoredTable constructor for a bundle must therefore copy the data into newly allocated
memory, and the destructor must deallocate that memory.
