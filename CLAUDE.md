# Working on this repo

A spinning name wheel whose characters react to being chosen. One
self-contained file, `wheel.html` — no build step, no dependencies, no network
calls, no framework. Keep it that way; the whole point is that it opens by
double-clicking it and survives being emailed to someone.

## Verify with `?inspect`, not with a spin

Open `wheel.html?inspect`. It renders every state on every character, frozen,
**at the size they actually render on the wheel** (~90px tall), plus
frame-by-frame strips for the animated states.

Use it before claiming any visual change works. Almost every art bug in this
file was invisible at 300px and obvious at true size:

- "worried" brows drawn inner-end-down, so everyone scowled through the whole
  spin
- panic eyes as white spheres with pinprick pupils, which read as possessed
- an angry pose with no left arm at all
- a notch in the curly hair that read as a widow's peak
- a kick whose silhouette barely changed, so nobody understood it

Judging poses at large size alone will mislead you. Check both.

## Three seams — keep them

1. **`characters` is the only source of truth.** `{name, state, hue,
   headImage, look}`. Rendering reads from it; nothing else holds state.
2. **`setState(index, state)` is the only place appearance changes.** It has
   already drifted (it also picks a speech-bubble line and computes the
   bubble's counter-rotation). Don't let it grow further.
3. **Poses are SVG groups tagged `data-show="stateA stateB"`.** A new state is
   new groups plus one CSS block, with no JS branching. This has held through
   nine states, including swapping an entire limb rig. Per-character
   appearance is data in `SEED_LOOKS`, resolved through the `HAIR`,
   `HAIR_BACK`, `BEARD`, `GLASSES` and `SAY` tables. Adding a character should
   only add table entries.

## Three traps that have already bitten

**A CSS animation silently overwrites an element's SVG `transform` attribute.**
Hit three times: the sweat drop rendered at the body origin, the head shape
snapped round mid-animation, the speech bubble text came out tilted. The fix is
always the same — one group carries the placement, a child carries the
animation. `?inspect` audits for this and turns red; there is a deliberate test
for the audit (plant a `transform` attribute on `.breathe`, confirm it flags,
revert).

**`CHAR_MARKUP` is built before the lookup tables exist.** It is a module-level
string, so anything it references must go through a `%TOKEN%` replaced inside
`charMarkup()`. Writing `+ FOO +` directly yields a silent `undefined` that a
syntax check passes clean. This caused an empty shirt-stripe clip and
invisible speech bubbles.

**Never duplicate a CSS duration in JS.** `syncTimings()` reads them off the
stylesheet at start-up; only keyframe *percentages* belong in JS. A hand-copied
duration once truncated the cave animation to 56%, cutting it off mid-lunge.

## Animation has to survive a screen share

This gets shared as a Chrome tab on calls, which caps at ~30fps and adapts
lower. Keep looping animations at or below ~5Hz. Two were above it and aliased
into noise when captured; the fix was lower frequency with larger amplitude,
which reads *better* at small size, not worse.

When a gesture needs to be legible at 90px, change the whole silhouette. A
swinging limb is not enough — the kick was replaced by a jump because a leg is
small, low, and half-hidden by the torso.

Two poses that lead to different outcomes must look different *from the start*.
Caving used to replay the full shove lunge before slumping, so it looked like a
push that failed to move the wheel. It now has its own wind-up that cocks the
arms backward and never completes.

Caving is currently switched off (`ALLOW_CAVE = false`), so a tantrum always
moves the wheel. The `bristle` and `accept` poses stay in the rig and in
`?inspect`; flip the flag to bring the outcome back.

## Adding a character

Measure, don't eyeball. For each existing character the head aspect, lens box
(against pupil distance), hairline and beard line were read off a gridded
photo, and colours sampled from masked regions and then checked by pasting the
swatch next to the face. Literal pixel values usually come out muddy — photos
are lit, cartoons want albedo.

## Testing notes

- A background tab throttles `setTimeout` to ~1/sec and stops rAF, so live
  timing measurements taken from a hidden tab are worthless. Reason from the
  constants instead, or bring the tab to the front.
- `setState` calls `Math.random()` for bubble lines. If you stub randomness to
  force a branch, that call is in the sequence and will shift your indices.
  The order for one spin is: duration, turns, angle, winner's bubble,
  `shouldShove`, outcome pick, then a bubble per subsequent state change.
