# Multi-Screen Sequencing & Navigation

How the individual animated screens compose into the full YIM story.

## Target behaviour

Carried over from last year's Lottie implementation:

- Screens play one after another
- When a screen's animation ends, it **auto-advances** to the next
- The user can tap the **right** side to go forward, **left** side to go back
- The sequence ends on a **final static screen**

Essentially an Instagram/Stories model.

## What the host must provide

**Artboards do not chain.** There is no "next artboard" concept in Rive — an
artboard is an independent scene. With one artboard per screen, nothing advances
on its own; the host has to swap artboards.

So exactly one thing is structurally required:

> **A signal for when a screen's animation has finished**, so the host knows when
> to advance.

**This has now been tested.** See `harness/README.md`.

> **Measured:** 11 seconds into the 9-second one-shot, `rive.isPlaying` is still
> `true` and no completion event fires. State machines run continuously and do
> **not** report that an animation state finished.

So there is no free completion callback. Two real options:

1. **Authored Rive Event** fired at the end of the outro, listened for via the
   `riveevent` callback. **Recommended.** Timing stays owned by the animation and
   survives retiming.
2. **Host timer with a hardcoded duration** (9.0s). Works, but breaks silently
   the moment a designer retimes the animation. Not recommended.

An earlier draft of this doc suggested a runtime completion callback might make
an authored event unnecessary. Testing disproved that — **the event is required.**

The current auto-firing outro is otherwise fine: it is part of each screen's own
9 seconds, and the host swaps artboards when the event arrives.

### Also confirmed during the harness test

`reset()` detaches the view model. After a reset you must re-acquire
`viewModelInstance`, call `bindViewModelInstance(...)`, and re-apply all data
including images — otherwise writes silently do nothing. This matters for
back-navigation if a screen replays.

## Optional: trigger-driven outro

Not required for basic playback. Only needed if you decide either of these:

- **Tap-forward should animate out** instead of hard-cutting. Stories apps
  typically hard-cut, and hard-cutting is one line of host code. UX call.
- **Hold duration should be host-controlled** — tunable per screen or adjustable
  without a re-export. Currently 1.2s is baked into the timeline.

If either becomes a requirement, the change is: split the timeline so the intro
plays and holds, and an `exit` trigger starts the outro. Plus a skip path so
tap-forward during the intro can jump to the resting state. These are state
machine changes, not keyframe tweaks — cheaper to do before more screens exist.

## The alternative: one long timeline

Worth naming because it genuinely does auto-continue. All screens laid end to end
in a **single artboard on one timeline**. Linear playback needs zero host logic.

Costs:

- Tap navigation needs **hardcoded seek offsets** ("screen 4 starts at frame
  1620"), so the host must know the animation's internal structure.
- **Retiming silently breaks navigation.** Add 10 frames to screen 2 and every
  later offset is wrong, with no error.
- Screens stop being independently editable.

Rejected for those reasons, but it is a legitimate design if navigation
requirements ever soften.

## File architecture

**Recommendation: one `.riv`, one artboard per screen.**

    yim.riv
    |
    +-- view models
    |   +-- YearInMotion
    |       +-- theme          : Theme          <- shared by every screen
    |       +-- topInstructor  : TopInstructor
    |       +-- totalClasses   : TotalClasses   (future)
    |       +-- minutesInClass : MinutesInClass (future)
    |
    +-- artboards
        +-- TopInstructor      -> reads theme + topInstructor
        +-- TotalClasses       -> reads theme + totalClasses
        +-- MinutesInClass     -> reads theme + minutesInClass
        +-- FinalScreen        -> reads theme

One section per artboard, one artboard per section, theme shared by all.

### Why not one file per screen

The instinct to split comes from Lottie, where each JSON was genuinely
independent because there was no shared data model. Rive changes that:

**View model definitions live inside a `.riv` and are not shared across files.**
Six files means six copies of the `Theme` definition, the host setting studio
colors six separate times into six unrelated model instances, and six re-exports
whenever `Theme` changes.

Other reasons one file wins here:

- A member's YIM is **one payload**. One view model instance maps to
  it directly.
- Fonts and shared assets are embedded once, not once per file.
- Host wiring is far simpler: acquire one instance, set all data, swap artboards.

### The costs, honestly

- **Designers are coupled.** One file, one export — parallel work on different
  screens collides. This is the real cost, and it is organisational, not
  technical. Needs clear ownership or sequencing.
- **The whole payload loads upfront.** No lazy-loading later screens. Likely fine
  for mostly-vector content, but measure it in the spike.
- **Every release touches every screen.** A tweak to one screen re-exports the
  file containing all of them, so the regression surface is always all screens.

### To verify during the spike

- Can **Rive Libraries** share view model definitions across files? If so,
  splitting becomes cheaper and this recommendation softens.
- Does the runtime allow **one view model instance to drive multiple artboards**?
  This design assumes yes.
- Total file size and load time with all screens included.

> Supersedes an earlier recommendation in this doc that favoured one file per
> screen. That was reasoned from the Lottie workflow; it did not account for view
> model definitions being file-scoped, which is the deciding factor.

## Open questions

- **Back-navigation:** does the previous screen replay its animation from the
  start, or appear already-completed? Stories usually replay. Replaying means
  the host must be able to reset a screen to frame 0.
- **Hold duration:** currently baked into the animation at 1.2s. Should the host
  own this instead, so all screens hold consistently and the duration is tunable
  without a re-export? Probably yes.
- **Tap-forward mid-animation:** skip to the resting state, or cut immediately to
  the next screen?
- **Progress indicator:** last year's design should be checked — segmented
  progress bars are standard in this pattern and need per-screen duration, which
  argues for the host owning timing.
- **Image preloading:** if instructor photos load slowly (see
  [integration-constraints.md](integration-constraints.md#1-instructor-images--the-hard-one)),
  screens may need to preload assets for screen N+1 while N is playing. This
  could significantly shape the playback model — resolve the image spike first.
