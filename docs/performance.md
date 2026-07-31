# Performance

Measured on a G2 (firmware 2.2.6.10, Even App 2.2.6, SDK 0.0.13) over ~40 sends
in one session, from a real app. Treat the absolute numbers as one data point,
not a platform specification — but the *shape* (a large fixed cost per host
call, payload barely mattering) has held across every call type tested.

## The headline: per-call overhead dominates

Every host call carries a large fixed cost. Payload size is close to irrelevant.

| Call | Cost |
|---|---|
| `updateImageRawData` | **~104 ms fixed** + ~3.9 ms per KB of gray4 |
| `rebuildPageContainer` | **~165 ms**, flat, regardless of container count |
| `textContainerUpgrade` | **~83 ms** per call |
| `createStartUpPageContainer` | ~100-135 ms |

`updateImageRawData` measured across four container sizes:

| Container | gray4 bytes | Time |
|---|---|---|
| 58x58 | 1,682 | ~110 ms |
| 30x135 | 2,025 | ~118 ms |
| 216x68 | 7,344 | ~145 ms |
| 272x130 | 17,680 | ~173 ms |

Fits `ms ≈ 104 + 0.0039 × gray4Bytes`. **10.5x the data costs 1.6x the time.**

The figure that predicts transfer time is the 4-bit greyscale buffer the host
expands your image into (`width × height / 2` bytes), not your PNG size. A
1.5 KB PNG of a 272x130 container becomes ~17.7 KB of gray4.

### Corollary: LZ4 compression cannot help much

SDK 0.0.12+ compresses image data for the phone-to-glasses leg. The data says
it works — 17,680 B of gray4 costing only ~69 ms of marginal time implies a
nominal rate no BLE link achieves with raw bytes. But it optimises the part of
the pipeline that was already nearly free. A screen change is dominated by
`104 ms × images + 165 ms rebuild`; the pixels are a rounding error.

## What this means in practice

**Fewer host calls beats cheaper host calls.** Every optimisation that reduces
call count wins; every optimisation that shrinks payloads barely registers.

### Merge image containers where geometry allows

Two image containers cost ~208 ms in fixed overhead alone. If both fit inside a
single container's 288x144 limit, compositing them onto one canvas and sending
once is a direct saving.

Worked example: a screen with a 272x130 number and a 58x58 icon overlapping its
area merged into one 281x141 send. Two calls became one, saving ~69 ms — the
merged canvas carried ~2,100 more bytes of gray4, costing ~2 ms, against ~104 ms
of eliminated overhead.

The 288x144 cap is the binding constraint. Strips that would need 30x270 or
32x216 merged cannot be combined and stay at their original call count.

### Do not replace rebuilds with in-place text updates

Tempting on screens that share a layout: skip `rebuildPageContainer` and update
each text container in place instead. Measured, it loses.

At ~83 ms per `textContainerUpgrade` against ~165 ms for a whole rebuild,
**break-even is 2 containers**. A screen with 5-6 text containers costs ~415-500
ms that way versus ~165 ms for the rebuild — around 3x worse.

`textContainerUpgrade` is still the right call for updating *one or two*
containers on a page that is already correct. It is not a rebuild replacement.

### Clearing the display: crossover at 2 containers

| Containers to clear | Blank image sends | One rebuild | Winner |
|---|---|---|---|
| 1 | ~110 ms | ~165 ms | blank send |
| 2 | ~221 ms | ~165 ms | rebuild |
| 4 | ~442 ms | ~165 ms | rebuild |

Because a rebuild is flat regardless of container count while image sends are
per-container, moving containers off-screen (or rebuilding to an empty page)
beats blanking them individually as soon as you have more than one.

## Animation frame rates

Image sends must be serial, so one frame costs one call per animated container.

**One container:**

| Size | ms/frame | fps |
|---|---|---|
| 20x20 | 105 | 9.5 |
| 58x58 | 111 | 9.0 |
| 128x64 | 120 | 8.3 |
| 200x100 | 143 | 7.0 |
| 288x144 (max) | 185 | 5.4 |

**The ceiling is ~9.5 fps** and shrinking the image barely moves it — a 74x
reduction in area buys under 2x the frame rate, because ~104 of the ~105 ms is
fixed. Counter-intuitively, **animate one large container rather than several
small ones**.

**Multiple containers** (58x58 each, serial):

| Containers | ms/frame | fps |
|---|---|---|
| 1 | 111 | 9.0 |
| 2 | 221 | 4.5 |
| 4 | 442 | 2.3 |

They also update one at a time as each send lands, so a multi-container frame
visibly reveals in sequence rather than appearing at once.

Practical ceiling: one container at 200x100 gives ~7 fps — workable for a
spinner, a progress indicator or a slow transition, not for anything that must
read as continuous motion.

**Caveats:** these are call rates, not verified on-glass frame rates. The timing
is the round trip from the JS call to the host's reply; whether the host
resolves after the glasses have painted or earlier once the BLE write is queued
is unknown, so real visual fps may be lower. The SDK docs also warn against
sending images frequently due to limited glasses memory — expect memory or
thermal limits before the frame rate becomes the constraint.

## Startup costs worth checking in your own app

Two bugs found by timing rather than by reading code, both worth ruling out:

**Double initialisation.** Two independent auto-connect paths (one at boot, one
from a UI effect) ran the whole init twice, interleaving their image sends —
which the SDK explicitly forbids. It showed up as one send taking 263 ms against
a ~175 ms baseline, and ~700 ms of duplicated startup work. A guard on the init
entry point fixed it.

**Awaiting location before first paint.** `getAppLocation` took ~3 s on device.
Awaiting it before the first render left the glasses blank for that whole time.
Painting with the last known position and repainting only if the new fix moved
the user removed the delay entirely. The same call awaited inside the event
dispatcher also caused ~5 s of dropped taps, since the dispatcher's busy guard
rejects events while a handler is in flight.
