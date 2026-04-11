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

Updates text in an existing container without rebuilding the whole page. Faster and flicker-free (on real hardware).

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

Note: `GetGlassesInfo` is the underlying native method name; the SDK's public `getDeviceInfo()` wraps it.
