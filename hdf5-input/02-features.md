# Features

## File types

SKIRT currently supports several input file types, each suited to a different kind of
source data:

- **Text column file** — a plain text table with one row per item (wavelength, particle,
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

## Command-line syntax

The SKIRT command-line option that specifies the input location, `-i`, now accepts up to
three components combined into a single argument:

```
-i <dir>/<hdf>:<suite>
```

- `<dir>` — the input directory, exactly as before.
- `<hdf>` — optional: the name of an HDF5 file located inside `<dir>`, including the `.hdf5`
  extension.
- `<suite>` — optional, and only meaningful together with `<hdf>`: a path within the HDF5
  file, used as described below.

Only `<dir>` is required.

## How input files are located

If only `<dir>` is given, SKIRT behaves exactly as before: every input file it needs is
expected to be a plain file inside `<dir>`. (The ski file itself is not an input file in this
sense — its full path is given directly as a separate command-line argument, independent of
`-i`.)

If `<hdf>` is also given, SKIRT looks for each input file in two steps: first as a plain file
in `<dir>`, exactly as before; if that file is not found, SKIRT looks inside `<hdf>` instead,
for a bundle whose name matches the input file name.

If `<suite>` is given as well, it is prefixed to the input file name before that lookup, so
SKIRT looks for `<suite>/<bundle>` rather than `<bundle>` alone. This makes it possible to
store the input for several simulations inside a single HDF5 file, each under its own suite,
and select the right one per run through the `-i` option. On the other hand, multiple SKIRT
simulations may simply share the same input file, for example a common custom spectrum.

## Examples

Assuming a ski file `mysim.ski`, an input directory `in`, and an HDF5 file `data.hdf5`:

- `skirt -i in mysim.ski` — unchanged behavior: all input files are read from plain files in
  `in`.
- `skirt -i in/data.hdf5 mysim.ski` — input files are sought as plain files in `in` first,
  then as bundles in `in/data.hdf5`.
- `skirt -i in/data.hdf5:run01 mysim.ski` — as above, but bundles are looked up under the
  `run01` group, i.e. as `run01/<bundle>`.
- `skirt -i in/data.hdf5:campaign7/galaxy042 mysim.ski` — the suite can itself be a
  multi-level path, here selecting the input for one galaxy out of many stored in the same
  file.
- `skirt -i ./data.hdf5 mysim.ski` — edge case: the HDF5 file sits directly in the current
  directory (`<dir>` is `.`).

## Concurrency

Any number of independent SKIRT processes may open and read the same HDF5 file at the same
time, with no special mode or coordination needed.

## PTS

To help users migrate existing input files, and to serve as a Python coding example, PTS will
be extended with functions and commands to convert the file types discussed above to
their corresponding HDF5 bundle form.
