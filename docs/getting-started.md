# Getting Started

For engineers picking this up. Assumes you worked on (or have seen) last year's
Lottie-based Year in Motion.

## What this is

A Rive animation rendered in a WebView. Rive replaces Lottie. The screen is
data-driven: the animation file ships with placeholder content, and the app
supplies real studio theming and instructor data at runtime.

## Mental model vs. Lottie

If you carry Lottie assumptions into this, four of them will break:

| | Lottie (last year) | Rive (this year) |
|---|---|---|
| Asset format | `.json` — text, diffable, hand-editable | `.riv` — **compiled binary** |
| Runtime | Pure JS | JS **+ a WebAssembly binary** |
| Dynamic data | Patch the JSON, or use slots | Real runtime **data binding API** |
| Images | Can be base64-embedded in the JSON | Must be **fetched and decoded at runtime** |

The data binding API is a genuine upgrade. The WASM dependency and the image
handling are genuine new costs. See
[integration-constraints.md](integration-constraints.md) before you write code.

## The workflow

    Designer (Rive editor)  →  export .riv  →  commit to riv/  →  app loads it

1. Design and animate in the Rive editor. The editable source lives in the Rive
   workspace, **not** in this repo.
2. Export "for runtime" to produce a `.riv`.
3. Commit the `.riv` to `riv/` with a version bump.
4. The app loads it and drives it through the data binding API.

Because `.riv` is a binary, you cannot review a design change in a PR diff.
Review happens in Rive, not GitHub. Treat the `.riv` as a build artifact.

## Current file

One artboard, `Artboard`, **375 × 812**, with artboard clipping enabled.

- State machine `State Machine 1` (default, autoplays)
- Timeline `TopInstructor Sequence` — 540 frames @ 60fps = **9.0s**, one-shot
- Structure: intro (~5.8s) → hold (1.2s) → outro (~1.8s) → empty screen

The outro currently fires automatically on a fixed timer. That conflicts with
tap-to-advance navigation and will need to change — see
[sequencing.md](sequencing.md).

## Before you start

1. Run the browser harness — see [harness/](../harness/README.md). It loads the
   committed `.riv` and drives the whole data contract with zero app code, so it
   is the fastest way to see what you are integrating against.
2. Read [integration-constraints.md](integration-constraints.md).
3. Run the spike in [spike-plan.md](spike-plan.md) in order. Step 0 is already
   done; **start at Step 1**. Steps 1–2 are go/no-go for the whole approach — do
   not build feature code before they pass.
