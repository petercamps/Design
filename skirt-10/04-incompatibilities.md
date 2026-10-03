# Incompatibilities

This chapter collects the incompatible changes in SKIRT 10, including those proposed by the other
SKIRT 10 design notes, which describe the reasons and the details.

## Affected features

### Full instrument

The `FullInstrument` is removed. It can be replaced by an `SEDInstrument` and a
`FrameInstrument` configured with the same line of sight. This allows streamlining the
code because each instrument now has a well-defined aperture (a `FullInstrument`'s
aperture was ambiguous).

The performance cost of handling two separate instruments is minimal. When instruments
with the same line of sight are placed consecutively in the ski file, the same peel-off
photon packet is sent to all of these instruments, so the extinction along that line of
sight is calculated only once.

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
| Property | `lyaAccelerationScheme` | `accelerationScheme` |
| Enumeration | `LyaAccelerationScheme` | `AccelerationScheme` |
| Property | `lyaAccelerationStrength` | `accelerationStrength` |

The related configuration setup messages will be adjusted accordingly.

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

Support for the `scol` format, a less-frequently-used SKIRT-specific binary alternative
to regular text column files, is deprecated and will be removed in some future minor
release version. Users should migrate to the HDF5 alternative presented in the [HDF5
input](hdf5-input/01-introduction.md) design note, which accomplishes the same goals with an
industry-standard file format. There is, however, no technical reason to remove the
feature right away.

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

- Replace each `FullInstrument` by consecutive `SED`- and `FrameInstrument`s.

- Replace `FileTreeSpatialGrid` by `OctTreeSpatialGrid` with the `TopologyTreePolicy`
  policy and a very wide `minLevel`..`maxLevel` range. Because `FileTreeSpatialGrid`
  reads the tree type from file, this upgrade will be incorrect for a binary tree.

- Replace `PolicyTreeSpatialGrid` by `OctTreeSpatialGrid` or `BinTreeSpatialGrid`
  depending on the configured tree type, and replace the configured policy as follows:

  - `DensityTreePolicy` becomes one or more of `DustDensityTreePolicy`,
    `DustOpticalDepthTreePolicy`, or `DustDispersionTreePolicy` depending on the
    configured criteria.

  - `NestedDensityTreePolicy` becomes one or more `BoxTreePolicy` plus `DustXxxTreePolicy`,
    again depending on the configured criteria.

  - `SiteListTreePolicy` retains the same name.

- Move the `defaultWavelengthGrid` of the `InstrumentSystem`, if present, into a new
  `WavelengthGridPool` placed just before the instrument system, as a named grid called
  `default`, and set the pool's `defaultGridName` property to `default`.

- Rename Lya simulation modes and options as proposed above.

- Remove the `maxFractionOfPrevious` property from `DustEmissionOptions`. The new dust
  emission convergence and rebalance properties receive their default values.
