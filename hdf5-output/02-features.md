# Features

## File types

SKIRT currently writes the following kinds of output files:

- **Text column files** — employed for all single-axis tables: SEDs, per-cell and
  per-position probe output, and instrument statistics tables.
  Uses the same header-comment convention as the input side (`# column N: ... (unit)`).
- **FITS files** — employed for instrument frames and data cubes,
  and for planar cuts or projections produced by probes.
- **Spatial grid plot files** — a polyline format where
  each line holds 2 or 3 coordinates meaning "draw a line to this point", and a blank line
  starts a new, disconnected segment. Every spatial grid writes one of these to plot its own
  cell geometry.
- **Unstructured text files** — the `convergence.dat` plain text file intended for human
  consumption, written by `ConvergenceInfoProbe`.
- **XML file** — the `parameters.xml` file, a reformatted version of the ski file
  governing the simulation.
- **Log file** — the `log.txt` plain text file with progress, warning and error messages.
  Because this file grows as the simulation runs, and users should be able to view the
  simulation's progress in real time, the log file is always written as a regular file
  and then stored in the HDF5 file after the simulation has ended.
- **Iteration history files** — the text column files written by the `HistoryProbe` proposed in
  the [Iteration history](iteration-history/01-introduction.md) note, one for each iteration loop,
  with a row per iteration. Like the log file, these files grow as the simulation runs, so that the
  progress of the simulation can be followed and the output survives an aborted run. They are therefore always
  written as regular files and then stored in the HDF5 file, in the text column format, after the
  simulation has ended.

The [Data model](hdf5-output/03-data-model.md) chapter explains how each of these maps to bundles
in HDF5, and how to read them from Python.

## Command-line syntax

The command-line option that specifies the output location, `-o`, follows the same
`<dir>/<hdf>:<suite>` format as `-i`, with the same meaning for each component (see Command-line
syntax in the [HDF5 input](hdf5-input/02-features.md) note).

## How output files are written

If only `<dir>` is given, SKIRT behaves exactly as before: every output file is written as a
plain file inside `<dir>`.

If `<hdf>` is also given, SKIRT writes into that HDF5 file instead. The file as a whole is
never replaced: if it already exists, SKIRT opens it and adds to it. Each individual output
is written as a new bundle, or replaces an existing one if a bundle with the same name is
already present; every other bundle in the file is left untouched. The bundle name matches
the plain output file that would otherwise have been produced, prefixed with `<suite>` if
given, exactly as on the input side.

This makes it possible to collect the output of several simulations in a single HDF5 file,
each under its own suite, or to store both the input and the output of a single simulation
together in one file.

## Examples

Assuming a ski file `mysim.ski`, an input directory `in`, an output directory `out`, and an
HDF5 file `data.hdf5`:

- `skirt -i in -o out/data.hdf5 mysim.ski` — plain-file input, HDF5 output: input files are
  read from `in` as before, while every output file is written as a bundle inside
  `out/data.hdf5` instead of as a plain file in `out`.
- `skirt -i in/data.hdf5 -o out mysim.ski` — the reverse: input files are sought as bundles
  in `in/data.hdf5`, while every output file is written as a plain file in `out`, as before.
- `skirt -i in/data.hdf5 -o in/data.hdf5 mysim.ski` — input and output share the same HDF5
  file: input bundles are read from it, and output bundles are added into that same file
  alongside them.

`-i` and `-o` are independent: each may target a plain directory or an HDF5 file regardless
of what the other one uses.

## Concurrency

HDF5 does not support safe, uncoordinated writes to the same file from more than one
independent process — only a single writer is allowed at a time. Therefore, simulations that
run concurrently should each write to their own HDF5 file. SKIRT output files can be combined
afterward, either by merging them, or by building a small "umbrella" HDF5 file that uses
[external links](https://docs.h5py.org/en/stable/high/group.html#external-links)
to present the separate files as a single navigable hierarchy, without copying any data.

Any number of processes may read the same HDF5 file at the same time, as described in the HDF5
input note, provided that no process still has the file open for writing. A write-open imposes
this restriction as long as it stays open — not just while a write call is actually in progress —
so in practice the writer needs to have closed the file, not merely paused, before readers open
it.

Using the same HDF5 file for both `-i` and `-o` in a given simulation, as in the last
example above, does not violate this concurrency rule: a single process reading from and
writing to a file it already has open is explicitly allowed.

When a simulation runs in MPI multi-processing mode, SKIRT ensures that there are no
concurrency conflicts between these processes.
