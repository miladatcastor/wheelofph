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
stands. This has caused three separate bugs: the shove announcing the wrong
name, bubbles rotated out by exactly one slice, and - for a long time -
`stop()` itself. There are two functions now and the difference is the whole
point:

- `pointerIndex()` reads the matrix. Correct **only** for the fear loop, which
  wants the live position while the wheel is still turning.
- `targetIndex()` reads `rotation`. Anything deciding **who is picked** uses
  this. `stop()` runs off either the animation's `finished` promise or the
  stopTimer backstop, and a background tab never advances the timeline - so
  the backstop fired while the matrix still held the PRE-SPIN angle and the
  wheel announced whoever was under the pointer before you spun, while
  visibly sitting somewhere else entirely. The two agree whenever the
  animation really finished, since `mod(round(-x/seg), n)` is unchanged by
  adding whole turns.

**Mind the descendant space in the state CSS.** `.char[data-state="x"]
[data-show~="x"]` needs the space; without it the selector matches a single
element that has both attributes, which nothing does, and that state renders
as a faceless blob with no error anywhere.

**`transitionend` bubbles - and that is why the wheel no longer uses one.**
The listener that ended a spin lived on `.wheel`, so without an
`e.target === wheel` check *any* transition on *anything* inside the wheel
ended it. Adding a 0.7s transition to the characters was enough: it landed
40ms before `launch()` and settled the wheel without it ever turning.
`turnWheel()` uses the Web Animations API now and waits on the animation's own
`finished` promise, so there is no listener and nothing to bubble into it. If
you are ever tempted to go back to a CSS transition here, this is the bug you
are re-inviting.

**Never duplicate a CSS duration in JS - ask the browser instead.** A
hand-copied duration once truncated the cave animation to 56%, cutting it off
mid-lunge. There used to be a `syncTimings()` that flipped a character through
every state at start-up and parsed durations back out of the stylesheet; it is
gone. `running(el)` returns the Animations actually on an element and its
subtree, `animMs(el)` the longest of them, and `settled(el)` a promise for when
they are done. CSS transitions come back from `getAnimations()` too, so the
CEO's slide is covered by the same three functions.

Two things that are load-bearing there:

- **Infinite animations are filtered out.** An idle breath never finishes, and
  waiting on one hangs the sequence for ever.
- **`settled()` races `finished` against a timeout** taken from the same
  Animation objects. A background tab never advances animations, so `finished`
  would simply never settle - and the spin button would stay disabled for ever
  if someone switched tabs mid-spin. The timeout is not a duplicated duration:
  it is read off the very animation being waited on.

Only keyframe *percentages* still live in JS (`STOMP_IMPACT_PCT`,
`SHOVE_HEAVE_PCT`), because nothing fires on reaching a keyframe - but they are
multiplied by a duration measured at runtime, not by one written down twice.

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

**The stage has to be genuinely square, not merely `aspect-ratio:1`.** In a
flex column the block axis is flex-driven, so `flex:1 1 auto` grew the height
and `max-width` then capped the width - aspect-ratio loses that argument and
you get a tall box. The wheel still draws round, because the SVG letterboxes,
so it is easy to miss; what gives it away is `.wheel-wrap`, which is `inset:0`
with `border-radius:50%` and so stretches its pale shadow ring into a big egg
around a circular wheel. The pointer detaches and floats above the rim too. At
520x1000 the stage came out 496x863. It is sized from the container now
(`width:min(100cqw, calc(100cqh - 3rem))`), so neither axis is left to the
flex algorithm; the 3rem is the result line plus the gap and only bites when
height is the limit. Wrapped in `@supports (width: 1cqw)` so a browser without
container queries keeps the old behaviour instead of collapsing the stage.

