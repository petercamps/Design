# Irrelevant options

This chapter is about ski file properties that, according to the SMILE schema, are irrelevant
because of an earlier configuration choice, and about whether ski files should include them.

## Current behavior

When SKIRT or MakeUp writes a ski file in XML format:

- Irrelevant compound properties are omitted. This is desirable: for example, a `NoMedium`
  simulation should not show a `MediumSystem`.
- Irrelevant scalar properties are included. This has pros and cons.

## Pros and cons

### Con: confusing to occasional users

Beginning and occasional users in particular may wonder why an irrelevant property is there and
what it does. For example:

```xml
<SourceSystem minWavelength="1 micron" maxWavelength="10 micron"
              wavelengths="0.55 micron, 0.77 micron" sourceBias="0.5">
```

Without context, it is unclear whether this source system uses a range from 1 to 10 micron or two
specific wavelengths. The answer depends on the simulation mode, configured elsewhere in the file.

### Pro: easier editing

If a property becomes relevant because the property controlling it changes, it is much easier to
edit the previously irrelevant property. For example:

```xml
<IntegratedLuminosityNormalization wavelengthRange="All" minWavelength="0.09 micron"
                        maxWavelength="100 micron" integratedLuminosity="1 Lsun"/>
```

With `wavelengthRange="All"`, the minimum and maximum wavelengths are irrelevant. When editing the
ski file by hand to change this to `Custom`, however, it is a lot easier to adjust the custom range
than to remember and type the names `minWavelength` and `maxWavelength`.

## Relation to wavelength grids

The question also arises for the wavelength grid pool considered in the
[Wavelength grids](ski-file-issues/01-wavelength-grids.md) chapter. Identifying pool WLGs by name
would require a name property on every WLG. That property might be made irrelevant for WLGs
configured outside the pool, which would hide it, provided irrelevant scalar properties are no
longer written to the ski file.

## Other suggestions

### Omit irrelevant scalar properties, and compensate for editing

A middle road omits irrelevant scalar properties from the ski files written by SKIRT and MakeUp,
and makes up for the loss of editing convenience in other ways:

- When a hand-edited ski file lacks a property that has become relevant and has no default value,
  the error message names the missing property and lists the valid property names for the element.
  A property with a default value is silently assigned that value, as now.
- A MakeUp preference or a SKIRT command-line option writes complete ski files, including the
  irrelevant scalar properties, for users who edit ski files by hand frequently.
- MakeUp keeps the values of irrelevant properties in memory during a session, so that switching
  a choice back and forth still preserves the values entered by the user.

Before adopting this, two consequences need to be checked. PTS scripts that read attributes of
irrelevant properties would break, although scripts that set attributes are not affected, because
setting an attribute creates it. And resaving the ski files of the tutorials and functional tests
would produce large differences, so that a one-time reformatting of these files is worth planning.

## Conclusion

Ideas, and votes on whether to keep including irrelevant scalar
properties in ski files, are welcome.
