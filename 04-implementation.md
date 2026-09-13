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
- If the input HDF5 file contains bundles equivalent to today's memory-mapped stored
  tables (`.stab`) and the output target is the same file, memory-mapping them is unsafe.
  In that case, fall back to copying the data into memory instead of memory-mapping it.
- Considering redesigning `PolicyTreeSpatialGrid` to hold a _list_ of `TreePolicy` instances
  rather than one, subdividing a node as soon as any single policy asks for it (so density,
  electron/gas density, particle-list, etc. criteria can be freely combined). Under this
  design, `FileTreeSpatialGrid` could become just another policy that replays a previously
  recorded topology, instead of a separate spatial grid class — this also gives "refine a
  snapshot with an extra criterion" for free. Gotcha: tree construction is breadth-first,
  level by level (see `DensityTreePolicy::constructTree()`), so the snapshot policy must
  match live nodes to its own recorded ones by tree position — parsed into a real tree at
  setup — rather than by consuming the recorded sequence in order; once a live node falls
  past what was recorded, it correctly answers no from there down.
