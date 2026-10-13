# Open questions

## Design decisions

### Uniform import

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

**Finding the temperature for line emission.** The `LineGasSecondarySource` adds thermal motion to
the emitted lines using the state variable named `T`. A virtual function of the mix
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
for an intensive variable; for an extensive variable the aggregate is the plain sum. Its examples
build the variables of a mix as a vector; with this proposal, a mix adds them to its variable list
instead.

**Dynamic grid refinement.** That note selects state variables by the names proposed here. The
kinds proposed here replace its requirement that all state variables be intensive: an extensive
variable is divided among the child cells in proportion to their volume.

**Dust self-heating.** The arrays with the absorbed luminosity per cell are not part of the medium
state, and are not affected.

**Checkpointing.** The datasets for the metallicity and the temperature of a component disappear,
since both become custom variables. An array variable is more naturally stored as a single
two-dimensional dataset than as one dataset per element, and the name of a variable can be stored
as an attribute.
