# Page lifecycle

## `createStartUpPageContainer`

Must be called **exactly once** at app startup. Establishes the initial page layout. Returns `StartUpPageCreateResult` (0=success, 1=invalid, 2=oversize, 3=outOfMemory).

```typescript
const result = await bridge.createStartUpPageContainer(
  new CreateStartUpPageContainer({
    containerTotalNum: 2,
    textObject: [textContainer],
    listObject: [listContainer],
  })
)
```

**Fields:** `containerTotalNum`, `listObject?`, `textObject?`, `imageObject?`

## `rebuildPageContainer`

Replaces the entire page. Can change container count, types, and layout. This is the primary way to navigate between screens.

```typescript
await bridge.rebuildPageContainer(
  new RebuildPageContainer({
    containerTotalNum: 1,
    textObject: [newTextContainer],
  })
)
```

**Behaviour:** Full redraw – all containers are destroyed and recreated. Any internal scroll position or list selection state is lost. On real hardware this causes a brief flicker.

**Fields:** Same as `createStartUpPageContainer`.

## `textContainerUpgrade`

Updates text in an existing container without rebuilding the whole page. Flicker-free on real hardware.

**Faster only for one or two containers.** It costs ~83 ms per call against
~165 ms for a whole page rebuild, so break-even is 2 containers — updating 5-6
in place is roughly 3x slower than simply rebuilding. See
[performance.md](performance.md).

```typescript
await bridge.textContainerUpgrade(new TextContainerUpgrade({
  containerID: 1,
  containerName: 'main-text',
  contentOffset: 0,
  contentLength: 50,
  content: 'New content',
}))
```

**Fields:** `containerID`, `containerName`, `contentOffset?`, `contentLength?`, `content`

## `updateImageRawData`

Updates image data for an existing image container. Must be called after page creation – image containers are empty placeholders until this is called.

```typescript
// Send as PNG byte array
const pngBytes = Array.from(new Uint8Array(await pngBlob.arrayBuffer()));
await bridge.updateImageRawData(new ImageRawDataUpdate({
  containerID: 3,
  containerName: 'logo',
  imageData: pngBytes,
}))

// Or send as base64 PNG string
const base64 = canvas.toDataURL('image/png').replace(/^data:image\/png;base64,/, '');
await bridge.updateImageRawData(new ImageRawDataUpdate({
  containerID: 3,
  containerName: 'logo',
  imageData: base64,
}))
```

Returns `ImageRawDataUpdateResult` (success/imageException/imageSizeInvalid/imageToGray4Failed/sendFailed). Do not send concurrent image updates – wait for one to complete before starting the next.

Costs **~104 ms of fixed overhead per call**, plus ~4 ms per KB of gray4. Payload
size barely matters; call count is what costs. See [performance.md](performance.md).

### `compressMode` and the 0.0.12 image regression

**SDK 0.0.12 breaks image sends on Even App below 2.2.7.** It stamps
`compressMode: 2` (LZ4) onto every `updateImageRawData` payload while passing
`imageData` through **uncompressed** — verified byte-identical from 512 B to
200 KB, with no size threshold. On an older host this produces `imageException`
or `sendFailed`, or images that render as garbage.

Symptoms seen on firmware 2.2.6.10 with Even App 2.2.6:

- small container (1,682 B gray4): `success`, but rendered garbled
- large container (17,680 B gray4): `sendFailed`
- same app on 0.0.11: correct

The field cannot be suppressed through the public API — passing
`compressMode: 0` is ignored whether the model is an `ImageRawDataUpdate`
instance or a plain object, because the SDK overwrites it during serialisation.
The only interception point is the static `ImageRawDataUpdate.toJson`:

```typescript
// Workaround for hosts below Even App 2.2.7. Prefer updating the app.
const original = ImageRawDataUpdate.toJson.bind(ImageRawDataUpdate)
ImageRawDataUpdate.toJson = (model) => {
  const json = original(model)
  delete json.compressMode
  return json
}
```

