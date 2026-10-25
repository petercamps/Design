# Open questions

**Names.** The proposal uses `EscapeInstrument` for the instrument, `binning` for its property, and names
ending in `EscapeBinning` for the binnings. Alternatives are `EscapeSEDInstrument`, which emphasizes that it
records spectra, or names that refer to the sky, such as `SkyBins`. The output style names `SEDFiles` and
`DataCubes` are equally provisional.

**Polarization.** The Stokes vector of an escaping photon packet is defined relative to a reference
direction perpendicular to its direction of travel. Adding the Stokes vectors of the packets in a bin
requires a common reference direction for each bin, for example the projection of the z axis, and a
rotation of each contribution into that frame. For bins that cover a wide range of directions, the
resulting polarization is hard to interpret, so that polarization is left out of the first version.

**Arrival times.** Time-dependent simulations record arrival times with light curve and spectral-time map
instruments. An escape instrument could record the time at which a packet leaves the model, corrected for
the light travel time to a distant observer in its direction. This is not needed for the current use cases.

**Statistics buffers.** With statistics, the flux recorder buffers the contributions of the current photon
packet history per thread and sorts them at the end of the history. An escape instrument contributes at
every step of every packet, so that these buffers grow larger than for peel-off instruments. Whether this
affects the performance must be measured; if so, the contributions could be merged per bin as they arrive.

**Checkpointing.** The [Checkpointing](checkpointing/01-introduction.md) design note must include the state
of escape instruments. Because they keep their state in a flux recorder, like all other instruments, this
should require no special treatment.

**Other binnings.** Further binnings could be added later, for example bins of line-of-sight column
density and inclination, as the sky bins of XARS used for UXClumpy, with each direction assigned to a bin
through a map of the column density from a reference position calculated during setup; bins defined by a
list of cones around given directions; or bins defined by a table of directions and solid angles read from
a file.
