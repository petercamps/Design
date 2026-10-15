# Implementation

## Descriptors

### Kind

A new enumeration describes the kind of a quantity. It lives in the `utils` module, next to
`SnapshotParameter`, so that both descriptor classes can use it.

```cpp
/** The kind of a quantity that is imported from a snapshot or stored in the medium state. An
    intensive quantity is continuous and does not depend on the amount of material; a categorical
    quantity is a label or an index; an extensive quantity is proportional to the amount of
    material. */
enum class QuantityKind { Intensive, Categorical, Extensive };
```

### Snapshot parameter

`SnapshotParameter` gains a kind, which is intensive unless stated otherwise:

```cpp
/** Returns a copy of this parameter marked as categorical. */
SnapshotParameter categorical() const;

/** Returns the kind of this parameter. */
QuantityKind kind() const;
```

A material mix family that selects a mix by index declares
`SnapshotParameter::custom("dustmix index").categorical()`. The class offers no way to mark a
parameter as extensive, because a material mix cannot request an extensive quantity (see the
Features chapter). The existing identifiers, including `Metallicity` and `Temperature`, are kept:
they let other code recognize a standard parameter in the list of a material mix, and SED families
use them as today.

### State variable

`StateVariable` loses the identifiers `Metallicity` and `Temperature` and their factory functions.
A custom variable gains a name, a kind, and a number of elements:

```cpp
/** Returns a copy of this custom variable with the specified short name. The name consists of
    letters, digits, and underscores, and must be unique among the variables of a material mix. */
StateVariable named(const string& name) const;

/** Returns a copy of this custom variable marked as categorical, or as extensive. A variable is
    intensive unless marked otherwise. */
StateVariable categorical() const;
StateVariable extensive() const;

/** Returns a copy of this custom variable that represents an array with the specified number of
    elements. The variable occupies that number of consecutive custom indices, starting at its own
    index. */
StateVariable array(int count) const;

/** Return the name (empty if none), the kind, and the number of elements (one for a scalar). */
const string& name() const;
QuantityKind kind() const;
int count() const;
```

These functions follow the pattern of the `aggregated()` function proposed in the Iteration
history design note, so that the attributes can be combined. The factory function `custom()` gains
an overload without an index, for a variable that receives its index when it is added to the
variable list of a mix (see Declaring and accessing variables below):

```cpp
StateVariable::custom("hydrogen neutral fraction", "").named("x_HI")
StateVariable::custom("extinction opacity", "").named("kappa_ext").array(n)
```

Two factory functions create the custom variables with the standard names, so that all mixes
declare them in the same way:

```cpp
/** Return a custom variable that holds the temperature or the metallicity of the material,
    with the standard name (T or Z), description, and quantity. */
static StateVariable temperature();
static StateVariable metallicity();
```

The function `specificStateVariableInfo()` of `MaterialMix` returns a reference to a variable
list, described under Declaring and accessing variables below, instead of a vector that is built
for each call:

```cpp
/** Returns the specific state variables of this material mix. The default implementation
    returns a list that holds just the number density. */
virtual const StateVariableList& specificStateVariableInfo() const;
```

The function is called several times, by the medium system and by probes. Returning a reference
avoids a copy for each call, and requires the list to outlive the call. A mix therefore assembles
its list of variables once, during setup, and keeps it. This also removes the need to cast away
the constness of the mix, which is how most mixes record the indices today.

## Material mix interface

The temperature and the metallicity leave the signatures:

```cpp
// MaterialMix
virtual vector<SnapshotParameter> parameterInfo() const;
virtual void initializeSpecificState(MaterialState* state, const Array& params) const;

// MaterialMixFamily
virtual const MaterialMix* mix(const Array& parameters) = 0;
```

A mix that needs the temperature lists `SnapshotParameter::temperature()` in `parameterInfo()`,
at the position where it wants the column, and finds the value at that position in `params`. The
array is empty for a geometric medium, in which case the mix uses its default values, as today.
The ban on requesting the temperature or the metallicity as a parameter is lifted.

