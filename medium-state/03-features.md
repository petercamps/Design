# Features

Two of the five proposed changes are visible to users: the way quantities are imported for a
medium, and the names and kinds of the quantities. The other three are internal, and show only as
new possibilities for the authors of material mixes. The Implementation chapter describes them.

## Importing quantities

### Behavior

An imported medium imports every quantity that its material mix, or its material mix family,
requests. The temperature and the metallicity are requested like any other quantity, by the mixes
that need them. The properties `importMetallicity`, `importTemperature`, and `maxTemperature` are
removed from all imported media and from all imported geometries.

The `massFraction` property remains available for all imported media, whatever their material
type. The imported mass or density is multiplied by this fraction, as today.

The columns of an import file are, in this order:

1. the position and size, or the cell geometry, as defined by the type of snapshot;
2. the mass or the density;
3. the velocity, if `importVelocity` is enabled;
4. the magnetic field, if `importMagneticField` is enabled;
5. the quantities requested by the material mix or family, in the order it lists them.

The user no longer needs to know which combination of options a material mix expects. The log
lists the columns that are read, with their units, as it does today.

For a geometric medium nothing is imported, and a material mix uses its default values, set by
its own properties, as today. These defaults no longer apply to an imported medium: every
requested quantity must be present in the import file.

### Material mix families

An imported medium has a single `materialMix` property, which accepts a material mix or a
material mix family. The `importVariableMixParams` option and the separate `materialMixFamily`
property are removed: configuring a family is what makes the material mix vary with position. For
example, a medium that was configured as

```xml
<ParticleMedium filename="dust.txt" importVariableMixParams="true" ...>
    <materialMixFamily type="MaterialMixFamily">
        <SelectDustMixFamily>
            ...
        </SelectDustMixFamily>
    </materialMixFamily>
</ParticleMedium>
```

becomes

```xml
<ParticleMedium filename="dust.txt" ...>
    <materialMix type="MaterialMixProvider">
        <SelectDustMixFamily>
            ...
        </SelectDustMixFamily>
    </materialMix>
</ParticleMedium>
```

The import file does not change. The list of choices for the material mix of an imported medium
includes the families, under the conditions that applied to the option. A geometric medium
accepts a material mix only, as today.

### Quantities requested by each material mix

The following table lists the proposed requests. The metallicity and the temperature come first,
in that order, so that an existing import file without velocity or magnetic field columns keeps
its column order, provided that it already holds all the listed quantities.

| Material mix or family | Imported quantities, in order |
| --- | --- |
| `ElectronMix` | temperature, only if thermal dispersion is included |
| `LyaNeutralHydrogenGasMix` | temperature |
| `SpinFlipAbsorptionMix` | temperature |
| `SpinFlipHydrogenGasMix` | metallicity, temperature, neutral hydrogen mass surface density |
| `NonLTELineGasMix` | temperature, number density of each collisional partner, turbulence velocity |
| `DiffuseIonizedGasMix` | metallicity, temperature and, in per-cell abundance mode, the nine abundance ratios |
| `FragmentDustMixDecorator` | weight for each fragment |
| `SelectDustMixFamily` | dust mix index |
| `XRayIonicGasMixFamily` | temperature, abundance of each ion |

A mix whose need for a quantity depends on its configuration expresses this through its own
properties, as the `ElectronMix` does with `includeThermalDispersion` and the
`DiffuseIonizedGasMix` with its abundance mode. The author of a mix can use the same means to make
a quantity optional.

### Dust mass

The mass of a dust medium is the imported mass times `massFraction`. SKIRT no longer multiplies
the mass by an imported metallicity, and no longer discards the mass above a maximum temperature.
A user who derives the dust distribution from the gas of a hydrodynamical snapshot calculates the
dust mass, or the mass of the metals in sufficiently cold gas, before handing it to SKIRT.

For example, a dust medium that was configured as

```xml
<ParticleMedium filename="gas.txt" massFraction="0.3" importMetallicity="true"
                importTemperature="true" maxTemperature="8000 K" ...>
```

with columns for the position, the smoothing length, the gas mass, the metallicity, and the
temperature, becomes

```xml
<ParticleMedium filename="metals.txt" massFraction="0.3" ...>
```

with columns for the position, the smoothing length, and the mass of the metals, which is the gas
mass times the metallicity for the particles at or below 8000 K, and zero or absent for the
others.

PTS offers a command that performs this conversion for an existing import file, given the maximum
temperature. It multiplies the mass or density column by the metallicity column, removes the
particles above the maximum temperature, or sets the density to zero for cells, and removes the
two columns.

The same holds for an imported geometry, which is normalized to unit mass and has no mass
fraction.

### Existing import files for gas and electrons

An import file for a gas or electron medium that holds the velocity or the magnetic field has its
metallicity and temperature columns in a different position than the new order expects. If the
file has a header with column names, the existing `useColumns` property maps the columns by name,
or reorders them automatically, without changing the file. Otherwise the columns must be
reordered.

## Kinds of quantities

Each imported quantity, and each state variable, has a kind, declared by the material mix:

| Kind | Meaning | Examples |
| --- | --- | --- |
| intensive | a continuous quantity that does not depend on the amount of material | temperature, metallicity, velocity, abundance ratio, number density of a species |
| categorical | a label or an index, for which an average has no meaning | dust mix index |
| extensive | a quantity proportional to the amount of material | mass, number of particles, luminosity |

