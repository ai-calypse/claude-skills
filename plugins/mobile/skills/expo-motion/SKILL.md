---
name: expo-motion
description: >-
  How to make motion in an Expo / React Native app feel native — springs over
  durations, interruptible where the user can redirect it, gesture velocity
  carried into the release, things emerging from where they came from, and the
  failure modes that render fine and still feel wrong. Use when adding,
  changing, or reviewing any animation, transition or gesture response in an
  Expo app: a sheet, a modal, a card that expands, a list that reorders, a
  hover/press state, a value that changes while on screen, a swipe or drag, or
  any request phrased as "animate X", "make X smooth", "add motion to X", "make
  it feel native/fluid/like iOS", or "it feels janky/cheap/laggy".
---

# Motion in Expo apps

The feel is not a number. It comes from four decisions, made the same way every
time, and any screen in an app that makes them the same way will feel like the
rest of it.

Everything here assumes `react-native-reanimated` (v4) and
`react-native-gesture-handler` — both ship inside Expo Go, so none of it needs a
development build unless a section says otherwise. `Animated` from `react-native`
is not a substitute: its `useNativeDriver` cannot touch layout props and its
springs do not carry velocity across a retarget.

Put the app's values in one `src/motion.ts` and import them everywhere. One file
means one damping ratio and a handful of responses; values scattered across
components are how an app stops feeling like one app.

```ts
// src/motion.ts — the whole system. Vary `duration`, never `dampingRatio`.
export const SPRING = { dampingRatio: 0.82, duration: 380 } as const; // default
export const SPRING_SNAPPY = { ...SPRING, duration: 240 }; // frequent, small
export const SPRING_CLOSE = { ...SPRING, duration: 460 }; // ~1.2x the open
export const SPRING_FLAT = { dampingRatio: 1, duration: 300 } as const; // values, not objects
```

## 0. What Expo Go can and cannot do

Expo Go bundles a fixed set of native modules. Motion work stays inside it right
up to the point where these three things start to matter:

- **No Skia.** `@shopify/react-native-skia` is not in Expo Go. Custom silhouettes
  go through `react-native-svg` (which *is* included) — enough for animated
  paths, not enough for per-frame blur, shaders, or backdrop distortion.
- **Blur is a static material, not an animated one.** `expo-blur`'s `BlurView`
  is included, but animating `intensity` per frame is expensive on Android and
  visibly steps. Treat blur as a state you cross-fade between, not a value you
  animate.
- **`Info.plist` flags do not apply.** Notably `CADisableMinimumFrameDurationOnPhone`,
  which is what lets a ProMotion iPhone run animation above 60Hz. It comes from
  your app config, so it only lands in a development or production build. Judge
  final smoothness there, never in Expo Go.

None of these change the decisions below. They change what the last 5% costs.

## 1. Pick the mechanism first

This is the call that separates motion that feels alive from motion that
stutters, and it is made before any value is chosen.

**Can the target change while the thing is still moving?**

- **No** — it runs start to finish unmolested. Use the declarative layer:
  Reanimated's CSS transitions (`transitionProperty` / `transitionDuration` in a
  style) or `entering` / `exiting` / `layout` layout-animations. Least code, runs
  on the UI thread, nothing to unwind. This is the default and most motion should
  be this.
- **Yes** — a sheet the user is still dragging, a value that changes three times
  in two seconds, anything retargetable mid-flight. Use a shared value driven by
  `withSpring`.

The reason it matters: **`withSpring` is the only mechanism that carries
velocity across a retarget.** Assign a new `withSpring` to a shared value that is
already springing and the in-flight velocity is inherited — the motion bends
toward the new target without breaking stride. `withTiming`, keyframes, CSS
transitions and layout animations all restart from the current value at zero
velocity, which you see as a stall-then-restart: the thing is travelling,
hitches for a frame, sets off again. No curve tuning removes it.

Cost of getting it wrong in each direction: a shared value where a `layout` prop
would do is thirty lines that a one-line prop already does correctly. A
declarative animation where the target moves produces the hitch.

## 2. The rules that produce the feel

**Springs, not durations.** A duration is a schedule; a spring settles. Reach for
`withTiming` only for things that are not physical — a shimmer, a skeleton
pulse, a crossfade of pure opacity.

