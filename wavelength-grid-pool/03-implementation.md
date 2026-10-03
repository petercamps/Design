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

- the nonempty `wavelengthGrids` list inserts a condition that allows the
  `ReferenceWavelengthGrid` type, so that MakeUp offers it only when there is something to
  reference;
- the nonempty `defaultGridName` property inserts the existing `DefaultInstrumentWavelengthGrid`
  condition, so that instruments and probes keep their current rule: a WLG of their own is required
  only if there is no default.

Because a condition inserted by a property only affects the properties that follow it, the
`wavelengthGrids` list precedes the `defaultGridName` property within the pool, so that the user
first names the grids and then selects the default. For the same reason, the condition allowing
references is not yet in effect while the pool itself is being configured, so that a pool grid
cannot reference another pool grid, which excludes reference cycles.

## Validation

Neither MakeUp nor the schema can verify names. During setup, the pool reports a fatal error for
duplicate names and for a default name that does not occur in the list, and a reference reports a
fatal error for an unknown name; each of these errors lists the available names. During setup, the
order of the properties does not matter, because a reference locates the pool through `find()`,
which sets up the pool when needed.
