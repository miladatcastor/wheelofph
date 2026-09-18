# Wheel of PH

### ▶ Play it: **https://miladatcastor.github.io/wheelofph/**

A spinning name wheel where the characters react to being chosen. Single
self-contained HTML file — no build, no dependencies, no network. Open the
link above, or `index.html` in a browser.

## Using it

- **Spin** — the hub in the middle of the wheel.
- **Click a character** — make them the one who jumps to start the next spin,
  or drop them from this session. A reload brings everyone back.
The names are the five in `DEFAULT_NAMES`. There is no settings drawer, no
name editor, no photo upload and no switches — the page is the wheel and
nothing else. Changing who is on it means editing that array and the matching
`SEED_LOOKS` entry.

The opening arrangement is shuffled on every load.

## What happens on a spin

Before anyone presses anything, the characters close in toward the hub, join
hands and the whole ring turns slowly. Pressing spin releases them.

The character under the pointer crouches and body-slams the wheel; it takes off
on the landing frame. While it spins, fear travels round the ring — the slice
under the pointer panics, its neighbours go nervous, everyone else idles.

On stop the winner is doomed, and **every** landing then ends one of exactly
three ways: the winner shoves the wheel a notch right, shoves it a notch left
(landing a neighbour in it), or squares up and thinks better of it. Taking the
hit is much the likeliest ending. Whoever shoves it away points at the person they just landed it on.
Everyone who escapes gets one of four reactions — relieved, gloating, smug or
cheering — dealt so that no two of them react the same way.

There is a fourth ending too, as likely as a shove either way — not a chance
layered on top of the others. That makes it **60% take the hit**, and 13.3%
each for a shove right, a shove left, and Derk. Derk walks
in, towers over the ring, plants himself and jabs a finger at somebody picked
at random, and the wheel goes wherever he says regardless of where it actually
landed. Everyone is in that draw, including whoever the wheel just landed on —
so now and then he draws them straight back, gives the wheel a full ceremonial
lap and changes nothing. Then he puts his hands on his hips, says something a
CEO says, and strides off the side of the screen.

## `?design`

Open `index.html?design` to dial in a new character: every colour, hair,
beard and glasses option live, the head sliders with an area lock, the figure
drawn at true wheel size and large at once, a drop zone for their photo to
compare against, and the `SEED_LOOKS` entry written out ready to paste.

## `?inspect`

Open `index.html?inspect` for the contact sheet: every state on every
character, frozen, at the size they actually render on the wheel, plus
frame-by-frame strips for the animated ones.

Use it. Nearly every art bug in this file was invisible at 300px and obvious
at true size — scowling "worried" brows, bug eyes, a missing arm, a notch in
the hair. It also runs three audits and turns red on a hit: that every
`@keyframes` is defined, used and actually moves something; the transform trap
(below); and that every speech line fits its bubble.

## How it is put together

Three seams, all of which have held up through sixteen states and five
characters:

1. **`characters`** is the only source of truth — `{name, state, hue,
   headImage, look}`. Rendering reads from it; nothing else holds state. The
   CEO is an overlay and deliberately stays out of it.
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
