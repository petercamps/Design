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

Nodes stay pointer-based and are allocated in a std::deque during construction — growing
a tree incrementally is what that representation is good at, and evaluation does not care
whether nodes are reached through pointers or indices. Once construction finishes, the
tree is converted to the flat, index-linked array `BinTreeSpatialGrid` and
`OctTreeSpatialGrid` use for path segment generation (see below), establishing neighbor
links for every node in a single top-down pass over the now-complete tree. The
pointer-based nodes can then be discarded.

## Policies

Many policies fit the shared per-node test directly, since their subdivision criterion is
purely a property of the individual cell — e.g. a sampled density — with no dependency on
other nodes. Some policies need a closer look.

The owning `TreeSpatialGrid` class provides a way for a policy's setup code to obtain the
domain extent and the splitting convention (2 or 8 children per split), so that a
policy's implementation can use this information. Furthermore, with each
`needsSubdivide()` call, the policy is passed the bounding box and level of the node to
be evaluated.

`SiteListTreePolicy` must subdivide nodes until every leaf holds at most one site, then
refine each of those leaves — and everything below them — a fixed number of further
levels. To recast this as a per-node test, the policy calculates at setup the isolation
box for each site in the list, defined as the bounding box of the node where this site
first becomes isolated (no other sites are inside the node). This is a cheap, sequential,
geometry-only walk to find the extent of the node that site-only splitting would first
leave alone. That walk needs the raw site positions, so the policy builds a temporary
`BoxSearch` over them, only for this one-time setup computation. The per-node test itself
queries a second `BoxSearch`, built over the resulting isolation boxes, which is the only
structure that needs to survive setup.

The per-node test then becomes a single query: how many of the precomputed per-site
isolation boxes does the node's own box overlap? Since tree nodes are always either
disjoint or strictly nested, a box overlaps two or more isolation boxes exactly when it
still spans multiple, not-yet-separated sites; it overlaps exactly one when it lies at or
below the point where some site became isolated; and it overlaps none when it has no
relation to any site at all. Two or more overlaps means the node needs subdividing
outright, matching the "more than one site" rule. Exactly one overlap means it needs
subdividing only if its level is still less than that isolation box's own level plus
`numExtraLevels`. No overlaps means this policy has nothing to say about the node.

`TopologyTreePolicy` parses the recorded topology file — unchanged, still the depth-first
sequence of subdivision flags written by `TreeSpatialGridTopologyProbe` — at setup into a
small in-memory tree that mirrors it. Given a node's box and level, the per-node test
descends the parsed tree from its root, `level` times, choosing at each step the child
whose sub-box contains the query box's center. The domain extent and splitting convention
make this descent exact, without needing to compare boxes by floating-point equality. If
the descent reaches a recorded node at that depth, the test answers yes if that node is
marked subdivided in the recording and no otherwise; if it runs out of recorded depth
first — because another policy in the list kept subdividing past where this recording
stops — it answers no. Each call repeats this descent independently from the root, so the
test needs no state carried between calls and no particular calling order.

`ResolvedSpheresTreePolicy` builds a `BoxSearch` at setup, bulk-loaded with the spheres'
bounding boxes — the same pattern `ParticleSnapshot` already uses for its own smoothed
particles. The per-node test queries `entitiesFor(box)` for candidate spheres, verifies
each with an exact box-sphere intersection test, and subdivides if any of them is still
under-resolved in this node. No precomputation walk is needed here, unlike the site list:
each sphere's required cell size follows directly from its own properties, with nothing
that depends on the tree's evolving structure.

`AdaptiveMeshTreePolicy` and `CellTreePolicy` need no comparable treatment: given a node,
the per-node test asks the imported snapshot for the cell at the node's center and
compares that cell's extent to the node's. There is no cross-node bookkeeping or ordering
concern to design around: each node's test only looks at its own center against an
already-built, read-only snapshot.

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

`OctTreeSpatialGrid` follows the same approach, adapted to eight children per node.
