# Introduction

> Status: draft
>
> Depends on: [SKIRT 10](skirt-10/01-introduction.md)

## Motivation

The medium state holds, for each spatial cell and each medium component, the quantities that
define the material in that cell: its number density and any further variables that its material
mix needs. The mechanism dates from the time when SKIRT handled dust only. It has since been
stretched to serve gas mixes that store hundreds of variables per cell, import many of them from a
snapshot, and update them from the radiation field in every iteration. Several conventions that
were adequate for dust now get in the way:

- **Temperature and metallicity are treated specially.** They have their own import options,
  their own place in the import file, their own state variables, and their own arguments in the
  material mix interface. All other quantities are handled by a generic mechanism. The special
  treatment exists because a dust medium can be derived from the gas in a hydrodynamical snapshot
  through the metallicity and the temperature. For a gas mix, the two are quantities like any
  other, and the user must remember which combination of import options a given mix expects.
- **Overlapping entities are combined in a fixed way.** Where smoothed particles overlap, the
  density is summed, the temperature and the metallicity are averaged, and all other imported
  quantities are taken from the particle with the largest weight. This is right for an index, but
  wrong for a velocity or an abundance ratio.
- **State variables are anonymous.** A custom state variable is known by its index, which changes
  with the configuration of the mix. A probe or a refinement criterion cannot ask for "the neutral
  hydrogen fraction".
- **The capabilities of a medium are asked from a material mix.** For a medium with a family of
  mixes, this means constructing a mix before the simulation has been set up, just to answer a
  few yes-or-no questions, and nothing verifies that all members of the family give the same
  answers.
- **A state update sees one cell at a time.** A mix that needs an expensive external calculation
  for some cells must perform it on the spot, in the thread that happens to handle the cell, and
  cannot share the result between cells that need the same calculation.
- **Array-valued state is accessed one value at a time.** A mix that stores an opacity table per
  cell copies it into a temporary array for every use.

## Background

The proposal answers suggestions made in response to the SKIRT 10 design notes. Andrea Gebek
suggested dropping the separate treatment of temperature and metallicity. Arno Lauwers suggested
the other changes, from his experience with adding X-ray gas mixes to SKIRT, and pointed out that
the first suggestion requires the second: once temperature and metallicity are ordinary imported
quantities, they must still be averaged over overlapping particles.

## Proposal

This design note proposes five changes. The first two change what users see; the others are
internal.

- **Uniform import.** A material mix requests all the quantities it needs from the snapshot,
  including the temperature and the metallicity, through a single list. The import options for
  temperature and metallicity disappear, and so does the built-in recipe that derives the dust
  mass from them.
- **Richer descriptors.** The descriptor of an imported quantity or a state variable states its
  kind: intensive, extensive, or categorical. The kind determines how overlapping entities are
  combined, and how a variable is passed on when a spatial cell is subdivided. The descriptor of
  a state variable also carries a short name, and can describe an array of values.
- **Capabilities descriptor.** A material mix and a material mix family describe their
  capabilities in a small data structure, which a family provides without constructing a mix.
- **Staged updates.** A material mix can split a state update into a preparation pass over all
  cells, a serial step that sees the complete state, and the actual update of each cell.
- **Array access.** A material mix reads and writes a block of state variables through a view,
  without copying.

## Scope and assumptions

- The note describes changes relative to the current code. It assumes that the changes proposed in
  the [SKIRT 10](skirt-10/01-introduction.md) design note have been implemented, including the
  incompatible ski file changes it allows, and adds incompatible changes of its own, listed in the
  Features chapter.
- The proposal concerns media. Sources are not affected: a source imports the parameters of its
  SED family as before, and keeps its own handling of metallicity and age.
- The radiation field, the photon life cycle, and the physics of the material mixes are not
  changed.
- The five changes can be implemented one after the other, in the order given in the
  Implementation chapter. The last two are independent of each other.
- Three later design notes touch the same classes. The Open questions chapter lists how they
  relate to this proposal.

## Overview

- **[Current state](medium-state/02-current-state.md)** describes how the medium state is
  declared, imported, stored, and updated today, and where temperature and metallicity are treated
  specially.
- **[Features](medium-state/03-features.md)** describes the changes as seen by the user, including
  the incompatibilities.
- **[Implementation](medium-state/04-implementation.md)** describes the new descriptors and
  interfaces, the staged update, and the changes to existing classes.
- **[Open questions](medium-state/05-open-questions.md)** collects the design decisions that need
  input, with their alternatives, and the relation to other design notes.
