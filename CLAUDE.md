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

**Patching this file with a substitution script? Do not let the script's own
variables leak into the emitted JavaScript.** Twice now a Python local has
ended up written out as `+ CHEER +` or `+ A +`, which is a valid identifier
reference, so `node --check` passes and the page dies on load with the wheel
blank. Grep the diff for bare capitalised identifiers before trusting it.

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

Whoever escapes gets one of four reactions - relieved, gloat, smug, cheer -
dealt from a shuffled deck, so with four or fewer survivors no two react the
same way. Drawing independently gave three the same pose about half the time.

Every landing ends one of exactly three ways, with equal odds: push right,
push left, or cave. There is deliberately no probability gate on top and no
no-repeat rule - both existed once and made the behaviour impossible to reason
about or to see. Resist adding a layer here.

## Adding a character

Measure, don't eyeball. For each existing character the head aspect, lens box
(against pupil distance), hairline and beard line were read off a gridded
photo, and colours sampled from masked regions and then checked by pasting the
swatch next to the face. Literal pixel values usually come out muddy — photos
are lit, cartoons want albedo.

Speech bubbles size themselves to their line. The box was a fixed 34 units
wide and longer lines ran under the border; `fitBubble()` measures the text as
rendered and rebuilds the box around it, so any wording added to `SAY` works
without hand-tuning. `?inspect` renders every line and audits that each one
fits.

## Testing notes

- A background tab throttles `setTimeout` to ~1/sec and stops rAF, so live
  timing measurements taken from a hidden tab are worthless. Reason from the
  constants instead, or bring the tab to the front.
- Do not stub `Math.random()` by index. The call order shifts whenever a state
  gains a speech bubble or `settle()` needs another shuffled deck of
  reactions, and it has silently invalidated tests twice. Wrap it and log the
  calls first to learn the current order, or force an outcome some other way.
