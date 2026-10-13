# Current state

This chapter summarizes how the current code declares, imports, stores, and updates the medium
state, as a basis for the proposal. Class and function names refer to the current code.

## Declaring the state

For each spatial cell, the medium state holds a set of common variables shared by all medium
components, and a set of specific variables for each component.

| Variable | Set | Present |
| --- | --- | --- |
| volume | common | always |
| bulk velocity | common | if any medium has a velocity |
| magnetic field | common | if any medium has a magnetic field |
| number density | specific | always |
| metallicity | specific | if requested by the material mix |
| temperature | specific | if requested by the material mix |
| custom variable *k* | specific | zero or more, as requested by the material mix |

A material mix lists its specific variables in `specificStateVariableInfo()`, as `StateVariable`
objects. A custom variable carries an index, a description, a quantity, and an output format. It
has no name: the rest of the code knows it by its index, which depends on the configuration of the
mix. The function is `const`, and most mixes record the indices of their custom variables while
building the list, which requires casting away the constness.

A variable that represents an array, such as an opacity at each of a number of wavelengths, is
declared as that number of separate custom variables.

## Storing the state

`MediumState` allocates a single array that holds all variables of all components for every cell,
contiguously per cell. Each access calculates the position of the variable from the cell index and
an offset. A `MaterialState` object bundles a cell index and a component index, and offers
`numberDensity()`, `metallicity()`, `temperature()`, and `custom(i)` with the corresponding
setters. A custom variable is read or written one value at a time.

Two further "fake" cells at the end of the array hold the current and the previous aggregate
state, used by convergence criteria. The
[Iteration history](iteration-history/01-introduction.md) design note proposes to remove them.

## Importing quantities

An imported medium reads one row per particle or cell. The columns are, in this order: the
position and size or the cell geometry, the mass or density, the metallicity, the temperature,
the velocity, the magnetic field, and the parameters requested by the material mix. All but the
first two are optional.

| Property of `ImportedMedium` | Effect |
| --- | --- |
| `massFraction` | the fraction of the imported mass to be included |
| `importMetallicity` | import a metallicity column |
| `importTemperature` | import a temperature column |
| `maxTemperature` | the maximum temperature for included mass (zero means no maximum) |
| `importVelocity`, `importMagneticField` | import the corresponding three columns |
| `importVariableMixParams` | select a material mix per position from a family, using imported parameters |

The `importVariableMixParams` option tells which of two further properties applies: `materialMix`,
which holds a single material mix, or `materialMixFamily`, which holds a family. The classes
`MaterialMix` and `MaterialMixFamily` have no common base class.

The meaning of the metallicity and temperature columns depends on the material type:

- For **dust**, they define the mass. The mass of an entity is the imported mass times the
  metallicity times `massFraction`, and it is zero if the temperature exceeds `maxTemperature`.
  This derives a dust distribution from the gas of a hydrodynamical snapshot. The values are not
  passed to the material mix.
- For **gas and electrons**, they do not affect the mass. The values are passed to the material
  mix when it initializes its state.

`ImportedGeometry` offers the same three options, with the meaning they have for dust.

A material mix requests further columns through `parameterInfo()`, which returns a list of
`SnapshotParameter` objects, each with a description, a quantity, and a default unit. The class is
shared with the SED families of imported sources, and knows a few standard parameter types,
including metallicity, temperature, and age. A material mix is not allowed to request the
metallicity or the temperature in this list, because they are imported through the options above.

As a result, each gas mix documents which combination of `importMetallicity` and
`importTemperature` it expects, in addition to its own parameters:

| Material mix | Metallicity | Temperature | Parameters |
| --- | --- | --- | --- |
| `ElectronMix` | — | optional, if thermal dispersion is included | — |
| `LyaNeutralHydrogenGasMix` | — | optional | — |
| `SpinFlipAbsorptionMix` | — | optional | — |
| `SpinFlipHydrogenGasMix` | optional | optional | neutral hydrogen surface density |
| `NonLTELineGasMix` | — | optional | number density of each collisional partner, turbulence velocity |
| `DiffuseIonizedGasMix` | optional, or required in per-cell abundance mode | optional | nine abundance ratios in per-cell abundance mode |
| `FragmentDustMixDecorator` | defines the mass | defines the mass | weight for each fragment |
| `SelectDustMixFamily` | defines the mass | defines the mass | dust mix index |
| `XRayIonicGasMixFamily` | — | optional | abundance of each ion |