`MaterialState` loses `temperature()`, `metallicity()`, and their setters. A mix reads and writes
these quantities as custom variables, through the handle it recorded (see Declaring and accessing
variables below).

## Common base class

`MaterialMix` and `MaterialMixFamily` derive from a new abstract class, so that an imported medium
can hold either of them in a single property:

```cpp
/** MaterialMixProvider is the abstract base class of MaterialMix and MaterialMixFamily. */
class MaterialMixProvider : public SimulationItem
{
    ITEM_ABSTRACT(MaterialMixProvider, SimulationItem, "a material mix or a family of material mixes")
    ITEM_END()

public:
    /** Returns the descriptors of the quantities to be imported for this mix or family. */
    virtual vector<SnapshotParameter> parameterInfo() const = 0;

    /** Returns the capabilities of this mix, or of all members of this family. */
    virtual Capabilities capabilities() const = 0;

    /** Returns true if the imported quantities select a material mix for each position, as for
        a family, and false if they initialize the state of a single mix. */
    virtual bool hasVariableMix() const = 0;

    /** Returns the material mix for the specified values of the imported quantities. */
    virtual const MaterialMix* mix(const Array& parameters) = 0;

    /** Returns a representative material mix. */
    virtual const MaterialMix* mix() = 0;
};
```

- A `MaterialMix` has no variable mix, and returns itself from both `mix()` functions. Its
  `parameterInfo()` keeps the default implementation, which returns an empty list.
- A `MaterialMixFamily` has a variable mix. Its subclasses implement the other functions, as
  today.
- The enumerations `MaterialType` and `DynamicStateType` move from `MaterialMix` to the base
  class, together with the capabilities structure described below. They remain accessible through
  `MaterialMix`, so that existing code is not affected.

`ImportedMedium` loses the properties `importVariableMixParams` and `materialMixFamily`. Its
`materialMix` property holds a `MaterialMixProvider`, to which the medium forwards
`capabilities()`, `parameterInfo()`, `hasVariableMix()`, and the selection of a mix.
`GeometricMedium` keeps a `materialMix` property that holds a `MaterialMix`. The conditions under
which the option was displayed move to the family classes, as conditions on the type.

## Capabilities

The capabilities of a material mix are gathered in a small structure, nested in
`MaterialMixProvider`:

```cpp
/** Describes the capabilities of a material mix, or of all members of a material mix family. */
struct Capabilities
{
    MaterialType materialType{MaterialType::Dust};
    bool polarizedScattering{false};
    bool polarizedAbsorption{false};
    bool polarizedEmission{false};
    bool resonantScattering{false};
    bool negativeExtinction{false};
    bool stochasticDustEmission{false};
    bool extraSpecificState{false};
    bool scatteringDispersion{false};
    bool scatteringEmulatesSecondaryEmission{false};
    bool continuumEmission{false};
    bool lineEmission{false};
    DynamicStateType dynamicMediumState{DynamicStateType::None};

    bool operator==(const Capabilities& other) const;
};
```

- `MaterialMix` implements `capabilities()` once, for all mixes. It fills the structure from the
  existing virtual functions, such as `materialType()` and `hasPolarizedScattering()`, which
  remain. The existing mixes therefore need no change.
- Each family implements `capabilities()` itself, from its own configuration, without
  constructing a mix. The `SelectDustMixFamily` returns the capabilities of its first configured
  mix, and verifies at setup that the others are equal. The `XRayIonicGasMixFamily` derives them
  from its properties, and no longer forces its setup ahead of time.
- `Medium::capabilities()` is a new pure virtual function. A geometric medium forwards it to its
  mix, and an imported medium to its mix or family.

`ConfigurationSetup` replaces its calls to `medium->mix()->hasXxx()` by
`medium->capabilities()`. The other code that asks a representative mix for the material type
does the same. The remaining uses of the representative mix returned by `Medium::mix()`, such as
the mass per entity and the list of state variables, occur after the media have been set up.

