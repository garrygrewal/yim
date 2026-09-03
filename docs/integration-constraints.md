# Integration Constraints

Known limitations and hard problems. **Read before spiking.**

Originally derived from inspecting the Rive file and from general Rive runtime
behaviour. Several items have since been **validated in a desktop browser** via
[harness/](../harness/README.md), and are marked **Validated** below.

What remains unproven is behaviour inside the app's **WebView** — WASM
instantiation and CSP — which is still the go/no-go item in
[spike-plan.md](spike-plan.md). Treat anything not marked Validated as a
hypothesis the spike is designed to confirm or kill, and verify API specifics
against current Rive documentation rather than trusting this file.

---

## 1. Instructor images — the hard one

**This is the item most likely to cost real time. Spike it first.**

`image`, on each `Instructor`, is a Rive **image** property, not a URL string. You
cannot hand Rive a CDN link. The runtime needs actual decoded image data.

Roughly, per instructor, the app must:

1. Fetch the instructor photo from wherever it lives
2. Decode it into whatever image type the Rive web runtime accepts
3. Assign it to the view model's image property

With Lottie we could base64-embed images directly in the JSON. That option does
not exist here.

**Validated in the harness.** The decode path is
`await rive.decodeImage(new Uint8Array(bytes))` — async, takes raw bytes — and the
result assigns to `instructorVm.image('image').value`. It renders. Decoded images
do **not** survive a `reset()` and must be re-applied.

Open questions still to answer during the spike:

- What happens on the artboard before an image resolves — placeholder, blank, or
  the design-time asset?
- What happens if a fetch fails, or the studio has no photo for an instructor?
- Are we blocking the whole animation on 3 image loads, or letting them pop in?
- Memory cost of 3 decoded images per screen, across all screens in the story.

Recommendation: **do not design the sequencing until this is understood.** If
images must be preloaded before a screen plays, that changes the whole playback
model.

## 2. WebAssembly in the WebView

The Rive web runtime ships a WASM binary alongside its JS. Lottie was pure JS, so
this is new.

Verify early:

- The WebView can instantiate WebAssembly at all
- The Content Security Policy permits it (`wasm-unsafe-eval` or equivalent)
- If the WebView loads **bundled local content** rather than a hosted URL, the
  WASM file can still be fetched — this is a common failure with `file://`
  origins
- The runtime and WASM load from **the origin production will use**. The harness
  pulls both from a CDN (unpkg), so it proves nothing about CSP or bundled
  origins — the two things most likely to fail here
- The WASM binary's size is acceptable in the app bundle or over the network

**This is go/no-go for the entire WebView approach.** If WASM can't run in our
WebView, we need native Rive runtimes instead, which is a different project.

## 3. `.riv` is an opaque binary

- Not diffable, not reviewable in a PR, not hand-editable
- A one-pixel design tweak means a full re-export and a new commit
- Design review happens in the Rive editor, not GitHub
- The editable source lives in the Rive workspace — **the repo is not the source
  of truth for the design**

Practical consequence: version `.riv` files deliberately and record which Rive
revision produced each one, or you will not be able to trace a rendering bug back
to a design change.

## 4. Colors are ARGB integers

Theme colors are not hex strings. They're packed 32-bit ARGB integers.

    #FF000000  (opaque black)  ->  4278190080
    #FFFFFFFF  (opaque white)  ->  4294967295

Whatever supplies white-label studio colors will almost certainly hand us hex or
CSS strings, so a conversion layer is needed. Include the alpha byte — omitting
it yields a fully transparent color and a silently blank screen.

**Validated in the harness.** Note colors also read back **signed**: `#FF000000`
goes in as `4278190080` and comes out as `-16777216`. Same 32 bits. Normalise
with `>>> 0` before comparing.

## 5. `.riv` export is plan-gated — resolved

Export was previously blocked on this workspace (`exportRiv is not available on
this workspace's plan`), which gated `.riv` and `.rev` together and blocked every
downstream step.

**This is resolved** — a runtime file was exported on 2025-08-28 and is committed
at `riv/testingyim2025.riv`. Recorded here because it stays a live cost: the
workspace must remain on an export-capable tier for the life of the project, or
no design change can ship.

## 6. Fixed artboard size

The artboard is **375 × 812** — a fixed logical size, not a responsive layout.

Decide how it should behave on other aspect ratios (tall phones, tablets, small
devices): letterbox, crop, or scale-to-fit. Rive's fit/alignment settings handle
this at the runtime level, but somebody has to choose the behaviour, and the
choice affects whether the design reads correctly on real hardware.

## 7. Fonts

Fonts used in the design are embedded in the `.riv`. Confirm:

- The fonts we're using are licensed for embedding and redistribution in an app
- Embedded font weight is acceptable in the file size budget

## 8. Text overflow with real data

Instructor names use shrink-to-fit inside a 96px-wide pillar, with wrapping to a
second line. This was tuned against short test names ("Sarah J.", "Jessica P.").

Real studio data will contain longer names. Test with deliberately long and
short names, and with non-Latin characters, before shipping.
