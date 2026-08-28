# Data Contract

The view model API Engineering codes against. Rive data binding drives everything
dynamic; the `.riv` ships with placeholder content only.

## Structure

    YearInMotion                      <- bound to artboard "Artboard" (375x812)
    |
    +-- theme: Theme
    |   +-- primary     : color
    |   +-- secondary   : color
    |   +-- background  : color
    |   +-- foreground  : color
    |
    +-- topInstructor: TopInstructor
        +-- first  : Instructor       -> center pillar (tallest)
        +-- second : Instructor       -> right pillar  (medium)
        +-- third  : Instructor       -> left pillar   (shortest)

    Instructor
    +-- name       : string
    +-- image      : image
    +-- classCount : number

`Instructor` is a reusable type. `topInstructor` is a **section container** — as
more stats are added they become siblings of it under the root, keeping
`YearInMotion` organised by screen rather than as a flat pile of every field:

    YearInMotion
    +-- theme
    +-- topInstructor
    +-- totalClasses     (future)
    +-- favoriteClass    (future)

Adding a fourth instructor is one new property on `TopInstructor`, not three.

## Where each property is used

| Property | Drives |
|---|---|
| `theme.secondary` | Artboard background |
| `theme.foreground` | Heading, pillar fills, class counts, "classes" labels |
| `theme.background` | Instructor name text (reads against the pillar) |
| `theme.primary` | **Currently unused** — no element bound to it |
| `topInstructor.*.name` | Name text on each pillar |
| `topInstructor.*.image` | Photo at the top of each pillar |
| `topInstructor.*.classCount` | Floating number above each pillar |

**`theme.primary` is bound to nothing.** Either wire it up or remove it before
publishing the contract — an unused property in a published API is a trap.

## Runtime notes

- **`classCount` is a number, not a string.** An in-file converter renders it as
  text with zero decimals. Supply a number.
- **Colors are ARGB integers**, not hex strings. `#FF000000` -> `4278190080`.
  Include the alpha byte; omitting it yields a transparent (invisible) color. See
  [integration-constraints.md](integration-constraints.md#4-colors-are-argb-integers).
- **`image` needs decoded image data, not a URL.** The hardest integration item —
  see [integration-constraints.md](integration-constraints.md#1-instructor-images--the-hard-one).

## Playback

- Artboard: `Artboard`, 375 x 812, clipping enabled
- State machine: `State Machine 1` (default, autoplays)
- Timeline: `TopInstructor Sequence` — 540 frames @ 60fps = 9.0s, one-shot
- Shape: intro (~5.8s) -> hold (1.2s) -> outro (~1.8s) -> empty

The outro fires on a fixed internal timer, which is **not compatible** with
tap-to-advance navigation. See [sequencing.md](sequencing.md).

## Test instances in the file

Two `YearInMotion` instances exist for editor testing:

| Instance | Theme | Instructors |
|---|---|---|
| `Studio A` | Light Studio | Sam T. (17) / Dana W. (21) / Jess P. (6) |
| `Studio B` | Dark Studio | Sam T. (17) / Dana W. (21) / Jess P. (6) |

Note both studios currently reference the **same three instructors**, so
switching between them only exercises theming. Each now has its own
`TopInstructor` instance (`Studio A Instructors` / `Studio B Instructors`), so
they can be given different instructors to make the A/B switch a fuller test.

## Known cleanup — outstanding

Three elements carry an **orphaned duplicate binding** left over from the
migration. Each has one correct binding plus one with a null path:

| Element | Correct binding | Orphan to delete |
|---|---|---|
| `Pillar3 Image` | `topInstructor -> second` / `image` | null-path binding to `Instructor.image` |
| `Pillar3` name run | `topInstructor -> second` / `name` | null-path binding to `Instructor.name` |
| `Pillar2 Image` | `topInstructor -> third` / `image` | null-path binding to `Instructor.image` |

**These must be deleted by hand in the Rive editor.** A null-path binding can
fall back to the view model's default instance, which would make those elements
ignore the active `YearInMotion` instance entirely — the exact bug this contract
is designed to avoid. Delete before exporting a `.riv`.

## Gotchas when changing bindings

Learned the hard way on this file:

1. **Rive tooling retargets bindings but cannot delete them.** If a rebind creates
   a second binding instead of retargeting, the element ends up bound to two
   sources and behaves nondeterministically. Always re-list bindings after
   editing and confirm one binding per element.
2. **Retargeting can silently drop an attached converter.** After any rebind,
   confirm the three `classCount` bindings still have the number-to-string
   converter, or counts will render wrong.
3. **Deleting a view model property orphans its bindings rather than removing
   them** — they persist with a null path and must be cleaned up manually.