When the medium system obtains the mix of each cell for a medium with a family, it verifies for
each distinct mix that its capabilities equal those of the medium and that it declares the same
state variables as the representative mix, and reports a fatal error otherwise.

## Import

### Snapshot

- The functions `importMetallicity()` and `importTemperature()` are removed.
- The mass density policy keeps only its multiplier: `setMassDensityPolicy(double multiplier)`.
  The temperature cutoff and the metallicity multiplier disappear from
  `calculateDensityAndMass()`, from the row filter of `ParticleSnapshot`, and from the logged mass
  statistics, together with the functions `maxTemperature()`, `useMetallicity()`, and
  `useTemperatureCutoff()`.
- `importParameters()` remembers the kind of each parameter. It still records the column of each
  standard parameter, so that `hasTemperature()`, `temperature(m)`, and their metallicity
  counterparts keep working for an individual entity, which the probes of the input model use.
- `parameters(Position, Array&)` combines the values of the overlapping entities according to the
  kind of each parameter. In a single pass over the entities at the position, it accumulates the
  weighted sum of the intensive parameters, with the weight that `temperature(Position)` uses
  today, and remembers the entity with the largest weight for the categorical ones.
- The functions `metallicity(Position)` and `temperature(Position)` are removed.

### Medium

`Medium` loses `hasMetallicity()`, `metallicity()`, `hasTemperature()`, and `temperature()`, and
gains, next to `capabilities()`:

```cpp
/** Returns the descriptors of the quantities that this medium imports for its material mix or
    material mix family, or an empty list if there are none. */
virtual vector<SnapshotParameter> parameterInfo() const = 0;
```

`parameters(Position, Array&)` returns the values of these quantities, now also for a medium with
a material mix family. The medium system passes them to `initializeSpecificState()` only for a
medium with a single mix, as today.

`ImportedMedium` and `ImportedGeometry` lose the three properties and the corresponding calls,
and no longer distinguish dust from gas when setting the mass density policy.

### Sampling in a cell

The property sampler of the medium system loses its functions for metallicity and temperature.
Its function for the parameters averages the samples of an intensive parameter, weighted by
density, and takes a categorical parameter from the sample with the highest density.

## Staged updates

A material mix opts in to the staged protocol through a new virtual function. The existing
`updateSpecificState()` becomes the third of four stages:

```cpp
/** Returns true if the state of this material mix is updated in stages. The default
    implementation returns false. */
virtual bool hasStagedStateUpdate() const;

/** Stage 1, called for each cell, in parallel. Prepares the update of the specified cell, and
    may store in the specific state of that cell the information that the next stage needs. */
virtual UpdateStatus prepareSpecificStateUpdate(MaterialState* state, const Array& Jv) const;

/** Stage 2, called once, serially, on each process, after the state has been synchronized. The
    function has access to the state of all cells for the specified medium component. */
virtual void beginSpecificStateUpdate(const MediumState& state, int h) const;

/** Stage 3, called for each cell, in parallel, as today. */
virtual UpdateStatus updateSpecificState(MaterialState* state, const Array& Jv) const;

/** Stage 4, called once, serially, on each process, after the state has been synchronized. */
virtual void endSpecificStateUpdate(const MediumState& state, int h) const;
```

For the components that opt in, `MediumSystem::updateDynamicStateMedia()` performs an extra
parallel pass over the cells for stage 1, synchronizes the state, and calls stage 2. It then
performs the existing pass for stage 3, for all components with a dynamic state, synchronizes,
calls stage 4 for the components that opted in, and finally calls `isSpecificStateConverged()` as
today. A component that does not opt in is updated exactly as today.

The stages communicate through two channels:

- **The medium state.** Stage 1 stores what it has determined for a cell in state variables that
  the mix declares for this purpose. Because the state is synchronized before stage 2, each
  process sees the values of all cells, and all processes arrive at the same result in stage 2
  without further communication.
- **The material mix.** Stage 2 may store data for use by stage 3 in `mutable` data members of
  the mix, and stage 4 releases them. This is safe because stages 2 and 4 are serial, and
  stage 3 only reads the data. The functions remain `const`, like all functions of a mix that are
  called after setup.

