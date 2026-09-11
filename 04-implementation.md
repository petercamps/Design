# Implementation

> Status: draft

TODO: implementation plan, milestones, and affected SKIRT components.

## Notes (to revisit)

- Build HDF5 with `HDF5_ENABLE_THREADSAFE=ON`, and likely also `HDF5_USE_FILE_LOCKING=FALSE`
  (or `BEST_EFFORT`) to avoid lock failures on HPC filesystems that don't fully support
  `flock`.
- When running under MPI with input and output pointed at the same HDF5 file, all ranks
  must close their read access to that file before the root process opens it for writing —
  reading and writing must not overlap in time (see Concurrency under Output in Features).
- If the input HDF5 file contains datasets equivalent to today's memory-mapped stored
  tables (`.stab`) and the output target is the same file, memory-mapping them is unsafe.
  In that case, fall back to copying the data into memory instead of memory-mapping it.
