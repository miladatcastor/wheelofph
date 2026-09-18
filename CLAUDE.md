# Working on this repo

A spinning name wheel whose characters react to being chosen. One
self-contained file, `index.html` — no build step, no dependencies, no network
calls, no framework. Keep it that way; the whole point is that it opens by
double-clicking it and survives being emailed to someone.

## Verify with `?inspect`, not with a spin

Open `index.html?inspect`. It renders every state on every character, frozen,
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

**CSS never tells you it ignored you, and that is the most expensive class of
bug in this file.** Two silent deaths, both of which have cost an afternoon:
an invalid value inside a keyframe is dropped and the block still parses
(`translateY(2.6)` with no unit - every stop using it vanished, the animation
ran perfectly and moved nothing); and a stray brace makes the parser swallow a
whole `@keyframes`, after which the name still resolves - `getComputedStyle`
happily reports `animationName: diveRight` - but the element sits at the
identity matrix. Neither logs anything, and both look plausible in a
screenshot, which is worse than looking broken.

`animationAudit()` in `?inspect` catches both. It does not read the
declarations to decide whether a block works, because it cannot: a dropped
value leaves a stop that still carries its timing function, so the block looks
fully populated. It **runs** each block on a probe element and samples whether
anything actually changes. Four things about it are load-bearing, all learned
by getting them wrong:

- the probe animation needs **`both`**, or the last sample lands after the
  animation has ended, the property snaps back to its base value, and a
  completely dead block reads as alive;
- the probe needs **`--dir`** set, because some keyframes are written as
  `calc(-6deg * var(--dir))` and an unset custom property makes the whole
  calc invalid - the block then animates nothing *on the probe* and gets
  reported as dead. Anything a keyframe reads has to exist on the probe;
- walking the stylesheet must read each rule **before** recursing. Since CSS
  nesting landed, every `CSSStyleRule` has a `cssRules` list of its own and an
  empty list is truthy, so branching on it first skips every real rule and the
  audit calls the whole file unused;
- a `var()` anywhere in the `animation` **shorthand** makes it a pending
  substitution and every longhand reads back as an empty string, so the rules
  that stagger with `var(--d)` look unused to the CSSOM. There is a text
  fallback for exactly those.

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

**Anything positioned relative to a character must be placed in *screen*
space, not the character's own frame.** Characters are rotated with their
slice, so a speech bubble pinned at a fixed local point swings wherever the
slice happens to point - at about a quarter turn it landed on the face.
`aimBubble()` rotates the desired screen offset back into local space, so the
bubble always sits above and right of the head at any angle.

**Never measure the wheel mid-transition.** Use the `rotation` variable, which
is the target, not `liveAngle()`, which reads the matrix as it currently
stands. This has caused two separate bugs: the shove announcing the wrong
name, and bubbles rotated out by exactly one slice.

**Mind the descendant space in the state CSS.** `.char[data-state="x"]
[data-show~="x"]` needs the space; without it the selector matches a single
element that has both attributes, which nothing does, and that state renders
as a faceless blob with no error anywhere.

**`transitionend` bubbles.** The listener that ends a spin lives on `.wheel`,
so without a `e.target === wheel` check *any* transition on *anything* inside
the wheel ends the spin. Adding a 0.7s transition to the characters was enough:
it landed 40ms before `launch()` and settled the wheel without it ever turning.
Note this cannot be reproduced in a background tab, where transitions never run
at all - dispatch a synthetic bubbling `transitionend` to test it.

**Never duplicate a CSS duration in JS.** `syncTimings()` reads them off the
stylesheet at start-up; only keyframe *percentages* belong in JS. A hand-copied
duration once truncated the cave animation to 56%, cutting it off mid-lunge.

**The page must never be able to raise a scrollbar.** `.wheel` is a square
that is permanently rotating, and the axis-aligned bounding box of a rotating
square swells by 41% at 45deg and shrinks back, so the document's scroll
height pulses every frame. macOS hides this on a laptop screen, where
scrollbars are overlays that float above the layout. Plug in a monitor and you
have usually plugged in a mouse, and macOS switches to classic scrollbars that
take real layout width: the bar appears, steals 15px, the narrower viewport
shrinks `.stage`, the smaller wheel stops overflowing, the bar goes, the wheel
grows back - every frame, which reads as the wheel shaking itself.
`html:has(.app){overflow:hidden}` severs it. The `:has()` is load-bearing:
`?inspect` replaces `.app` outright and that contact sheet does need to
scroll. Nothing is lost by clipping - what spills past the viewport is the
empty corner of the wheel's bounding box.

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