**One damping ratio across the app; vary only the response.** Configure springs
with `{ duration, dampingRatio }`, not `{ mass, damping, stiffness }` — the
physics triple couples bounciness to speed, so tuning one silently changes the
other, and the app drifts apart component by component. The `duration` form is
perceptual, and `dampingRatio` stays a constant you set once. When something
needs to feel faster, make it faster; do not also make it bouncier.
`dampingRatio < 1` overshoots and reads as an object landing; `1` is critically
damped and reads as a value changing. Pick by which one it is. (Reanimated's own
defaults — `duration: 550`, `dampingRatio: 1`, and note actual settle time is
1.5× the perceptual duration — are a starting point, not your system.)

**A gesture's velocity is part of the animation.** This is the single biggest
difference between web motion and native motion. On release, feed the gesture's
velocity into the spring:

```ts
.onEnd((e) => {
  x.value = withSpring(snapTarget, { ...SPRING, velocity: e.velocityX });
})
```

Without it, a fast flick and a slow drag end identically and the whole surface
feels dead in the hand. Pair it with a projection — where the finger *would*
have landed at that velocity — to choose the snap target, so a hard flick clears
the threshold a slow drag would not.

**Close a little slower than open.** Roughly a fifth. An equal open and close
reads as a box toggling; a close that lingers reads as something with mass
settling back. This asymmetry is doing more work than any single curve.

**Things emerge from where they came from.** A popover belongs to the button that
opened it; a card detail belongs to the row that was tapped. React Native scales
from the centre by default, which is almost always the wrong origin and is the
most common reason a correct-looking animation feels generic. Set
`transformOrigin` on the style, or translate by half the delta alongside the
scale.

**A size change *is* the animation.** A container whose contents change height
must animate to the new height. Snapping between two sizes is the clearest tell
of an app built by someone who did not look. `layout={LinearTransition.springify()}`
covers most cases for free; when you need the number yourself, measure once with
`onLayout`, cache it, and drive a shared value.

**Squash anisotropically.** When content is replaced, compress it more on one
axis than the other. Equal `scale` on both axes reads as a zoom; unequal reads as
something being squeezed through a slot. Tune the *ratio* between the axes, not
either number alone.

**Frequency governs duration.** The more often a user sees it, the shorter and
subtler it must be. A tab-bar press gets a beat; a screen the user deliberately
opens can take its time.

**Haptics land on the event, not the gesture.** A snap, a threshold crossing, a
commit — one `expo-haptics` impact at that instant, fired from a worklet through
`runOnJS`. Haptics on gesture *start* or on every frame read as noise and
actively cheapen the interaction. On Android, check the effect exists before
leaning on it; several map to nothing.

**Everything that enters must exit — and the parent has to outlive the exit.** An
element cut off mid-frame is worse than one that never animated. An `exiting`
animation does not run if the parent unmounts, if the navigator pops the screen,
or if a `Modal` closes on the same tick. Hold the parent open, then tear it down.

**Reduced motion is a hard switch, not a slower one.** The OS setting is the user
saying movement makes them unwell. Reanimated respects it by default
(`ReduceMotion.System`): entering and layout animations jump to their endpoint,
exiting animations are skipped. For anything you drive yourself, read
`useReducedMotion()` and collapse the *travel* — never merely lengthen it.

## 3. Surfaces that change shape

A thing that merely moves or fades needs only the section above. A thing that
*grows, opens, or changes silhouette* has four more decisions, and getting any of
them wrong caps how good the rest can look.

**Never animate the container's layout. Oversize it once and animate inside it.**
Give the surface a fixed absolutely-positioned host — full screen,
`pointerEvents="none"` where it should not swallow taps — mounted once and never
resized. Animating `width`/`height`/`margin`/`flex` on a parent re-runs layout for
the subtree every frame; animating `transform` and `opacity` inside a still host
does not. This is the decision that makes everything else possible.

The same applies to `Modal`: it runs its own uninterruptible OS presentation and
cannot be dragged mid-open. For any sheet the user can grab, render into your own
host (or a library built on one) instead.

**A silhouette is parameters that interpolate, not a shape you swap.** If the
outline itself changes — corners opening, a concave shoulder, anything
`borderRadius` cannot express — build it as an SVG path whose *inputs* are
animated, and rebuild `d` inside `useAnimatedProps` each frame on an
`Animated.createAnimatedComponent(Path)`. Do not cross-fade between two shapes
and do not reach for a mask. Keep the curve type faithful: a quadratic with its
control point at the corner being cut does not look like an arc or a cubic
through the same points, and that difference is most of the character.

