# Incompatibilities

This chapter collects the incompatible changes in SKIRT 10, including those proposed by the other
SKIRT 10 design notes, which describe the reasons and the details.

## Affected features

### Full instrument

The `FullInstrument` is removed. It can be replaced by an `SEDInstrument` and a
`FrameInstrument` configured with the same line of sight. This allows streamlining the
code because each instrument now has a well-defined aperture (a `FullInstrument`'s
aperture was ambiguous).

The performance cost of handling two separate instruments is negligeable. SKIRT 10 groups the
instruments that share a line of sight, regardless of their order in the ski file, and sends
a single peel-off photon packet to each group, so the extinction along that line of sight is
calculated only once. For distant instruments, the distance does not matter, and the roll
angle matters only if one of the instruments records polarization. In a test with three lines
of sight, after replacing each `FullInstrument` by an `SEDInstrument` and a `FrameInstrument`,
SKIRT 10 is slightly faster than SKIRT 9, in whatever order the instruments are listed.

### Tree-based spatial grids

SKIRT 10's tree-based spatial grids are significantly reorganized to support an extended
feature set and improved performance. Unfortunately, this invalidates all existing ski
files that configure an octree or binary tree spatial grid. The [Tree-based spatial
grids](tree-based-spatial-grids/01-introduction.md) design note describes this in detail.

### Default instrument wavelength grid

The [Wavelength grid pool](wavelength-grid-pool/01-introduction.md) design note proposes a pool of
named wavelength grids that instruments and probes can reference, placed just before the
instrument system. The pool also designates the default grid, which replaces the
`defaultWavelengthGrid` property of the `InstrumentSystem`. This invalidates all ski files
that configure a default instrument wavelength grid, but the upgrade is mechanical: the grid
moves into the pool under a name, and the pool's `defaultGridName` property is set to that name.

The pool is offered only from the Regular user level on. At the Basic level, each instrument
and probe configures its own wavelength grid. Existing ski files at the Basic level keep
working after the upgrade, because SKIRT reads the pool from a ski file at any user level.

### Resonant scattering options

SKIRT offers a simulation mode and some options specifically intended for use by the
Lyman-alpha resonant scattering implemented by the `LyaNeutralHydrogenGasMix` material
mix. The more recently added `XRayIonicGasMix` material mix supports resonant scattering
for a range of hydrogen-like and helium-like ions, and honors those same options.
It is therefore appropriate to adjust the simulation mode and option names to reflect a
general resonant scattering context. While this invalidates all ski files configured with
either of the material mixes mentioned above, the upgrade is a trivial rename:

| Item | Before | After |
| --- | --- | --- |
| Simulation mode | `LyaExtinctionOnly` | `ResonanceExtinction` |
| Option block | `LyaOptions` | `ResonanceOptions` |
| Property of `MediumSystem` | `lyaOptions` | `resonanceOptions` |
| Property | `lyaAccelerationScheme` | `accelerationScheme` |
| Enumeration | `LyaAccelerationScheme` | `AccelerationScheme` |
| Property | `lyaAccelerationStrength` | `accelerationStrength` |

The related configuration setup messages refer to resonant line scattering accordingly.

### Dust self-absorption

The [Dust self-absorption](dust-self-absorption/01-introduction.md) design note proposes
changes to the calculation of dust emission and to the secondary emission iterations,
which affect existing ski files and simulation results as follows:

- The `maxFractionOfPrevious` property of `DustEmissionOptions` is removed, because it
  can hide large errors when the absorbed secondary luminosity is much larger than the
  absorbed primary luminosity. It is replaced by the new convergence criteria
  `maxLuminosityDeficit` and `maxSecondaryChange`, evaluated over
  `numConvergenceIterations` iterations. The properties `rebalanceSecondaryEmission`,
  `numCoreBins`, and `numHeatingBins` configure the new regional rebalance of the
  secondary radiation field, which is enabled by default.
