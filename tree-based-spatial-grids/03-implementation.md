# Implementation

## Construction

`TreeSpatialGrid` builds its tree level by level, the same way today's `DensityTreePolicy`
does: starting from a one-node list holding the root, each pass evaluates every node at
the current level, subdivides the ones that need it, and appends their children to the
end of the same list, so the newly appended range becomes the next level. Subdivision
only ever appends, so a node's position in the list is a stable, level-ordered id — the
list doubles as the tree's ownership list and its implicit breadth-first queue.

The two phases per level stay split as today, for the same reason: evaluating whether a
node needs subdivision is read-only and can be expensive — sampling the density field is
the typical case — so it runs in parallel across the level's nodes, using SKIRT's
existing `Parallel` machinery exactly as today's `DensityTreePolicy::constructTree()` does.
Subdividing a flagged node mutates the tree (new children, updated neighbor links) and
stays sequential. The one real change is what gets evaluated per node: instead of a
single policy's `needsSubdivide()`, it is now the configured list of policies, tested in
order and short-circuiting on the first one that asks for subdivision. Since
`TreeSpatialGrid` now owns that combined list, it drives the loop itself, rather than
each policy implementing its own as today's `TreePolicy` subclasses do. A policy narrows
to a per-node test that this shared loop calls into.

The nodes are small value objects that refer to each other by index from the start; there are
no pointer-based nodes. During construction they are appended to a `std::deque`, which never
moves its existing elements, so that a reference to a parent node remains valid while its
children are appended. Once construction finishes, the nodes are copied into the flat array
that `BinTreeSpatialGrid` and `OctTreeSpatialGrid` use for path segment generation (see
below), and the subclass establishes the neighbor links for every node in a single top-down
pass over the now-complete tree. The leaf nodes are numbered as cells in array order, i.e. level
by level. Node and cell indices are 32-bit integers; setup reports a fatal error if the tree
grows beyond that range.

The nodes evaluated by the policies are represented by a `TreeNodeEvaluation` object rather
than by just their bounding box and level. Besides the extent and level of the node, this
object offers the properties of the media in the node that the material policies need, each
calculated when first requested and then cached, so that all policies evaluating the node share
them: the random sample positions (drawn lazily, only if a policy needs them), the density of
each medium component at these positions (the mass density for dust, the number density for
electrons and gas), the summed samples per material type, and the amount of material of each
medium component in the node, calculated exactly for a component that offers the
`MassInBoxInterface` and estimated from the samples otherwise. Each thread uses its own
instance, reset for each node, so that the cache needs no locking and its memory is reused. The
lists of medium component indices per material type are built by `MediumSystem` before the
spatial grid is set up.

## Policies

Many policies fit the shared per-node test directly, since their subdivision criterion is
purely a property of the individual cell — e.g. a sampled density — with no dependency on
other nodes. Some policies need a closer look.

The owning `TreeSpatialGrid` class provides a way for a policy's setup code to obtain the
domain extent and the splitting convention (2 or 8 children per split), so that a
policy's implementation can use this information. Specifically, the grid offers its extent,
its number of children per node, and the public functions `childExtent()` and `childIndex()`,
which return the extent of a given child, and the index of the child containing a given
position, for a node with a given extent and level. The tree itself uses `childExtent()` to
subdivide its nodes, so that a policy mirroring the tree structure uses exactly the same rule.
Furthermore, with each `needsSubdivide()` call, the policy is passed the `TreeNodeEvaluation`
object for the node to be evaluated (see above).

`SiteListTreePolicy` must subdivide nodes until every leaf holds at most one site, then
refine each of those leaves, including the empty ones, a fixed number of further levels. To
recast this as a per-node test, the policy builds a reference tree at setup that mirrors the
site-only subdivision, using the splitting convention of the grid. The tree is built level by
level: a node is subdivided if it contains more than one site, as long as its level is below the
grid's `maxLevel`. The sites in each node at the current level form a consecutive range in a list
of site indices, which is reordered with a counting sort when the node is subdivided, so that the
sites in each child again form a consecutive range. The reference tree is not subdivided where a
node holds at most one site, even at levels below the grid's `minLevel`, so that its size does not
depend on `minLevel`.

The per-node test descends the reference tree from its root toward the center of the node, up
to the level of the node. If the reference tree is still subdivided at that level, the node
contains multiple sites and must be subdivided. Otherwise, the descent ended at a leaf of the
reference tree at some level, and the node is subdivided if its own level is below the larger of
that level and `minLevel`, plus `numExtraLevels`. Because the grid always subdivides nodes up to
`minLevel`, counting the extra levels from `minLevel` for coarser reference leaves reproduces the
SKIRT 9 behavior, in which the sites were inserted into a tree that was first subdivided up to
`minLevel`. The test needs no state carried between calls and no particular calling order. This
departs from the approach considered earlier, which built search structures over the isolation
boxes of the sites and refined only the nodes overlapping an isolation box: that approach would
not refine the empty leaves, which SKIRT 9 does.

