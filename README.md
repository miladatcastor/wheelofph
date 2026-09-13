# Wheel of PH

A spinning name wheel where the characters react to being chosen. Single
self-contained HTML file — no build, no dependencies, no network. Open
`wheel.html` in a browser.

## Using it

- **Spin** — the hub in the middle of the wheel.
- **Click a character** — make them the one who jumps to start the next spin,
  or remove them from the wheel.
- **Names** — the pill in the top corner opens a drawer to edit the list, give
  anyone a photo head, or turn the "sore losers" gag off.

The opening arrangement is shuffled on every load.

## What happens on a spin

The character under the pointer crouches and body-slams the wheel; it takes off
on the landing frame. While it spins, fear travels round the ring — the slice
under the pointer panics, its neighbours go nervous, everyone else idles. On
stop the winner is doomed and the rest are relieved.

About a third of the time the winner then squares up and either shoves the
wheel a notch left or right (landing a neighbour in it) or thinks better of it
and takes the hit. Never twice in a row.

## `?inspect`

Open `wheel.html?inspect` for the contact sheet: every state on every
character, frozen, at the size they actually render on the wheel, plus
frame-by-frame strips for the animated ones.

Use it. Nearly every art bug in this file was invisible at 300px and obvious
at true size — scowling "worried" brows, bug eyes, a missing arm, a notch in
the hair. It also runs the transform audit (below) and turns red on a hit.

## How it is put together

Three seams, all of which have held up through nine states and five
characters:

1. **`characters`** is the only source of truth — `{name, state, hue,
   headImage, look}`. Rendering reads from it; nothing else holds state.
2. **`setState(index, state)`** is the only place a character's appearance
   changes.
3. **Poses are SVG groups tagged `data-show="stateA stateB"`.** A new state is
   new groups plus one CSS block — no JS branching. Appearance (hair, beard,
   glasses, head shape, stripes...) is per-character data in `SEED_LOOKS`,
   resolved through the `HAIR` / `HAIR_BACK` / `BEARD` / `GLASSES` / `SAY`
   tables.

## Three traps, learned the hard way

- **A CSS animation silently overwrites an element's SVG `transform`
  attribute.** This bit three times (sweat drop rendered at the body origin,
  head shape snapping round, tilted bubble text). Anything animated gets a
  wrapper group so placement and animation never share an element.
  `?inspect` audits for it.
- **`CHAR_MARKUP` is built before the lookup tables exist.** Anything it
  references must go through a `%TOKEN%` replaced in `charMarkup()`. Writing
  `+ FOO +` directly yields a silent `undefined` that passes a syntax check.
- **Never duplicate a CSS duration in JS.** `syncTimings()` reads them off the
  stylesheet at start-up; only keyframe *percentages* live in the JS.

## Accessibility

Respects `prefers-reduced-motion` (idle loops off, the stomp and shove skip to
their outcome), keyboard focus is visible on every control, the result is an
`aria-live` region, and colours are custom properties with a dark-scheme
override.
