# Architecture

## Overview

This is a single-file static site: `index.html` contains all HTML, CSS,
and JS with no build step, no bundler, and no other source files.
`nunta-calculator-personalizat.html` is a byte-identical twin (served for
a differently-branded URL) — **there is no sync mechanism**. Any edit to
one must be manually copied to the other, or it will silently drift out
of date. This duplication is a known rough edge, not an intentional
abstraction.

## File map

Line numbers below are accurate as of this writing but will drift as the
file changes. Don't trust them blindly — grep for the banner comments
(`/* ===== ... ===== */` and `/* ---------- ... ---------- */`) to
relocate a section after edits.

| Section | Anchor (approx. line) |
|---|---|
| CONFIG | `index.html:741` |
| Guest model (`buildGuestModel`) | `index.html:811` |
| Event profile multipliers (`getEventMultipliers`) | `index.html:828` |
| Selection reader (`readSelection`) | `index.html:839` |
| Calculation engine (`computeResults`) | `index.html:855` |
| State + DOM wiring init | `index.html:929` |
| Stepper navigation (`initStepper`, `goToStep`) | `index.html:1066` |
| Formatting helpers (`fmtLiters`, `fmtMl`, `fmtMoney`) | `index.html:1122` |
| Render results (`renderResults`) | `index.html:1134` |
| Pricing refresh (`refreshPricingDisplay`) | `index.html:1257` |
| Summary text (`buildSummaryText`) | `index.html:1296` |
| Main recalculation (`recalculateAll`) | `index.html:1319` |
| Copy / Print / Reset actions (`initActions`) | `index.html:1332` |

## CONFIG (`index.html:741`)

Single source of truth for every tunable number. Nothing else in the
file should hardcode a bottle size, consumption baseline, or multiplier.

- **`BOTTLE_SIZES`** — ml per bottle, keyed by drink key. Used for both
  math and the label shown to the user.
- **`CATEGORY_META`** — `{ label, group }` per drink key. Iteration
  order of `Object.keys(CONFIG.CATEGORY_META)` drives category order in
  a few places (see Gotchas below), so key order matters, not just
  content.
- **`GROUP_ORDER`**, **`WINE_KEYS`**, **`SPUMANT_KEYS`**, **`SPIRIT_KEYS`**
  — group the individual drink keys for the split logic in
  `computeResults`.
- **`CONSUMPTION`** — baseline liters per relevant person for a
  *standard* event (5–7h, non-summer, "Normal" profile):
  - `apaPlata`, `apaMinerala` — per **all guests** (adults + teens + children)
  - `suc` — per **soft consumer** (teens + children + non-drinking adults)
  - `wineTotal`, `spumantTotal`, `spiritsTotal` — per **drinking adult**,
    then split across the selected types within that group

The non-obvious part is *which multiplier tables apply to which
categories* — easy to get wrong when adding a new category:

| Category group | PROFILE | DURATION | SEASON | RESERVE |
|---|---|---|---|---|
| Water (`apaPlata`, `apaMinerala`) | — | — | ✓ | ✓ |
| Soft drinks (`suc`) | — | ✓ | ✓ | ✓ |
| Wine / Spumant / Spirits | ✓ | ✓ | — | ✓ |

- `PROFILE_MULTIPLIERS` and `DURATION_MULTIPLIERS` apply only to alcohol.
- `DURATION_MULTIPLIERS` additionally applies to `suc` (soft drinks) —
  event length affects both drinking pace and general thirst.
- `SEASON_MULTIPLIER` applies only to water and `suc` — deliberately
  excluded from alcohol.
- `RESERVE_OPTIONS` applies uniformly to every category, once, at the
  very end of the pipeline.

## Calculation pipeline (`computeResults`, `index.html:855`)

`computeResults(guestModel, eventProfile, selection, reservePct)` is a
**pure function** — no DOM access, no price logic, no side effects.
Preserve this constraint in future changes; DOM reading and pricing
belong in the caller (`recalculateAll`) and renderer respectively.

Order of operations:

1. Look up `profileMult`, `durationMult`, `seasonMult`, `reserveMult`
   from `CONFIG` via the event profile and reserve percentage.
2. Compute non-alcoholic liters directly per category (water, soft
   drinks) using the relevant guest sub-population and applicable
   multipliers (see table above).
3. Compute a **group total** in liters for wine, spumant, and spirits
   (`drinkingAdults × baseline × profileMult × durationMult`).