The one who *shoved* it away gets their own pose, `gotcha-left`/`gotcha-right`
- pointing at the slice they just landed it on, other hand on the hip. They
were in the firing line a second ago and got out of it by putting someone else
in, which is not the bystanders' feeling. The mirror pair exists because which
side the victim ends up on depends on which way the wheel was shoved.

Everyone else who escapes gets one of four reactions - relieved, gloat, smug, cheer -
dealt from a shuffled deck, so with four or fewer survivors no two react the
same way. Drawing independently gave three the same pose about half the time.

Every landing ends one of exactly four ways: push right, push left, cave, or
the CEO. Caving is deliberately twice as likely as any single one of the
others; the three that pass it on or bring Derk in are equally likely to each
other.

    accept 2/5 = 40%    right 1/5    left 1/5    ceo 1/5

**The weighting is the shape of the list, nothing else** - `OUTCOMES` holds
`accept` twice, and one flat pick is taken from it. Repeating an entry is not a
layer: it is still one roll you can read in one line, and changing the balance
means changing that line.

What IS a layer is a second roll in front of it, a probability gate, or a
no-repeat rule. All three existed once and made the behaviour impossible to
reason about or to see. If you ever find yourself writing a second
`Math.random()` above the pick, stop and change the shape of the list instead.

**There are no switches, and there is no UI.** The page is the wheel, the hub
and the result line. A settings drawer existed and held a names textarea, a
face/photo uploader and tick boxes for "sore losers" and "the CEO"; all of it
is gone, along with `parseNames`, `renderFaces`, `setHeadImage`,
`squareDataURL`, the whole `headImage` photo-head path and the `MAX` cap that
only ever limited the textarea. Every option was another thing to reason about
and another state to test, and one of them - "sore losers" off - made the
wheel land on a name and then do nothing at all. `shouldShove()` is now only
"are there at least two names".

Names live in `DEFAULT_NAMES`, looks in `SEED_LOOKS`, and that is the only way
to change who is on the wheel. The one surviving control is the per-character
menu (click a slice): make them the jumper, or drop them for this session -
which is recoverable by reloading, now that there is no textarea to restore
them from. Resist adding a toggle back for the same reason you resist adding a
probability gate.

## The CEO

His name is Derk, and it is on a clip-on name tag on his shirt. The tag is
`look.badge`, resolved through `%BADGE%` like every other feature - reaching
for `BADGE` directly from `CHAR_MARKUP` would emit a silent `undefined`,
because that string is built before the constant exists. Nobody else has one,
and giving anyone one is a single line of look data. `fitBadge()` squeezes a
name too long for the tag rather than resizing it, so the badge keeps its shape
whoever wears it; the squeeze is meant to be the exception, so the box is sized
so that ordinary names clear it untouched. It measures as rendered, which means
it only works while he is on screen - a hidden node reports zero width.

He is not on the wheel and is deliberately **not in `characters`** - he is an
overlay (`#ceo`) that sits in the stage, outside `.wheel` so he never rotates
with it. That keeps the first seam intact: `characters` is still the only
source of truth for what is on the wheel.

He reuses the character rig wholesale. `buildCeo()` calls the same
`charMarkup()` everyone else gets, so his poses are ordinary `data-show`
groups (`walk`, `decree`) and his face comes out of the same `HAIR` / `BEARD` /
`GLASSES` tables - `tousled` and `shades` were added for him and are available
to anyone. `lookVars()` is shared with `render()` for the same reason.

Two things about him are worth not relearning:

- **The mirrored lenses are filled, and that is load-bearing.** `%GLASSES%` is
  drawn *after* the eye groups, so a filled lens simply covers whatever eyes
  the current state shows and he never needs an eye pose of his own. His brows
  do need their own group though - at the shared height they weld onto the top
  of the frame and read as a thicker rim.
