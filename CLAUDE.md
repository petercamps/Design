# CLAUDE.md

Project-local instructions for Claude Code working in this repo. This file is not part of
the design note itself — never link it from `_sidebar.md`, `_coverpage.md`, `README.md`,
or any chapter file.

## What this repo is

A standalone, multi-chapter design note ("HDF5 in SKIRT") on implementing HDF5
input/output/checkpoint support in SKIRT. Plain Markdown, no LaTeX/math. Rendered locally
via Docsify and, eventually, on GitHub Pages. Portions will later be copied/converted into
Doxygen-format pages for the SKIRT website (Doxygen has native Markdown support since 1.8,
so this mainly means adding `\page`/`\subpage` labels, not a full rewrite).

## Conventions

- Keep lines in `.md` files under 120 characters; rewrap paragraphs as needed.
- Chapter files are numbered to fix their order: `01-introduction.md`, `02-features.md`,
  `03-data-model.md`, `04-implementation.md`. New chapters follow the same pattern, and
  the corresponding link goes into `_sidebar.md` in the same change.
- The user edits files directly in this working copy alongside Claude. If a file has
  changed on disk since it was last read, treat that as the current, intentional state
  rather than reverting it.
- Don't commit or push unless explicitly asked to.

## Site structure (Docsify)

- `index.html` — Docsify bootstrap and config (site name, sidebar, cover page, TOC depth).
- `_sidebar.md` — chapter navigation.
- `_coverpage.md` — cover/landing page.
- `README.md` — homepage content, rendered below the cover page on the same route.
- `.nojekyll` — required empty file so GitHub Pages serves underscore-prefixed files
  (`_sidebar.md`, `_coverpage.md`) instead of Jekyll silently dropping them.

### Known Docsify gotcha

A Markdown link `[text](#/route)` written inside `_coverpage.md` gets rewritten by
Docsify's cover-page compiler into a same-page anchor scroll, no matter what it targets —
it cannot do cross-route navigation. Use a raw HTML anchor instead, e.g.
`<a href="#/01-introduction">Get Started</a>`.

## Local preview

```bash
cd /Users/pcamps/SKIRT/HDF5design
docsify serve .
```

Serves at `http://localhost:3000` with live-reload. `docsify-cli` is installed globally.
The plain install command fails on a broken `husky install` postinstall hook in a
transitive dependency; install with `npm install -g docsify-cli --ignore-scripts` instead
(safe to skip — that hook only matters if you were developing docsify itself).

## GitHub / Pages

- Repo: `github.com/petercamps/HDF5design` — private, personal account, remote `origin`,
  branch `main`.
- GitHub Pages is intentionally **off** for now. Even from a private repo, a published
  Pages site is publicly reachable by URL on a personal (non-Enterprise) plan, so this
  gets turned on deliberately once the note is ready for reviewers — not as a side effect
  of pushing.
