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

## Conclusion

Both behaviors have merit. Ideas, and votes on whether to keep including irrelevant scalar
properties in ski files, are welcome.