**Never let it settle smaller than the thing it grew out of.** A surface that
emerges from an origin — a row, a button, a thumbnail — must be at least as large
as that origin in every dimension, floored explicitly. Short content will
otherwise ask for less, and a panel narrower than the row it hangs off leaves the
trigger sticking out either side. This looks fine with long placeholder text and
wrong the moment real content is short.

**Derive radii from the origin, not from the element.** Corner radii should come
from the size of the thing it grew out of, so the shoulders stay put as it grows
and then open up as it expands past its origin. Deriving them from the element's
own current size makes the corners swell with the content, which stops reading as
one continuous object immediately. Compute them from the *current* size each
frame so they interpolate rather than jumping to their final value on frame one.

## 4. Failure modes that render fine and still feel wrong

These all pass typecheck, pass review by eye, and are wrong.

**Anything that touches the JS thread per frame.** `runOnJS` inside
`useFrameCallback` or a gesture's `onUpdate`, `setState` per frame, a console log
in a worklet. The animation still runs on the UI thread, but the queue behind it
backs up and everything else in the app — the list you are dragging over, the
next screen's mount — stutters. Do the whole loop in worklets; cross to JS once,
at the end.

**Reading a plain JS value inside a worklet.** Only shared values and captured
constants survive. A `ref.current`, a value from a closure that later changed, a
prop read outside the animated hook — all silently stale, all render fine.

**Writing a shared value during render.** It must happen in an effect, a handler
or a worklet. A render-phase write is dropped or applied out of order, and shows
up as an animation that plays only every other time.

**`useSharedValue` treated as reactive.** The argument is an initial value read
once; a changed prop does not update it. Sync it in an effect or you will animate
from a stale origin.

**Measuring in the wrong place.** `onLayout` is a JS-thread callback that arrives
a frame late — fine for caching a size, wrong for reading one mid-animation. When
a worklet genuinely needs the current geometry, use `measure()` with an animated
ref on the UI thread. Either way, measure when layout changes and cache it; do
not measure inside the frame loop.

**Animating layout props.** `width`, `height`, `top`, `left`, `padding`, `flex`
cost a layout pass every frame. Sometimes unavoidable (a container genuinely
resizing) — when it is, keep the subtree small and its children independent of
the parent's size. `transform` and `opacity` are free by comparison; reach for
them first, always.

**An unstable `transform` array.** Changing the number or order of entries between
frames re-maps the properties and jumps. Keep every transform present for the
whole animation, even at its identity value.

**Every screen flying in at cold start.** `entering` animations fire on first
mount too, so the whole app performs itself on launch. Wrap the tree in
`<LayoutAnimationConfig skipEntering>` for the initial mount.

**Tearing down the element before its exit plays.** A state change that both
starts the exit and unmounts in the same commit produces a hard cut. The "it's
over" signal and the "remove it" signal must be separate, with removal downstream
of the animation finishing.

**A floor that makes a morph inert.** A `minHeight` larger than anything the
content ever needs means the "animated" dimension never actually changes. It
looks implemented and does nothing. Check the floor against real content.

## 5. Verify by measuring, not by watching

**Judge feel on a physical device, in a release-mode or development build, with
no debugger attached.** Expo Go's dev bundle, a simulator, and a connected
debugger each add enough jitter to hide a real problem and to invent a fake one.
Expo Go is for building the animation; it is not where you sign off on it.

Build a hatch for every animated surface — a dev-only route that reaches any
state without performing the gesture that produces it:

```tsx
// app/_dev/motion.tsx — every state, one tap away
// ?surface=sheet&phase=open&dismissAfter=800
```

Then sample the animation rather than trusting your eye. Push frames from a
worklet into a shared array and read the shape afterwards:

```ts
const samples = useSharedValue<number[]>([]);
useFrameCallback(({ timeSinceFirstFrame }) => {
  'worklet';
  if (timeSinceFirstFrame < 1200) samples.value.push(y.value);
});
```

What the numbers should show: movement on the very first frames, most of the
distance covered early, a long quiet tail, and — if underdamped — a peak
slightly past the target. A linear ramp means you got a duration, not a spring.
Identical values for several frames at the start means something is stalling.

Two things worth asserting in a test rather than watching: that a value the user
sees never snaps, and that a component is never unmounted while its exit is
supposed to be running.