The lifetime of such mutable data must be limited to a single update. A mix must not carry them
over from one iteration to the next. They are not part of the medium state, so they are not
stored in a checkpoint, for example, and a resumed simulation would start without them. Anything
that must survive an update belongs in the medium state, or in the iteration history proposed in
the [Iteration history](iteration-history/01-introduction.md) design note.

Stage 2 may itself start a parallel calculation through the `ParallelFactory`, and communicate the
results between processes with the `ProcessManager`, like the medium system does.

For example, a gas mix that obtains its state from an external photoionization code proceeds as
follows. Stage 1 derives the input parameters of the external calculation from the radiation
field of the cell, rounds them to the nodes of a parameter grid, and stores the node indices in
the state. Stage 2 collects the distinct nodes that occur in any cell, runs the external code
once for each, distributed over all threads and processes, and stores the results in a table.
Stage 3 looks up the result for each cell. Stage 4 saves the table to a file for a later run, and
releases it. The external code thus runs once per distinct node instead of once per cell, and
the calculations are balanced over the threads and processes regardless of which cells need them.

For a medium with a material mix family, stages 2 and 4 are called on the mix of the first cell,
as `isSpecificStateConverged()` is today.

## Array access

A new class template in the `fundamentals` module represents a view on a contiguous sequence of
values owned by someone else. It is a minimal stand-in for `std::span`, which requires C++20:

```cpp
/** A Span refers to a contiguous sequence of values that it does not own. */
template<typename T> class Span
{
public:
    Span(T* first, size_t size);
    size_t size() const;
    T& operator[](size_t i) const;
    T* begin() const;
    T* end() const;
};
```

`MaterialState` returns such a view on a range of custom variables:

```cpp
/** Returns a view on the n consecutive custom state variables starting at index i. */
Span<const double> customs(int i, int n) const;
Span<double> customs(int i, int n);
```

The custom variables of a component for a cell are stored contiguously, so the view refers
directly to the stored values. A mix declares an array-valued variable with `array(n)`, and
accesses it without copying:

```cpp
auto kappa = state->customs(_indexFirstOpacityExt, _numOpacityPoints);
double k = kappa[j] + (kappa[j + 1] - kappa[j]) * fraction;
```

The view offers `begin()` and `end()`, so it also works with the standard algorithms and with a
range-based loop.

## Declaring and accessing variables

A material mix with many variables, such as the `DiffuseIonizedGasMix` or the `NonLTELineGasMix`,
records the index of each variable by hand while it builds its list, and defines preprocessor
macros that make `state->custom(_indexHNeutralFraction)` look like a named accessor function. Three
small classes in the `material` module replace both.

A handle refers to a custom variable of a mix. The mix stores a handle where it stores an index
today, and applies it to a state:

```cpp
/** A StateScalar refers to a scalar custom state variable of a material mix. */
class StateScalar
{
public:
    /** Returns the value of the variable in the specified state. */
    double operator()(const MaterialState* state) const;

    /** Returns a reference to the value of the variable in the specified state. */
    double& operator()(MaterialState* state) const;
};

/** A StateArray refers to an array-valued custom state variable of a material mix. */
class StateArray
{
public:
    /** Return a view on the elements of the variable in the specified state. */
    Span<const double> operator()(const MaterialState* state) const;
    Span<double> operator()(MaterialState* state) const;
};
```

A list assembles the variables of a mix, assigns their indices, hands out the handles, and finds
a variable by its name:

```cpp
/** A StateVariableList assembles the specific state variables of a material mix. It starts with
    the number density, assigns consecutive custom indices to the variables that are added, and
    returns a handle for each of them. */
class StateVariableList
{
public:
    /** Adds the specified scalar custom variable, and returns a handle for it. */
    StateScalar add(const StateVariable& variable);

    /** Adds the specified array-valued custom variable, and returns a handle for it. */
    StateArray addArray(const StateVariable& variable);

    /** Returns the custom index of the variable with the specified name, or -1 if the list has
        no such variable. For an array variable, this is the index of its first element. */
    int index(const string& name) const;

    /** Return the number of variables in the list, and iterators over them. */
    size_t size() const;
    vector<StateVariable>::const_iterator begin() const;
    vector<StateVariable>::const_iterator end() const;
};
```