`?inspect` runs a **layout audit** over eight viewports from 380x820 up to
1920x1080, checking the stage is square, the halo is round, the pointer is on
the rim, the wheel fits and the result line is not pushed off. It squeezes the
real app by constraining `.app` itself, because JS cannot resize the window -
everything that matters follows from that, including the container query units
the stage is sized from. It also covers sizes you cannot test by hand: Chrome
refuses to make a window narrower than 500px, so 380 is only reachable this
way. Verified both directions - clean as it stands, and red on every affected
viewport when the container-query sizing is disabled.

`?inspect` also runs a **name-count audit** (2 to 24 names: everything inside
the rim, figures not colliding, nothing collapsed) and offers a **scrubber** -
a state picker and a slider that steps any animation by hand. The scrubber is
not a luxury: a background tab does not run animations at all and headless
Chrome advances timers but not the animation clock, so stepping frames is the
only way anything here can be watched moving.

Two measurement traps the name-count audit had to get around, both of which
caught me first:

- **`getBoundingClientRect` is axis-aligned.** A character rotated onto its
  side reports a box far wider than the drawing, so every neighbour looks
  like a collision from about twelve names up. Use `getBBox()` scaled by the
  CTM for the width that is actually drawn.
- Rotated **label** boxes read as poking past the rim for the same reason, so
  labels are left out of the containment check.

**Sizes that were fitted at five names have to be derived, not typed in.**
`ceoSayScale()` is the worked example: his bubble is built in his units and he
draws far larger than a wheel character, so the ratio comes from
`charScale * CEO_VB / (400 * CEO_WIDTH)`. It lands on 0.45 at five names,
which is exactly what had been hand-fitted there - and on 0.17 at twenty-four,
where the typed-in value would have towered over the characters.

## Animation has to survive a screen share

This gets shared as a Chrome tab on calls, which caps at ~30fps and adapts
lower. Keep looping animations at or below ~5Hz. Two were above it and aliased
into noise when captured; the fix was lower frequency with larger amplitude,
which reads *better* at small size, not worse.

When a gesture needs to be legible at 90px, change the whole silhouette. A
swinging limb is not enough — the kick was replaced by a jump because a leg is
small, low, and half-hidden by the torso.

**The spin is a velocity profile, and the keyframes are derived from it.**
Written the obvious way - each stage as (fraction of the clock, fraction of
the angle), eased out - every stage's curve dies to a standstill at its own
end, so the wheel braked to 80deg/s and then leapt back to 1300 in the next
one, four times a spin. That reads as dropped frames, not as friction. So
`SPIN_SPEEDS` gives the speed at each stage boundary and `SPIN_STAGES` how
long each stage lasts; the angles fall out of the two, and so do the easings
(a stage running v_in -> v_out about its own mean is `cubic-bezier(1/3,
v_in/3m, 2/3, 1 - v_out/3m)`).

The opposite mistake is just as easy: a profile whose speed falls *evenly*
joins back up into exactly the single smooth glide it was meant to replace.
What anyone notices is not a step in the speed - an instant drop looks like a
glitch too - it is a step in the RATE of slowing. So the drops between those
speeds are deliberately uneven and the stages alternate between shedding most
of a speed and very nearly keeping it. Both failures were sitting in the file
at different points in the same afternoon.

**A means-only audit cannot see any of that, and said it could.** Easing does
not move a stage's mean, so checking that each stage is slower than the last
passes the lurch version happily - it passed while the audit text claimed it
checked "no step at the joins". `spinAudit()` now parses each
`cubic-bezier` back out and compares `mean*(1-y2)/(1-x2)` leaving one stage
against `mean*y1/x1` entering the next. Verified by reinstating the original
flat easing and watching it flag join 5.

**An audit that reads the constant it is checking says only that the code
agrees with itself.** The close-call bound was checked against `CLOSE_EDGE`,
so setting `CLOSE_EDGE` to `[0.05, 0.12]` - landings nowhere near a line -
passed clean. The bounds are written out in the audit now. Assume any check
phrased in terms of the thing it audits is vacuous until a deliberately wrong
value makes it red.