The kind determines how values are combined when an imported medium is evaluated:

| Kind | Overlapping entities at a position | Several samples in a spatial cell |
| --- | --- | --- |
| intensive | average, weighted by the kernel weight times the mass of each entity | average, weighted by density |
| categorical | value of the entity with the largest kernel weight | value of the sample with the highest density |
| extensive | not supported for quantities requested by a mix | not supported for quantities requested by a mix |

The kinds apply to the quantities that a material mix requests. A material mix that needs an
extensive quantity requests the corresponding intensive quantity instead, such as an abundance
ratio or a number density.

Importing the mass or density of a medium keeps its own treatment, as today. 
The velocity and the magnetic field are combined as today, and so are the temperature and the
metallicity of a material mix. The other quantities requested by a material mix were all treated
as categorical; they are now intensive, with two exceptions. The dust mix index of the
`SelectDustMixFamily` remains categorical. All quantities of the `XRayIonicGasMixFamily`,
including its temperature, are categorical, because the family creates a mix for each distinct
set of values and therefore must not receive averaged values.

For a snapshot without overlapping entities, such as a cell-based or Voronoi snapshot, and with a
single property sample per cell, which is the default, the kind makes no difference.

The kind of a state variable also determines what happens to it when a spatial cell is
subdivided, as proposed in the
[Dynamic grid refinement](dynamic-grid-refinement/01-introduction.md) design note: intensive and
categorical variables are copied to the child cells, and extensive variables are divided among
them in proportion to their volume.

## Named state variables

A material mix can give a short name to each of its state variables. A name consists of letters,
digits, and underscores, and is unique among the variables of the mix. Two names are standard, so
that the rest of the code can find these quantities in any mix that stores them:

| Name | Quantity |
| --- | --- |
| `T` | the temperature of the material, in kelvin |
| `Z` | the metallicity of the material, as a mass fraction |

Other names are chosen by the mix. The `DiffuseIonizedGasMix`, for example, names its ionization
parameter `logU` and its neutral hydrogen fraction `x_HI`.

Names serve wherever a user selects a state variable:

- The refinement criteria and conditions of the Dynamic grid refinement design note select a
  variable through their `variable` property.
- The `CustomStateProbe` selects variables through a `variables` property, which replaces its
  `indices` property. The value is a comma-separated list of names, indices, and index ranges; an
  empty list selects all variables, as today. For an array variable, the name selects all its
  elements.

Selecting by name is more robust than selecting by index, because the indices depend on the
configuration of the mix, and they change with this proposal: the temperature and the metallicity
become custom state variables of the mixes that store them.

## Probes

- The `TemperatureProbe` and the `MetallicityProbe` produce the same output as today. They locate
  the temperature and the metallicity by name.
- The `CustomStateProbe` also lists the temperature and the metallicity, for the mixes that store
  them, and selects variables by name or by index, as described above.
- The `ImportedMediumTemperatureProbe` and the `ImportedMediumMetallicityProbe` produce output for
  the media whose material mix requests the corresponding quantity. They no longer produce output
  for dust media.
- The `ImportedMediumVelocityProbe` weighs the velocity by the imported density for all media. For
  a dust medium, it used to divide the density by the metallicity to recover the gas density.

## Incompatibilities

- The properties `importMetallicity`, `importTemperature`, and `maxTemperature` are removed from
  all imported media and imported geometries.
- The properties `importVariableMixParams` and `materialMixFamily` are removed from all imported
  media. A material mix family is configured through the `materialMix` property, of which the
  base type changes for all imported media.
- The dust mass is no longer derived from an imported metallicity and temperature. The import
  files of dust media and imported geometries that used these options must be converted.
- For gas and electron media, the metallicity and temperature columns move behind the velocity
  and magnetic field columns, and every quantity requested by the material mix must be present in
  the import file. Import files that hold a velocity or a magnetic field, or that relied on the
  default temperature or metallicity of the mix, must be adjusted, or mapped with `useColumns`.
- The quantities requested by the `NonLTELineGasMix`, the `SpinFlipHydrogenGasMix`, the
  `DiffuseIonizedGasMix`, and the `FragmentDustMixDecorator` are averaged over overlapping
  entities, where they used to be taken from the entity with the largest weight. Conversely, the
  temperature of the `XRayIonicGasMixFamily` is now taken from the entity with the largest weight.
  This changes the results of simulations that import these media from smoothed particles.
- With more than one property sample per cell, a categorical quantity is no longer averaged.
- The `indices` property of the `CustomStateProbe` is renamed to `variables`, and the indices of
  the custom state variables shift for the mixes that store a temperature or a metallicity.
- The `ImportedMediumTemperatureProbe` and the `ImportedMediumMetallicityProbe` no longer produce
  output for dust media.
- Material mixes maintained outside the SKIRT repository must be adapted to the changed
  interfaces, listed in the Implementation chapter.

The PTS ski file upgrade removes the three properties from all imported media and geometries, and
renames the property of the `CustomStateProbe`. For each imported medium, it removes the
`importVariableMixParams` property, renames a `materialMixFamily` element to `materialMix`, and
adjusts the base type of the `materialMix` element. It cannot convert import files. It reports
each medium and geometry that had `importMetallicity` or `importTemperature` enabled, so that the
user can convert or remap the corresponding file.
