# Data model

## Bundles

As on the input side, each output file corresponds to a **bundle**: an HDF5 group containing one
or more datasets whose names are fixed by SKIRT (see the
[HDF5 input](hdf5-input/03-data-model.md) data model). The bundle name is the name of the plain
output file that would otherwise have been written.

Every bundle written by SKIRT carries three standard attributes, never required when SKIRT only
reads a bundle:

| Attribute | Type | Description |
| --- | --- | --- |
| `producer` | string | SKIRT version and build that wrote the bundle (as in SKIRT's welcome message). |
| `format` | 32-bit integer | Version of this bundle format itself, for future changes; `1` for now. |
| `created` | string | When the bundle was written, as an ISO 8601-with-milliseconds string. |

For example, `producer` might read `SKIRT v10.0 (git 584edd6 built on 03/09/2026 at 10:51:20)`, and
`created` might read `2026-09-03T10:53:22.305`. Individual bundle descriptions below list only
attributes beyond these three.

Upon output, SKIRT writes data types exactly as described in the tables of this chapter. The
possible use of chunking and/or compression is left for future consideration.

## Text column file

Wraps the same `TextInFile`/`TextOutFile` convention as the input side (see Text column file
bundle in the [HDF5 input](hdf5-input/03-data-model.md) data model), used for SEDs, per-cell and
per-position probe output, and instrument statistics tables. Every output column is written as a
64-bit float and numbered from 0 in the order `addColumn()` is called, so the same per-dataset
`column` and `unit` attributes apply.
Most call sites also write a free-form comment as the very first line, by convention rather
than by any mechanism `TextOutFile` itself enforces (via the public `writeLine()`, before
adding columns) — this becomes the bundle's own `description` attribute.

**Attributes**

| Attribute | Type | Description |
| --- | --- | --- |
| `description` | string | Free-form comment describing the output. |

**Datasets** — one per column, as on the input side. Shown here for a 2-column SED output
file, where `N` is the number of rows:

| Dataset | Dimensions | Type | Attributes |
| --- | --- | --- | --- |
| `lambda` | (N) | 64-bit float | `column = 0`, `unit = "micron"` |
| `F_nu` | (N) | 64-bit float | `column = 1`, `unit = "Jy"` |

## FITS file

Wraps SKIRT's existing `FITSInOut::write()` and `FITSInOut::writeMap()` helpers (built on
`cfitsio`) for 2-D data frames and 3-D data cubes (a stack of frames along a third axis).
These are used for instrument fluxes (IFUs and STMs) and statistics, and for planar
cuts or projections produced by probes.

Data values are stored at 64-bit float precision as opposed to 32-bit float in actual
FITS output files. The three axis coordinate values are the equivalent of information
otherwise stored in the FITS header and/or in FITS table extensions named `GRID_POINTS`.
The quantities represented by data and axes differ for the various use cases, as shown
in the table below. The axis and data datasets each carry attributes defining the
quantity type and unit.

| Output type | x-axis | y-axis | z-axis | data |
| --- | --- | --- | --- | --- |
| Flux IFU | spatial | spatial | spectral | flux |
| Statistics for flux IFU | spatial | spatial | spectral | contribution moments |
| Flux STM | spectral | time lag | - | flux |
| Statistics for flux STM | spectral | time lag | - | contribution moments |
| Planar cut or projection | | | | |
| ... for scalar quantity | spatial | spatial | - | depends on probe |
| ... for velocity | spatial | spatial | (vx, vy, vz) | velocity |
| ... for spectral grid | spatial | spatial | spectral | depends on probe |

**Attributes**

| Attribute | Type | Description |
| --- | --- | --- |
| `description` | string | Free-form comment describing the output. |

Only for distant-instrument IFUs:

| Attribute | Type | Description |
| --- | --- | --- |
| `inclination` | 64-bit float, degrees | Viewing inclination (FITS `CROTA1`). |
| `azimuth` | 64-bit float, degrees | Viewing azimuth (FITS `CROTA2`). |
| `roll` | 64-bit float, degrees | Viewing roll angle (FITS `CROTA3`). |
| `redshift` | 64-bit float | Redshift of the source (FITS `REDSHIFT`). |
| `luminosity_distance` | 64-bit float | Luminosity distance to the source (FITS `DISTLUMI`). |
| `angular_diameter_distance` | 64-bit float | Angular-diameter distance to the source (FITS `DISTANGD`). |
| `distance_unit` | string | Unit of the two distance attributes (FITS `DISTUNIT`). |

The remaining FITS header information is represented by the datasets and their attributes,
as listed below, and by the `producer` and `created` attributes carried by the bundle.

**Datasets** — shown here for a 3-D IFU data cube produced by a FrameInstrument:

| Dataset | Dimensions | Type | Attributes |
| --- | --- | --- | --- |
| `x` | (nx) | 64-bit float | `quantity = "length"`, `unit = "pc"` |
| `y` | (ny) | 64-bit float | `quantity = "length"`, `unit = "pc"` |
| `z` | (nz) | 64-bit float | `quantity = "wavelength"`, `unit = "micron"` |
| `data` | (ny, nx) or (nz, ny, nx) | 64-bit float | `quantity = "frequencysurfacebrightness"`, `unit = "MJy/sr"` |

If `z` is missing or has a single value, `data` is 2-D representing a single data frame.
If `z` is present with 2 or more values, `data` is 3-D representing a data cube.

## Spatial grid plot file

Wraps SKIRT's existing `SpatialGridPlotFile` helper (itself a thin wrapper around
`TextOutFile`), used by every spatial grid to plot its own cell geometry: 2-D plane-cut
variants (`_grid_xy`, `_grid_xz`, `_grid_yz`) and a 3-D variant (`_grid_xyz`). Unlike the
other `TextOutFile`-based formats, it writes no header at all — just raw coordinate pairs
or triples, one point per line, with a blank line marking the start of a new, disconnected
polyline (a "moveto" in the class's own terminology; consecutive points with no blank line
between them are connected, a "lineto"). No unit is recorded in the plain-text file itself,
even though the coordinates do have one (the simulation's configured output length unit) —
the bundle adds an explicit `unit` attribute to close that gap.

**Attributes**

| Attribute | Type | Description |
| --- | --- | --- |
| `unit` | string | Length unit of every `points` coordinate — whatever the simulation's output uses. |

**Datasets**

| Dataset | Dimensions | Type | Description |
| --- | --- | --- | --- |
| `points` | (N, 2) or (N, 3) | 64-bit float | One coordinate pair or triple per row. |
| `moveto` | (N) | boolean | `true` marks the start of a new polyline; always `true` for row 0. |

`N` is the total number of points across every polyline in the file.

## Unstructured text file

Covers three otherwise-unrelated plain-text output types — `convergence.dat` (free-form
human-readable text written by `ConvergenceInfoProbe`), `parameters.xml` (a reformatted
copy of the ski file), and `log.txt` (progress, warning, and error messages) — each stored
the same way: as a single string holding the entire file's content, unparsed. A `type`
attribute records which of the three it is.

The log file is the one special case. Because it grows throughout the run and users should
be able to watch progress in real time, it continues to be written incrementally to a
regular file during the simulation, exactly as today (see File types in the Features chapter);
only the finished, complete text is added to its bundle at the very end
of the run — the bundle itself is never updated incrementally.

**Attributes**

| Attribute | Type | Description |
| --- | --- | --- |
| `type` | string | One of `free form`, `XML`, or `log`. |

**Datasets**

| Dataset | Dimensions | Type | Description |
| --- | --- | --- | --- |
| `text` | scalar | variable-length string | The entire file's content, unparsed. |