`TopologyTreePolicy` parses the recorded topology file — the depth-first sequence of
subdivision flags written by `TreeSpatialGridTopologyProbe`, in the same format as today — at setup into a
small in-memory tree that mirrors it. Given a node's box and level, the per-node test
descends the parsed tree from its root, `level` times, choosing at each step the child
whose sub-box contains the query box's center. The domain extent and splitting convention
make this descent exact, without needing to compare boxes by floating-point equality. If
the descent reaches a recorded node at that depth, the test answers yes if that node is
marked subdivided in the recording and no otherwise; if it runs out of recorded depth
first — because another policy in the list kept subdividing past where this recording
stops — it answers no. Each call repeats this descent independently from the root, so the
test needs no state carried between calls and no particular calling order. The descent uses the
grid's `childIndex()` and `childExtent()` functions. Because the descent follows the center of
the node, which is never close to a splitting plane of an ancestor, it is not affected by
rounding errors.

`ResolvedSpheresTreePolicy` builds a `BoxSearch` at setup, bulk-loaded with the bounding
boxes of the spheres enlarged by their reach — the same pattern `ParticleSnapshot` already uses
for its own smoothed particles. The per-node test queries `entitiesFor(box)` for candidate
spheres, verifies each with an exact box-sphere intersection test against the enlarged sphere,
and subdivides if any of them is still under-resolved in this node, i.e. if the largest width of
the node exceeds the radius of the sphere divided by its number of bins. No precomputation
walk is needed here, unlike the site list: each sphere's required cell size follows directly
from its own properties, with nothing that depends on the tree's evolving structure.

`AdaptiveMeshTreePolicy` and `CellTreePolicy` need no comparable treatment: given a node,
the per-node test asks the imported snapshot for the cell at the node's center and
compares that cell's extent to the node's. There is no cross-node bookkeeping or ordering
concern to design around: each node's test only looks at its own center against an
already-built, read-only snapshot. The node is subdivided if it is wider than the cell along any
axis, allowing for a small relative tolerance for rounding errors. Two additions support these
policies. A new `CellMeshInterface`, implemented by `CellGeometry` and `CellMedium` like the
existing `AdaptiveMeshInterface`, gives access to the cell snapshot of a medium, and
`CellSnapshot` offers the index of the cell containing a given position (the first listed cell
if several cells contain it). To import an adaptive mesh without interpreting its data columns,
`TextInFile::readRow()` accepts a file for which no columns were declared, returning an empty
row for each data line instead of reporting an error.

`ParticleFieldTreePolicy` imports its own `ParticleSnapshot`, configured with a mass density
policy that keeps the imported masses, so that the snapshot offers the total mass, the mass in a
box, and the density at a position. The mass in a node is calculated by integrating the smoothing
kernel of each overlapping particle over the node, as for the `MassInBoxInterface` of a
`ParticleMedium`. If the smoothing kernel lacks the cumulative kernel table required for this,
the mass is estimated from the density of the field sampled at the shared sample positions of the
node.

`GasThermalEnergyTreePolicy` estimates the thermal energy of each gas medium component in a node
as the number of entities in the node, calculated exactly if the component offers the
`MassInBoxInterface`, multiplied by the average temperature in the node, i.e. the temperature
sampled at the shared sample positions weighted by the sampled number density. If none of the
samples hits any material while the node does contain material, which can happen for a small
particle in a large node, the average temperature of the medium component is used instead. That
average temperature is estimated at setup from a large number of positions (100 000) drawn from
the spatial distribution of the component, and the total thermal energy of the component as its
total number of entities multiplied by its average temperature. A small hot particle that is
missed by the samples of a node is thus counted at the average temperature, which underestimates
the thermal energy of that node; an exact alternative for particle media would integrate the
kernel of each particle weighted by its temperature.

## Path segment generation

`BinTreeSpatialGrid` stores its tree as a flat array of small, fixed-size nodes that
refer to each other by index rather than by pointer, avoiding pointer chasing and virtual
function calls. Each node holds its own extent, its cell index, the index of its first
child (the second child is stored consecutively), and the index of its neighbor across
each of its six walls. This data structure is established once, for the whole tree,
before actual path segment generation begins.

The path segment generator follows the neighbor links directly when a path crosses a
wall, descending into the neighbor only if it still has children of its own. A top-down
search from the root is needed only to locate a path's first cell, and as a fallback for
the rare case where rounding error leaves a neighbor link inconsistent. Per-path
quantities — the reciprocal of each direction component, and which wall it can cross —
are computed once rather than for each step, and the handful of nodes a path could move
to next are prefetched while the current step is still being finished.

A node occupies 88 bytes: the coordinates of its six walls, the index of its neighbor across
each wall, the index of its first child, its cell index, its level, and (for a binary tree) the
axis perpendicular to its splitting plane, which alternates between x, y, and z with the level.
The neighbor across a wall is the node at the same level that touches the wall, if there is one,
and otherwise the leaf node at a lower level that covers the other side of the wall. A position
on a splitting plane is assigned to the child on the upper side. The prefetch requests use a small
header-only `Prefetch` namespace in the `utils` target, which wraps the compiler's builtin
prefetch function (available for GCC, Clang, and compatible compilers, and a no-op otherwise).
This is a deliberate exception to the rule that system-dependent code belongs in the `System`
class, because the functions must be inlined at the call site to be useful.

`OctTreeSpatialGrid` follows the same approach, adapted to eight children per node.
