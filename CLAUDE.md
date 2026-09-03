# CLAUDE.md

Guidance for working in this repo. Read this before editing.

## What this is

Martin Fir's personal portfolio — a **single-column vertical scroll** of
projects. Each project is one full-width 16:9 **preview frame** (its hero clip)
above a caption bar holding just the **title and year** — no index number, no
discipline labels. Clicking a frame opens a **full-screen modal** whose media is
a **horizontal scroll-snap carousel**: arrows, dots, ←/→ and swipe step through
that project's media. There is no side rail and no viewer pane.

On **phones (≤760px) the feed goes full-bleed and full-height**: no column
gutters, `.proj` is exactly `100dvh - nav` tall, and `.proj-stage` drops its
16:9 ratio (`flex:1; aspect-ratio:auto`) to take all the height left over above
the title bar. The media is `object-fit:cover`, so it crops to fill the screen.

**Exactly one video plays at a time.** Every `<video>` is watched by a single
IntersectionObserver; the most visible clip plays and all others pause. While
the modal is open it *owns* playback (`vidScope`), so the feed behind it stays
paused. Feed clips carry a small **icon-only** mute toggle and a bare blinking
dot marking the live one; **modal clips use the browser's native player**
(`controls`). Media can be an image, a video, a before/after **compare** wiper,
or a **3D model**.

Aesthetic: monochrome and quiet, on a **light warm-grey canvas** with near-black
ink — colour comes only from the work. **One sans across the whole site**
(Helvetica Neue/Arial): `--font-d` and `--font-m` are the same stack, no serif,
no webfont request by default. No grain/scanline/vignette FX (the layers still
exist and are wired to the CMS sliders, but all default to 0).

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
- `buildFeed()` — renders one `<article class="proj">` per project: preview
  frame + a caption bar of title/year only.
- `heroOf(p)` — the clip that represents a project: first `video` with a `src`,
  else the first media item.
- `openProjectModal(i)` / `closeProjectModal()` — the carousel is built **on
  open** and torn down on close, so a project never loads its media until asked.
  Open sets `vidScope = pmTrack` and locks page scroll; close reverses both,
  unobserves the modal's videos and returns the 3D stage.
- `slideHtml(m)` — one `.pm-slide`; routes by `m.type` to an `<img>`, a video,
  `compareHtml`, or a 3D slot, each inside a contain-fitted `.pm-frame`.
- `pmGoTo(n)` / `pmSync()` / `pmPaint(n)` — every slide is exactly one
  track-width wide, so index ↔ `scrollLeft` is just multiplication. `pmGoTo`
  scrolls and paints optimistically; `pmSync` (debounced on `scroll`) is the
  source of truth and covers swipes. **Don't call `pmSync()` straight after
  `pmGoTo()`** — the smooth scroll hasn't landed yet and it will snap the
  counter back to the old slide.
- Modal videos are marked `data-native` and carry `controls`. `syncSound()`
  **returns early for them** — the browser owns their mute state, and forcing it
  would undo the viewer's own click on the player. Only feed clips are driven by
  the custom toggle.
- `mediaThumb(m)` — shared thumb source (image `src` / video `poster` /
  compare `after`). Use this whenever you need a representative thumbnail.
- **One-video-at-a-time**: `vidObserver` tracks the intersection ratio of every
  `video[data-feed]` in `vidRatio`; `updateActiveVideo()` plays the highest-ratio
  clip (≥0.3) and pauses the rest. `pauseAll(true)` stops everything (About /
  Sketchbook); `setVideoScope(el)` restricts playback to one subtree (the project
  modal) — without it the feed, still intersecting the viewport behind the
  overlay, would steal the slot. New videos need `registerVideos(root)`.
  - `play()` ignores **AbortError** — that just means a `pause()` superseded a
    still-buffering `play()`. Retrying there resurrects stopped videos.
  - Videos are `preload="none"`; `play()` promotes the active one to `auto`.
  - Chrome drops a `<video>`'s `poster` once `play()` is called, so the poster is
    *also* painted as the frame's CSS background (`posterBg()`) and the video is
    held at `opacity:0` until `loadeddata` adds `.ready`.
