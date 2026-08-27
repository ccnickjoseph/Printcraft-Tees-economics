# CLAUDE.md

Orientation for Claude Code sessions on this repo. Read this before editing `index.html`.

## What this is

A single-file unit-economics calculator for Printcraft Tees' Printful print-on-demand
catalog. It answers one question in many variations: **on this product, on this channel,
at this order size, what do we actually keep?**

- One file: `index.html`. Everything inline — markup, CSS, data, logic, chart.
- No build, no dependencies, no framework, no tests, no package.json.
- ES5-style vanilla JS in one IIFE. `var`, function declarations, string-concatenated HTML.
- Deployed by GitHub Pages from `main`. Commit, wait ~1 minute, it is live.

Live: https://ccnickjoseph.github.io/Printcraft-Tees-economics/ (case-sensitive path)

## Who uses it and how

Nick (owner) and whoever he shares a link with. It is a decision tool, not a report:
someone opens it, moves inputs until a scenario looks right, and sends the resulting URL
as the artifact. See `README.md` for the use cases, `docs/MODEL.md` for the economics,
`docs/ROADMAP.md` for what to build next and why.

## Non-negotiables

Break any of these and you have broken the product, not just the code.

1. **It stays one self-contained file.** No build step, no npm, no bundler, no external JS.
   Google Fonts via CDN is the only remote dependency. If a change seems to need tooling,
   the change is wrong for this repo.
2. **Every input round-trips through the URL hash.** Any new piece of state must be added
   to `encode()` *and* `decode()` (`index.html:462`, `:470`). A snapshot link that silently
   drops a setting is worse than no snapshot, because the recipient sees wrong numbers
   with no indication anything is missing.
3. **Every new input shows up in `diffList()`** (`:492`). The "Changed from baseline" panel
   is the approval artifact — it is how a reader knows which numbers were touched. An input
   that can change results but never appears in the diff is a silent lie.
4. **Unverified numbers stay visibly flagged.** Placeholder retail prices and low-confidence
   fee rates carry badges, banners, and footnotes on purpose. Do not quietly promote a guess
   to a fact. If you confirm a real value, update the data *and* clear its flag *and* note the
   source and date in `docs/MODEL.md`.
5. **The repo and the site are public.** Blank costs and margins are visible to anyone with
   the URL. That was a deliberate call (costs are not competitive today). Do not add anything
   here that would not survive being public — no API keys, no customer data, no supplier terms
   under NDA. There is no auth and a client-side password would not be one.

## Code map (`index.html`)

| Lines | What |
|---|---|
| 1–10 | head, Google Fonts, `noindex` |
| 11–126 | all CSS; palette in `:root` at `:13` |
| 128–327 | markup — left rail is inputs, right column is output panels |
| 330 | IIFE opens (`856` closes it) |
| 333–335 | `QTYS` quantity ladder, `LABEL_Q` chart-label thinning |
| 339–361 | `PRODUCTS`, `PIDS` — 5 Printful blanks, cost + spec, keyed by style id |
| 364 | `SHIP` — US Printful shipping by category (`tee` / `fleece`) |
| 366–383 | `CHANNELS`, `CIDS`, color maps |
| 386–398 | `baseDefaults()` → `DEFAULTS` (the baseline every diff compares to) and `S` (live state) |
| 400–408 | `baseUrl()`, `clone`, `money`, `pct`, `esc` |
| 410–427 | `unitCost`, `shipCost`, `shipCharged`, `feeRate` |
| 429 | **`calc()` — the entire economic model.** One function, returns a row object |
| 452 | `syncShipRates()` — shipping cost follows the selected product's category |
| 462–490 | `encode()` / `decode()` — base64url snapshot of `S` |
| 492 | `diffList()` — `S` vs `DEFAULTS` |
| 535 | `chartSVG()` — hand-built SVG, no chart library |
| 593 | `render()` — rebuilds every panel's innerHTML from scratch |
| 767–811 | event wiring; `bind()` at `:787` is the standard input helper |
| 812–831 | snapshot copy / load / reset |
| 833 | `syncInputs()` — pushes `S` back into the DOM inputs |
| 851–855 | boot: decode hash → sync → render |

**Render model:** every input event mutates `S` and calls `render()`, which rebuilds all
output HTML from scratch. Fine at this size. If it ever lags, that is the first thing to fix —
do not pre-optimize it now.

## How to make a change

Adding an input is the most common task. The full checklist:

1. Add the field to `baseDefaults()` so it has a baseline.
2. Add the markup in the left rail.
3. Wire it — `bind("id", fn)` for numeric/text inputs, an explicit listener for selects and
   checkboxes.
4. Read it in `calc()` (or wherever it applies).
5. Add it to `encode()` and `decode()`.
6. Add it to `diffList()`.
7. Add it to `syncInputs()` so Reset and snapshot-load push it back to the DOM.

Miss step 5, 6, or 7 and the bug is invisible until someone shares a link.

## Verify economics numerically, not by eye

Before claiming any finding about margins, cliffs, or thresholds, reproduce the arithmetic
in a scratch Node script rather than reasoning about it. The model is small enough to
transcribe in ~10 lines, and step functions (free-shipping threshold, Amazon fee tiers) make
intuition unreliable. Example of a real result that eyeballing gets wrong: raising the free
shipping threshold does not remove the 4-unit profit cliff, it relocates it — see
`docs/MODEL.md`.

There are no tests. Manual check after any edit: open the file, change an input, confirm the
table and chart move, hit Reset, then Copy link and load the link in a fresh tab.

## Known rough edges

- `syncShipRates()` (`:452`) uses a `S.lastCat` sentinel to decide when to reset ship rates on
  product switch. Fragile; worth a cleaner approach if you are in there anyway.
- `shipCost()` (`:413`) branches on `S.sfOverride` / `S.saOverride`, which are never assigned
  anywhere in the file. Dead code that reads like a feature.
- Fee overrides (`S.fo`) store raw strings from the input, not numbers; coerced at read time in
  `feeRate()`. The `!==undefined && !==null && !==""` guard is load-bearing — `0` is a valid
  override.
- The `#cliff` div (`:274`) is **not** a quantity-profit cliff detector. It holds the Amazon
  *fee* tier warning only (`:684`). The quantity cliff detector is unbuilt — pick a different id.
- No persistence beyond the URL hash. No saved scenario library.
- `history.replaceState` is wrapped in try/catch because sandboxed iframes reject it.
  `baseUrl()` falls back to the hardcoded Pages URL off http(s) so Copy link does not emit a
  `file://` URL during local preview.

## Conventions

- Match the surrounding style: `var`, no arrow functions, no template literals, string
  concatenation for HTML. Consistency beats modernity in a file this size.
- Run any user-supplied or data-derived string through `esc()` before putting it in innerHTML.
- Palette comes from the `:root` CSS variables. Do not introduce new hex values ad hoc; the
  chart and legend colors are already centralized in `CHAN_COLOR`, `PROD_COLOR`, and `C`.
- Comments in the file are sparse and explain *why* (data provenance, fragility). Keep that
  density — do not narrate the obvious.