4. Split each group's total across the types the user actually
   selected within that group:
   - Wine and spumant split **evenly** across selected types.
   - Spirits split by the user's slider **weights**; if all selected
     spirits have weight 0, fall back to an equal split so a selected
     type never silently gets 0 liters.
5. Apply `reserveMult` uniformly to every category's liters, once, at
   the end.
6. Convert liters to bottle count via ceiling division
   (`Math.ceil(litersWithReserve * 1000 / bottleMl)`); an unselected or
   zero-liter category produces `bottleCount: 0`, never a fabricated
   minimum.

Output is one row per **active** category:
`{ key, label, group, bottleMl, liters, litersWithReserve, bottleCount }`.

## State model (`index.html:929`)

```js
const state = {
  currentStep: 1,      // which wizard step is visible
  overrides: {},        // key -> manually set bottle count (overrides computed value)
  prices: {},            // key -> manual price per bottle, in MDL
};
```

`state.lastRows` (set in `recalculateAll`) caches the most recent
`computeResults` output so that `refreshPricingDisplay` can recompute
totals without re-running the calculation.

Pricing is currently a single binary mode, not a multi-mode system:
`readPricingMode()` (`index.html:1118`) reads a radio group and returns
either `'manual'` or `'none'`. `'manual'` means the user is expected to
type per-bottle prices into `state.prices`; there's no per-category
default price list.

## Render/refresh contract

This is the thing most likely to break if someone "simplifies" the
input handlers. There are two update paths with different costs, and
picking the wrong one for a given trigger causes visible bugs (lost
focus, dropped keystrokes):

- **`renderResults(rows)`** (`index.html:1134`) — full rebuild of the
  results table DOM. Call this after `computeResults` produces new
  rows (i.e. from `recalculateAll`, when guest counts, event profile,
  reserve, or drink selection change).
- **`refreshPricingDisplay()`** (`index.html:1257`) — targeted patch
  that only updates per-row total text and the grand total, touching no
  input's own DOM node. Call this after in-place edits to quantity
  override or price inputs.

Why the split exists (see inline comments at `index.html:1208-1234`):
qty-input fires `change` on blur, and price-input fires `input` on
every keystroke. If either handler triggered a full `renderResults()`
rebuild, it would destroy and recreate the input nodes mid-interaction
— stealing focus on blur-into-adjacent-cell, or silently dropping
characters mid-type. So both handlers write into `state.overrides` /
`state.prices` and then call only `refreshPricingDisplay()`.

## Copy / Print / Reset actions (`initActions`, `index.html:1332`)

- **`btnCopy`** — builds a plain-text summary of the visible rows,
  including per-row price/total and a grand total when pricing mode is
  `'manual'`. Copies via `navigator.clipboard.writeText`, falling back
  to `fallbackCopy` (`index.html:1382`, a hidden-textarea +
  `document.execCommand('copy')` shim) for browsers without Clipboard
  API support.
- **`btnPrint`** — just calls `window.print()`; print layout is handled
  entirely by the `/* ---------- Print ---------- */` CSS block
  (`index.html:491`).
- **`btnResetOverrides`** — clears `state.overrides` back to `{}` (does
  not touch `state.prices`).

## Stepper / UI wiring

DOM event wiring is kept separate from calculation logic. The entry
points — `initStepper`, `initCounters`, `initChipVisuals`,
`initDrinkingPct`, `buildSpiritSliders`, `initGlobalRecalcListeners` —
are all invoked from the init block at the bottom of the file. If
you're looking for where a UI control gets its event listener attached
(as opposed to how a value is calculated), start there rather than in
`computeResults` or `renderResults`.

## Known gotchas / invariants

- **Two-file duplication, no sync step.** `index.html` and
  `nunta-calculator-personalizat.html` must be kept identical by hand.
  Any change to one needs to be manually mirrored to the other.
- **Render vs. refresh split is load-bearing.** Full rebuilds
  (`renderResults`) must only follow a real recalculation; in-place
  edits (qty/price inputs) must use `refreshPricingDisplay` only, or
  focus/typing breaks (see above).
- **`computeResults` must stay pure.** No DOM reads, no price
  references — it takes plain data in and returns plain rows out.
  Keeping this pure is what makes it possible to reason about the
  calculation independent of the UI.
- **`CATEGORY_META` key order matters.** `Object.keys(CONFIG.CATEGORY_META)`
  drives category iteration order in `computeResults`'s row-building
  step and elsewhere — adding a new category means placing its key in
  the position where it should visually/logically appear, not just
  appending it.
