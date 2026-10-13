# Open questions

## Design decisions

### Uniform import

**Optional quantities.** The proposal imports every quantity that a material mix requests. Today
the temperature and the metallicity are optional for most gas mixes, which fall back on a default
value. With the proposal, a mix that wants to keep a quantity optional offers its own property for
it, and its default values otherwise serve geometric media only. Alternatives are a property of
the imported medium that lists the quantities to import, which reintroduces the question of which
combination a mix expects, or a convention by which a mix replaces a value of zero by its default,
so that a user can map the column to the virtual zero column that `useColumns` offers.

**The dust recipe.** The proposal removes the derivation of the dust mass from the metallicity and
the temperature of the gas. This is the most disruptive change, because nearly every pipeline that
processes a cosmological simulation relies on it. The conversion is simple, and the converted file
reproduces the previous results exactly, but every such pipeline must be adapted. The alternative
is to keep the recipe for dust media as an explicit feature of its own, with a metallicity column,
a temperature column, and a maximum temperature that have no other meaning. This keeps existing
pipelines working at the cost of a second meaning for the same two quantities, which is the
confusion the proposal sets out to remove.

**Which quantities each mix requests.** The table in the Features chapter keeps the quantities
that each mix can import today, and makes them all required. The authors of the mixes may prefer
otherwise. The `DiffuseIonizedGasMix`, for example, calculates its own temperature and uses the
imported one only as a starting value, so it could leave the temperature out.

**One descriptor or two.** The proposal keeps `SnapshotParameter` for imported quantities and
`StateVariable` for stored ones. Most imported quantities end up in a state variable of the same
meaning, so a mix could instead declare each state variable once and mark it as imported. The
medium system would then derive the import list and copy the imported values into the state,
leaving to the mix only the variables it derives. This removes the duplication in most mixes. It
does not fit the mixes that transform a quantity before storing it, such as the `NonLTELineGasMix`
with its turbulence velocity, nor the families, which import quantities without storing them. The
proposal therefore keeps the two descriptors and the explicit initialization.

**Names for imported quantities.** Imported quantities get no short name. Their description
already serves as the column name for `useColumns`, and the standard identifiers let other code
find the temperature and the metallicity.

### Kinds

**Extensive imported quantities.** The proposal does not support importing an extensive quantity,
such as the mass of a species in each particle. Supporting it requires summing the contributions
of the overlapping entities into a density, as for the mass itself, and multiplying by the cell
volume. No current mix needs this; a mix can import the corresponding intensive quantity, such as
a number density or an abundance ratio, instead.

**One attribute or two.** The kind combines two properties: whether a quantity scales with the
amount of material, and whether its values can be averaged. They could be declared separately,
which would allow a quantity that is extensive and discrete, such as a count. The proposal uses a
single attribute with three values, because these cover all current quantities.

**A categorical quantity with several samples.** With more than one property sample per cell, the
proposal takes a categorical quantity from the sample with the highest density. The value at the
center of the cell is an alternative that does not depend on the random samples.

**Standard names.** Only the temperature and the metallicity have a standard name, because other
code needs to find them. Further standard names can be added when another quantity is used
outside its own mix.

**Finding the temperature for line emission.** The `LineGasSecondarySource` adds thermal motion to
the emitted lines using the state variable named `temperature`. A virtual function of the mix
that returns the temperature for this purpose would avoid relying on the name, at the cost of one
more function in the interface.

### Capabilities

**Virtual functions.** The proposal keeps the virtual capability functions of `MaterialMix`, and
fills the capabilities structure from them. Replacing them by a single virtual function that
returns the structure is cleaner, but requires changing every mix that overrides one of them.

**State variables of a family.** The list of state variables of a medium with a family is still
taken from a representative member. This happens after the media have been set up, so it does not
require constructing a mix early. Letting the family declare the list would remove the last need
for a representative member during setup of the medium system.

### Staged updates

**Where the shared data live.** The proposal lets a mix keep the data shared between the stages
in `mutable` data members. Alternatively, stage 2 could return an object with these data, which
the medium system keeps and passes to the later stages. This avoids mutable members, but the mix
must cast the object back to its own type in every call.

**Recipes.** A dynamic state recipe already has a serial function before and after its pass over
the cells. It could be given a preparation stage as well, so that mixes and recipes follow the
same protocol.

**Stages for a family.** The serial stages are called on the mix of the first cell. A family whose
members need them must make the shared data reachable from all its members.

### Array access

**Own class or standard class.** The proposal adds a minimal `Span` class, because SKIRT 10 moves
to C++17 and `std::span` requires C++20. The class can be replaced by the standard one when SKIRT
adopts that standard.

**Avoiding const_cast.** The proposal recommends that a mix assemble its list of state variables
during setup, which removes the most common reason for casting away constness. Whether the coding
conventions should discourage `const_cast` in general is a separate question.

### Storage

**Compact storage.** The medium state allocates the variables of a component for every spatial
cell, including the cells in which the component has no material. Storing the variables other
than the number density only for the cells that hold material would save memory for a component
with many variables per cell that fills only part of the grid. This is not proposed, because it
complicates the setup of the state, the handling of cells whose density changes during the
simulation, and the growth of the state under dynamic grid refinement, and because it adds an
indirection to the access of these variables in the photon life cycle.

## Relation to other design notes

The following design notes touch the same classes, and need to be read with this proposal in mind.

**Tree-based spatial grids.** The `GasThermalEnergyTreePolicy` uses the temperature imported by
the gas media. With this proposal it finds the temperature among the quantities requested by the
material mix, instead of relying on the `importTemperature` option, and it also works for a medium
with a material mix family.

**Iteration history.** That note removes the aggregate states from the medium state, and marks
state variables for aggregation with `aggregated()`, in the same style as the attributes proposed
here. The volume integral it calculates sums a variable times the cell volume. It is meaningful
for an intensive variable; for an extensive variable the aggregate is the plain sum.

**Dynamic grid refinement.** That note selects state variables by the names proposed here. The
kinds proposed here replace its requirement that all state variables be intensive: an extensive
variable is divided among the child cells in proportion to their volume.

**Dust self-heating.** The arrays with the absorbed luminosity per cell are not part of the medium
state, and are not affected.

**Checkpointing.** The datasets for the metallicity and the temperature of a component disappear,
since both become custom variables. An array variable is more naturally stored as a single
two-dimensional dataset than as one dataset per element, and the name of a variable can be stored
as an attribute.
