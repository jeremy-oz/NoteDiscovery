# Fork notes — jeremy-oz/NoteDiscovery

Fork-specific notes for this branch. Not upstream documentation; upstream's docs
live in `documentation/`. This file exists so known issues and fork-maintenance
context don't have to be rediscovered.

## What this fork adds

An **Excalidraw editor** for `*.excalidraw` vector scenes, alongside upstream's
raster drawing editor. See [documentation/EXCALIDRAW.md](documentation/EXCALIDRAW.md)
for how it works.

Smaller fork changes to upstream behaviour:

- **A "Recently updated" panel** (clock icon in the icon rail, and a "Recent" tab
  in the mobile bottom bar). Every file in the vault, newest first, no folders,
  grouped Today / Yesterday / This week / This month / Older. The filter matches
  every typed word against the name and folder path, in any order and ignoring
  accents ("week 3 maths"); chips narrow it to notes, drawings or other files, and
  Enter opens the top match. It re-reads the vault when opened, when the tab
  regains focus (at most every 10 s) and from its refresh button, so edits made in
  Obsidian show up without a page reload. Logic is the `recent*` methods in
  `app.js`; strings are the `recent` section of every `locales/*.json`.
- **The homepage defaults to the list view** (upstream defaults to cards). A
  viewer who picks cards keeps it; the choice is per browser (`homepageView` in
  localStorage). Two lines in `app.js`: the `LOCAL_SETTINGS` default and the
  initial `homepageView` value.

| | |
|---|---|
| Branch | `feature/excalidraw-editor` |
| `origin` | `git@github.com:jeremy-oz/NoteDiscovery.git` (SSH — HTTPS has no stored credentials) |
| `upstream` | `https://github.com/gamosoft/notediscovery.git` (fetch only) |

## Local setup

The Excalidraw bundle is **built, not downloaded**, and is not committed. A fresh
checkout needs it once:

```bash
cd scripts/build_excalidraw && npm ci && node build.mjs
```

Without it the app runs fine, but opening a `.excalidraw` file reports that the
editor failed to load. Docker builds it automatically (stage 2b). The other
browser libraries come from `python scripts/vendor_assets.py`, which `run.py` runs
on first start.

---

## Known issues

### Cmd+Z undo not verified end-to-end

Excalidraw's undo/redo is handled by the component itself. The **toolbar undo
button works**, and a DOM probe confirmed the `Cmd+Z` keydown reaches both
`document` and the Excalidraw host untouched by our capture-phase listener — but
Excalidraw doesn't act on Playwright's *synthetic* key events, so automation
can't confirm the shortcut. **Needs one manual check in a real browser.**

If it turns out to be broken, the suspect is the capture-phase listener at the
bottom of `frontend/excalidraw-editor.js`, which calls `stopPropagation()` for
`z`/`y` only when focus is *outside* the canvas.

### elkjs is EPL-2.0 (weak copyleft)

The vendored bundle inlines 145 packages. All permissive except **elkjs 0.9.3**,
which is EPL-2.0 and arrives via `@excalidraw/mermaid-to-excalidraw` (the
"Mermaid to Excalidraw" dialog).

Redistribution in bundled form is permitted — the licence text ships in
`frontend/vendor/excalidraw/THIRD_PARTY_NOTICES.md`, elkjs is unmodified, and its
source is public. The obligation attaches to elkjs, not to NoteDiscovery or your
notes. Full reasoning in [documentation/THIRD_PARTY.md](documentation/THIRD_PARTY.md).

**Relevant if upstreaming:** upstream's `THIRD_PARTY.md` previously stated that
nothing bundled is copyleft. Dropping the mermaid-import feature would remove
elkjs, cytoscape and katex and restore that claim.

### CJK handwriting font excluded by default

Xiaolai is 12 MB, against ~480 KB for every other Excalidraw font combined, so
`build.mjs` skips it. CJK glyphs in a scene fall back to a system font. Pass
`--with-cjk` to include it.

---

## Fixed along the way (don't re-investigate)

- **Opening a scene rewrote its file.** `mount()` set `lastSavedJSON = null`, and
  Excalidraw emits an `onChange` on first render (loading normalises the scene —
  `"boundElements": null` becomes `[]`), so the dedupe guard in `save()` could
  never match and every open wrote the file. Harmless for content, but it churned
  mtime on read, which breaks sort-by-modified and makes Syncthing/Dropbox/git
  vaults re-upload on every view. `mount()` now seeds `lastSavedJSON` with the
  bytes it just read, so the comparison is against what is actually on disk.
  Safe because the server writes the PUT body verbatim. A scene that isn't yet
  canonical still costs exactly one normalising write, then settles.
- **Ctrl+S did nothing in a scene.** Excalidraw swallows the `s` keydown before it
  reaches `app.js`'s bubble-phase window listener. Now handled on the capture
  phase inside `excalidraw-editor.js`, which also stops Excalidraw reading the
  bare `s` as its stroke-colour shortcut.
- **React loaded from a CDN.** Replaced by the vendored bundle; the `importmap` in
  `index.html` is gone, since React is compiled in.

- **Opening a scene from another address rewrote it.** `serializeAsJSON` stamps
  `"source"` with `location.origin`, so a scene saved at `localhost:8000` and
  opened at a LAN hostname (or another port) differed from the file on its first
  `onChange` and was written back, on every open from that address. Vaults
  reached from more than one address churned indefinitely. `serializeMounted()`
  now keeps the `source` the scene was loaded with; new scenes keep the address
  they were created on.

## Versioning — bump on every fork change

`VERSION` carries a fork suffix: `<upstream version>+jc.<n>`, e.g. `0.31.7+jc.1`
(same style as the kanban-tui fork). **Bump `n` in every commit that changes
anything under `frontend/`**, and reset it to `+jc.1` when merging a new upstream
release (take upstream's number, add the suffix).

Why: the version is the browser cache key. Script URLs are `app.js?v=<version>`
and the service worker serves `/static/` cache-first under a cache named after the
version. Ship new frontend code under an unchanged version and every browser that
already has the app keeps running the old `app.js` against the new `index.html`
(which is not cached) — new buttons appear but do nothing, until a hard reload.

`+jc.n` is a valid PEP 440 local version, so `pyproject.toml` (which reads
`VERSION`) accepts it; after changing it, `uv sync --reinstall-package
notediscovery` refreshes the installed metadata. `release.ps1` is upstream's and
expects plain `X.Y.Z` — don't use it on this fork.

## Keeping up with upstream

```bash
git fetch upstream && git merge upstream/main
# VERSION will conflict on every upstream release: take theirs, append +jc.1
```

The feature is deliberately structured to keep this cheap:

- The editor lives in its own file, `frontend/excalidraw-editor.js`. New files
  never conflict.
- `app.js` carries only ~43 lines of wiring.
- `closeMediaViewer()` mirrors upstream's method, so the four navigation teardown
  paths auto-merge instead of conflicting.

Expect conflicts only around `closeMediaViewer()` / `viewMedia()` in `app.js`, the
script tags in `index.html`, the icon rail / mobile bottom bar in `index.html`
(the Recent button sits between Files and Search), the `homepageView` default lines, and the end of
`.gitignore` (both sides append there — keep both). Both are mechanical: keep upstream's version and
re-add the `ExcalidrawEditor.teardown()` call / the `excalidraw-editor.js` tag.

Last merged: **upstream v0.31.7** (3 Oct 2026).