- **Compare slider** (`compareHtml` / `initCompare`): clips the "after" layer
  with `clip-path: inset(0 X% 0 0)`; pointer drag with `setPointerCapture`.
  Left half shows `after`, so the left tag is `labelAfter`.
- **3D viewer** (`ensureThree` / `openModel`): lazy, one instance. The
  `#stage3d` element is *moved* into whichever `.model-slot` asks for it and
  returned to `#modelHost` by `closeModel()`. GLTFLoader + DRACOLoader +
  MeshoptDecoder, OrbitControls (auto-rotate), AnimationMixer, ACES tone
  mapping. Camera framing uses `ZOOM_MULT` (2.2). Exposes `window.__viewer`.
- Scroll reveal: `revealObserver` adds `.seen`. If IntersectionObserver is
  missing the feed gets `.no-reveal` — never leave the work at `opacity:0`.
- Keyboard: ↑/↓ move between projects, Enter/Space opens the focused one. Inside
  the modal ←/→ step slides and Esc closes.

### Theming (do not hardcode the accent)

Colors/fonts live in CSS custom properties in `:root`. The accent is a **single
variable** used everywhere:

```css
--bg:        #dedcd6;   /* light warm-grey canvas      */
--panel:     #ebeae6;   /* cards/modals above the canvas */
--frame:     #cecdc7;   /* media frame before art loads */
--line:      #c2c0b9;
--ink:       #16150f;   /* near-black, warm            */
--ink-dim:   #55534c;   /* secondary text              */
--ink-faint: #86847c;   /* tertiary text               */
--safe:      #16150f;   /* accent — colour comes from the work */
--on-media:  #ffffff;   /* overlays that sit ON footage */
--rim:       #ffffff;   /* 3D key light                */
```

Never hardcode the accent hex in new code — reference `var(--safe)` (CSS) or
read the property (JS). Two traps, both fixed once already:

- **Anything drawn over media** (the live dot, compare line/grip/tags, the model
  loader) must use `--on-media`, not `--safe`. It has to read over arbitrary
  footage regardless of the page theme.
- **The Three.js rim light reads `--rim`, not `--safe`** — on a light page the
  accent is near-black and would switch the key light off entirely.

`applyAccent()` derives `--safe-dim` by **mixing toward `--bg`**, not by
darkening. On a light canvas darkening makes the "dim" shade stronger than the
accent, inverting every active/inactive pair (the carousel dots caught this).
`about.accent` in `projects.json` is authoritative and overrides `:root` at
runtime — it must stay dark while the canvas is light.

Fonts: **one sans everywhere.** `--font-d` and `--font-m` are both the
Helvetica Neue/Arial stack; the `editorial` entry in `FONTS` has `url:null`, so
`<link id="gfonts">` ships with no `href` and no webfont is fetched unless a
different CMS preset asks for one. Do not reintroduce a serif display face, the
teal safelight, the occult sigils or the grain/glow filter stack — all
explicitly rejected.

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
- Preserve the existing patterns: media-type routing in `slideHtml`/`heroHtml`,
  the `mediaThumb` helper, CSS-variable theming, the
  JSON-first-with-inline-fallback loading model.
- All interpolated content goes through `esc()`. The feed is built with
  `innerHTML` from user-authored JSON — don't bypass it.
- Lowercase, hyphenated asset filenames (the CMS does this). Watch case.
- When adding a new media type: add a `slideHtml` branch, a `heroHtml` branch if
  it can lead a project, a `mediaThumb` case, and a CMS editor field + preview
  branch.
- `role` and `cat` are still written by the CMS but no longer displayed — the
  caption bar is title/year only and the modal shows media in authored order.
- Don't add a build step or external dependencies to keep Pages deployment a
  zero-config `git push`.
```
