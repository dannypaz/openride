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
different labels:

| Fit value | Zwift letter | Zwift meaning |
|---|---|---|
| SX | **B** | Saddle X |
| SY | *(cross-check only, see below)* | Saddle Y |
| HX | **C** | Handlebar Reach |
| HY | **D** | Handlebar Stack |

Because both systems use the same reference point and axes, **the numbers
carry over with no conversion** — SX is your target for B, HX is your target
for C, and so on. That equivalence is the whole trick; everything else in
the calculator is just range-checking and unit bookkeeping around it.

## The formulas

Given inputs `SX, SY, HX, HY` (mm, from your fit) and `saddleHeight` (mm,
your fit's own BB&rarr;saddle-top measurement, if it reports one):

```
E (Saddle Height)     = saddleHeight, or √(SX² + SY²) if saddleHeight is blank
B (Saddle Fore/Aft)   = SX
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

Each target is clamped to the Ride's range and compared against your raw
(unclamped) input to produce a status:

```
status(value, min, max):
  good  — min <= value <= max
  warn  — outside the range, but by <= 15mm
  bad   — outside the range by more than 15mm
```

### The Ride's ranges (from Zwift's Smart Frame Geometry chart)

| Letter | Meaning | Min | Max |
|---|---|---|---|
| E | Saddle Height (BB &rarr; saddle top) | 599mm | 865mm |
| B | Saddle X (setback) | 147mm | 251mm |
| C | Handlebar Reach | 390mm | 510mm |
| D | Handlebar Stack | 600mm | 761mm |
| F | Saddle&rarr;Bar Drop (derived) | 21mm | 69mm |

Everything else on the chart (chainstay, BB drop, seat/head angle, etc.) is
fixed frame geometry, not adjustable — shown in the "Fixed Frame Reference"
table for context only.

### Saddle tilt

Zwift doesn't publish a tilt range, so tilt is a straight passthrough of
your fit's own saddle angle (degrees, nose-up positive) with no in/out-of-range
check — the UI shows it as "Preference" rather than a pass/fail.

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
const CRANK_OPTIONS = [160, 165, 170, 172.5, 175];
```

If Zwift revises the Ride's geometry, or you're adapting this for a
different smart frame with its own published BB-relative range chart, only
these constants (plus the `CONSTANTS` reference table just below them) need
to change — the rest of the logic is frame-agnostic.

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

- [Adjusting Your Zwift Ride](https://support.zwift.com/en_us/adjusting-your-zwift-ride-SyUBRM8A)
- [Zwift Ride Adjustable Crank Arms](https://us.zwift.com/products/zwift-ride-adjustable-crank-arms)
- Zwift Ride Smart Frame Geometry chart (rider-supplied)
