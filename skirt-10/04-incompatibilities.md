# Incompatibilities

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
files that configure an octree or binary tree spatial grid. The chapter [Tree-based
spatial grids](skirt-10/05-tree-based-spatial-grids.md) describes this in detail.

### Binary column format

Support for the `scol` format, a less-frequently-used SKIRT-specific binary alternative
to regular text column files, is deprecated and will be removed in some future minor
release version. Users should migrate to the HDF5 alternative presented in [HDF5
input](skirt-10/06-hdf5-input.md), which accomplishes the same goals with an
industry-standard file format. There is, however, no technical reason to remove the
feature right away.

### Resonant scattering options

SKIRT offers a simulation mode and some options specifically intended for use by the
Lyman-alpha resonant scattering implemented by the `LyaNeutralHydrogenGasMix` material
mix. The more recently added `XRayIonicGasMix` material mix supports resonant scattering
for a range of hydrogen-like and and helium-like ions, and honors those same options.
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

## PTS automated ski file upgrade

The PTS function/command that upgrades ski files to the most recent version is extended
to perform the transformations corresponding to the changes in SKIRT 10 described above:

- Replace each `FullInstrument` by consecutive `SED`- and `FrameInstrument`s.

- Replace `FileTreeSpatialGrid` by `OctTreeSpatialGrid` with `TopologyTreePolicy`  policy
and very wide `minLevel` .. `maxLevel` range. Because `FileTreeSpatialGrid` reads the
tree type from file, this upgrade will be incorrect for a binary tree.

- Replace `PolicyTreeSpatialGrid` by `OctTreeSpatialGrid` or `BinTreeSpatialGrid`
  depending on the configured tree type, and replace the configured policy as follows:
  
  - `DensityTreePolicy` becomes one or more of `DustDensityTreePolicy`,
    `DustOpticalDepthTreePolicy`, or `DustDispersionTreePolicy` depending on the
     configured criteria.
    
  - `NestedDensityTreePolicy` becomes one or more `BoxTreePolicy` plus `DustXxxTreePolicy`,
  again depending on the configured criteria.
  
  - `SiteListTreePolicy` retains the same name.

- Rename Lya simulation modes and options as proposed above.
