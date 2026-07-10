# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file marketing landing page for **Glider Grove** — a boutique release of
fourteen homesites at Mossy Point on the NSW South Coast, marketed by Agent Team
(agentteam.com.au). The entire deliverable is `index.html` (~765 KB, self-contained).
There is no package.json, build system, test suite, or linter.

## Running / previewing

Open `index.html` directly in a browser, or serve it:

```bash
python3 -m http.server 8000   # then visit http://localhost:8000
```

Note: the page fetches React 18.3.1, ReactDOM, and Babel standalone from unpkg.com at
runtime, so previewing requires network access. Fonts (Bebas Neue, Inter) are embedded
in the file itself.

## Architecture: a self-extracting bundle

`index.html` is NOT a normal HTML page — it is a bundler wrapper around the real page.
Editing the visible top-level markup will not change what renders. The file has three
`<script type="__bundler/...">` payload blocks:

- `__bundler/manifest` — JSON map of UUID → `{data, mime, compressed}`. `data` is
  base64 (gzip-compressed when `compressed: true`). Holds 19 assets: 16 woff2 fonts,
  the dc-runtime, the design-system bundle, and the `<image-slot>` scaffold.
- `__bundler/template` — a single JSON-encoded string containing the REAL page HTML.
  Asset references inside it are bare UUIDs.
- `__bundler/ext_resources` — external resource ID map (currently empty).

On `DOMContentLoaded`, the inline unpacker script decodes each asset to a blob URL,
string-replaces the UUIDs in the template, then swaps `document.documentElement` with
the parsed template and re-creates its scripts so they execute.

### Editing the page content

To change anything the user sees, you must round-trip the template payload:

1. Extract: `json.loads()` the text content of the `__bundler/template` script tag.
2. Edit the resulting HTML string.
3. Re-encode with `json.dumps()` and splice it back between the same script tags.

Do this with a small Python script; the payload is one enormous line, so line-oriented
edits on `index.html` won't work. Leave the manifest and unpacker script alone unless
adding/removing assets.

### Inside the template: dc-runtime page

The unpacked page uses the **dc-runtime** declarative component system (the runtime JS
is a generated asset — header says "GENERATED from dc-runtime/src/*.ts — do not edit"):

- Markup lives in `<x-dc>` ... `</x-dc>`; `<helmet>` inside it holds styles and asset
  script tags.
- Logic lives in `<script type="text/x-dc" data-dc-script="">` as
  `class Component extends DCLogic` with React-style `state` / `setState`. Everything
  returned from `renderVals()` is available in the markup as `{{ name }}` bindings —
  values, event handlers (`onclick="{{ scrollToForm }}"`), and rendered React elements
  (`{{ gliderMark }}`).
- `<sc-if value="{{ cond }}">` renders children conditionally (used to swap the
  register-interest form for its thank-you state).
- `<x-import component-from-global-scope="AgentTeamDesignSystem_cbc85e.Logo" ...>`
  mounts design-system components. The DS bundle exposes: `Logo`, `Stripe`,
  `SocialPill`, `LowerThird`, `Subtitle` under the global
  `AgentTeamDesignSystem_cbc85e` namespace.
- `<image-slot id="..." shape fit placeholder src credit credit-href>` are
  user-fillable image placeholders (currently two, both empty: `gg-hero` and
  `gg-lifestyle`). Every slot needs a distinct `id`. If `src` points at an Unsplash
  host, `credit` ("Photo by {name} on Unsplash") and `credit-href` are mandatory or an
  error tile renders instead of the photo.

Page sections, in order: top bar, hero, intro, stats band, the glider story,
lifestyle, the release (lot table, anchor `gg-release`), register interest (form,
anchor `gg-register`), footer.

## Brand conventions (Agent Team)

All styling is inline styles + CSS custom properties defined in `<helmet>`. Use the
tokens, not raw hex:

- **Palette** (`--at-*`): Coyote `#896C44` (primary accent), Lion `#BCA277`, 50% Lion
  `#DACBB2`, Dune `#F7F5EF` (background), 50% Dune `#F9F7F2`, Paper `#FFFFFF`, Black.
  Black is never used as a background. Semantic aliases (`--surface-*`, `--text-*`,
  `--combo-*`) and top-to-bottom gradient tokens (`--at-grad-*`) build on these.
- **Typography**: Bebas Neue for display headings — ALL CAPS, always regular weight
  (never bold), tight leading (~0.9). Inter for body and subheads. Tracking/leading
  live in tokens (`--display-*`, `--subhead-*`, `--body-*`). Split two-tone headings
  use Coyote + Lion spans.
- **Logo**: wordmark only, single tone — black on light backgrounds, paper on dark,
  never mixed.