- **His head turns, and the head is split into a skull and a face to do it.**
  `.face-turn` (beard, freckles, both feature bands) slides across the skull
  and `.hair-turn` follows at 40%, so the parting shifts without the mop
  sliding off. The ear belongs to the skull, not the face, and only exists for
  a look with `faceTurn` and only shows in the turned states - front-on it
  pokes out of the silhouette. The far temple arm of the sunglasses hides when
  he is turned, because it points away from you.
  It is a **transition**, not a keyframe animation: he swings his head round
  as the state changes, which is what makes it read as looking at something
  rather than as a different drawing. Turned while he walks in and while he
  points, front for the line - so he looks where he is going, then at the
  person he has condemned, then at you.
  A transition on transform overwrites an SVG transform attribute exactly as
  an animation does, so those two groups carry no attribute of their own.
  `transformAudit()` now checks transitions as well - and note it must test
  the transition DURATION, not just the property list: `transition-property`
  defaults to `all`, so checking the property alone flags every element on
  the page. His torso is still square to the viewer; only the head turns.
- **His walk does not read from the legs.** House proportions put stubby legs
  behind the torso, so the stride is nearly invisible; what sells it is the
  horizontal travel, the arm swing and the body bob. Same lesson as the kick
  that became a jump.

His walk-in is a CSS transition on the container and his durations are read
back off the stylesheet (`transMs()`, and `animMs()` now takes a host so it can
be pointed at him) - no duration is written down twice.

Who he lands it on is a uniform draw over everyone, made *after* the outcome
roll has already come up `ceo` - it decides who wears it, not whether he turns
up, so it is not a second gate on the endings. The draw deliberately includes
the one the wheel just landed on: one time in N there is nowhere to move the
wheel to, and that is what triggers the lap below. There is no nominated
favourite and no drawer control for one - that existed briefly and was wrong.

He then holds the `verdict` pose - finger down, hands on hips - and says a line
before walking off. Two things that are easy to get wrong there:

- **A translate percentage is of HIS OWN width**, which is a fraction of the
  stage, and the stage is narrower than the window. `-118%`/`+132%` left him
  parked beside the wheel, in full view, vanishing only when `hidden` was set.
  The entrance and exit are anchored to the viewport instead. He therefore
  overflows the document by a few hundred pixels on the way out, which is
  exactly the overflow that used to raise a scrollbar and start the wheel
  shaking - `html:has(.app){overflow:hidden}` is what makes it safe, so those
  two things are now load-bearing for each other.
- **A speech bubble is built in its owner's units.** He draws 2.87x the size of
  a wheel character, so his bubble came out 2.87x too and covered most of the
  wheel. `aimBubble()` takes a scale for this; it goes on `.bubble-aim` after
  the translate, never on `.bubble`, which carries the pop animation - and an
  animation overwrites the transform. He is also upright rather than riding a
  slice, so `speak()` takes the net angle outright and his is simply 0.

## Before the first spin

Everyone closes in toward the hub, joins hands and the whole ring turns slowly
(`ring` state, `.resting` on the wheel, `drift()`). Pressing spin hands over:
`endDrift()` freezes the wheel where it visually is and carries on from there,
so there is no jump and `rotation` stays the single source of truth for where
the wheel is. An extra rotation layer would have broken `pointerIndex()`.

How far in they come is worked out per list length. Note the arm reaches along
the character's own **tangent**, not along the chord to the next character, so
the ring radius is not simply "half the gap between them" - sizing it that way
leaves everyone short by a factor of cos(seg/2). The formula is in `render()`.
Hands meet properly for 5-7 names, overlap slightly above that, and fall short
below it because the ring would otherwise sit inside the hub.

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
- **Browser-automation JS runs in an isolated world.** It shares the DOM but
  *not* JS globals, so overriding `Math.random` from it silently does nothing
  to the page while `document.getElementById(...)` works fine - which makes it
  look like the stub took. Force an outcome by editing `outcomes()` in the file
  and reverting, or drive it through the DOM. A `MutationObserver` is the
  reliable way to catch a moment mid-sequence, because it fires on mutation
  rather than on a timer that a background tab has throttled to ~1/sec.
- Do not stub `Math.random()` by index. The call order shifts whenever a state
  gains a speech bubble or `settle()` needs another shuffled deck of
  reactions, and it has silently invalidated tests twice. Wrap it and log the
  calls first to learn the current order, or force an outcome some other way.