- The luminosity absorbed by dust is accumulated from the dust opacity at each photon
  packet's exact wavelength, rather than calculated from the binned radiation field. This
  changes the results of all simulations with dust emission: typically by a few tenths of
  a percent for a radiation field wavelength grid with 50 bins, more for coarser grids,
  and by up to 10–15% in strongly self-absorbing models.
- The regional rebalance and the new convergence criteria change the number of
  iterations, and the results of models that did not converge, for all simulations that
  iterate over secondary emission.
- In simulations with merged primary and secondary emission iterations, the secondary
  radiation field is no longer discarded at the start of each iteration and before the
  final secondary emission. Dust self-absorption is therefore taken into account in these
  simulations, which increases the dust emission of models with substantial
  self-absorption.

### Binary column format

Support for the `scol` format, a less-frequently-used SKIRT-specific binary alternative to regular
text column files, is removed. The text column file bundles proposed in the
[HDF5 input](hdf5-input/01-introduction.md) design note accomplish the same goals, compact storage
and fast reading, with an industry-standard file format. SKIRT 10 reports a fatal error for an input
file with the `.scol` extension.

Existing `.scol` files must be converted, either to a text column file or to a text column file
bundle in an HDF5 file, and the ski file must refer to the converted file. PTS offers a command for
both conversions, built on its existing functions for reading `.scol` files.

The removal affects:

- SKIRT: the `StoredColumns` class and the `.scol` branch of `TextInFile`;
- PTS: the `convert_text_to_stored_columns` command, which is removed, while the functions for
  reading the format are kept for the conversion;

### Data parallelization

The command line option `-d` is removed, including the related help information and error message,
as it is no longer relevant. This option was deprecated during the transition to SKIRT 9 because
photon packets can change wavelength during their lifetime. As a result, packets can cross both the
spectral and spatial domains almost at random, and it is no longer feasible to split the large data
structures across processes. For the foreseeable future, SKIRT 10 will continue to always duplicate
all data to all MPI processes.

## PTS automated ski file upgrade

The PTS function/command that upgrades ski files to the most recent version is extended
to perform the transformations corresponding to the changes in SKIRT 10 described above:

- Replace each `FullInstrument` by an `SEDInstrument` (without an aperture) followed by a
  `FrameInstrument`, both with the same name and line of sight, and each with a copy of any
  instrument-specific wavelength grid.

- Replace `FileTreeSpatialGrid` by `OctTreeSpatialGrid` with the `TopologyTreePolicy`
  policy and a very wide `minLevel`..`maxLevel` range. Because `FileTreeSpatialGrid`
  reads the tree type from file, this upgrade will be incorrect for a binary tree.

- Replace `PolicyTreeSpatialGrid` by `OctTreeSpatialGrid` or `BinTreeSpatialGrid`
  depending on the configured tree type, and replace the configured policy as follows:

  - `DensityTreePolicy` becomes one policy for each configured criterion: a
    `DensityTreePolicy` for each of `maxDustFraction`, `maxElectronFraction`, and
    `maxGasFraction`, an `OpticalDepthTreePolicy` for `maxDustOpticalDepth`, and a
    `DispersionTreePolicy` for `maxDustDensityDispersion`, each with the corresponding
    `materialType`.

  - `NestedDensityTreePolicy` becomes one or more `BoxTreePolicy` plus the same policies,
    again depending on the configured criteria.

  - `SiteListTreePolicy` retains the same name.

- Move the `defaultWavelengthGrid` of the `InstrumentSystem`, if present, into a new
  `WavelengthGridPool` placed just before the instrument system, as a named grid called
  `default`, and set the pool's `defaultGridName` property to `default`.

- Rename the Lyman-alpha simulation mode, option block, and options as listed above,
  including the `lyaOptions` property of the `MediumSystem`.

- Remove the `maxFractionOfPrevious` property from `DustEmissionOptions`. The new dust
  emission convergence and rebalance properties receive their default values.

- Report each input file name with the `.scol` extension, since these files must be converted
  separately, as described above.
