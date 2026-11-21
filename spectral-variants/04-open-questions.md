# Open questions

**Several sources with variants.** Any source with a single SED could have variants, if all of them have the same
number K, so that variant k of the simulation combines variant k of each of these sources. The photon packet already
keeps the index of its source, and the source system would forward the factors to the source of each history. The
first source of a model with an AGN and its host galaxy suffices for the current use cases.

**Variants given by other kinds of SED.** Instead of the columns of its file, a source could get a list of SED items as
its variants, such as black bodies or analytic power laws, each normalized like the source's own SED. This needs a
property of the source and a coverage check on a fine wavelength grid, since the variants no longer share the
tabulated wavelengths of the source's own SED, and it makes the ski file grow with the number of variants.

**A choice of instruments.** The switch makes every SED and escape instrument record the variants. A property of
each instrument could exclude some of them, to save memory when only a few sight lines need the variants.

**Response matrix.** XARS records a response matrix from injected to escaping energy, from which the spectrum for any
incident spectrum follows after the simulation. In SKIRT, an instrument could likewise record each contribution in a
bin of launch wavelength as well as of detection wavelength. This allows arbitrary source spectra later, but the
memory grows with the product of the two grids, 25 371 × 25 371 values per line of sight for the clumpy torus model,
and the source spectrum is then approximated as constant within each launch bin. With coarse launch bins, the matrix
would be manageable but the approximation would affect the transmitted continuum and the lines.

**Fewer recorded spectra.** For a smooth family of variants, such as cutoff power laws over a grid of photon index
and cutoff energy, the K × N<sub>λ0</sub> table of factors r<sub>k</sub>(λ<sub>0</sub>) is close to a matrix of low
rank. The recorder could record M basis spectra, with factors given by the leading singular vectors of that table, and
compose the K variants from them when writing the output. With M well below K, this saves memory and time, at the
price of a small, controllable approximation.

**Sampling density instead of the source's own SED.** The factors are defined relative to the source's own specific
luminosity, the second column of the variants file, so that variants must not emit where the source's own SED does
not. Storing the probability density of the
launch wavelength in the photon packet would lift this condition: the variant luminosity of a launch is then
L<sub>k,s,λ</sub>(λ<sub>0</sub>) / (N<sub>s</sub> p<sub>s</sub>(λ<sub>0</sub>)), whatever the source's own SED.

**Variant statistics.** The statistics of the variants take four arrays per variant and most of the recording work.
They could be recorded for a selection of variants only, or be left out, with the statistics of the regular output as
a guide.

**Imported sources.** Sources imported from particles or cells have an SED for each entity, from an SED family. A
variant could replace the family or change its parameters for all entities, with factors that depend on the emitting
entity. This needs the entity index in the photon packet and a factor calculation per entity.

**Other variant properties.** The same reweighting applies to the angular distribution of a source, with factors that
depend on the launch direction, and to the luminosity alone. Neither is needed for the current use cases.

**Oligochromatic simulations and other instruments.** Variants could be supported in oligochromatic simulations, for
which the factors are evaluated at the discrete wavelengths, and in instruments with data cubes, at the cost of their
memory. Polarization and scattering levels could be recorded for the variants in the same way as the flux components.
