# Open questions

**Several sources with variants.** Any source with a single SED could have variants, if all of them have the same
number K, so that variant k of the simulation combines variant k of each of these sources. Each of these sources would
launch its photon packets from the envelope of its own variants, the photon packet already keeps the index of its
source, and the source system would forward the variant luminosities to the source of each history. The first source
of a model with an AGN and its host galaxy suffices for the current use cases.

**Variants given by other kinds of SED.** Instead of the columns of its file, a source could get a list of SED items as
its variants, such as black bodies or analytic power laws, each normalized with the normalization of the source. This
needs a property of the source, and an envelope calculated on a fine wavelength grid, since the variants no longer
share tabulated wavelengths, and it makes the ski file grow with the number of variants.

**A launch spectrum of the user's choice.** SKIRT calculates the envelope of the variants automatically. A property of
the source could instead let the user supply the spectrum term of the launch distribution, for example for a set of
variants whose envelope covers much more than the wavelengths where the model needs photon packets. The guarantee that
each variant is sampled at least as well as in its own simulation would then be lost.

**A choice of instruments and quantities.** The switch makes every SED and escape instrument record the variants,
with all the quantities it would record without variants. A property of each instrument could exclude some
instruments, or restrict the variants to the total flux, to save memory when only a few sight lines need the variants,
or when the flux components, scattering levels, or polarization are needed for a single variant only.

**Response matrix.** XARS records a response matrix from injected to escaping energy, from which the spectrum for any
incident spectrum follows after the simulation. In SKIRT, an instrument could likewise record each contribution in a
bin of launch wavelength as well as of detection wavelength. This allows arbitrary source spectra later, but the
memory and the output grow with the product of the two grids, 20 000 × 20 000 values per line of sight for the example
of the Features chapter, and the source spectrum is then approximated as constant within each launch bin. With coarse
launch bins, the matrix would be manageable but the approximation would affect the transmitted continuum and the
lines. Moreover, the user would have to convolve the matrix with each source spectrum, normalize the result, and
interpolate between the binning of the matrix and that of the spectra, work that the variants leave to SKIRT. When the
source spectra of interest are known in advance, as is usually the case, the variants are the simpler choice.

**Fewer recorded spectra.** For a smooth family of variants, such as cutoff power laws over a grid of photon index
and cutoff energy, the K × N<sub>λ0</sub> table of specific luminosities L<sub>k,λ</sub>(λ<sub>0</sub>) is close to a
matrix of low rank. The recorder could record M basis spectra, with launch luminosities given by the leading singular
vectors of that table, and compose the K variants from them when writing the output. With M well below K, this saves
memory and time, at the price of a small, controllable approximation.

**Variant statistics.** The statistics of the variants take four arrays per variant and most of the recording work.
They could be recorded for a few representative variants only, as a guide for the others.

**Imported sources.** Sources imported from particles or cells have an SED for each entity, from an SED family. A
variant could replace the family or change its parameters for all entities, with variant luminosities that depend on
the emitting entity. This needs the entity index in the photon packet, a calculation of the variant luminosities for
each entity, and an envelope for the launch distribution of each entity.

**Other variant properties.** The same reweighting applies to the angular distribution of a source, with weights that
depend on the launch direction, and to the luminosity alone. Neither is needed for the current use cases.

**Oligochromatic simulations and other instruments.** Variants could be supported in oligochromatic simulations, for
which the variant luminosities are evaluated at the discrete wavelengths, and in instruments with data cubes, at the
cost of their memory.
