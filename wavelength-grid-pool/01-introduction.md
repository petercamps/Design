# Introduction

> Status: draft
>
> Depends on: [SKIRT 10](skirt-10/01-introduction.md)

## Motivation

The instrument system offers a default instrument wavelength grid (DIWLG), which each instrument
uses unless it specifies its own wavelength grid (WLG). This causes two problems.

1. **Only one default grid.** Many use cases require more than one WLG, for example a
   high-resolution SED combined with a low-resolution IFU, or a coarse full-range SED combined
   with a fine zoom-in SED. Only one grid can be the DIWLG; instruments that need another grid
   must configure it themselves. With multiple sight lines, every grid other than the DIWLG must
   be repeated for each sight line.

2. **Probes using the default grid.** Some probes also fall back on the DIWLG. This is
   conceptually wrong but seemed handy at the time, and it is convenient for probes such as the
   `LuminosityProbe` and `LaunchedPacketsProbe`. For the `OpacityProbe`, however, falling back on
   the DIWLG may produce output at many wavelengths, which is often not the intention.

   Several other probes automatically select a WLG on their own: the radiation field WLG, the dust
   emission WLG, or the instrument WLGs.

This note proposes a pool of named WLGs that instruments and probes can reference, and that
designates the default grid.

## Overview

- **[Features](wavelength-grid-pool/02-features.md)** describes the pool, the way instruments and
  probes reference its grids, and the effect on existing ski files.
- **[Implementation](wavelength-grid-pool/03-implementation.md)** describes the naming of the pool
  grids, the SMILE conditions, and the validation of names.
