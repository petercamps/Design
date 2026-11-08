# Open questions

**Polarization.** The Stokes vector of an escaping photon packet is defined relative to a reference
direction perpendicular to its direction of travel. Adding the Stokes vectors of the packets in a bin
requires a common reference direction for each bin, for example the projection of the z axis, and a
rotation of each contribution into that frame. For bins that cover a wide range of directions, the
resulting polarization is hard to interpret, so that polarization is left out of this version.

**Arrival times.** Time-dependent simulations record arrival times with light curve and spectral-time map
instruments. An escape instrument could record the time at which a packet leaves the model, corrected for
the light travel time to a distant observer in its direction. This is not needed for the current use cases
and is not implemented.

**Frame of inclination and azimuth bins.** In the data cubes, the bins of the inclination binning lie in a
single row, also with more than one azimuth bin. A two-dimensional frame, with the azimuth bins along one
axis and the inclination bins along the other, would be easy to add, but PTS already arranges these bins
in a map from the bin table.

**Instrument names.** SKIRT does not check whether two instruments have the same name. Upgraded ski files
rely on this for the pairs of SED and frame instruments that replace a `FullInstrument`, which write
different files. Two instruments with the same name that write a file with the same name, such as a frame
instrument and an escape instrument with the data cube style, or two escape instruments, overwrite each
other's output without a message. A check during setup could turn this into an error.

**Other binnings.** Further binnings could be added later, for example bins of line-of-sight column
density and inclination, as the sky bins of XARS used for UXClumpy, with each direction assigned to a bin
through a map of the column density from a reference position calculated during setup; bins defined by a
list of cones around given directions; or bins defined by a table of directions and solid angles read from
a file.
