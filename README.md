# openride

A single-file calculator (`index.html`) that translates a professional bike
fit into adjustment targets for a [Zwift Ride](https://www.zwift.com/) smart
frame. Open the file in a browser — no build step, no server.

## The core idea

Most professional bike fits (Retul, Purely Custom, and similar systems)
report your position as four numbers, all measured from the bottom bracket
(BB) axle centre:

| | Horizontal (X) | Vertical (Y) |
|---|---|---|
| **Saddle** | SX &mdash; setback | SY &mdash; height component |
| **Handlebar** | HX &mdash; reach | HY &mdash; stack |

Zwift's own "Smart Frame Geometry" chart for the Ride specifies its
adjustable range using **the same BB-relative coordinate system**, just with
different labels (SX&rarr;B, HX&rarr;C, HY&rarr;D), so those numbers carry
over with no conversion. That equivalence is the starting point; the rest of
the calculator is about how you actually *set* each one on the physical frame.

Three of the Ride's five adjustments turn out to be printed with a letter
scale directly on the frame (confirmed on a real unit): **saddle height**
(on the seatpost) and **handlebar height** and **handlebar reach** (on the
handlebar mast/mount), each spaced **10mm per letter, with A at the shortest
position**. The other two — **saddle fore/aft** and **saddle tilt** — share
one rail clamp with no printed scale at all.

## The formulas

Given inputs `SX, SY, HX, HY` (mm, from your fit) and `saddleHeight` (mm,
your fit's own BB&rarr;saddle-top measurement, if it reports one):

```
E (Saddle Height)     = saddleHeight, or √(SX² + SY²) if saddleHeight is blank
C (Handlebar Reach)   = HX
D (Handlebar Stack)   = HY
F (Saddle→Bar Drop)   = clamp(E) − clamp(D)      // derived, not a control
```

`√(SX² + SY²)` is the straight-line distance from BB to the saddle if you
don't have a separately-reported saddle height. It won't exactly match a
fit sheet's own printed saddle-height number — fit systems and Zwift's
"80mm-wide point" saddle reference sit at slightly different spots on the
saddle, so expect a few mm of difference. Prefer the fit sheet's own number
for `E` when you have it.

### Turning a target into a letter (E, D, C)

These three ride on a physical 10mm-per-letter scale, A = shortest:

```
letterIndex = round((target − rangeMin) / 10)     // 0 = A, 1 = B, 2 = C, ...
markMM      = rangeMin + letterIndex × 10          // the mm the letter actually sits at
```

The app shows both the letter *and* `markMM` next to your raw target, since
snapping to a 10mm step means up to &plusmn;5mm of rounding either way.

### Saddle Fore/Aft (B) — no letter scale

This is the one the letters don't cover, and it's not a simple independent
target either. Zwift's chart lists Saddle X as **147&ndash;251mm**, but that's
the combined envelope across the *entire* saddle-height range, not one
clamp's independent travel. As you raise or lower the saddle (the lettered
height adjustment), the seatpost's fixed 73.5&deg; angle drags a "neutral"
setback along with it:

```
t         = clamp((SY − 579) / (830 − 579), 0, 1)   // 0 at min height, 1 at max
neutralB  = 147 + t × (251 − 147)                    // where the height slide alone puts you
```

On top of that neutral point, the saddle's fore/aft rail clamp gives roughly
**&plusmn;17.5mm** of independent, saddle-rail-dependent fine-tune (confirmed
centred, not one-directional — though the exact total travel varies by
saddle model, nominally ~35mm):

```
achievedB = clamp(SX, neutralB − 17.5, neutralB + 17.5)
```

If your fit's `SX` falls outside that &plusmn;17.5mm window, the closest
you can get is whichever end of the window is nearer — the app reports the
shortfall in mm. There's no scale to read here, so measure it with a plumb
line (see below) rather than a tape pulled directly between the two points.

### Status thresholds

```
status(value, min, max):
  good  — min <= value <= max
  warn  — outside the range, but by <= 15mm
  bad   — outside the range by more than 15mm
```

### The Ride's ranges (from Zwift's Smart Frame Geometry chart, cross-checked
against the product spec page and geometrygeeks.bike)

