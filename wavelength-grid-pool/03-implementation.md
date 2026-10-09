# Implementation

## Names

WLGs outside the pool carry no name, and the list is an ordinary item list, so SMILE needs no new
property type. The property holding the default name cannot simply be called `default`, because
the property macros generate a function with the property's name, and `default` is a C++ keyword.

Identifying a pool grid by its position in the list, rather than by name, would be simpler to
implement but error prone. Requiring every instrument and probe to reference a pool grid, through
a scalar property holding the name, would force even a single-use grid into the pool.

## SMILE conditions

A nonempty item list and a nonempty string property each insert a name that can be used in SMILE
conditions. Using `ATTRIBUTE_INSERT` to make these names global:

- the nonempty `wavelengthGrids` list inserts the name `PoolWavelengthGrids`, which allows the
  `ReferenceWavelengthGrid` type, so that MakeUp offers it only when there is something to
  reference;
- the nonempty `defaultGridName` property inserts the existing `DefaultInstrumentWavelengthGrid`
  condition, so that instruments and probes keep their current rule: a WLG of their own is required
  only if there is no default.

Because a condition inserted by a property only affects the properties that follow it, the
`wavelengthGrids` list precedes the `defaultGridName` property within the pool, so that the user
first names the grids and then selects the default.

An item list, however, inserts its name as soon as its first item is added, before that item is
configured, so that `PoolWavelengthGrids` is already in effect while the pool grids themselves are
configured. To prevent a pool grid from referencing another pool grid, which excludes reference
cycles, the `ReferenceWavelengthGrid` type also requires the type name of the instrument system or
the probe system: it is allowed if `PoolWavelengthGrids&(InstrumentSystem|ProbeSystem)`. Each
instrument and probe is configured inside one of these systems, which follow the pool, and the type
name of a system is inserted when the system is added. Both names are needed, because a ski file may
omit the instrument system, which is then added with its default value only after the probe system
has been read.

The pool is displayed from the Regular user level on (`Level2`), and it is optional. The WLG
property of instruments and probes is displayed if `Level2|!DefaultInstrumentWavelengthGrid`, so
that a Basic user, who has no pool, configures it. When a WLG is required, the instruments suggest a
`LogWavelengthGrid` (default value `!DefaultInstrumentWavelengthGrid:LogWavelengthGrid;`), while
the probes suggest nothing, so that the user selects their WLG deliberately.

The default value of `defaultGridName` is empty. When reading a ski file, SMILE treats an empty
string attribute as a missing attribute and assigns the property's default value, so that with a
nonempty default value, an empty default name could not be read back.

## Validation

Neither MakeUp nor the schema can verify names. During setup, the pool reports a fatal error for
duplicate names and for a default name that does not occur in the list, and a reference reports a
fatal error for an unknown name; each of these errors lists the available names. A reference also
reports a fatal error if it is located inside the pool, which is possible only in a ski file with a
manually changed element order. During setup, the order of the properties does not matter, because
a reference locates the pool through `find()`, which sets up the pool when needed.

## Resolving references

A `ReferenceWavelengthGrid` forwards all requests to the referenced grid, which serves clients such
as the functions that determine the wavelength range of the simulation. The instruments and probes,
however, obtain their WLG through `Configuration::wavelengthGrid()`, which returns the referenced
grid itself rather than the reference. This preserves the behavior that depends on the type of the
grid, such as the faster detection for grids with non-overlapping bins in `FluxRecorder`, the band
convolution in `ImportedSourceLuminosityProbe`, and the bin widths for reversed output units in
`InstrumentWavelengthGridProbe`. Because the `tools` target, which holds `Configuration`, cannot know
the reference type, this function is implemented in the `ConfigurationSetup` subclass.
