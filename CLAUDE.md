# CLAUDE.md

Guidance for working in this repo. Read this before editing.

## What this is

Martin Fir's personal portfolio — a **fixed single-screen one-pager** (no page
scroll on desktop). Left is a slim vertical **belt of projects**; right is a
**media viewer** with Stills / BTS / Breakdown tabs. Media can be an image, a
video, a before/after **compare** wiper, or a **3D model** (GLB).

Aesthetic: darkroom / DI-suite. Near-black calibrated background, single teal
safelight accent, grain + scanlines + vignette, live timecode in the header.

## Stack & constraints

- **Vanilla HTML/CSS/JS. No framework, no build step, no package manager.**
  Do not introduce React/Vite/Tailwind/npm. Edits are made directly to the
  `.html` files.
- **Three.js only**, loaded via CDN importmap (`three@0.160.0`) for the model
  viewer. No other runtime deps.
- **Static hosting (GitHub Pages).** Everything must work as plain files served
  over http. No server-side code.
- Must be **served over http**, not opened via `file://` — both `fetch()` of the
  JSON and ES module imports require it.

## Run locally

```
python -m http.server 8000
# site: http://localhost:8000/
# cms:  http://localhost:8000/cms.html
```

There is nothing to build or install.

## Folder structure

```
index.html              the site
cms.html                local admin tool — edits data/projects.json (keep unlinked)
data/projects.json      single source of truth for all content
img/                    stills, video poster frames, compare plates (.webp)
video/                  clips (.mp4) + their poster frames (.webp)
models/                 GLB files for the 3D viewer
.nojekyll               disables Jekyll so assets serve verbatim
README.md               human setup notes
```

All asset paths in `projects.json` are **relative to the repo root**
(`img/...`, `video/...`, `models/...`). The file on disk must match the path
in the JSON exactly, **including case** (Pages is case-sensitive).

## Data model

The site renders from `data/projects.json` (an array). `index.html` fetches it
on load and falls back to the inline `DEFAULT_PROJECTS` array if it's missing or
opened over `file://`. **`data/projects.json` is authoritative** — if you change
content, change the JSON (via the CMS), not the inline fallback. Keep the inline
fallback only as a rough mirror.

Project shape:

```json
{
  "code": "001",
  "title": "Joker Out",
  "role": "Odsevi Sonca · Comp · Set Ext · DI",
  "year": "2025",
  "media": [ ...media items... ]
}
```

Media item — `type` is one of `image | video | model | compare`,
`cat` is one of `stills | bts | breakdown` (drives which tab it appears under):

```json
{ "type":"image",   "cat":"stills",    "src":"img/shot.webp", "caption":"..." }
{ "type":"video",   "cat":"breakdown", "src":"video/clip.mp4", "poster":"video/clip-poster.webp", "caption":"..." }
{ "type":"model",   "cat":"stills",    "src":"models/asset.glb", "caption":"..." }
{ "type":"compare", "cat":"breakdown", "before":"img/plate.webp", "after":"img/final.webp",
                    "labelBefore":"GREENSCREEN", "labelAfter":"FINAL", "caption":"..." }
```

Missing `src`/`before`/`after` render placeholder "plates" (gradient fills, green
for compare-before) so the site never looks broken with partial content.

## How the site works (`index.html`)

One `<script type="module">`. Key pieces:

- `loadProjects()` — boot: fetch JSON → fallback to `DEFAULT_PROJECTS` → build UI.
- `buildRail()` — renders the left project belt. Thumb via `mediaThumb(m)`.
- `selectProject(i)` — sets the active project, picks first non-empty tab.
- `buildTabs()` / `buildStrip()` — category tabs and thumbnail strip.
- `showMedia(idx)` — routes by `m.type` to one of:
  - `paintImage`, `paintVideo`, `paintCompare`, or `openModel`.
- `mediaThumb(m)` — shared thumb source (image `src` / video `poster` /
  compare `after`). Use this whenever you need a representative thumbnail.
- **Compare slider** (`paintCompare`): clips the "after" layer with
  `clip-path: inset(0 X% 0 0)`; pointer drag with `setPointerCapture`.
- **3D viewer** (`ensureThree` / `openModel`): lazy, reused across projects.
  GLTFLoader + DRACOLoader + MeshoptDecoder, OrbitControls (auto-rotate),
  AnimationMixer, ACES tone mapping. Camera framing uses `ZOOM_MULT` (2.2).
  Exposes `window.__viewer`.
- Keyboard: ↑/↓ switch project, ←/→ step media.

### Theming (do not hardcode the accent)

Colors/fonts live in CSS custom properties in `:root`. The accent is a **single
variable** used everywhere:

```css
--safe: #34b3bd;   /* DI-suite teal */
--safe-dim: #1d6a70;
```

The Three.js rim light reads `--safe` at runtime, so changing those two lines
recolors the whole UI **and** the 3D key light. Never hardcode the accent hex in
new code — reference `var(--safe)` (CSS) or read the property (JS).

Fonts: Big Shoulders Display (display/headings), IBM Plex Mono (everything else).

## How the CMS works (`cms.html`)

Standalone vanilla app, no deps. Three panes: project list (drag / ↑↓ reorder) ·
editor · live preview (mirrors the site viewer, compare slider included).

- Loads `data/projects.json` on open; autosaves a working draft to
  `localStorage` (`fir_cms_draft`) wrapped in try/catch.
- **Pick** buttons take a local file, create an object URL for instant preview,
  and set the media path to `FOLDER[type]/<sanitized-name>` (img/video/models).
  Browsers can't write to the repo, so the preview also renders a
  "Files to place in repo" checklist — the user copies those files in manually.
- **Export projects.json** downloads the file; **Copy JSON** to clipboard;
  **Reset** clears the draft and reloads from disk.
- `cms.html` is committed but must stay **unlinked** from the site. It's
  harmless (only exports a file) but shouldn't be surfaced publicly.

## Asset pipeline (reproduce these specs)

Keep the site light. Convert before committing.

```bash
# stills -> WebP, max 1600px wide
ffmpeg -i in.png -vf "scale='min(1600,iw)':-2" -c:v libwebp -quality 82 img/name.webp

# clips -> web mp4 (H.264, faststart, audio stripped for silent breakdowns)
ffmpeg -i in.mp4 -an -c:v libx264 -profile:v high -pix_fmt yuv420p \
  -crf 23 -preset slow -movflags +faststart video/name.mp4

# poster frame for a clip
ffmpeg -ss 4 -i in.mp4 -frames:v 1 -c:v libwebp -quality 80 video/name-poster.webp
```

- **Models:** export GLB with Draco (and meshopt where it helps). The loader is
  already configured for both. Keep hero GLBs lean.
- Do **not** host the full copyrighted Joker Out music video — link to YouTube
  (`s7Cl-4GZQU4`) instead.

## Conventions when editing

- Provide **complete files / full code**, not partial snippets, unless asked for
  a snippet specifically.
- Preserve the existing patterns: media-type routing in `showMedia`, the
  `mediaThumb` helper, CSS-variable theming, the JSON-first-with-inline-fallback
  loading model.
- Lowercase, hyphenated asset filenames (the CMS does this). Watch case.
- When adding a new media type: add a `paint*` renderer, a `showMedia` branch, a
  `mediaThumb` case, a strip badge, and a CMS editor field + preview branch.
- Don't add a build step or external dependencies to keep Pages deployment a
  zero-config `git push`.
```
