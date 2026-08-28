# Rive Test Harness

Standalone browser harness that loads `riv/testingyim2025.riv` and drives the
`YearInMotion` view model at runtime. No app code involved.

## Run

    python3 -m http.server 8765
    # open http://localhost:8765/harness/index.html

Must be served over HTTP — opening `index.html` from `file://` will fail to fetch
the `.riv` and the WASM binary.

Runtime is pulled from a CDN: `@rive-app/webgl2@2`.

## What it does

- Loads the `.riv` and auto-binds the `YearInMotion` view model
- Live-edits all four `theme` colors (hex is converted to the ARGB ints Rive needs)
- Live-edits `name` and `classCount` for `first` / `second` / `third`
- Sets `image` from a local file **or** a remote URL (exercises fetch + decode + CORS)
- **Dump view model** prints the resolved nesting and current values
- Replay re-runs the sequence

## Verified working

Tested in a Chromium browser against the 2025-08-28 export:

| Behaviour | Result |
|---|---|
| Rive web runtime (webgl2) loads | works |
| `.riv` loads and renders | works |
| `autoBind: true` yields a view model instance | works |
| Nested walk `topInstructor -> first/second/third` | works |
| `string()` / `number()` / `color()` get + set | works |
| `rive.decodeImage()` + assigning to an image property | works, renders |
| Remote image fetch -> decode -> bind | works (same-origin) |

The nested restructure is confirmed good at runtime:

    topInstructor resolved OK
      first:  name="Sam T."   classCount=17
      second: name="Dana W."  classCount=21
      third:  name="Jess P."  classCount=6

## Findings that affect the app integration

### 1. No completion signal from the state machine — an authored Event is required

Measured directly: **11 seconds into a 9-second one-shot, `rive.isPlaying` is
still `true` and no completion event fires.** State machines run continuously;
they do not report that an animation state reached its end.

Consequence: there is **no free "animation finished" callback**. To auto-advance
between screens you must either

- fire an authored **Rive Event** at the end of the outro and listen for
  `riveevent` (recommended), or
- run a host-side timer with a hardcoded duration (brittle — breaks silently
  whenever a designer retimes the animation).

This supersedes an earlier assumption in `docs/sequencing.md` that a runtime
callback might be enough. It is not.

### 2. `reset()` detaches the view model

After `rive.reset(...)` the old `viewModelInstance` reference is stale **and the
new artboard is not bound to it**. Writes appear to succeed but nothing renders.

    rive.reset({ stateMachines: SM, autoplay: true });
    rive.play();
    vmi = rive.viewModelInstance;      // re-acquire
    rive.bindViewModelInstance(vmi);   // AND rebind, or nothing renders
    // re-apply all data, including images — they do not survive a reset

### 3. Colors are ARGB ints and read back signed

`#FF000000` goes in as `4278190080` and reads back as `-16777216`. Same 32 bits,
signed on the way out. Compare with `>>> 0` if you compare at all. Always include
the alpha byte — omitting it yields a transparent, invisible color.

### 4. Images do not survive a reset

Decoded images must be re-applied after any `reset()`. Budget for keeping decoded
image references around rather than re-fetching per replay.

## Not verified here

- **Behaviour inside the app's actual WebView** — this ran in a desktop browser.
  WASM instantiation and CSP in the real WebView are still unproven and remain
  the go/no-go item in `docs/spike-plan.md`.
- **Frame-accurate animation timing.** The automated browser used for testing
  throttles `requestAnimationFrame`, so visual timing checks were unreliable.
  Verify the 9-second sequence by eye in a normal browser.
- **Cross-origin image fetches.** Only same-origin was tested. Real instructor
  photos will come from another origin and need CORS headers.
- Performance on low-end mobile hardware.

## Test asset

`test-instructor.png` is a generated 240x240 solid-colour PNG used to prove the
image path end to end. Replace with real photos when testing CORS.
