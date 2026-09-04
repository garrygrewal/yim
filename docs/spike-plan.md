# Spike Plan

Ordered validation before committing to Rive for the full YIM
experience. **Run in order.** Steps 1–2 are go/no-go for the whole approach —
stop and reassess if either fails.

The purpose of this spike is to answer one question: *can we build the rest of
YIM in Rive, in a WebView, without discovering a blocker halfway
through?* Optimise for killing the approach fast, not for polish.

**Timebox: 5 engineering days.** If Steps 1–2 are not both green by day 2, that
is itself the answer — stop and write up Step 6. Overrunning the box to make Rive
work is the failure mode this plan exists to prevent.

---

## Step 0 — Unblock export ✅ Done

`.riv` export was gated on the Rive workspace plan. **Resolved** — exported
2025-08-28 and committed to `riv/testingyim2025.riv`.

Data binding has also since been validated in a desktop browser via
[harness/](../harness/README.md): the nested view model, theming, text, counts and
runtime image decoding all work. **Start at Step 1.**

## Step 1 — WASM runs in our WebView

**Go/no-go.** Do not build anything else first.

Load the Rive web runtime in the app's WebView, exactly as the real feature would
(same WebView config, same CSP, same local-vs-hosted content strategy) and render
the `.riv` with its default placeholder data. No data binding yet.

**Serve the runtime JS, the WASM binary and the `.riv` from the origin production
will use** — bundled local content if that is the plan, otherwise the real hosted
origin. Do **not** copy the harness here: it pulls the runtime from a CDN
(unpkg), which is precisely the arrangement that will not survive the app's CSP.
Loading from a CDN proves nothing about this step.

Pass: the animation plays on a real iOS device and a real Android device, with
the production CSP in force.

Fail: WASM won't instantiate, CSP blocks it, or the binary can't be fetched from
the WebView's origin. → Rive in a WebView is not viable; evaluate native runtimes
instead, which is a different and larger project.

## Step 2 — Instructor images

**Go/no-go.** The highest-risk integration item — see
[integration-constraints.md](integration-constraints.md#1-instructor-images--the-hard-one).

Fetch three real instructor photos over the network, decode them via
`rive.decodeImage(...)`, and bind them to the `image` property on `first`,
`second` and `third`. These are **cross-origin** fetches, which the harness never
exercised — expect to need CORS headers on the photo origin.

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

The question is not "does Rive work" — it is **"is Rive better than the Lottie
setup we already have"**. Fill this in with real numbers before deciding:

| Measure | Lottie (2024, baseline) | Rive (measured) | Verdict |
|---|---|---|---|
| Payload over the wire (runtime + WASM + asset) | | | |
| Time from screen open to first frame, low-end Android | | | |
| Sustained fps through a screen, low-end Android | | | |
| Memory with 3 decoded images loaded | | | |
| Engineering effort to add one new screen | | | |
| Designer effort to change one screen, export to shipped | | | |
| Blockers hit that have no workaround | | | |

Suggested thresholds — agree these **before** measuring, not after:

- **Stop** if any Step 1–2 blocker has no workaround.
- **Stop** if sustained fps on the target low-end device is materially worse than
  the Lottie baseline, or first frame is more than ~500ms slower.
- **Continue** only if the data binding win (real white-labelling, no JSON
  patching) is worth the WASM dependency, the opaque binary, and the coupled
  single-file designer workflow described in
  [sequencing.md](sequencing.md#the-costs-honestly).

If no Lottie baseline numbers exist, measure last year's build on the same device
first. A comparison against a remembered impression is not a comparison.

Decide deliberately. The cost of discovering a blocker after building six
animations is far higher than the cost of this spike.

---

## What this spike does NOT cover

- Production error handling and offline behaviour
- Analytics
- Accessibility (reduced motion, screen readers) — **should be scoped separately;
  a 9-second auto-playing animation has real reduced-motion implications**
- Localisation and non-Latin name rendering
