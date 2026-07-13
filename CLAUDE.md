# CLAUDE.md — Succulent Care page spec

Spec for working on this repository. Read this before editing.

## What this is

A static web page documenting indoor care for a personal succulent & caudex collection.
Deployed via GitHub Pages at the custom domain in `CNAME` (`steven-xia.com`). No build step,
no framework, no dependencies, no backend. The **UI** (markup, CSS, logic) lives inline in
**`index.html`**; the **per-plant data** lives in **`plants.json`**, which `index.html`
`fetch()`es at runtime. The page is **English-only**.

### Files

| File | Role |
|------|------|
| `index.html` | The app shell — HTML, CSS, and JS (rendering/filtering/search), all inline. Holds **no** plant data. |
| `plants.json` | **All per-plant data.** `{ "$schema": "./plants.schema.json", "plants": [ … ] }`. Fetched at load. |
| `plants.schema.json` | Rigorous JSON Schema (draft 2020-12) that `plants.json` must validate against. |
| `plants.md` | **Canonical roster** of every plant, by academic Latin name. Human-facing source of truth. |
| `images/` | One `<id>.jpg` per plant (card + detail hero), plus `icon.png` (favicon). |
| `CNAME` | Custom domain for GitHub Pages. |
| `README.md` | Human-facing blurb. |

> **`fetch` requires HTTP.** Because the data is fetched, opening `index.html` directly from
> the filesystem (`file://`) shows a graceful "couldn't load plant data" message and an empty
> grid — browsers block `fetch` of local files. Serve over HTTP for local viewing/testing
> (`python3 -m http.server`); GitHub Pages serves over HTTPS so production is fine.

## Sources of truth & the sync invariant

Two files describe the plant set and must always agree:

- **`plants.md`** — the canonical roster (Latin names only, alphabetical, nothing else).
- **`plants.json`** — the full records that the page renders.

**Every Latin name in `plants.md` MUST have exactly one record in `plants.json`, and vice
versa.** When asked to **add** a plant, add it to both (see "Adding a plant"); when asked to
**remove** one, remove it from both. The join key between the two is the **Latin name**. After
any change to the plant set, re-verify they're in sync and that `plants.json` still validates
against `plants.schema.json` (see "Verifying").

## `plants.json` — the data contract

Top level: `{ "$schema": "./plants.schema.json", "plants": [ <plant>, … ] }`. Each `<plant>`:

```jsonc
{
  "id": "aloe-arborescens",        // stable kebab-case slug; also the image filename stem & #hash
  "latin": "Aloe arborescens",     // academic name — the ONLY name shown (card + sheet). Join key to plants.md.
  "family": "Asphodelaceae",       // shown as a chip on the detail sheet
  "habit": "Succulent shrub",      // shown as a chip on the detail sheet
  "lightLevel": "high",            // "high" | "medium" | "low"  — see Light level rubric
  "waterLevel": "medium",          // "high" | "medium" | "low"  — see Water level rubric
  "dormancy": "none",              // "summer" | "winter" | "none" — drives the Dormancy filter & detail-sheet chip
  "care": {                        // the detail-sheet copy; all six required, rendered in this order
    "light": "…", "water": "…", "dormancy": "…", "soil": "…", "feeding": "…", "tips": "…"
  }
}
```

Rules (enforced by `plants.schema.json` — keep it authoritative; update it if the shape ever
changes):
- **No additional properties** at either level. **No `common`/nickname field and no `chip`
  field** — deliberately omitted; do not reintroduce them. Only the Latin name is displayed.
- `id`: `^[a-z0-9]+(?:-[a-z0-9]+)*$`, unique across the array. It is a **stable, arbitrary
  slug**, NOT mechanically derived from `latin`. Several ids are shortened/renamed relative to
  the Latin name — e.g. `Adenium obesum ‘Godzilla’` → `adenium-godzilla`,
  `Astrophytum asterias ‘Onzuka’` → `astrophytum-onzuka`, `Greenovia (Aeonium aureum)` →
  `greenovia`, `Mammillaria compressa cv. ‘Yokan’` → `mammillaria-compressa-yokan`,
  `Haworthia emelyae var. comptoniana ‘Oyayubihime’` → `haworthia-comptoniana-oyayubihime`.
  **Never change an existing id** (it breaks image paths and `#hash` deep links).
- `care` values are plain text (no HTML). Use typographic punctuation (`—`, `‘’`, `°F`, `¼`);
  a leading `⚠` in `tips` flags toxicity/hazards. Preserve these conventions.
