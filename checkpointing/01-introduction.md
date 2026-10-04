# Introduction

> Status: under construction
>
> Depends on: [SKIRT 10](skirt-10/01-introduction.md),
> [Tree-based spatial grids](tree-based-spatial-grids/01-introduction.md),
> [Dynamic grid refinement](dynamic-grid-refinement/01-introduction.md),
> [Iteration history](iteration-history/01-introduction.md),
> [HDF5 input](hdf5-input/01-introduction.md),
> [HDF5 output](hdf5-output/01-introduction.md)

## Motivation

A checkpoint is a snapshot of a SKIRT simulation's complete internal state, written
periodically while the simulation runs. Resuming from a checkpoint continues the simulation
from that point onward, rather than restarting it from the beginning.

HDF5 is well suited to checkpointing. The same properties that make it attractive for input and
output, as proposed in the [HDF5 input](hdf5-input/01-introduction.md) and
[HDF5 output](hdf5-output/01-introduction.md) notes — a single, self-describing file holding
multiple structured datasets — make it equally suitable for capturing a simulation's state.

Beyond resuming after an interruption — a crash, or a cluster job hitting its time limit —
the checkpoint file is an ordinary, self-describing HDF5 file, which makes it useful in its
own right:

- **Inspection and visualization in Python.** Its contents — the radiation field, per-cell
  medium state, and so on — can be explored directly with `h5py`, independently of whether
  the run is ever actually resumed.
- **Iterating across simulations.** The saved state can seed a new run configured with more
  photon packets or additional iterations, to improve signal-to-noise or convergence, rather
  than starting that more expensive run from scratch. This makes it practical to chain
  several SKIRT runs together, with convergence judged externally — from the intermediate
  results — rather than automatically inside SKIRT itself.

## Impact on PTS

Checkpoints may enable new PTS capabilities that take advantage of the additional information
they hold. Working out these changes, however, is out of scope for this note.

## Overview

- **[Features](checkpointing/02-features.md)** describes the checkpoint probe, resuming from a
  checkpoint, and the other uses of checkpoints. It is aimed at SKIRT users.
- **[Data model](checkpointing/03-data-model.md)** specifies the checkpoint bundle that captures
  a running simulation's internal state. It is aimed at authors of Python scripts that inspect
  checkpoints.
- **[Implementation](checkpointing/04-implementation.md)** describes the extensions to the `hdf5`
  build target and the name resolution, the checkpoint probe, and the changes needed to resume a
  simulation.
