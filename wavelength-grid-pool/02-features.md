# Features

## Pool contents

A `WavelengthGridPool` item holds a list of named WLGs and the name of the default grid. Instead of
a name property on every WLG, each list entry is a small wrapper item that combines a name with a
WLG of any type:

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

## Referencing a grid

A `ReferenceWavelengthGrid` with a `name` property is a regular WLG subclass that forwards all
requests to the named pool grid. It can be configured wherever an instrument or probe accepts a
WLG, so that any WLG can still be configured directly instead. The default grid can be referenced
by its name like any other pool grid.

During setup, the pool reports a fatal error for duplicate names and for a default name that does
not occur in the list, and a reference reports a fatal error for an unknown name.

## Default grid

The pool replaces the DIWLG of the instrument system. An instrument or probe without a WLG of its
own uses the pool grid named by `defaultGridName`. If this property is empty, there is no default,
and each instrument and probe must configure a WLG.

## Scope and placement

The pool serves instruments and probes only, which are the consumers that need several grids and
repeat them. The radiation field WLG, the dust emission WLG, and the grid-based wavelength
distributions require a WLG with non-overlapping bins, a constraint that a general reference cannot
guarantee. The pool is therefore a property of the `MonteCarloSimulation`, placed just before the
instrument system.

Like the DIWLG today, the pool is relevant only in panchromatic simulations.

## Effect on the problems

The pool solves problem 1: several grids can be defined once and referenced from any number of
instruments. By itself, it does not change problem 2, because probes still fall back on the default
grid. However, a user can opt to define and reference pool WLGs without assigning a default WLG,
avoiding the silent fallback.

## Compatibility

Because the DIWLG moves from the instrument system to the pool, existing ski files must be
upgraded, as described in the SKIRT 10 [Incompatibilities](skirt-10/04-incompatibilities.md)
chapter.