**The close call moves where the wheel rests, never what it picks.**
`CLOSE_CALL` (0.6) parks the landing angle against the line between two
names so nobody can call it until the wheel stops. The slice is drawn before
this runs and the nudge only moves the resting angle *within* that slice;
which edge is a coin flip. `CLOSE_EDGE` tops out at **0.47 of a slice from
centre, and the 0.5 it stays under is load-bearing**: `targetIndex()` rounds,
so at 0.5 the pick becomes a rounding accident. `spinAudit()` checks the
index against the one `spinPlan()` drew, every time.

This is presentation, not a layer on the odds, for the same reason `TAKES` is
not one - it decides how an outcome is played, not which outcome. If you ever
find yourself making it a second roll over *who*, stop.

**What was tried first and is wrong: a false stop.** The wheel settled on the
previous name, hung for half a second, then crept a whole slice onto the
real one. It measured beautifully and it is not what a wheel does - the ask
was that you cannot tell which side of a line it is on, which is a smooth
approach to a boundary, not a stop and a second move. Two things from that
attempt are worth keeping if it ever comes back: a false stop a fixed 0.75
of a slice back lands on the *same* index for a quarter of landings, because
`targetIndex()` rounds; and pegging a creep below the mean of the final
stage - a stage that is already a crawl to zero - pushed every such spin past
eight seconds.

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

Caving is performed three ways - cave, facepalm, shrug - picked uniformly.
At 60% of landings one fixed beat stops reading as a reaction at all, which
is the same thing that happened to the bystanders before they got four
celebrations. It is not a layer on the odds: the outcome is still one flat
pick, and this only decides how that one outcome is played.

The variety that earns its keep is not the final pose, it is **whether there
is a wind-up**. Cave and facepalm square up first; a shrug does not bother.
That changes the opening half second rather than the last frame, and the
opening is the half you notice repeating. The result line follows - "almost
passed it on" is only true if they actually put up a fight.

Both new poses change the OUTLINE, because at 90px nothing else carries: the
shrug takes the body from about 36 units across to 50, and the facepalm needs
its own ellipse for the palm since a hand circle of r=4 cannot hide a head of
r=14. At 6.4 it still left an eye showing and read as a hand near a face; 7.9
covers both eyes and reads as a facepalm.

Every landing ends one of exactly four ways: push right, push left, cave, or
the CEO. Caving is the common one; the three that pass it on or bring Derk in
split what is left, equally.

    accept 60%    right 13.3%    left 13.3%    ceo 13.3%

**The weighting is the shape of the list, nothing else.** `WEIGHTS` is
`{accept:9, right:2, left:2, ceo:2}`, `OUTCOMES` expands it once at start-up,
and one flat pick is taken from that. 9:2:2:2 out of 15 is the smallest whole
number way of writing 60/13.3/13.3/13.3 exactly - 40% does not divide into
three neatly, and a shorter list can only approximate it by making the three
unequal, which is the one thing they must not be. Repeating entries is not a
layer: it is still one roll you can read in one line, and changing the balance
means changing those four numbers.

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

**Open `index.html?design` and drag until it looks like them.** A look is only
eight colours, three picks from the part tables and two head measurements -
a tiny space. What was slow was never expressing it, it was setting those
numbers off a photo and finding out at true size that the hair read as horns
or the head came out 0.80 wide when it wanted 0.69. The editor closes that
loop: every control live, the character drawn at true wheel size *and* large
side by side, their photo dropped in next to it, and the `SEED_LOOKS` entry
written out in the file's own style ready to paste.

Two things in there are worth knowing. **Hold area** keeps `rx * ry` at 196,
which is the base circle's area and what every existing look sits at - it is
the thing that stops a new face arriving bigger than everyone else's. And the
readout under the sliders gives the ratio against the 0.64-0.78 everyone else
occupies, so a head that is drifting out of family says so while you drag.

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
