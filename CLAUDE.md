# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A practice drill for jazz piano 2-5-1 voicings. It shows a random key, chord symbols, and
**degree formulas** (e.g. `B♭7 — 7-9-3-6`). The user works out the actual notes at the
keyboard. Deriving them is the practice, so the app must never display note names, a keyboard
diagram, or notation.

## Running it

Open `index.html` in a browser. There is no build, no dependencies, no server, and no tests.

To exercise the internals from a browser console, the top-level bindings `state`, `generate()`,
`playable()`, `blockers()`, `fixedCombos()`, `defaultState()`, `deck`, `renderKeys()` and
`renderDrill()` are all reachable —
that's how this was verified. `localStorage` needs a real origin, so persistence must be tested
over `http://` (`python3 -m http.server`), not from a `file://` snapshot.

## Single-file constraint

Everything lives in `index.html` — markup, `<style>`, and one classic `<script>`. **Do not split
it into modules or external files.** `<script type="module">` is blocked by CORS on `file://`,
which broke the app's whole reason for existing: opening it by double-clicking, with no server.
A classic external `<script src>` would survive that, but a self-contained file makes the
failure mode impossible and keeps it to one thing to AirDrop to a phone or drop on Pages.

## Data model

The `DATA` section at the top of the script is the source of truth. **The voicing set is final**
— these are all the voicings the user will ever use, so don't add extensibility (an add-voicing
form, migrations for new cells). Edit the data only to fix a transcription error. `FORMULAS` is `mode → quality → position → variant → formula`:

- **Formulas are opaque display strings.** Nothing parses them or converts them to notes. This
  is deliberate: the user's notation is internally inconsistent (`2` vs `9`, varying `♭`
  placement) and passing it through untouched means it can't produce bugs. Don't add theory code.
- **Variant keys.** `"_"` means the position has a single form and ignores the Extended/Grounded
  filter — minor 1st and 3rd have no Grounded variant, and they stay in the pool rather than
  vanishing when only Grounded is checked. `extended`/`grounded` apply only to minor 2nd and 4th.
- **`null` instead of a position object** means the cell doesn't exist and renders as a greyed-out,
  unselectable checkbox. Currently only mMaj7 outside 1st position.
- **Keys are stored by pitch class (0–11), never by name.** Names change with the spelling toggle,
  so storing `"A♭"` would silently drop a selected key when switching to conventional spelling.

## Invariants worth preserving

- **`formulasFor` dedupes by formula string.** Minor 4th position's ii and V are identical across
  both variants; without the dedupe that position would be drawn twice as often as the others.
- **Variant labels have two forms.** `VARIANT_LABELS` ("Extended (Ex)") is for the grid checkboxes;
  `VARIANT_SHORT` ("Ex") is for the drill card. A `"_"` position has no short form and shows no tag.
- **`poolForRole` flattens across every quality in the role.** The i chord draws from the maj7 and
  6 rows combined, so four maj7 cells and one 6 cell gives 80/20. Picking a quality first and then
  a cell within it would flatten that to 50/50.
- **Three drill shapes**, via `state.layout` and `state.posMode`:
  - *single* — one chord, drawn uniformly from every enabled cell in the mode.
  - *progression + shuffled* — each chord draws its own position. This deliberately breaks the
    voice-leading a position encodes, so each cell must be known in isolation. A choice, not an oversight.
  - *progression + fixed* — one position for all three, preserving that voice-leading. Built from
    `fixedCombos()`, which returns only the (position, variant) pairs enabled for **every** role at
    once. Note this creates a dead end the other shapes don't have: each role can have voicings while
    no single position serves all three, so `playable()` and `blockers()` both branch on the mode.
- **Keys come from a shuffled deck**, reset to `[]` whenever the key selection changes, so every
  enabled key is drilled once before any repeats.
- **Storage access is wrapped in try/catch** and the app must render correctly when it throws.

## Verification

Changing `FORMULAS` or the key tables calls for a transcription diff against the user's class
notes before anything else — a wrong formula teaches something incorrect and no amount of UI
testing catches it.
