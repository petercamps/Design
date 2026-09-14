# Implementation

> Status: draft

## Notes to revisit

- Build HDF5 with `HDF5_ENABLE_THREADSAFE=ON`, and likely also `HDF5_USE_FILE_LOCKING=FALSE`
  (or `BEST_EFFORT`) to avoid lock failures on HPC filesystems that don't fully support
  `flock`.

- When running under MPI with input and output pointed at the same HDF5 file, all ranks
  must close their read access to that file before the root process opens it for writing —
  reading and writing must not overlap in time (see Concurrency under Output in Features).

- The spatial grid checkpoint bundle's topology is recorded and replayed breadth-first, not
  depth-first as today's `TreeSpatialGridTopologyProbe`/`FileTreeSpatialGrid` do — see Tree
  grids under Reusing grid topology in Features for why this is required, not optional.

- Related to the redesigning of tree policies: tree construction is breadth-first,
  level by level (see `DensityTreePolicy::constructTree()`), so the checkpoint policy must
  match live nodes to its own recorded ones by tree position — parsed into a real tree at
  setup — rather than by consuming the recorded sequence in order; once a live node falls
  past what was recorded, it correctly answers no from there down.

## Stored table bundle

The data in a stored table input bundle cannot be memory mapped (as for a regular .stab file)
for several reasons:

- The quantity values are ordered differently (interleaved in .stab, per dataset in bundle).
- The bundle datasets may be compressed and/or chunked.
- If the same HDF file is used for input and output, memory mapping a portion of the file
  is unsafe because writing to the file might relocate its contents.

The StoredTable constructor for a bundle must therefore copy the data into newly allocated
memory, and the destructor must deallocate that memory.
