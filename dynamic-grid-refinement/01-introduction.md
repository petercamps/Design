# Introduction

> Status: under construction

## Motivation

Photoionization simulations depend on spatial structure that is not known when the spatial grid
is built. An ionization front, or the thin layer in which a given ion, and hence a given emission
line, is concentrated, only emerges once the radiation field has been calculated. A grid fine
enough everywhere such a layer might appear is prohibitively large. A grid built from the density
distribution alone, on the other hand, typically under-resolves these layers, which biases line
luminosities and line ratios.

Dynamic grid refinement addresses this by subdividing cells between iterations, wherever a
quantity derived from the radiation field varies too steeply across a cell. The
simulation continues on the finer grid, and the iterations proceed until both the medium state and
the grid have stopped changing. Child cells start from their parent's state, so each iteration
after a refinement round starts close to the previous solution rather than from scratch.

## Background

This design note is based on the grid refinement code written by Anand Utsav Kapoor for the
photoionization runs of an upcoming paper, including a 48-process MPI run on a galaxy grid with 40
million cells. The code is a patch against the current SKIRT 9 master (commit `646cfdc`) and
consists of two parts:

- **Static refinement** (seeding): before the simulation starts, the tree is refined around a list
  of ionizing sources, so that each source's Strömgren radius is resolved by a given number of
  cells. This part has already been recast into the new tree policy framework in the SKIRT 10
  chapter on [Tree-based spatial grids](skirt-10/05-tree-based-spatial-grids.md), notably as the
  `ResolvedSpheresTreePolicy`.

- **Dynamic refinement** (on the fly): during the dynamic medium state iterations, cells across
  which a chosen gas quantity changes steeply are subdivided between iterations. This is the subject of
  this note.

The design follows the reference implementation wherever it proved itself in production, and
departs from it where the SKIRT 10 grid design or existing SKIRT conventions suggest a cleaner
structure.

## Scope and assumptions

- The note assumes that the SKIRT 10 restructuring of the
  [tree-based spatial grids](skirt-10/05-tree-based-spatial-grids.md) has been
  implemented, including the flat, index-linked node array used for path segment generation.
- The note also assumes that the central iteration history described in the
  [Convergence history](convergence-history/01-introduction.md) design note has been implemented.
  Dynamic refinement keeps all of its historical data in that history.
- Only tree-based spatial grids (octree and binary tree) support dynamic refinement. Cells are
  only ever subdivided, never merged.
- Refinement can happen in each of the iteration loops: primary, secondary, and merged primary and
  secondary emission iterations.
- The mechanism is generic, but at present only the `DiffuseIonizedGasMix` offers quantities that
  are useful to drive refinement.

## Overview

- **[Features](dynamic-grid-refinement/02-features.md)** describes the configuration, the
  refinement criteria, and the behavior of the iteration loop, as seen by the user.
- **[Implementation](dynamic-grid-refinement/03-implementation.md)** describes the refinement
  step, the grid operations, and the growth of the per-cell data structures.
- **[Open questions](dynamic-grid-refinement/04-open-questions.md)** collects the design
  decisions that need input, and observations on the reference implementation.
