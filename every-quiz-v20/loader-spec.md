# Loader — timing spec

Source: `renderLoad()` in app.js, styles under `.load` / `.lphase` / `.lfoot` in styles.css.
All times are ms from the moment the loader is triggered (t=0), i.e. right
after the last quiz answer is tapped.

## Overview

1. Overlay washes in over the last question (brown, `--quizbg`).
2. Picture + "Going over your responses" + 4-line checklist fade/slide up as one group.
3. The 4 checklist lines appear one at a time, each animating in then ticking "done".
4. At 4.0s a "Continue to my result" button fades in at the bottom. Nothing
   moves on by itself (a testing aid); tapping it makes the group lift away and
   the overlay crossfade brown → green.
5. Overlay clears to reveal the results page underneath, which was built silently behind it.

## Timeline

| t (ms) | Event | Duration / easing |
|---|---|---|
| 0 | `.load` overlay gets `.in` (rAF) — opacity 0→1 | 0.7s ease |
| 0 | Background/text colour transition armed | 0.9s ease (only fires later, at green swap) |
| 320 | `.lphase` (pic + title + checklist wrapper) gets `.in` — opacity 0→1, translateY 10px→0 | opacity .55s / transform .6s, ease |
| 750 | Results page (`renderResult()`) is built off-screen, behind the opaque overlay | instant (not visible yet) |
| 620 | Line 0 "Assessing your goal" gets `.in` (fade up) | .55s/.6s ease |
| 1360 | Line 0's icon gets `.done` (spinner → ✓) | .35s spring |
| 1400 | Line 1 "Reading how your week runs" gets `.in` | .55s/.6s ease |
| 2140 | Line 1's icon gets `.done` | .35s spring |
| 2180 | Line 2 "Weighing up your challenges" gets `.in` | .55s/.6s ease |
| 2920 | Line 2's icon gets `.done` | .35s spring |
| 2960 | Line 3 "Choosing your plan" gets `.in` | .55s/.6s ease |
| 3700 | Line 3's icon gets `.done` | .35s spring |
| **4000** | **`LOAD_MS` — `#lgo` "Continue to my result" loses `.hid`** (fades up in the footer spot). The loader then waits. | .3s ease |
| **T (tap)** | **handover fires**: `.lphase` gets `.out` (fades down, opacity 1→0, translateY 0→-10px); button hides; overlay gets `.green` (bg/text crossfade to `--reveal`/`--ink`) | lphase.out: .45s/.45s ease · colour: .9s ease |
| T+600 | Overlay gets `.out` — opacity 1→0, revealing results underneath | 0.7s ease |
| T+900 | Results hero children start revealing (staggered) | see below |
| T+1500 | `.load` node removed from DOM | — |

## Per-line cadence

Each of the 4 checklist lines follows the same local pattern once it starts:
- **arrive**: fades up over 0.6s (opacity 0.55s, transform 0.6s)
- **+740ms after arriving**: spinner fades out / checkmark pops in over 0.35s (spring)

Line starts are spaced roughly every **780ms** (620, 1400, 2180, 2960), so with
the 740ms arrive→tick offset each line's checkmark lands ~40ms before the next
line starts — a near-continuous chain, tuned to fit 4 lines inside the 4000ms
`LOAD_MS` window with the last tick landing just inside it (3700ms, 300ms of
headroom before handover).

## Handover / results reveal detail

- `finish()` runs when the button is tapped, t=T (same moment as `handover()`;
  the times below were t=4000-based before the button, and are now relative to T):
  - `.load` gets `.green` → background/text crossfade to results colours (0.9s)
  - `phone` gets `.bundle` class (swaps chrome mode)
  - +600ms (T+600): `.load` gets `.out` → overlay opacity 1→0 (0.7s)
  - +1500ms (T+1500): `.load` element is removed from the DOM
  - +900ms (T+900): results hero elements matching `REVEAL_SEL` start revealing,
    each staggered **110ms** apart (`i*110`), each using the `.rv.in` reveal
    animation (`rvIn`, 0.85s, `--soft` easing, fade + translateY 12px→0)

The 900ms offset is deliberate: the overlay fade is 700ms, so starting the
hero reveal at 900ms means it begins ~200ms after the overlay is fully
transparent — early enough to feel continuous with the clear, but never
starting while still hidden behind the overlay (which would read as a jump
cut when it became visible).

## Key constants (app.js)

```js
const LOAD_MS = 4000;               // total time the "working" state is shown
const STARTS  = [620,1400,2180,2960]; // line-in start times
// each line's tick = its start + 740ms
```

## Notes for changing timing

- If `LOAD_LINES` grows/shrinks, `STARTS` must be re-spaced manually (not
  computed) — keep ~780ms between starts and leave ~300ms headroom before
  `LOAD_MS` for the last tick.
- Reduced-motion: only the heart's idle float (`heartFloat`, 1.6s loop) is
  disabled under `prefers-reduced-motion`; the sequencing above still runs at
  full speed regardless — no reduced-motion path shortens or skips it.
