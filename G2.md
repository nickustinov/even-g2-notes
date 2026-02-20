# Even Realities G2 – development notes

> **Disclaimer:** This document is the result of independent research and reverse-engineering of the public SDK. It is not official documentation, may contain inaccuracies, and is updated as new information becomes available. The author is not affiliated with Even Realities.

- [Overview](#overview)
- [Architecture](#architecture)
- [Development flow](#development-flow)
- [SDK initialisation](#sdk-initialisation)
- [G2 hardware](#g2-hardware)
- [Display and UI system](#display-and-ui-system)
  - [Canvas](#canvas)
  - [Container model](#container-model)
  - [Shared container properties](#shared-container-properties)
  - [Border and decoration](#border-and-decoration)
  - [Text containers](#text-containers-textcontainerproperty)
  - [List containers](#list-containers-listcontainerproperty)
  - [Image containers](#image-containers-imagecontainerproperty)
- [Input events](#input-events)
  - [Event types](#event-types-oseventtypelist)
  - [Event delivery](#event-delivery)
  - [Event quirks](#event-quirks-critical-for-real-world-apps)
- [Page lifecycle](#page-lifecycle)
  - [createStartUpPageContainer](#createstartuppagecontainer)
  - [rebuildPageContainer](#rebuildpagecontainer)
  - [textContainerUpgrade](#textcontainerupgrade)
  - [updateImageRawData](#updateimagerawdata)
  - [shutDownPageContainer](#shutdownpagecontainer)
  - [callEvenApp (generic escape hatch)](#callevenapp-generic-escape-hatch)
- [Audio](#audio)
- [Device info](#device-info)
- [User info](#user-info)
- [Local storage](#local-storage)
- [What the SDK does NOT expose](#what-the-sdk-does-not-expose)
- [Error codes](#error-codes)
- [SDK JSON compatibility](#sdk-json-compatibility)
- [UI patterns from real apps](#ui-patterns-from-real-apps)
- [Browser UI component library](#browser-ui-component-library)
- [Even-dev simulator environment](#even-dev-simulator-environment)
  - [Running apps](#running-apps)
  - [App structure](#app-structure)
  - [Packaging and deployment](#packaging-and-deployment)
- [Example apps](#example-apps)

---

## Overview

The G2 are smart glasses with dual micro-LED displays (one per lens), no camera, no speaker. Privacy-focused design. Paired with the R1 control ring for input. Connected to iPhone via BLE 5.x (~28m range).

## Architecture

Even Hub apps are **regular web apps hosted on your own server** (any hosting – Vercel, Cloudflare, your own VPS, etc.). The iPhone acts purely as a proxy between your web app and the glasses.

```
[Your server] <--HTTPS--> [iPhone WebView] <--BLE--> [G2 Glasses]
```

- **Your web app** runs on your server, with its own backend, database, API keys – like any web app.
- **The iPhone** runs the Even App (Flutter), which opens your web app's URL in a `flutter_inappwebview`. The iPhone does not run your app logic – it just loads your page and relays messages to the glasses over BLE.
- **The glasses** are a display + input peripheral. They render UI containers sent via BLE and send back input events (taps, scrolls). No code runs on the glasses.

The SDK injects a JS bridge (`EvenAppBridge`) into the WebView's `window` object. Your frontend JS calls this bridge to control the glasses display and receive input events. The bridge uses `flutter_inappwebview`'s `callHandler('evenAppMessage', ...)` – native WebView message passing, not HTTP.

### Two-way communication

- **Web -> Glasses:** Your JS calls `bridge.callEvenApp(method, params)` which sends a message through the WebView bridge. The Flutter app relays it over BLE to the glasses.
- **Glasses -> Web:** Input events and status updates travel back over BLE to the Flutter app, which calls `window._listenEvenAppMessage(...)` to push them into your WebView.

### Implications for app design

- Your backend can hold API keys, do heavy computation, call third-party APIs – the glasses UI is just a thin frontend
- Standard web security applies: session tokens, server-side secrets, HTTPS
- The WebView has normal browser capabilities (fetch, localStorage, etc.) in addition to the G2 bridge

## Development flow

1. Build a web app (any framework) that imports `@evenrealities/even_hub_sdk`
2. During development, run locally – the Even App loads your `localhost` URL in its WebView
3. For production, deploy to any hosting – the Even App loads it from that URL

SDK: <https://www.npmjs.com/package/@evenrealities/even_hub_sdk>

```bash
npm install @evenrealities/even_hub_sdk
```

## SDK initialisation

Two ways to get the bridge instance:

```typescript
import { waitForEvenAppBridge, EvenAppBridge } from '@evenrealities/even_hub_sdk'

// Method 1: async wait (recommended) – resolves when bridge is ready
const bridge = await waitForEvenAppBridge()

// Method 2: synchronous singleton – only use after bridge is already initialised
const bridge = EvenAppBridge.getInstance()
```

## G2 hardware

- Dual micro-LED displays (green), synced via physical FPC (not wireless)
- 576x288 pixel canvas per eye
- 4-bit greyscale – 16 shades of green. White pixels appear as bright green on the display; black pixels are off
- BLE 5.x – ~28m real-world range, ~9dB more power than G1
- Microphone (accessible via SDK)
- Touch gestures on temple tips
- R1 ring – separate BLE device for scroll/click input
- Wearing detection
- No camera, no speaker

## Display and UI system

### Canvas

576x288 pixels. Coordinate system: origin (0, 0) at top-left, X right, Y down. Green micro-LED display – all colours are converted to 4-bit greyscale (16 levels of green) by the host app before transmission. Border colour ranges vary slightly by container type (0–15 for list, 0–16 for text).

### Container model

The UI is built from **containers** – rectangular regions positioned absolutely on the canvas. There is no CSS, no flexbox, no DOM. You define containers with pixel coordinates, and the glasses firmware renders them.

**Rules:**
- Max **4 containers per page** (mixed types allowed)
- Exactly **one** container must have `isEventCapture: 1` – this is the container that receives input events
- Container count is set via `containerTotalNum` and must match the actual number of containers passed
- Containers can overlap (later containers draw on top)
- No z-index control beyond declaration order

### Shared container properties

All container types share these layout properties:

| Property | Type | Range | Notes |
|---|---|---|---|
| `xPosition` | number | 0–576 | Left edge in pixels |
| `yPosition` | number | 0–288 | Top edge in pixels |
| `width` | number | 0–576 | 20–200 for images |
| `height` | number | 0–288 | 20–100 for images |
| `containerID` | number | any | Unique per page, used for updates |
| `containerName` | string | max 16 chars | Unique per page, used for updates |
| `isEventCapture` | number | 0 or 1 | Exactly one container must be 1 |

### Border and decoration

Available on list and text containers (not images):

| Property | Type | Range | Notes |
|---|---|---|---|
| `borderWidth` | number | 0–5 | 0 = no border |
| `borderColor` | number | 0–15 (list), 0–16 (text) | Greyscale level. Practical values: 5 is a subtle grey, 13 is brighter |
| `borderRdaius` | number | 0–10 | Rounded corners. Note: SDK uses `borderRdaius` (typo in protobuf, preserved in SDK) |
| `paddingLength` | number | 0–32 | Uniform padding on all sides |

No background colour property. No fill colour. The only visual decoration is the border.

### Text containers (`TextContainerProperty`)

The workhorse container. Renders plain text, left-aligned, top-aligned. No text alignment options (no centre, no right-align). No font size control. No bold/italic/underline.

```typescript
new TextContainerProperty({
  xPosition: 0,
  yPosition: 0,
  width: 576,
  height: 288,
  borderWidth: 0,
  borderColor: 5,
  paddingLength: 4,
  containerID: 1,
  containerName: 'main-text',
  content: 'Hello from G2',
  isEventCapture: 1,
})
```

**Content limits:**
- `createStartUpPageContainer`: max **1000 characters**
- `textContainerUpgrade`: max **2000 characters**
- `rebuildPageContainer`: max **1000 characters** (same as startup)

**Text rendering behaviour:**
- Text wraps at container width. If the content overflows the container height and the container has `isEventCapture: 1`, the firmware scrolls the text internally – the user can scroll through the overflow with swipe/ring gestures
- `SCROLL_TOP_EVENT` and `SCROLL_BOTTOM_EVENT` are emitted when the user reaches the top or bottom boundary of the internal scroll – they are boundary events, not raw gesture events
- In practice, most apps manually paginate text anyway (splitting into ~400-char pages) for better UX: controlled page breaks, progress indicators, and staying within the 1000-char content limit of `rebuildPageContainer`
- Containers without `isEventCapture: 1` do not receive scroll input, so overflow text will be clipped
- `\n` works for line breaks
- Unicode characters work (e.g. `▲`, `━`, `─`, arrows) – useful for progress bars and indicators
- Approximate capacity: ~400–500 characters fill a full-screen (576x288) text container, depending on character width
- No font selection, no font size – the firmware uses a single fixed-width-ish font
- To "centre" text, you must manually pad with spaces

**Partial updates via `textContainerUpgrade`:**
- Updates text content in-place without a full page rebuild
- Requires matching `containerID` and `containerName`
- Has `contentOffset` and `contentLength` for partial replacement within the string
- Returns `boolean` (success/failure)
- **Caveat:** On the simulator, this still causes a visual redraw. On real hardware, it's smoother.

```typescript
await bridge.textContainerUpgrade(new TextContainerUpgrade({
  containerID: 1,
  containerName: 'main-text',
  contentOffset: 0,
  contentLength: 100,  // length of content to replace
  content: 'Updated text here',
}))
```

### List containers (`ListContainerProperty`)

Native scrollable list widget rendered by the glasses firmware. The glasses handle scroll highlighting natively – you don't need to manually track selection for scroll events.

```typescript
new ListContainerProperty({
  xPosition: 0,
  yPosition: 0,
  width: 576,
  height: 288,
  borderWidth: 1,
  borderColor: 13,
  borderRdaius: 6,
  paddingLength: 5,
  containerID: 1,
  containerName: 'my-list',
  isEventCapture: 1,
  itemContainer: new ListItemContainerProperty({
    itemCount: 5,
    itemWidth: 560,
    isItemSelectBorderEn: 1,
    itemName: ['Item 1', 'Item 2', 'Item 3', 'Item 4', 'Item 5'],
  }),
})
```

**`ListItemContainerProperty` fields:**

| Property | Type | Range | Notes |
|---|---|---|---|
| `itemCount` | number | 1–20 | Must match `itemName.length` |
| `itemWidth` | number | pixels | Width of each item row. `0` = auto-fill (firmware calculates width). Other values = fixed width in pixels. Usually `containerWidth - 2*padding` |
| `isItemSelectBorderEn` | number | 0 or 1 | Show selection highlight border on current item |
| `itemName` | string[] | max 64 chars each | The text label for each item |

**List behaviour:**
- The firmware handles scrolling natively – scroll gestures move the selection highlight on the device without any app intervention. No `rebuildPageContainer` needed for scrolling through items
- `SCROLL_TOP_EVENT` and `SCROLL_BOTTOM_EVENT` are emitted only when the user reaches the top or bottom boundary of the list (boundary events, not every scroll gesture)
- Click events report `currentSelectItemIndex` and `currentSelectItemName` via `listEvent`
- No custom styling per item (no per-item colours, icons, or secondary text)
- No item height control – the firmware calculates item height from `containerHeight / itemCount`
- Items are plain text only, single line per item
- Cannot update list items in-place – must `rebuildPageContainer` to change items
- No separator lines between items (border is around the whole list)

**Gotcha:** List containers take over scroll handling. If you have a list on screen, scroll events arrive as `listEvent` (not `textEvent`/`sysEvent`), and the device moves the selection highlight automatically. You only need to respond to click/double-click.

### Image containers (`ImageContainerProperty`)

Displays greyscale images on the glasses. The host app converts all image data to 4-bit greyscale (16 shades of green) before sending to the device.

```typescript
new ImageContainerProperty({
  xPosition: 200,
  yPosition: 100,
  width: 100,
  height: 50,
  containerID: 3,
  containerName: 'logo',
})
```

| Constraint | Value |
|---|---|
| Width | 20–200 px |
| Height | 20–100 px |
| Colour | Converted to 4-bit greyscale (`imageToGray4`) by the host |
| Data formats | `number[]`, `Uint8Array`, `ArrayBuffer`, or base64 string |
| Concurrent sends | **Not allowed** – queue image updates sequentially |
| Startup phase | Cannot send image data during `createStartUpPageContainer` – create an empty placeholder, then update via `updateImageRawData` |
| Tiling | If image data is smaller than the container dimensions, the hardware tiles (repeats) it – always match image size to container size |

**`ImageRawDataUpdate` fields:**
- `containerID` – must match the image container
- `containerName` – must match the image container
- `imageData` – image data as `number[]`, `Uint8Array`, `ArrayBuffer`, or base64 string

**`imageData` format:**

The SDK accepts multiple formats and the host app handles decoding:
- **`number[]`** (recommended) – can be either PNG file bytes (`Array.from(new Uint8Array(pngBuffer))`) or raw greyscale pixel values (one byte per pixel)
- **base64 `string`** – base64-encoded PNG (e.g. from `canvas.toDataURL('image/png')` with the `data:image/png;base64,` prefix stripped)
- **`Uint8Array`** or **`ArrayBuffer`** – the SDK auto-converts these to `number[]` during serialisation

The host app's `imageToGray4` process handles the final conversion to 4-bit greyscale regardless of input format.

**Advanced fields** (in `ImageRawDataUpdateFields`, for fragmented transfers):
- `mapSessionId`, `mapTotalSize`, `compressMode`, `mapFragmentIndex`, `mapFragmentPacketSize`, `mapRawData`

**Image processing advice:**
- Send greyscale images – colour information is discarded by the host's 4-bit conversion
- Do not perform 1-bit dithering in your app – the host does better 4-bit downsampling; manual Floyd-Steinberg dithering creates noisy green dots on the display
- Match image dimensions to container dimensions to prevent tiling
- For best results: resize to fit container (preserve aspect ratio), centre on a black canvas (black = off on micro-LED), convert to greyscale (BT.601 luminance: `0.299R + 0.587G + 0.114B`), encode as PNG, send as `number[]` or base64
- Keep images simple – the 4-bit conversion (16 levels) means detailed images lose quality
- Glasses memory is limited – avoid frequent image updates

## Input events

### Event types (`OsEventTypeList`)

| Event | Value | Source |
|---|---|---|
| `CLICK_EVENT` | 0 | Ring tap, temple tap |
| `SCROLL_TOP_EVENT` | 1 | Emitted when internal scroll reaches the top boundary |
| `SCROLL_BOTTOM_EVENT` | 2 | Emitted when internal scroll reaches the bottom boundary |
| `DOUBLE_CLICK_EVENT` | 3 | Ring double-tap, temple double-tap |
| `FOREGROUND_ENTER_EVENT` | 4 | App comes to foreground |
| `FOREGROUND_EXIT_EVENT` | 5 | App goes to background |
| `ABNORMAL_EXIT_EVENT` | 6 | Unexpected disconnect |

### Event delivery

Events arrive via `bridge.onEvenHubEvent(callback)`. The callback receives an `EvenHubEvent` object with one of these populated:

```typescript
type EvenHubEvent = {
  listEvent?: List_ItemEvent    // from list containers
  textEvent?: Text_ItemEvent    // from text containers
  sysEvent?: Sys_ItemEvent      // system-level events
  audioEvent?: AudioEventPayload // microphone PCM
  jsonData?: Record<string, any> // raw payload for debugging
}
```

**`List_ItemEvent` fields:**
- `containerID` – which list container
- `containerName` – which list container
- `currentSelectItemName` – text of the selected item
- `currentSelectItemIndex` – 0-based index of selected item
- `eventType` – `OsEventTypeList` value

**`Text_ItemEvent` fields:**
- `containerID`, `containerName`, `eventType`

**`Sys_ItemEvent` fields:**
- `eventType` only

### Event quirks (critical for real-world apps)

1. **`CLICK_EVENT = 0` becomes `undefined`:** The SDK's `fromJson` normalises `0` to `undefined` in many cases. Always check `eventType === OsEventTypeList.CLICK_EVENT || eventType === undefined` to catch clicks.

2. **Missing `currentSelectItemIndex`:** The simulator (and sometimes real hardware) omits `currentSelectItemIndex` for the first list item (index 0). Fall back to tracking `selectedIndex` in your app state.

3. **Event routing depends on `isEventCapture`:** The container with `isEventCapture: 1` determines whether you get `listEvent` vs `textEvent`. If a list has capture, scroll gestures are handled natively by the firmware and boundary events come as `listEvent`. If text has capture, the firmware scrolls the text internally and boundary events come as `textEvent`.

4. **Simulator vs real device:** The simulator sends `sysEvent` for button clicks, while real hardware sends `textEvent` or `listEvent` depending on the active container. Handle all three event sources.

5. **Swipe throttling:** Scroll events can fire rapidly. Use a cooldown (e.g. 300ms) to prevent duplicate actions.

## Page lifecycle

### `createStartUpPageContainer`

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

### `rebuildPageContainer`

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

### `textContainerUpgrade`

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

### `updateImageRawData`

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

### `shutDownPageContainer`

Exits the app.

```typescript
await bridge.shutDownPageContainer(0) // 0 = immediate exit
await bridge.shutDownPageContainer(1) // 1 = show exit confirmation to user
```

### `callEvenApp` (generic escape hatch)

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

## Audio

- `bridge.audioControl(true/false)` – open/close microphone
- Requires `createStartUpPageContainer` to be called first
- PCM data arrives via `onEvenHubEvent` as `audioEvent.audioPcm` (`Uint8Array`)
- PCM format: 16kHz sample rate, 10ms frame length (dtUs 10000), 40 bytes per frame, PCM S16LE (signed 16-bit little-endian), mono

## Device info

```typescript
const device = await bridge.getDeviceInfo()
// device.model – DeviceModel.G1 | DeviceModel.G2 | DeviceModel.Ring1
// device.sn – serial number (string)
// device.status.connectType – DeviceConnectType (none/connecting/connected/disconnected/connectionFailed)
// device.status.batteryLevel – 0-100
// device.status.isWearing – boolean
// device.status.isCharging – boolean
// device.status.isInCase – boolean
```

Real-time monitoring via `bridge.onDeviceStatusChanged(callback)`. Returns an unsubscribe function.

**`DeviceInfo.updateStatus(status)`** – Updates the device's status in-place, but only when `status.sn === device.sn` (the serial numbers must match; mismatched updates are silently ignored).

Helper methods on `DeviceInfo`: `isGlasses()`, `isRing()`.
Helper methods on `DeviceStatus`: `isConnected()`, `isConnecting()`, `isDisconnected()`, `isConnectionFailed()`, `isNone()`.

Serialisation helpers on all models: `toJson()`, `fromJson(json)` (static), `createDefault()` (static, on `UserInfo` and `DeviceStatus`).

## User info

```typescript
const user = await bridge.getUserInfo()
// user.uid – number
// user.name – string
// user.avatar – string (URL)
// user.country – string
```

## Local storage

Key-value storage persisted on the phone side:

```typescript
await bridge.setLocalStorage('key', 'value') // returns boolean
const value = await bridge.getLocalStorage('key') // returns string
```

## What the SDK does NOT expose

- No direct BLE access
- No arbitrary pixel drawing – limited to list/text/image container model
- No `imgEvent` (defined in protocol but not in SDK types)
- No audio output (no speaker on hardware)
- No text alignment (no centre, right-align)
- No font size, weight, or family control
- No background colour or fill on containers
- No per-item styling in lists
- No programmatic scroll position control – the firmware handles internal scrolling, but there is no API to get or set the scroll offset
- No animations or transitions
- Image containers are greyscale only (4-bit, 16 levels)

## Error codes

### Page creation (`StartUpPageCreateResult`)

| Code | Meaning |
|---|---|
| 0 | Success |
| 1 | Invalid container configuration |
| 2 | Oversize – data too large for BLE transfer |
| 3 | Out of memory on glasses |

### Page rebuild

Returns `boolean`. The SDK has `APP_REQUEST_REBUILD_PAGE_SUCCESS` and `APP_REQUEST_REBUILD_PAGE_FAILD` (sic) error codes internally.

### Text upgrade

Returns `boolean`. Internal codes: `APP_REQUEST_UPGRADE_TEXT_DATA_SUCCESS` / `APP_REQUEST_UPGRADE_TEXT_DATA_FAILED`.

### Image update (`ImageRawDataUpdateResult`)

| Code | Meaning |
|---|---|
| success | OK |
| imageException | Image processing error |
| imageSizeInvalid | Image dimensions don't match container or are out of range |
| imageToGray4Failed | Greyscale conversion failed |
| sendFailed | BLE send failed |

### Shutdown

Returns `boolean`. Internal codes: `APP_REQUEST_UPGRADE_SHUTDOWN_SUCCESS` / `APP_REQUEST_UPGRADE_SHUTDOWN_FAILED`.

## SDK JSON compatibility

The SDK handles multiple key naming conventions from the host:
- camelCase: `containerID`
- PascalCase: `ContainerID`
- Proto-style: `Container_ID`

All are accepted via `pickLoose()` which normalises keys by removing underscores and lowercasing. You only need to use camelCase in your code – the SDK handles the rest.

Event data is also accepted in multiple shapes:
- `data: { type: 'listEvent', jsonData: {...} }`
- `data: { type: 'list_event', data: {...} }`
- `data: [ 'list_event', {...} ]` (array format)
- `data: { type: 'audioEvent', jsonData: { audioPcm: [...] } }`

## UI patterns from real apps

### Fake "buttons" with text cursor

Since there are no button widgets, apps simulate interactive items by appending action labels to text content and tracking a cursor position with `>` prefix:

```
> Return
  Delete note
```

Scroll moves the `>` between items. Click triggers the selected action. Updated via `textContainerUpgrade` to avoid full page flicker.

### Selection highlight with text borders

Without lists, selection can be shown by toggling `borderWidth` (0 = unselected, 1–3 = selected) on individual text containers, each representing a "row".

### Multi-slot text layout

Multiple text containers act as rows (e.g. 3 containers at `height: 96` each = 288px total). Each container shows one "item" with its own border state. This simulates a list without the native list widget's scroll-takeover.

### Progress bars with Unicode

Text containers can display Unicode progress bars:
```typescript
const filled = '━'.repeat(n)
const empty = '─'.repeat(total - n)
const bar = filled + empty
```

### Event capture for image-based apps

When your primary display is an image (Canvas-rendered UI, games, etc.), you need a separate container with `isEventCapture: 1` to receive input events since image containers don't have this property.

**Use a text container, not a list.** A hidden 1×1 list with 1 item does NOT generate scroll events – there's nothing to scroll, so `SCROLL_TOP_EVENT` and `SCROLL_BOTTOM_EVENT` never fire. Only click/double-click events come through.

Place a full-screen text container (content: `' '`) **behind** the image. The text container receives all event types including scroll. The image container renders on top (higher container ID = drawn later), so the text is invisible behind it:

```typescript
const config = {
  containerTotalNum: 2,
  textObject: [
    new TextContainerProperty({
      containerID: 1,
      containerName: 'evt',
      content: ' ',
      xPosition: 0,
      yPosition: 0,
      width: 576,
      height: 288,
      isEventCapture: 1,
      paddingLength: 0,
    }),
  ],
  imageObject: [
    new ImageContainerProperty({
      containerID: 2,
      containerName: 'screen',
      xPosition: 188,
      yPosition: 94,
      width: 200,
      height: 100,
    }),
  ],
}
```

Note: image containers are limited to 200×100 px max, so they cannot cover the full 576×288 canvas. Centre the image container on screen (xPosition = (576−200)/2 = 188, yPosition = (288−100)/2 = 94) and use the text container's text or a black background to fill the rest.

This gives you click, double-click, scroll-top and scroll-bottom events while the image is visible. Events arrive as `textEvent` (not `listEvent`), so adjust your event handler accordingly.

### Page flipping for long text

Although text containers scroll internally for overflow content, many apps choose to pre-paginate text into ~400–500 char pages at word boundaries. This gives better UX: controlled page breaks, progress indicators, and avoidance of the 1000-char `rebuildPageContainer` content limit. Track `pageIndex`, rebuild page on `SCROLL_BOTTOM_EVENT`/`SCROLL_TOP_EVENT`. Show page indicator in header or footer container.

## Browser UI component library

The companion settings/config page that runs in the WebView (not the glasses display) can use **`@jappyjan/even-realities-ui`** – a React component library aligned to Even Realities design guidelines.

NPM: <https://www.npmjs.com/package/@jappyjan/even-realities-ui>

```bash
npm install @jappyjan/even-realities-ui
```

- **Peer dependencies:** React 19, ReactDOM 19
- **Styling:** Tailwind-based. Uses `clsx` + `tailwind-merge` internally. Import `@jappyjan/even-realities-ui/styles.css` once in your app.
- **License:** Proprietary
- **No public source repo or docs** – the package includes Storybook config but no hosted instance is available.

### Entry points

| Import path | Description |
|---|---|
| `@jappyjan/even-realities-ui` | Re-exports everything (components + icons + tokens + utils) |
| `@jappyjan/even-realities-ui/components` | UI components only |
| `@jappyjan/even-realities-ui/icons` | Icon components only |
| `@jappyjan/even-realities-ui/tokens` | Design tokens (currently empty) |
| `@jappyjan/even-realities-ui/styles.css` | CSS stylesheet |

### Components

| Component | Notes |
|---|---|
| `Button` | `variant`: `default`, `accent`, `primary`, `negative`. `size`: `sm`, `md`, `lg` |
| `IconButton` | Same variants/sizes as Button, wraps an icon child |
| `Card`, `CardHeader`, `CardContent`, `CardFooter` | Card layout |
| `Text` | Polymorphic (`as` prop). `variant`: `title-xl`, `title-lg`, `title-1`, `title-2`, `body-1`, `body-2`, `subtitle`, `detail` |
| `Input` | Standard input, `ForwardRef<HTMLInputElement>` |
| `Textarea` | Standard textarea |
| `Select` | Standard select |
| `Checkbox` | `label`, `description` props |
| `Radio` | `label`, `description` props |
| `Switch` | `label` prop |
| `Badge` | Span-based |
| `Chip` | `size`: `sm`, `lg`. `onRemove` callback |
| `Divider` | `<hr>` wrapper |

### Icons (90+)

All icons extend `IconBase` (SVG `ForwardRef` with `size` and `title` props). Notable categories:

- **Hardware:** `GlassesIcon`, `GlassesBatteryIcon`, `GlassesChargingIcon`, `CaseIcon`, `CaseBatteryIcon`, `BluetoothIcon`, `BluetoothDisconnectedIcon`, `WearDetectIcon`, `ScreenOffIcon`
- **Battery:** `BatteryFullIcon`, `Battery75PercentIcon`, `Battery50PercentIcon`, `BatteryLowIcon`, `BatteryDyingIcon`
- **Navigation:** `BackIcon`, `ChevronBackIcon`, `ChevronDrillDownIcon`, `ChevronDrillInIcon`, `HomeMenuIcon`, `EvenHubMenuIcon`
- **Actions:** `AddIcon`, `EditIcon`, `CopyIcon`, `TrashIcon`, `UndoIcon`, `RedoIcon`, `ShareIcon`, `PlayIcon`, `PauseIcon`
- **Features:** `TranslateIcon`, `TranscribeIcon`, `TelepromptIcon`, `QuickNoteIcon`, `NavigateIcon`, `CalendarIcon`, `WeatherIcon`, `EvenAiIcon`
- **Settings:** `SettingsIcon`, `BrightnessIcon`, `LanguagesIcon`, `InterfaceSettingsIcon`, `ThemeIcon`
- **General:** `InfoIcon`, `HintIcon`, `NotificationIcon`, `CrossIcon`, `CompleteIcon`, `DotIcon`

### Design tokens (CSS custom properties)

- **Backgrounds:** `--color-bc-1` through `--color-bc-4`, `--color-bc-accent`, `--color-bc-highlight`
- **Surfaces:** `--color-sc-*`
- **Text:** `--color-tc-1`, `--color-tc-2`, `--color-tc-accent`, `--color-tc-green`, `--color-tc-red`
- **Typography:** `--font-size-app-title-xl` through `--font-size-app-detail`
- **Spacing:** `--space-0`, `--space-8`, `--space-12`, `--space-16`, `--space-20`, `--space-24`, `--space-28`, `--space-32`
- **Layout:** `--layout-card-margin`, `--layout-page-padding`, `--layout-section-gap`
- **Borders/Radii:** `--radius-sm`, `--radius-md`

### Utility

- **`cn(...inputs): string`** – `clsx` + `tailwind-merge` for composing class names.

## Even-dev simulator environment

[even-dev](https://github.com/BxNxM/even-dev) is a shared development environment for building and testing G2 apps with the Even Hub Simulator. It handles app discovery, dependency installation, Vite dev server configuration, and simulator launch.

Repository: <https://github.com/BxNxM/even-dev>

### Running apps

Install and run:

```bash
git clone https://github.com/BxNxM/even-dev.git
cd even-dev
npm install
./start-even.sh
```

The launcher lists all available apps and prompts you to pick one. You can also select directly:

```bash
APP_NAME=demo ./start-even.sh
```

To run a local app by path without any configuration:

```bash
APP_PATH=../my-app ./start-even.sh
```

External apps can be registered in `apps.json` at the even-dev root (git URLs are auto-cloned on first run):

```json
{
  "chess": "https://github.com/dmyster145/EvenChess",
  "my-local-app": "../my-local-app"
}
```

### App structure

A G2 app is a regular web app. There is no special framework or required interface – just HTML, TypeScript, and the Even Hub SDK.

Minimal file structure:

```
my-app/
  index.html          <- entry point
  package.json        <- dependencies and scripts
  vite.config.ts      <- dev server config
  app.json            <- app metadata (for packaging)
  src/
    main.ts           <- app bootstrap
    styles.css        <- stylesheet
```

**`index.html`** – a standard HTML page that loads your app:

```html
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>My App</title>
  <link rel="stylesheet" href="/src/styles.css">
</head>
<body>
  <div id="app"></div>
  <script type="module" src="/src/main.ts"></script>
</body>
</html>
```

**`package.json`** – at minimum, the Even Hub SDK and Vite:

```json
{
  "name": "my-even-app",
  "scripts": {
    "dev": "vite --host 0.0.0.0 --port 5173",
    "build": "vite build"
  },
  "dependencies": {
    "@evenrealities/even_hub_sdk": "^0.0.7"
  },
  "devDependencies": {
    "typescript": "^5.9.3",
    "vite": "^7.3.1"
  }
}
```

**`vite.config.ts`** – keep it minimal:

```typescript
import { defineConfig } from 'vite'

export default defineConfig({
  server: {
    host: true,
    port: 5173,
  },
})
```

**`src/styles.css`** – import the even-realities-ui stylesheet for consistent browser UI, then add your own styles:

```css
@import "../node_modules/@jappyjan/even-realities-ui/dist/even-realities-ui.css";

body {
  font-family: system-ui, -apple-system, sans-serif;
  max-width: 480px;
  margin: 0 auto;
  padding: 16px;
}
```

**Tips:**

- **Use inline styles or plain CSS** – avoid CSS frameworks like Tailwind that require build plugins. Your app may be served through even-dev's Vite config, which won't have those plugins.
- **Keep it standalone** – your app should work with just `npm run dev`. Don't depend on even-dev's infrastructure.
- **If your app needs a backend server**, put it in a `server/` directory with its own `package.json`. Even-dev auto-detects and starts it alongside Vite.
- **Use `@jappyjan/even-realities-ui`** for the browser settings page – Card, Button, Input, Text components give a consistent look across apps. See [browser UI component library](#browser-ui-component-library) above.

**Reference apps:**

| App | Description | Source |
|-----|-------------|--------|
| [chess](https://github.com/dmyster145/EvenChess) | Full app with tests, linting, modular architecture | Complex reference |
| [reddit](https://github.com/fuutott/rdt-even-g2-rddit-client) | Clean app with `app.json` packaging, API proxy, evenhub-cli integration | Packaging reference |
| [weather](https://github.com/nickustinov/weather-even-g2) | Settings UI with even-realities-ui components, plain CSS | Simple reference |

### Even Hub CLI (`@evenrealities/evenhub-cli`)

The official CLI for Even Hub development and app management. Provides QR code generation, app packaging, and account login.

NPM: <https://www.npmjs.com/package/@evenrealities/evenhub-cli>

```bash
npm install -D @evenrealities/evenhub-cli
```

The package installs two binaries: `evenhub` and `eh` (alias). When installed locally (not globally), use via `npx evenhub` or npm scripts.

#### CLI commands

| Command | Description |
|---|---|
| `evenhub login` | Log in to your Even Hub developer account |
| `evenhub init` | Generate a starter `app.json` manifest in the current directory |
| `evenhub qr [options]` | Generate a QR code for sideloading via the Even App |
| `evenhub pack <json> <project>` | Package a built app into a `.ehpk` file for distribution |

#### `evenhub qr`

Generates a QR code to scan with the Even App on your phone, which loads your dev server URL on the glasses.

```bash
# Specify full URL directly (recommended)
npx evenhub qr --url "http://192.168.0.138:5173"

# Or build from parts – prompts for IP on first run, caches it
npx evenhub qr --http --port 5173

# With explicit IP (no prompt)
npx evenhub qr --http --ip 192.168.0.138 --port 5173

# Reset cached settings
npx evenhub qr --clear
```

| Option | Description |
|---|---|
| `--url <url>` | Full URL to encode (ignores other options) |
| `--ip <ip>` | IP address or hostname |
| `--port <port>` | Port number |
| `--http` / `--https` | Scheme (default: HTTPS) |
| `--path <path>` | URL path |
| `--external` | Open QR in external program instead of terminal |
| `--clear` | Clear cached scheme, IP, port and path |

**Tip:** Use your machine's local network IP (e.g. `192.168.x.x`), not `localhost` or `0.0.0.0` – the phone needs to reach your dev server over the network. Vite prints the network URL when starting with `--host 0.0.0.0`.

#### `evenhub pack`

Packages a built app into a `.ehpk` file for distribution via the Even Hub portal.

```bash
# Build first, then pack
npx vite build
npx evenhub pack app.json dist -o myapp.ehpk

# Or combine in an npm script
# "pack": "npm run build && evenhub pack app.json dist -o myapp.ehpk"
```

| Option | Description |
|---|---|
| `<json>` | Path to `app.json` manifest (required) |
| `<project>` | Path to built output folder, e.g. `dist` (required) |
| `-o, --output <name>` | Output filename (default: `out.ehpk`) |
| `--no-ignore` | Include hidden files (starting with `.`) |
| `-c, --check` | Check if the package ID is available on Even Hub |

#### `evenhub login`

Authenticates with your Even Hub developer account (required for publishing).

```bash
npx evenhub login
# Prompts for email and password
```

#### `evenhub init`

Generates a starter `app.json` manifest file.

```bash
npx evenhub init
# Creates ./app.json with template values
```

### Packaging and deployment

#### `app.json` manifest

Every Even Hub app needs an `app.json` at the project root. This describes the app for packaging and the Even Hub portal.

```json
{
  "package_id": "com.example.myapp",
  "edition": "202601",
  "name": "My app",
  "version": "1.0.0",
  "min_app_version": "0.1.0",
  "tagline": "Short description shown in Even Hub",
  "description": "Longer description of what the app does",
  "author": "Your Name",
  "entrypoint": "index.html",
  "permissions": {
    "network": ["api.example.com"],
    "fs": ["./assets"]
  }
}
```

**`package_id` rules:** Must be a valid reverse-domain name where each segment starts with a lowercase letter and contains only lowercase letters or numbers. No hyphens. Example: `com.myname.myapp` (valid), `com.my-name.my-app` (invalid).

**`permissions.network`:** List domains your app needs to access. Use `["*"]` for unrestricted network access (e.g. apps that connect to user-configured servers).

#### Recommended npm scripts

```json
{
  "scripts": {
    "dev": "vite --host 0.0.0.0 --port 5173",
    "build": "vite build",
    "qr": "evenhub qr --http --port 5173",
    "pack": "npm run build && evenhub pack app.json dist -o myapp.ehpk"
  }
}
```

#### Development workflow

1. Start the dev server: `npm run dev` (keep running in terminal 1)
2. Generate QR code: `npm run qr` (terminal 2 – or use `npx evenhub qr --url "http://<your-ip>:5173"`)
3. Scan the QR code with the Even App on your phone
4. The app loads on your glasses – Vite hot reload works for code changes

#### Production packaging

1. `npm run pack` – builds and creates a `.ehpk` file

The `.ehpk` format is for future submission to the Even Hub portal (not yet available). Add `*.ehpk` to `.gitignore`.

## Example apps

| App | Description | Source |
|-----|-------------|--------|
| [tesla](https://github.com/nickustinov/tesla-even-g2) | Tesla vehicle status and controls | Image-based rendering, backend server |
| [weather](https://github.com/nickustinov/weather-even-g2) | Weather forecast display | Settings UI with even-realities-ui, plain CSS |
| [pong](https://github.com/nickustinov/pong-even-g2) | Pong game | Canvas-rendered game, image container |
| [snake](https://github.com/nickustinov/snake-even-g2) | Snake game | Canvas-rendered game, image container |
