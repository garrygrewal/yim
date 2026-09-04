# YIM

Rive-based animated YIM experience, rendered in a WebView inside the consumer
mobile app.

This replaces the Lottie JSON approach used in the previous year. The goal is a
white-label experience: Engineering loads a compiled `.riv` file and supplies
studio colors, instructor data, and other stats at runtime via Rive's data
binding API.

## Status

| Item | State |
|---|---|
| Top Instructor screen — design | Done |
| Top Instructor screen — animation (9.0s, in → hold → out) | Done |
| Data binding contract | Restructured & validated — 1 orphaned binding left to clean up |
| `.riv` export | Exported 2025-08-28 → `riv/testingyim2025.riv` |
| Multi-screen sequencing / navigation | Not started — **needs an authored Rive Event** (tested) |
| Browser harness | Working — see [harness/](harness/README.md) |
| WebView spike | Not started — see [spike-plan.md](docs/spike-plan.md) (Step 0 done, start at Step 1) |

Data binding is now **validated in a desktop browser** via `harness/` — the
nested view model, theming, text, counts and runtime image decoding all work.
What remains unproven is the **app's WebView** (WASM instantiation and CSP),
which is still the go/no-go item in [spike-plan.md](docs/spike-plan.md).

> **The committed `.riv` is slightly stale.** It was exported 2025-08-28 12:17,
> before two `Pillar2/Pillar3 Image` bindings were repointed at the root
> (`YearInMotion -> topInstructor -> ...`). In this build those two instructor
> photos will not follow the active studio instance. One orphaned binding on the
> `Pillar3` name run also still needs deleting by hand in the Rive editor.
> **Delete the orphan, then re-export and replace `riv/testingyim2025.riv`.**

## Docs

- **[getting-started.md](docs/getting-started.md)** — read this first. What Rive
  is here, how it differs from Lottie, and the designer → engineer workflow.
- **[data-contract.md](docs/data-contract.md)** — the view model API Engineering
  codes against.
- **[integration-constraints.md](docs/integration-constraints.md)** — known
  limitations and the hard problems. **Read before spiking.**
- **[spike-plan.md](docs/spike-plan.md)** — ordered validation steps with
  go/no-go criteria.
- **[sequencing.md](docs/sequencing.md)** — multi-screen playback and tap
  navigation.

## Repo layout

    docs/          documentation
    harness/       standalone browser test harness (run this first)
    riv/           compiled .riv runtime files

`.riv` files are compiled binaries, not source. The editable source of truth
lives in the Rive workspace; this repo holds the build output.
