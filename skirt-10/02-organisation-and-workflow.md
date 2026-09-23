# Organisation and workflow

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

## Workflow

### SKIRT/PTS

*(Placeholder — content to be written.)*

### Web

Unlike `SKIRT` and `PTS`, the `Web` repository does not follow the versioning and pull
request workflow described in this chapter. At all times, there is only one version of
the web site, maintained and published by the core SKIRT team, reflecting the most
recent commit to SKIRT 10 and PTS 10. The team simply does not have the resources to
maintain multiple versions of the web site and the documentation it contains, in parallel.