- Note the two `dormancy`s: the top-level `dormancy` **enum** (classification, drives the
  filter and detail-sheet chip) is distinct from `care.dormancy` (the prose paragraph about
  the plant's cycle).

### Light level — definitions (use these exactly)
`lightLevel` classifies how much light the species needs to actually thrive, not merely
survive. Assign it from the plant's real requirement, using these fixed definitions. They are
absolute categories, so the same species always lands in the same bucket:

- **`"high"` — requires full, intense sun.** The plant genuinely needs maximum light to grow
  well: direct, unobstructed, super-intense outdoor **afternoon** sun is beneficial (not
  harmful) to it. Indoors this means it wants the strongest position available and/or a
  powerful grow light running long hours; it will etiolate, bloat, or lose form/color without
  it. Typical of most cacti and desert/full-sun succulents (e.g. Lithops, Astrophytum,
  Adenium, most Euphorbia).
- **`"medium"` — bright indirect sun, or intense direct **morning** sun.** Thrives in strong
  but not scorching light: an unshaded bright spot out of harsh afternoon rays, or a few hours
  of gentler direct morning sun. Fierce afternoon sun can scorch it. Typical of forest/edge
  or soft-leaved species.
- **`"low"` — enough light to do fine on an indoor windowsill.** Content with modest light: a
  regular bright-ish indoor windowsill is sufficient for it to be healthy; it does **not**
  require intense direct sun and can be bleached or stressed by it. Typical of shade-tolerant,
  window-leaved, or thin-leaved species (e.g. Haworthia).

When adding a plant, pick the bucket by asking "what does this species **need to thrive**?" —
map "needs full/intense sun" → `high`, "wants bright but not harsh / morning sun only" →
`medium`, "happy on an ordinary windowsill" → `low`. Keep it consistent with the plant's
`care.light` copy.

### Water level — definitions (use these exactly)
`waterLevel` classifies how thirsty the species is **during its active growing season**. All
watering here uses the **soak-and-dry** method in the gritty mix (drench thoroughly, then let
it dry) — the levels differ only in **how long the substrate is left dry before the next
drench**. They are absolute categories, so the same species always lands in the same bucket:

- **`"high"` — drench with essentially no dry resting period.** In active growth, water
  thoroughly and then re-drench **as soon as the substrate has dried out**. The mix cycles
  wet → dry → wet with little to no time spent dry. This is the **wettest sustainable regime**
  for these plants. Typical of thirsty growers like Adenium and some leafy/warm-growing
  caudiciforms (e.g. Dorstenia, Sinningia).
- **`"medium"` — drench once the substrate is fully dry.** Standard soak-and-dry with a normal
  dry interval: let it dry out completely, then water. No extended bone-dry rest.
- **`"low"` — let it go bone dry, then wait a bit longer before drenching.** An extended dry
  rest between waterings; err toward under-watering. Typical of rot-prone, water-storing, or
  mesemb-type plants (e.g. Lithops, Fenestraria, Euphorbia obesa).

When adding a plant, pick the bucket by asking "in active growth, how soon does this species
want water again after the mix dries?" — "immediately / keep it moving" → `high`, "when fully
dry" → `medium`, "well after bone dry" → `low`. Keep it consistent with the plant's
`care.water` copy.

**Growing-season only.** `waterLevel` describes the *active-growth* baseline. Dormant-season
watering is a separate, much lower regime (often near zero) and is conveyed by the `dormancy`
field and the `care.dormancy` copy — NOT by `waterLevel`. A `"low"`-water plant may still
receive *zero* water while dormant. Never let this collection's substrate stay continuously
wet; the gritty mix plus soak-and-dry is assumed throughout.

## `index.html` structure & rendering

- `<head>`: meta, `<link rel="icon" href="images/icon.png">`, inline `<style>`, static
  `<title>Succulent Care — Indoor Collection</title>`.
- Toolbar: `#facets` (filter buttons, built by JS) and `.tb-search` with `<input id="search">`.
- `<main class="grid" id="grid">`: the card grid. `<footer>`: static care-assumptions note.
- Off-screen `<svg><defs>` of reusable icon symbols (`#i-sun`, `#i-water`, …).
- `.overlay#overlay > .sheet#sheet`: the modal detail sheet, filled by JS.
- Inline `<script>`: constants + rendering/interaction logic. Holds **no** plant data —
  `let PLANTS = []` is populated from `plants.json` in `init()`.

Constants in the script:
- `DORM_LABEL = {summer:"Summer dormant", winter:"Winter dormant", none:"Evergreen"}` — the
  dormancy label text (detail-sheet chip and search), keyed by a plant's `dormancy`.
- `CARE_FIELDS = [key, label, iconId]` per care section, e.g.
  `["light","Sunlight & Grow Lights","i-sun"]` — controls render order, section heading, and
  which `#i-*` icon each care section uses.
- `FACETS` — the three filter groups: `dormancy` (record field `dormancy`), `water` (field
  `waterLevel`), `light` (field `lightLevel`), each with a `label` and `opts:[[value,label]]`.
  Option values must match the data values.

Flow:
- `init()` (async): `fetch("plants.json")` → `PLANTS = (await res.json()).plants` → `renderFacets()`
  + `renderGrid()` → open a plant if the URL has a `#<id>` hash. On fetch failure it writes a
  load-error message into the grid and returns.
- `renderFacets()` builds the filter buttons from `FACETS`; `syncPressed()` sets `aria-pressed`.
  Clicking a facet toggles it (re-click clears); "All plants" clears all.
- `renderGrid()`: filters `PLANTS` by active facets, then by the search query, sorts by Latin
  name, renders cards (image + Latin name). Empty → "No plants match …".
- **Search** (`haystack`): matches Latin name, family, habit and dormancy label. Multi-token
  AND, case-insensitive. No nicknames to match. Typing any printable key while not focused in a
  field focuses the search box.
- Detail sheet: `openPlant(id)` → `renderSheet(p)` fills `#sheet` with the hero image
  (`images/<id>.jpg`), Latin name `<h2>`, family/habit/dormancy chips, and the six care
  sections. Opens from a card click or `#<id>` URL hash. Closes on backdrop click, the ×
  button, or Escape; syncs the URL hash.

## Styling / theming

- Inline `<style>` only. CSS custom properties in `:root` define the palette; a
  `@media (prefers-color-scheme: dark)` block overrides them for dark mode. Prefer editing
  variables over hardcoding colors.
- Serif display face for Latin names & headings (`--serif`), sans for everything else.
- Responsive card grid via `grid-template-columns:repeat(auto-fill,minmax(210px,1fr))`.
- No external fonts, scripts, or stylesheets — the shell stays dependency-free.

## Images

- Each plant needs `images/<id>.jpg` (used for both the card thumbnail and the sheet hero).
  Missing image → broken-image box; the layout still works. When you add a plant, the image
  file must be supplied separately — you generally can't create the photo, so call it out.
- Do not rename existing image files; ids/filenames are load-bearing.

## Adding a plant (checklist)

1. `plants.md`: add the Latin name (keep the list alphabetical, Latin-only).
2. `plants.json`: add a record with a new stable `id`, `latin`, `family`, `habit`,
   `lightLevel`, `waterLevel`, `dormancy`, and all six `care` fields. Follow the level/dormancy
   rubrics above.
3. Confirm `plants.json` still validates against `plants.schema.json`.
4. Add `images/<id>.jpg` (or flag that it's needed).
5. Verify (below).

Removing a plant reverses steps 1–2 (and optionally its image).

## Verifying

No test suite. The page fetches data, so verify over **HTTP**, not `file://`:

```bash
python3 -m http.server 8000 &                      # serve the repo
CHROME="/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"
"$CHROME" --headless --disable-gpu --dump-dom \
  "http://localhost:8000/index.html" | grep -c 'class="card"'   # card count == plant count
```
Deep-link a sheet with `…:8000/index.html#<id>`. To exercise a search/filter headlessly, inject
a line inside `init()` after `renderGrid();` (set `searchEl.value`+`curQuery`, or set `facets`
+ `syncPressed()`, then `renderGrid()`), save a temp copy, and dump its DOM. Also validate the
data file directly with Python (`json.load`; use the `jsonschema` package if present, else a
manual check of required keys / enums / unique ids). Sanity checks after edits:
- `plants.json` validates against `plants.schema.json`;
- card count equals the number of entries in `plants.md`;
- every `latin` in `plants.md` appears in the rendered grid, and none extra;
- a new plant's care text and family/habit/dormancy chips render at `#<id>`.

## Conventions & guardrails

- Two artifacts: an inline **`index.html`** shell and a **`plants.json`** data file. Keep them
  separate — no plant data in `index.html`, no rendering logic in the JSON. Don't add a build
  step or runtime dependencies.
- `plants.schema.json` is the authoritative data contract; when the record shape changes, change
  the schema in the same edit and keep `plants.json` valid.
- English-only. Only the Latin name is shown for a plant — no common/nickname names anywhere
  (data, display, or search). These were explicit product decisions.
- Preserve the `⚠` toxicity convention and typographic punctuation in care copy.
- Never change an existing `id` (breaks images and shared `#hash` links).
- Deployment is automatic via GitHub Pages on push to the default branch (served over HTTPS, so
  the runtime `fetch` works). For local viewing, serve over HTTP.
