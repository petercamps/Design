# Introduction

> Status: implemented
>
> Depends on: [SKIRT 10](skirt-10/01-introduction.md)

## Motivation

This note proposes a significant restructuring of SKIRT's hierarchical tree classes
and their helpers, including the classes that define cell subdivision policies. This
change invalidates any existing ski file that configures an octree or binary tree spatial
grid. This is annoying, even if PTS provides a procedure to automatically upgrade ski
files to the new structure. However, the additional capabilities unlocked by this change
seem to outweigh the nuisances.

The key objectives are to:

- Combine multiple cell subdivision policies; for example refining a previously
  stored tree topology with an extra criterion or refining multiple regions in the domain
  with their own distinct criteria.

- Enable new types of cell subdivision policies, for example based on an imported scalar
  field (other than density), an imported grid, or a list of regions to be resolved.

- Increase performance of path segment generation by taking advantage of a new flat-node
  tree structure and by implementing several targeted opimizations.

### Performance results

Memory required for the tree structure itself (i.e. not counting the medium state) is
reduced by a factor of 2 or more.

Performance after implementation of the new tree grids , measured on a 10+10 core system
(this may vary between computer architectures):

- Tree construction: similar or slightly faster (e.g. 10%).
- Tree grid traversal: 2 to 3 times faster.

An extinction-only dust simulation (i.e. constant optical properties across the domain,
no kinematics) runs 2 times faster. Simulations with more complex physics show a smaller
gain because the grid traversal time represents a smaller fraction of the total
simulation time.

## Overview

- **[Features](tree-based-spatial-grids/02-features.md)** describes the new grid and policy
  classes and their properties.
- **[Implementation](tree-based-spatial-grids/03-implementation.md)** describes the tree
  construction, the policies, the path segment generation, and the use of a known initial
  cell by the tree grids and other grid types.