**0.0.13 does not change this code path.** Its type definitions are identical to
0.0.12 and it still sends `compressMode: 2` over uncompressed bytes; the only
change is a `minAppVersion: "2.2.6"` field in the SDK's own `package.json`. The
resolution is a host requirement, not an SDK fix — **update the Even App** and
set `min_app_version` accordingly.

**Do not trust that `minAppVersion: "2.2.6"`.** The SDK declares 2.2.6, but in
practice images were still broken there; the fix landed in **Even App 2.2.7**
(confirmed working on firmware 2.2.7.14). Set `min_app_version` to `2.2.7` in
your `app.json`, not to the value the SDK advertises.

### The exit dialogue inverts the foreground events, and can wedge the image channel

`shutDownPageContainer(1)` does **not** produce the event sequence you would
expect. After the request the host fires:

| Event | When | Meaning |
|---|---|---|
| `FOREGROUND_ENTER` (4) | dialogue **appears**, ~100 ms later | the page has already been cleared host-side — **this is the cue to `rebuildPageContainer`** |
| `FOREGROUND_EXIT` (5) | user answers **"No"** | cancelled, keep running — *not* backgrounded |
| `SYSTEM_EXIT` (7) | user answers **"Yes"** | really exiting, clean up |

So the polarity is inverted relative to a real background. Treating that
`FOREGROUND_EXIT` as "backgrounded" leaves timers suspended and the app
half-dead after the user cancels. Arm a flag synchronously when you call
`shutDownPageContainer(1)` and branch on it until the user answers.

Outside the dialogue window the events have their normal meaning.

The firmware also emits **duplicate sys events** for one physical transition,
~50-100 ms apart. Dedupe by (type, timestamp) over ~600 ms before acting, or a
single transition rebuilds the page twice.

**The image-channel defect.** Observed on firmware 2.2.7.14 / Even App 2.2.7 /
SDK 0.0.13 — i.e. it persists on the versions that fixed the `compressMode`
bug, and is a separate problem. After the dialogue is shown and dismissed,
`rebuildPageContainer` still returns `ok`, text still renders and input events
still arrive — but every
`updateImageRawData` returns `sendFailed` **in 1-3 ms**, an immediate rejection
with no transfer attempted, against ~110-215 ms for a normal send. It does not
recover until the app restarts.

**There is no in-app recovery.** Three approaches were tried and all failed:

| Approach | Result |
|---|---|
| Rebuild when the dialogue appears | Wipes the images. Containers are recreated as empty placeholders and the sends that would refill them fail behind the modal. |
| Rebuild after the user cancels | Sends still fail in 1-3 ms; a rebuild does not clear the wedge. |
| Recreate the page via `createStartUpPageContainer` | Rejected with `invalid` (1) after blocking **~2.1 s**. The call is genuinely one-shot per session. |

That last one is a trap worth knowing independently: retrying a failed
`createStartUpPageContainer` costs ~2.1 s per attempt, and the event
dispatcher's busy guard discards every input during it. Latch "startup has been
called" rather than "startup succeeded", or a single failure leaves the app
permanently unresponsive.

**Avoid triggering it rather than recovering from it.** EvenChess reaches the
dialogue only through an explicit "Exit" item in a menu, so a stray double-tap
cannot trip the defect — its source calls this out directly. This still
satisfies the submission requirement that a **root-page** double-tap invoke the
exit dialogue: the root double-tap opens the menu, and the menu item fires the
dialogue. Firing `shutDownPageContainer(1)` directly from every screen, as an
app with no menu layer would, maximises exposure.

## Check the return values

`createStartUpPageContainer` and `rebuildPageContainer` both report whether the
page was actually built, and both results are easy to discard by accident. A
rejected page leaves no containers, so the only visible symptom is that every
following `updateImageRawData` fails — which looks like an image problem rather
than a page problem.

Two related traps:

