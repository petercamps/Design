# Introduction

> Status: draft
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

- Increase performance of path segment generation by providing a specific implementation
  for each tree type (octree or binary tree).

## Overview

- **[Features](tree-based-spatial-grids/02-features.md)** describes the new grid and policy
  classes and their properties.
- **[Implementation](tree-based-spatial-grids/03-implementation.md)** describes the tree
  construction, the policies, and the path segment generation.
