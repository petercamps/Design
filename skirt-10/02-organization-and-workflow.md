# Organization and workflow

## Repositories

The SKIRT GitHub organization currently holds eight repositories:

| Repository | Description | Proposed |
| --- | --- | --- |
| `SKIRT7` | SKIRT version 7 | removed |
| `SKIRT8` | SKIRT version 8 | removed |
| `PTS` | Python toolkit for SKIRT 7/8, plus unrelated DustPedia, EAGLE, and image-processing tooling | renamed to `LegacyPTS` |
| `Web8` | The SKIRT 8 web site | removed |
| `SKIRT9` | SKIRT version 9 | renamed to `SKIRT` |
| `PTS9` | Python toolkit for working with SKIRT version 9 | renamed to `PTS` |
| `Web9` | The SKIRT 9 web site | renamed to `Web` |
| `CosTuuM` | C++ T-Matrix code for spheroidal dust grain emission properties | unaffected |

**Removed.** `SKIRT7`, `SKIRT8`, and `Web8` are proposed to be removed outright, since
they are superseded by `SKIRT9`/`Web9` and no longer actively maintained. `CosTuuM` is a
separately maintained tool; it is not part of this proposal and is thus unaffected.

**Renamed.** The four repositories below are proposed to be renamed. The three carrying
an explicit version number simply drop the suffix; the bare `PTS` repository — the old
SKIRT 7/8-era Python toolkit — is renamed to `LegacyPTS` instead of `PTS`, for the reasons
given below.

- `PTS` → `LegacyPTS`
- `SKIRT9` → `SKIRT`
- `PTS9` → `PTS`
- `Web9` → `Web`

GitHub automatically routes requests for the old name — both `git` operations and the
web page — to the new one, so existing clones, links, and CI configurations keep working
without changes.

**Preserving `PTS`.** Next to its outdated SKIRT 7/8-era functionality, `PTS` also holds
a set of capabilities unrelated to SKIRT altogether — an image-processing toolkit,
tooling built for the DustPedia project and the EAGLE simulations, a genetic-algorithm
fitting framework — that other authors have relied on independently of SKIRT for years.
None of this appears to be cited as standalone software, but it clearly underpins
DustPedia's own foundational papers, which remain heavily and recently cited by
researchers outside the SKIRT team. That is reason enough to keep this material publicly
available under its own name — `LegacyPTS` — rather than delete it outright when `PTS9`
takes over the `PTS` name.

This does come with one unavoidable side effect: renaming `PTS` to `LegacyPTS` must
happen before `PTS9` can be renamed to `PTS`, since both cannot hold that name at the
same time. Once `PTS9` claims it, GitHub's redirect from the old `PTS` name now points
there instead of to `LegacyPTS` — an old citation or link to `github.com/SKIRT/PTS` ends
up at the new, unrelated SKIRT-10-era toolkit, with no automatic way back to `LegacyPTS`.

## Versioning