"Optional" means that the mix falls back on a default value, set by one of its own properties,
when the column is not imported.

## Initializing the state

During setup, the medium system determines the volume, the number densities, the bulk velocity,
and the magnetic field of each cell. It then calls, for each component,

```cpp
mix->initializeSpecificState(&state, metallicity, temperature, params);
```

where the metallicity and the temperature are negative if they are not imported, and `params` is
empty if the mix requested no parameters or the medium is not imported. For a geometric medium,
nothing is imported, and the mix uses its default values.

For a medium with a material mix family, the family selects a mix for each cell through
`mix(metallicity, temperature, params)`, evaluated at the center of the cell. The members of the
family do not receive the parameters.

## Combining values

Where the entities of a snapshot overlap, as smoothed particles do, the value at a given position
combines the values of several entities. The rule depends on the quantity:

| Quantity | Rule |
| --- | --- |
| density | sum of the contributions of the overlapping entities |
| metallicity, temperature, velocity, magnetic field | average, weighted by the kernel weight times the mass of each entity |
| parameters requested by the material mix | value of the entity with the largest kernel weight |

The value of a cell is by default taken at the center of the cell. If the user configures more
than one property sample per cell, the samples are averaged, weighted by density, for all
quantities including the parameters.

Taking the entity with the largest weight is appropriate for a parameter that selects something,
such as the dust mix index of the `SelectDustMixFamily`. It is not appropriate for a continuous
quantity, such as the turbulence velocity of the `NonLTELineGasMix` or the abundance ratios of the
`DiffuseIonizedGasMix`. Averaging the samples, conversely, is wrong for an index.

## Where temperature and metallicity are special

| Location | Special treatment |
| --- | --- |
| `ImportedMedium`, `ImportedGeometry` | the properties `importMetallicity`, `importTemperature`, and `maxTemperature` |
| `Snapshot` | the functions `importMetallicity()` and `importTemperature()`; the mass density policy with a metallicity multiplier and a temperature cutoff; averaged values at a position |
| the snapshot subclasses | skipping or zeroing entities above the temperature cutoff, and logging the metallic mass |
| `Medium` | the functions `hasMetallicity()`, `metallicity()`, `hasTemperature()`, and `temperature()` |
| `MediumSystem` | sampling both in each cell; the functions `metallicity(m,h)` and `temperature(m,h)` |
| `StateVariable`, `MediumState`, `MaterialState` | dedicated identifiers, offsets, getters, and setters |
| `MaterialMix`, `MaterialMixFamily` | dedicated arguments of `initializeSpecificState()` and `mix()`; the ban on requesting them as parameters |
| six material mixes | declaring and using the dedicated state variables |
| `LineGasSecondarySource` | looking for the temperature state variable to add thermal motion to emitted lines |
| `GasThermalEnergyTreePolicy` | reading the imported temperature through `Medium::temperature()` |
| `TemperatureProbe`, `MetallicityProbe` | looking for the dedicated state variables |
| `ImportedMediumTemperatureProbe`, `ImportedMediumMetallicityProbe`, `ImportedMediumVelocityProbe` | reading the imported columns; dividing the weight by the metallicity for dust |

## Capabilities

A material mix describes its capabilities through a dozen virtual functions, such as
`materialType()`, `hasPolarizedScattering()`, `hasResonantScattering()`, `hasExtraSpecificState()`,
and `hasDynamicMediumState()`. The configuration of the simulation is established early during
setup, before the media are set up, from the answers of a representative mix returned by
`Medium::mix()`.

For a medium with a material mix family, the representative mix is a default member of the family.
A family that creates its members on demand, such as the `XRayIonicGasMixFamily`, must construct
one for this purpose and force its own setup ahead of time. All members of a family must have the
same capabilities and the same state variables, but nothing verifies this.

## Updating the state

A material mix with a dynamic medium state updates its specific state at the end of a primary or
secondary emission segment. The medium system calls `updateSpecificState()` for each cell, in
parallel over threads and processes, then synchronizes the state between processes, and finally
calls `isSpecificStateConverged()` once. The update of a cell may only touch the state of that
cell, and the mix has no opportunity to act before or after the pass over the cells.

A dynamic state recipe, by contrast, has a `beginUpdate()` and an `endUpdate()` function around
its per-cell `update()`.
