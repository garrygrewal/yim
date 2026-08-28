# Spike Plan

Ordered validation before committing to Rive for the full Year in Motion
experience. **Run in order.** Steps 1–2 are go/no-go for the whole approach —
stop and reassess if either fails.

The purpose of this spike is to answer one question: *can we build the rest of
Year in Motion in Rive, in a WebView, without discovering a blocker halfway
through?* Optimise for killing the approach fast, not for polish.

---

## Step 0 — Unblock export

**Blocker.** `.riv` export is gated on the Rive workspace plan. Nothing below can
start until a runtime file exists.

Pass: a `.riv` is exported and committed to `riv/`.

## Step 1 — WASM runs in our WebView

**Go/no-go.** Do not build anything else first.

Load the Rive web runtime in the app's WebView, exactly as the real feature would
(same WebView config, same CSP, same local-vs-hosted content strategy) and render
the `.riv` with its default placeholder data. No data binding yet.

Pass: the animation plays on a real iOS device and a real Android device.

Fail: WASM won't instantiate, CSP blocks it, or the binary can't be fetched from
the WebView's origin. → Rive in a WebView is not viable; evaluate native runtimes
instead, which is a different and larger project.

## Step 2 — Instructor images

**Go/no-go.** The highest-risk integration item — see
[integration-constraints.md](integration-constraints.md#1-instructor-images--the-hard-one).

Fetch three real instructor photos over the network, decode them, and bind them
to `instructorImage` on all three pillars.

Pass: three different photos render in the three pillars, sourced at runtime.

Also record: how long images take to resolve, what the artboard shows before they
land, and what happens when a fetch fails.

Fail or unacceptably slow → rethink. Options include preloading before the screen
plays, baking images into the `.riv` per studio (kills white-labelling), or
dropping photos from the design.

## Step 3 — Data binding end to end

Drive the full contract from app code: theme colors, three instructor names,
three class counts.

Pass: switching between two different studio payloads visibly changes colors,
names, and counts with no file changes.

Also verify: ARGB conversion is correct (no transparent screens), class counts
render without a decimal, long names don't overflow the pillar.

## Step 4 — Performance on real hardware

Measure on a low-end Android device, not just a simulator.

Record: frame rate through the animation, time from screen open to first frame,
memory with images loaded, and the same across several screens in sequence.

This is where a WebView approach is most likely to disappoint. Get a number
before committing to building 6+ screens.

## Step 5 — Sequencing and navigation

See [sequencing.md](sequencing.md). Prove with two screens before building more:

- A screen signals completion and the host auto-advances
- Tap right advances, tap left goes back
- Tapping mid-animation does something sensible
- Back-navigation shows the previous screen in a correct state

Pass: two screens play in sequence with working tap navigation.

## Step 6 — Decide

Write up the results and make an explicit call: continue in Rive, or stop.

Decide deliberately. The cost of discovering a blocker after building six
animations is far higher than the cost of this spike.

---

## What this spike does NOT cover

- Production error handling and offline behaviour
- Analytics
- Accessibility (reduced motion, screen readers) — **should be scoped separately;
  a 9-second auto-playing animation has real reduced-motion implications**
- Localisation and non-Latin name rendering