The functions that add a variable report a fatal error if its name is already in use. Other code
finds a variable through the list of a mix, as in `mix->specificStateVariableInfo().index("T")`,
and loops over the variables as it does over the vector today.

A mix keeps the list and the handles as data members. It fills them during setup, and returns the
list from `specificStateVariableInfo()`:

```cpp
// during setup
_temperature = _variables.add(StateVariable::temperature());
_hNeutralFraction = _variables.add(StateVariable::custom("hydrogen neutral fraction", "").named("x_HI"));
_opacityExt = _variables.addArray(StateVariable::custom("extinction opacity", "").named("kappa_ext").array(n));

// in the functions that use the state
double h0 = _hNeutralFraction(state);
_hNeutralFraction(state) = 1.;
auto kappa = _opacityExt(state);
double k = kappa[j] + (kappa[j + 1] - kappa[j]) * fraction;
```

- The accessors are inline, so a handle compiles to the same code as the macro it replaces. For
  an array, the position of the variable is calculated once instead of once per element.
- The overloads distinguish a constant state from one that can be modified. A function that
  receives a constant state, such as an opacity function, reads a value or a read-only view
  without casting away the constness of the state.
- The mix no longer keeps track of the indices itself, and no longer modifies its data members
  from a `const` function.
- The functions `custom(i)` and `setCustom(i, value)` of `MaterialState` remain, for code that
  works with an index, such as the probes.

A macro that abbreviates an expression rather than an accessor, such as the fragment weight in the
`FragmentDustMixDecorator`, becomes a private inline function of the mix.

## Changes to existing classes

| Class | Change |
| --- | --- |
| `SnapshotParameter` | gains a kind |
| `StateVariable` | loses the metallicity and temperature identifiers; gains name, kind, and number of elements, with factory functions for the standard names and a `custom()` overload without index |
| `StateScalar`, `StateArray`, `StateVariableList` | new classes: the handles for the custom variables of a mix, and the list that assigns their indices and finds a variable by name |
| `MaterialState` | loses the metallicity and temperature functions; gains `customs()` |
| `MediumState` | loses the metallicity and temperature functions and offsets; gains `numCells()` and access to a range of custom variables |
| `MaterialMixProvider` | new abstract base class of `MaterialMix` and `MaterialMixFamily`; holds the `MaterialType` and `DynamicStateType` enumerations and the `Capabilities` structure |
| `MaterialMix` | derives from `MaterialMixProvider`; changed `initializeSpecificState()`; `specificStateVariableInfo()` returns a `StateVariableList`; gains `capabilities()` and the staged update functions |
| `MaterialMixFamily` | derives from `MaterialMixProvider`; changed `mix()`; gains `capabilities()` |
| `Snapshot` and subclasses | lose the metallicity and temperature import functions, the temperature cutoff, and the metallicity multiplier; combine parameters by kind |
| `Medium`, `GeometricMedium` | lose the four metallicity and temperature functions; gain `capabilities()` and `parameterInfo()` |
| `ImportedMedium`, `ImportedGeometry` | lose the three properties; `ImportedMedium` also loses `importVariableMixParams` and `materialMixFamily`, holds a `MaterialMixProvider`, and returns parameters for families as well |
| `MediumSystem` | changed property sampler; verification of family members; staged update; loses `metallicity(m,h)` and `temperature(m,h)` |
| `ConfigurationSetup` | uses `Medium::capabilities()` |
| `GasThermalEnergyTreePolicy` | finds the temperature among the parameters of each gas medium, through its identifier; this also works for a medium with a material mix family, which lifts a current limitation |
| `LineGasSecondarySource` | finds the temperature state variable by name |
| `TemperatureProbe`, `MetallicityProbe` | find the state variables by name |
| `CustomStateProbe` | selects variables by name, index, or index range |
| `ImportedMediumTemperatureProbe`, `ImportedMediumMetallicityProbe`, `ImportedMediumVelocityProbe` | lose the special weighting for dust |
| `VoronoiMeshSpatialGrid`, `TetraMeshSpatialGrid`, `TreeNodeEvaluation` | ask the material type through `Medium::capabilities()` |

