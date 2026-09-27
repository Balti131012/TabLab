# AGENTS.md — Fret Sheet (Tab Writer)

## What this is
A single-file HTML app for writing guitar tabs by tapping a fretboard-style
keypad (no typing required). Styled after vintage songbook covers, with a
Songsterr/Guitar-Tabs-X-inspired input keyboard. No build step, no
dependencies to install — everything lives in one `.html` file.

- **File:** `fret-sheet.html`
- **Runtime:** plain HTML/CSS/vanilla JS (ES5-ish, no framework, no bundler)
- **Fonts:** Abril Fatface (headings) + JetBrains Mono (tab grid), loaded from
  Google Fonts
- **Persistence:** `localStorage`, per-viewer only (see Storage below)
- **Deployment:** published as a Claude Artifact; can also be opened directly
  as a static HTML file

## Data model
```
state = {
  title: string,
  tuning: "e,B,G,D,A,E",   // comma-separated string labels, high to low
  bpm: number,
  measures: [ measure, ... ]
}

measure = {
  sig: { num: 4, den: 4 },   // time signature, per bar
  heads: [ head, ... ]       // ordered list of beats, spans the whole bar
}

head = {
  duration: number,          // in 16th-note units (w=16, h=8, q=4, e=2, s=1)
  grid: ["-","-","-","-","-","-"]   // one symbol per string, high to low
}
```

**Invariant:** for every measure, `sum(head.duration for head in heads) ===
sigCapacity(measure.sig)` where `sigCapacity = num * (16 / den)`. This must
never be violated — it's what keeps every bar the length its time signature
says it is. All mutation goes through `resizeHead()` (grow/shrink a beat,
absorbing or releasing time from later beats) or `applySig()` (change a bar's
time signature, trimming or padding heads to fit). Don't mutate
`heads[i].duration` directly anywhere else.

## Key functions (in the inline `<script>`)
- `blankMeasure(sig)` — new empty bar in a given time signature
- `applySig(measure, sig)` — reflow an existing bar to a new time signature
- `resizeHead(mIdx, hIdx, newUnits)` — change one beat's length, keeping the
  bar's total constant
- `renderMeasures()` — full re-render of the tab staff from `state`; cell
  widths are set inline (`style.width`) proportional to `head.duration`
- `updateExport()` — builds the plain-text tab in the `<pre>` at the bottom
- `pressNumber / pressTechnique / pressRest / pressBack / pressDuration` —
  keypad handlers, all operate on `selected = {m, h, s}`
- `advance()` — auto-advance to the next beat after entering a fret (creates
  a new bar at the end if needed, inheriting the previous bar's time sig)

## Conventions
- Selection is always `{m: measureIndex, h: headIndex, s: stringIndex}`.
  Duration changes only need `m` and `h`; fret/technique entry needs `s` too.
- Technique tokens are just appended strings on a cell's value (e.g. `"3h"`,
  `"5PM"`). There's no separate technique layer — keep this simple unless a
  real reason to change it comes up.
- String order is always high to low: `[e, B, G, D, A, E]` for standard
  tuning, matching how tab is conventionally written top-to-bottom.
- Plain-text export pads each beat's symbol with trailing dashes to the
  widest value needed at that beat (across strings), not to a fixed width —
  this is what keeps the six string lines vertically aligned.

## Storage / versioning
State is saved under a single `localStorage` key (currently
`fret-sheet-v4`). **Bump this key any time you change the shape of `state`,
`measure`, or `head`** — there is no migration logic, so a stale key would
otherwise load mismatched data and throw. When you do bump it, tell the user
their saved tab will reset, and say why.

Storage is per-viewer and local to the browser (per the artifact platform's
rules) — there is no shared/multi-user state and no server. Don't add
`window.storage` or any network calls; this page is meant to work standalone.

## Style / design constraints
- Palette and type are intentional (vintage songbook: brass/wood/parchment,
  Abril Fatface display type) — keep new UI consistent with the existing
  tokens in `:root` rather than introducing new colors ad hoc.
- Mobile-first: the person primarily uses this from a phone. Keep tap targets
  ≥ ~30px, avoid hover-only affordances, and test that the keypad doesn't
  require horizontal scrolling to reach.
- No external JS libraries. If a feature seems to need one, prefer a small
  hand-rolled implementation to keep this a single dependency-free file.

## When adding a feature
1. Decide if it changes the data model. If yes, bump `STORE_KEY` and update
   `blankMeasure`/`load()` accordingly.
2. Keep the bar-length invariant intact — route any duration change through
   `resizeHead`/`applySig`, never by hand.
3. Re-render via the existing `renderMeasures()` / `updateExport()` calls
   after every state mutation; there's no reactive framework doing this for
   you.
4. After editing, republish the same artifact URL rather than creating a new
   one, so the person keeps one continuous link.