| Letter | Meaning | Min | Max | Scale |
|---|---|---|---|---|
| E | Saddle Height (BB &rarr; saddle top) | 599mm | 865mm | Lettered, 10mm/letter |
| B | Saddle X (setback) | 147mm | 251mm | No scale — see above |
| C | Handlebar Reach | 390mm | 510mm | Lettered, 10mm/letter |
| D | Handlebar Stack | 600mm | 761mm | Lettered, 10mm/letter |
| F | Saddle&rarr;Bar Drop (derived) | 21mm | 69mm | Derived |

Everything else on the chart (chainstay, BB drop, seat/head angle, etc.) is
fixed frame geometry, not adjustable — shown in the "Fixed Frame Reference"
table for context only.

### Saddle tilt

Zwift doesn't publish a tilt range, so tilt is a straight passthrough of
your fit's own saddle angle (degrees, nose-up positive) with no in/out-of-range
check — the UI shows it as "Preference" rather than a pass/fail. It shares
the same rail clamp as fore/aft and has no scale either; match it with a
smartphone level app.

### Crank length

Matched to the nearest hole on Zwift's aftermarket Adjustable Crank Arms:
`160 / 165 / 170 / 172.5 / 175` mm. The stock crank is a fixed 170mm with no
alternative holes.

## Adjusting the ranges for a different frame or accessory

All of the numbers above live at the top of the `<script>` block in
`index.html`:

```js
const HEIGHT  = { min: 599, max: 865 };
const SETBACK = { min: 147, max: 251 };
const REACH   = { min: 390, max: 510 };
const STACK   = { min: 600, max: 761 };
const DROP    = { min: 21,  max: 69  };
const LETTER_STEP_MM   = 10;    // mm per printed letter
const RAIL_HALF_WINDOW = 17.5;  // half of the fore/aft rail clamp's nominal travel
const CRANK_OPTIONS = [160, 165, 170, 172.5, 175];
```

If Zwift revises the Ride's geometry, or you're adapting this for a
different smart frame with its own printed letter scale and range chart,
only these constants (plus the `CONSTANTS` reference table just below them)
need to change — the rest of the logic is frame-agnostic.

## Measuring your own fit's BB-relative coordinates

If your fit report already lists SX/SY/HX/HY (or "Saddle X/Y" and
"Handlebar X/Y"), use those numbers directly. If it only gives you a
straight-line "saddle height" and "reach," those are usually already the
same X/Y pair under a different name — check with your fitter.

To physically measure setback (`B`) yourself: a tape pulled straight from
the BB to the saddle measures the *diagonal* distance (that's closer to `E`),
not the horizontal-only setback, because the BB and saddle sit at different
heights. Use a plumb line instead — see the "Saddle Fore/Aft" step in the
app for the full plumb-line procedure with a diagram.

## Sources

- [Zwift Ride Smart Frame product page](https://us.zwift.com/products/zwift-ride-smart-frame)
- [Adjusting Your Zwift Ride](https://support.zwift.com/en_us/adjusting-your-zwift-ride-SyUBRM8A)
- [Zwift Ride Adjustable Crank Arms](https://us.zwift.com/products/zwift-ride-adjustable-crank-arms)
- [Geometry Details: Zwift Smart Frame 2025 — geometrygeeks.bike](https://geometrygeeks.bike/bike/zwift-smart-frame-2025/) (independent cross-check of the range chart, plus the "Saddle Fore/Aft rail slide only: 35mm" figure)
- Zwift Ride Smart Frame Geometry chart (rider-supplied)
- Letter-scale spacing and saddle-fore/aft behavior confirmed directly by the rider on their own frame