- Latch your "startup done" flag **whether or not the call succeeded**. It
  records that the one-shot call has been spent, not that it worked. Latching
  only on success looks more correct but is worse: the retry is rejected after
  ~2.1 s of blocking, on every render, and the busy guard drops all input in
  between — a single failed startup then stalls the app permanently. After a
  failure, fall through to `rebuildPageContainer`; it is the only route left.
- Reset that flag on app exit, so the next launch creates the page rather than
  rebuilding one that no longer exists.

## `shutDownPageContainer`

Exits the app.

```typescript
await bridge.shutDownPageContainer(0) // 0 = immediate exit
await bridge.shutDownPageContainer(1) // 1 = show exit confirmation to user
```

### Submission requirement: root-page double-tap must invoke the exit dialogue

The Even Hub review team **rejects** any app whose root (home) page does not call `shutDownPageContainer(1)` on a double-tap (`DOUBLE_CLICK_EVENT`). Reviewer message:

> Please ensure double tapping at the root page on OS can invoke exit dialogue (shutDownContainer(1)).

Non-root screens should treat double-tap as "go back" (the usual convention). On the root page specifically, double-tap must instead ask the host to show its exit confirmation. Use `exitMode: 1` so the user gets the host's native confirmation dialog – do not use `0` from the root page (that bypasses the dialogue and fails the same check).

**Minimal example** – dispatcher branches on the current screen:

```typescript
import { OsEventTypeList, type EvenHubEvent } from '@evenrealities/even_hub_sdk'

function onEvent(event: EvenHubEvent) {
  const type = resolveEventType(event) // see input-events.md

  if (type === OsEventTypeList.DOUBLE_CLICK_EVENT) {
    if (state.screen === 'main-menu') {
      // Root page: hand control back to the host for the exit dialogue.
      void bridge.shutDownPageContainer(1)
      return
    }
    // Any other screen: pop one level.
    void goBack()
    return
  }
  // ...other event types
}
```

If you build on top of `even-toolkit`'s `useGlasses()` hook this is handled for you: it routes `GO_BACK` on a home-mode screen through `showShutdownContainer(1)` by default (`shutdownOnHomeBack: true`). You only need to implement the call yourself if you dispatch events manually.

## `callEvenApp` (generic escape hatch)

Low-level method for calling any native Even App function by name. All the typed methods (`getDeviceInfo`, `createStartUpPageContainer`, etc.) are wrappers around this.

```typescript
import { EvenAppMethod } from '@evenrealities/even_hub_sdk'

// Using the enum
const user = await bridge.callEvenApp(EvenAppMethod.GetUserInfo)

// Using a raw string (for undocumented or future methods)
const result = await bridge.callEvenApp('someNativeMethod', { param: 'value' })
```

**`EvenAppMethod` enum:**

| Enum value | Native method string |
|---|---|
| `GetUserInfo` | `'getUserInfo'` |
| `GetGlassesInfo` | `'getGlassesInfo'` |
| `SetLocalStorage` | `'setLocalStorage'` |
| `GetLocalStorage` | `'getLocalStorage'` |
| `CreateStartUpPageContainer` | `'createStartUpPageContainer'` |
| `RebuildPageContainer` | `'rebuildPageContainer'` |
| `UpdateImageRawData` | `'updateImageRawData'` |
| `TextContainerUpgrade` | `'textContainerUpgrade'` |
| `AudioControl` | `'audioControl'` |
| `ShutDownPageContainer` | `'shutDownPageContainer'` |
| `GetAppLocation` | `'getAppLocation'` |
| `StartAppLocationUpdates` | `'startAppLocationUpdates'` |
| `StopAppLocationUpdates` | `'stopAppLocationUpdates'` |
| `PickImageFromAlbum` | `'pickImageFromAlbum'` |
| `CaptureImageFromCamera` | `'captureImageFromCamera'` |

The location, album and camera entries are SDK 0.0.11+.

Note: `GetGlassesInfo` is the underlying native method name; the SDK's public `getDeviceInfo()` wraps it.
