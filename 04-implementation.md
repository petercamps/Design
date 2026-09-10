# Implementation

> Status: draft

TODO: implementation plan, milestones, and affected SKIRT components.

## Notes (to revisit)

- Build HDF5 with `HDF5_ENABLE_THREADSAFE=ON`, and likely also `HDF5_USE_FILE_LOCKING=FALSE`
  (or `BEST_EFFORT`) to avoid lock failures on HPC filesystems that don't fully support
  `flock`. Revisit once the threading and concurrent-access design is worked out.
