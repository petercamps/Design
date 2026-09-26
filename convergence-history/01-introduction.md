# Introduction

> Status: draft

## Motivation

An iterative calculation needs a criterion to decide when to stop, and such a criterion almost
always compares the current iteration with one or more earlier ones. SKIRT keeps this historical
data in several unrelated places: a helper object in the iteration loops, "fake" aggregate cells
appended to the medium state, and `mutable` data members of material mixes that are otherwise
stateless configuration objects. Each instance has its own granularity, depth, and lifetime, and
none of them can be inspected in a uniform way.

The proposed dynamic grid refinement adds more instances of the same kind, including per-cell data
that must grow along with the spatial grid. This is a good moment to bring all historical data
under a single, central object with a well-defined lifetime, and to make it available to any
client that needs it.

## Scope

This design note proposes a refactoring. The convergence criteria themselves remain unchanged,
except for one small behavioral change in the `DiffuseIonizedGasMix`, discussed in the Call sites
chapter. Specifically:

- In scope: the historical values kept by the iteration loops, the aggregate medium states, and
  the history kept by material mixes, as well as the per-cell history needed by the proposed
  dynamic grid refinement.
- Out of scope: the per-cell comparisons that material mixes and recipes make between a new value
  and the old value still stored in the medium state. These need no storage of their own and remain
  unchanged.

## Overview

- **[Current state](convergence-history/02-current-state.md)** catalogs the existing instances of
  historical convergence data and summarizes their shortcomings.
- **[Design](convergence-history/03-design.md)** introduces the central object and the concepts
  behind it: series, identification, on-demand creation, lifetime, and parallelization.
- **[API](convergence-history/04-api.md)** proposes the concrete C++ interface.
- **[Call sites](convergence-history/05-call-sites.md)** describes, for each client, how it gains
  access to the central object and how it uses it.
- **[Open questions](convergence-history/06-open-questions.md)** collects the decisions that need
  input.
