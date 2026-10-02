# Wavelength grids

## Problems

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
   the DIWLG may produce output at many wavelengths, which is often not the intention. Configuring
   it for just one or a few wavelengths is awkward because it requires an explicit grid, for
   example a `ListWavelengthGrid`.

   Many other probes select a WLG on their own: the radiation field WLG (RFWLG), the dust
   emission WLG, or the instrument WLGs.

## Possible solutions

### Remove the DIWLG

Each instrument and probe configures its own WLG. This makes the configuration fully symmetric,
but it penalizes the simpler use cases that need just a single WLG.

### Stop using the DIWLG for probes

Each probe selects or configures its own WLG. The impact is much smaller. This solves problem 2
to some extent, but not problem 1, and it also penalizes the simple use cases, although to a
lesser extent.

### Add a WLG pool

Offer a pool of zero or more WLGs, placed at the same level as the instrument system and the
probe system, and before them. Instruments and probes then reference a WLG in the pool. This
raises two questions.

**Identifying a grid.** Identification by position (a 0-based or 1-based index) is simple to
implement but error prone. Identification by name would require either a name property on the WLG
class, which is unattractive because it puts a name on every WLG, even those outside the pool, or
a new SMILE dictionary type to use instead of a list, which would be complex.

**Referencing a grid.** There are two approaches:

- (a) Introduce a `ReferenceWavelengthGrid` that mentions the pool index or name. This still
  allows configuring any WLG directly, instead of a pool WLG.
- (b) Use a scalar property that holds the index or name. This requires **all** instrument and
  probe WLGs to be in the pool.

**Scope.** One could propose a pool for the whole simulation, but this causes extra
complications. For example, the RFWLG does not accept WLGs with overlapping bins.

## Other suggestions

### A named wavelength grid pool

This proposal makes the pool concrete, resolving the questions raised above.

**Pool contents.** A `WavelengthGridPool` item holds a list of named WLGs and the name of the
default grid. Instead of a name property on every WLG, each list entry is a small wrapper item that
combines a name with a WLG of any type:

```xml
<wavelengthGridPool type="WavelengthGridPool">
    <WavelengthGridPool defaultGridName="broad">
        <wavelengthGrids type="NamedWavelengthGrid">
            <NamedWavelengthGrid name="broad">
                <wavelengthGrid type="WavelengthGrid">
                    <LogWavelengthGrid minWavelength="0.1 micron" maxWavelength="1000 micron" numWavelengths="200"/>
                </wavelengthGrid>
            </NamedWavelengthGrid>
            <NamedWavelengthGrid name="zoom">
                ...
            </NamedWavelengthGrid>
        </wavelengthGrids>
    </WavelengthGridPool>
</wavelengthGridPool>
```

WLGs outside the pool carry no name, and the list is an ordinary item list, so SMILE needs no new
property type. The property holding the default name cannot simply be called `default`, because
the property macros generate a function with the property's name, and `default` is a C++ keyword.

**Referencing a grid.** A `ReferenceWavelengthGrid` with a `name` property is a regular WLG
subclass that forwards all requests to the named pool grid. It can be configured wherever an
instrument or probe accepts a WLG, as in approach (a), so that any WLG can still be configured
directly instead. The default grid can be referenced by its name like any other pool grid.

**Default grid.** The pool replaces the DIWLG of the instrument system. An instrument or probe
without a WLG of its own uses the pool grid named by `defaultGridName`. If this property is empty,
there is no default, and each instrument and probe must configure a WLG.

**Scope and placement.** The pool serves instruments and probes only, which are the consumers that
need several grids and repeat them. The RFWLG, the dust emission WLG, the grid-based wavelength
distributions, and the `CompositeWavelengthGrid` are each configured once, and require a WLG with
non-overlapping bins, a constraint that a general reference cannot guarantee. The pool is therefore
a property of the `MonteCarloSimulation`, placed just before the instrument system:

```
userLevel, random, units, simulationMode, iteratePrimaryEmission, iterateSecondaryEmission,
numPackets, cosmology, sourceSystem, mediumSystem, wavelengthGridPool, instrumentSystem,
probeSystem
```

Like the DIWLG today, the pool is relevant only in panchromatic simulations.

**SMILE conditions.** A nonempty item list and a nonempty string property each insert a name that
can be used in SMILE conditions. Using `ATTRIBUTE_INSERT` to make these names global:

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

**Validation.** Neither MakeUp nor the schema can verify names. During setup, the pool reports a
fatal error for duplicate names and for a default name that does not occur in the list, and a
reference reports a fatal error for an unknown name; each of these errors lists the available
names. During setup, the order of the properties does not matter, because a reference locates the
pool through `find()`, which sets up the pool when needed.

**Effect on the problems.** The pool solves problem 1: several grids can be defined once and
referenced from any number of instruments. By itself, it does not change problem 2, because probes
still fall back on the default grid. Because the DIWLG moves from the instrument system to the
pool, existing ski files must be upgraded, as described in the SKIRT 10
[Incompatibilities](skirt-10/04-incompatibilities.md) chapter. The proposal also removes the
dependency of the pool on the treatment of irrelevant options (see the
[Irrelevant options](ski-file-issues/02-irrelevant-options.md) chapter).

### Instruments with multiple sight lines

The repetition in problem 1 stems from the coupling of the viewing direction and the instrument
configuration. An instrument that accepts a list of viewing directions, for example several
inclinations or azimuths, would configure its WLG once and produce separate output for each
direction. This solves the repetition even without a pool. It would also benefit models that need
many sight lines with identical instrument settings.

## Conclusion

None of these options is fully satisfactory. Suggestions for other approaches, and votes for one
of the options above, are welcome.