The documentation of all imported media and imported geometries, which describes the columns of
the import file, changes accordingly, and so does the documentation of the material mixes that
refers to the removed options.

## Changes to material mixes

| Material mix or family | Change |
| --- | --- |
| `ElectronMix`, `LyaNeutralHydrogenGasMix`, `SpinFlipAbsorptionMix` | request the temperature as a parameter; store it as a custom variable named `T` |
| `SpinFlipHydrogenGasMix`, `DiffuseIonizedGasMix` | request the metallicity and the temperature as parameters; store them as custom variables with the standard names |
| `NonLTELineGasMix` | requests the temperature as a parameter; stores the effective temperature as the custom variable named `T`, next to its kinetic temperature |
| `DiffuseIonizedGasMix` | names the variables listed in the Dynamic grid refinement design note; declares its opacity tables as array variables and reads them through views |
| `DiffuseIonizedGasMix`, `NonLTELineGasMix` | replace their accessor macros by handles |
| `FragmentDustMixDecorator`, `AbsorptionOnlyMaterialMixDecorator` | follow the changed signatures; the `FragmentDustMixDecorator` replaces its weight macro by a private function |
| `SelectDustMixFamily` | marks its index as categorical; provides `capabilities()` |
| `XRayIonicGasMixFamily` | requests the temperature as a parameter; provides `capabilities()` from its properties |
| all mixes with custom variables | assemble the list of variables during setup, through a `StateVariableList` |

The `NonLTELineGasMix` stores two temperatures. The effective temperature, which includes the
unresolved turbulence, takes the standard name, because it is the one that broadens the emitted
lines and that the `TemperatureProbe` reports today.

## Performance

- **Photon life cycle.** The state is stored and accessed as today. A mix that reads an array
  variable through a view no longer copies its values for every use, and a handle compiles to the
  same code as the macro it replaces.
- **Setup.** Combining the imported quantities by kind still takes a single pass over the
  overlapping entities at a position.
- **Updates.** A component that does not opt in to the staged protocol sees no change. One that
  does pays an extra pass over the cells and an extra synchronization.
- **Memory.** The memory use does not change: the temperature and the metallicity occupy the same
  space as custom variables.

## Testing

- **Uniform import.** For each affected functional test, the import file is converted as
  described in the Features chapter. A dust medium must then reproduce its previous results
  exactly, and so must a gas or electron medium without overlapping entities.
- **Kinds.** A new test imports two overlapping particles with different values for an intensive
  and a categorical quantity, and probes the resulting state.
- **Capabilities.** The existing tests must reproduce their results, since the configuration is
  derived from the same answers. This includes the tests that configure a material mix family,
  after their ski files have been upgraded. A new test configures a family with mismatching
  members and expects the fatal error.
- **Staged updates.** A test-only material mix, in the spirit of the `TrivialGasMix`, uses the
  four stages to derive the state of each cell from a table built in stage 2, with one and with
  several processes.
- **Variable access.** Unit-style checks through the test-only mix, and unchanged results for the
  mixes that adopt the views, the handles, and the variable list.

## Order of implementation

The changes can be implemented and verified in five steps:

1. The common base class and the capabilities descriptor, which change no results.
2. The kinds of imported quantities and their combination, which changes the results of a few
   tests with overlapping particles.
3. The uniform import, including the conversion of the temperature and the metallicity to named
   custom state variables, the removal of the import options, the single property for a material
   mix or family, and the PTS commands. This is the incompatible step.
4. Array variables, views, handles, and the variable list.
5. Staged updates.

Steps 4 and 5 are independent of each other, and each is useful without the others.
