# CLAUDE.md

Project-local instructions for Claude Code working in this repo. This file is not part of
the design notes themselves — never link it from `_sidebar.md`, `README.md`, or any
chapter file.

## What this repo is

A collection of standalone, multi-chapter design notes for the SKIRT radiative transfer
code. Each note lives in its own top-level folder (e.g. `full-hdf5-support/` for "Full
HDF5 support"); the repo root is a landing page listing all notes. Plain Markdown, no
LaTeX/math.
Rendered locally via Docsify and on GitHub Pages. Portions will later be copied/converted
into Doxygen-format pages for the SKIRT website (Doxygen has native Markdown support since
1.8, so this mainly means adding `\page`/`\subpage` labels, not a full rewrite).

## Conventions

- Keep lines in `.md` files under 120 characters; rewrap paragraphs as needed. This does
  not apply to Markdown table rows (`| ... |`), which may run longer since a table row
  cannot be wrapped without breaking the table.
- Each design note lives in its own top-level folder, named in kebab-case after the note's
  title (e.g. `full-hdf5-support/`). Within a note, chapter files are numbered to fix their
  order: `01-introduction.md`, `02-features.md`, etc. New chapters follow the same
  pattern, and the corresponding link goes into that note's own `_sidebar.md` in the same
  change.
- A new design note gets its own folder with numbered chapters and its own `_sidebar.md`
  (see "Site structure" below), plus one added entry in the top-level `_sidebar.md` and
  `README.md` so it shows up on the landing page.
- The user edits files directly in this working copy alongside Claude. If a file has
  changed on disk since it was last read, treat that as the current, intentional state
  rather than reverting it.
- Don't commit or push unless explicitly asked to.

## Site structure (Docsify)

- `index.html` — Docsify bootstrap and config (site name, sidebar, TOC depth).
- `_sidebar.md` and `README.md` at the repo root — the landing page, listing every design
  note. There is no `_coverpage.md`; the home route is a regular content page like any
  other, not a separate hero section.
- `.nojekyll` — required empty file so GitHub Pages serves underscore-prefixed files like
  `_sidebar.md` instead of Jekyll silently dropping them.
- Each design note's folder has its own `_sidebar.md` with that note's chapter
  navigation, plus a `[← All design notes](/)` link back to the root. Docsify loads
  `_sidebar.md` from whichever folder the current page is in (falling back up the tree if
  none exists there) — this is default Docsify behavior with `loadSidebar: true`, no
  plugin or extra config required.

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

- Repo: `github.com/petercamps/Design` — public, personal account, remote `origin`,
  branch `main`. Made public specifically to enable Pages: on a personal (non-Enterprise)
  plan, Pages on a private repo requires a paid plan.
- GitHub Pages is enabled, serving from `main` at the repository root, live at
  `https://petercamps.github.io/Design/`.
- Renamed from `HDF5design` once the repo became a multi-note hub. GitHub redirects the
  old name for git and web access, but a further rename would move the live Pages URL
  again, so update this file's URLs if that ever happens.