`SKIRT` and `PTS` currently have no version-tagging practice at all: no git tags, no
GitHub releases, and the human-readable version string printed at startup
(`BuildInfo::projectVersion()`, currently `"v9.0"`) was hand-set once, in January 2019,
and never touched since, despite hundreds of substantive changes recorded in the Recent
Changes list. This proposal adopts
[Semantic Versioning 2.0.0](https://semver.org) (`MAJOR.MINOR.PATCH`), the de facto
standard for citable software version numbers: `MAJOR` increases for incompatible
changes — a ski file that can no longer be auto-upgraded, or a removed feature — `MINOR`
for backward-compatible new functionality, and `PATCH` for backward-compatible fixes.
This also settles, mechanically, whether a given change belongs in the next minor
release or has to wait for the next major one.

Each release is marked with a git tag (`v10.0.0`, `v10.1.0`, ...) on the appropriate
branch (see Workflow, below). The build already computes a git-derived identifier via
`git describe --dirty --always` (in `SMILE/build/CMakeLists.txt`); with actual tags to
describe, this starts producing a real version string — `v10.1.0` for an exact release,
or `v10.1.0-5-gabc1234` a few commits past one — instead of always falling back to a
bare commit hash. The separate, hand-maintained `PROJECT_VERSION` constant in
`SMILE/build/Version.cmake` is dropped in favor of deriving `BuildInfo::projectVersion()`
from this same tag, so the two pieces can no longer drift apart the way they have since
2019.

`PTS` is versioned in lockstep with `SKIRT` on `MAJOR.MINOR`, but not on `PATCH`. Every
SKIRT minor or major release aligns PTS to the same `MAJOR.MINOR`, but each repository
then accumulates its own independent patch releases from that point, since a patch is by
definition a backward-compatible fix with no effect on the SKIRT features PTS needs to
track. This keeps the coordination to what actually matters. The rule for users is
correspondingly simple: for a given `MAJOR.MINOR`, always install the latest available
patch of both `SKIRT` and `PTS`.

## Workflow

Both repositories already follow a fork-and-pull-request model. Contributors work from
their own fork and open a pull request against the shared repository, reviewed and
squash-merged by a core team member. This matches "trunk-based development": a single
active branch (`master`) with no long-lived feature branches and with changes landing
through short-lived pull requests. This proposal keeps that unchanged; only what a pull
request targets, and when a new branch gets created, changes.

`master` remains the only branch that accepts new features. What's added is a
*maintenance branch* per major version that still needs support after the next one takes
over `master`. When SKIRT 10 is ready to become the actively developed line, a branch
named `9` is cut from the current tip of `master` before any SKIRT 10 work lands, so `9`
and `master` diverge at that exact point. This is the same pattern [CPython
uses](https://devguide.python.org/versions/) to keep several release lines alive at once,
scaled down. Rather than CPython's five release phases and roughly five-year support
window, SKIRT keeps to a single phase — bug fixes only — for as long as the core team
judges SKIRT 9 still needs it, a decision made as it comes up rather than fixed in
advance.

The `9` branch accepts only backward-compatible fixes (`PATCH`, occasionally `MINOR`;
see Versioning, above), each tagged as its own release. A fix needed on both lines means
two pull requests, one against each branch; there is no tooling in place to backport
this automatically. To establish a clean starting point, the current tip of `master` is
tagged `v9.9.0` — reflecting the substantial development that has accumulated since the
2019 release, even without formal version numbers to mark it — and the `9` branch is cut
from it before any SKIRT 10 work is merged.

## Implementation

The following off-the-shelf tooling will help manage the version numbers and the
SKIRT/PTS coordination.

**Classifying changes.** Pull request titles are required to follow the
[Conventional Commits](https://www.conventionalcommits.org) format — `fix: ...` for a
PATCH, `feat: ...` for a MINOR addition, `feat!: ...` for
a MAJOR one. Because both repositories already squash-merge, the PR title becomes the
commit message on `master`, so this is the only thing a contributor — including the
maintainer — has to get right by hand: picking one of three prefixes when opening the
PR, not computing a version number. A PR-title-linting GitHub Action (e.g.
`amannn/action-semantic-pull-request`) fails the check immediately if the prefix is
missing or malformed, catching the mistake before merge.

**Releasing.** [release-please](https://github.com/googleapis/release-please) reads that
same commit history and computes the correct next version automatically, but does not tag
anything by itself. It keeps a single, always-current "Release" pull request open,
showing the accumulated changelog and the version it is about to become. Merging it — the
one deliberate action the maintainer takes — creates the tag, the changelog, and the
GitHub release together. A `Release-As: 10.2.0` footer on any commit overrides the
computed version, for the rare case it needs correcting by hand. The same setup runs on
the `9` maintenance branch too, configured so that a `feat:` commit there fails the PR
check outright, mechanically enforcing "backward-compatible fixes only."

**Coordinating `PTS`.** Because `PTS` only needs to track SKIRT's `MAJOR.MINOR` (see
Versioning, above), the two repositories do not need a single, unified release pipeline —
each runs its own release-please setup, independently producing its own patch releases.
What is added is a narrow trigger: the last step of SKIRT's release workflow — whether
that release happened on `master` or on a maintenance branch — checks whether the
`MAJOR.MINOR` of the tag it just created differs from the previous one, and only then
fires a `repository_dispatch` event at the `PTS` repository, carrying the new
`MAJOR.MINOR`. A small workflow there tags the current tip of the matching `PTS` branch
as `<MAJOR.MINOR>.0`, whether or not `PTS` itself has any pending changes — from that
point on, `PTS`'s own release-please instance takes over and accumulates further patches
for that `MAJOR.MINOR` on its own, exactly like `SKIRT` does. A SKIRT patch release
changes nothing on the PTS side, and vice versa.

## Web

Unlike `SKIRT` and `PTS`, the `Web` repository does not follow the versioning and pull
request workflow described in this chapter. At all times, there is only one version of
the web site, maintained and published by the core SKIRT team, reflecting the most
recent commit to SKIRT 10 and PTS 10. The team simply does not have the resources to
maintain multiple versions of the web site and the documentation it contains, in parallel.

## Resources

SKIRT's "built-in" resources are published as downloadable resource packs. The `core`
resource pack must always be installed; the other packs need to be installed only when
the features that depend on them are actually used.

Resource packs are versioned, and the SKIRT code includes a list of the specific versions
expected for that commit. This scheme is agnostic to the SKIRT versioning proposed here,
and can therefore be used unchanged. Checking out another commit, whether on the `master`
or `9` branch, will automatically update the expected resource pack versions.

