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

## Conclusion

None of these options is fully satisfactory. Suggestions for other approaches, and votes for one
of the options above, are welcome.
