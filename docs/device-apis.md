# Device APIs

## Audio

- `bridge.audioControl(true/false, source?)` – open/close microphone
- `source` (SDK 0.0.11+) is an `AudioInputSource`: `Glasses` (`'glasses'`) or `Phone` (`'phone'`). Omit for the default.
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

## Location (SDK 0.0.11+)

Reads the **phone's** location — the glasses have no GPS. Requires the
`location` permission in `app.json`.

```typescript
import { AppLocationAccuracy } from '@evenrealities/even_hub_sdk'

// One-shot fix. Returns null if denied, timed out, or no signal.
const fix = await bridge.getAppLocation({
  accuracy: AppLocationAccuracy.Medium,  // Low | Medium | High
  timeoutMs: 10_000,
})
// fix.latitude, fix.longitude
// fix.accuracy?, fix.altitude?, fix.speed?, fix.heading?, fix.timestamp?
```

Continuous updates, for the rare case that needs them:

```typescript
await bridge.startAppLocationUpdates({ accuracy, distanceFilter, intervalMs })
const off = bridge.onAppLocationChanged((location) => { /* ... */ })
await bridge.stopAppLocationUpdates()
```

**Prefer `getAppLocation` per launch over continuous updates.** Continuous
tracking keeps the GPS radio warm and drains the phone battery; most apps only
need a position when they open.

**A fix took ~3 s on device.** Do not await it before your first render — the
glasses stay blank for the whole time. Paint with a cached position and repaint
only if the new fix actually moved the user. Awaiting it inside the event
dispatcher is worse still: the busy guard drops every tap until the handler
settles. See [performance.md](performance.md).

`accuracy` must be the `AppLocationAccuracy` enum, not a bare string —
`accuracy: 'medium'` fails typecheck.

### Naming a fix

There is no reverse-geocoding in the SDK, and Open-Meteo (the usual forecast
source) has no reverse endpoint either. Turning coordinates into a place name
needs a third-party service, whitelisted in `app.json`.

Watch the granularity. BigDataCloud's `city` field is the *municipality* and its
`locality` is a *sub-city district*, so picking either alone is wrong somewhere:
`city` turns Corroios into Seixal, while `locality` turns London into "City of
Westminster", Lisbon into "Santo Antonio" and Paris into "Saint-Merri". The
admin hierarchy does not disambiguate — Corroios and Lisbon have structurally
identical trees, yet the intuitive answer sits at a different level in each.

What worked: take both candidate names, resolve them through Open-Meteo's
geocoder, and pick the **nearest result with a non-zero population**. The
population filter is what makes it work — parishes and quarters report 0 and can
sit fractionally nearer than the city containing them. This also names a
detected place identically to a searched one, since both come from the same
database.

## Album and camera (SDK 0.0.11+)

Reads from the **phone's** album and camera. The G2 has no camera. Require the
`album` and `camera` permissions respectively.

```typescript
const asset = await bridge.pickImageFromAlbum()      // single-select only
const shot  = await bridge.captureImageFromCamera()
// asset.path, asset.name, asset.mimeType, asset.size, asset.base64
```

Both return `AppImageAsset | null` (null if the user cancels). Use `.base64` in
the WebView.

## User info

```typescript
const user = await bridge.getUserInfo()
// user.uid – number
// user.name – string
// user.avatar – string (URL)
// user.country – string
```

## SDK storage

Key-value storage persisted on the phone side via the Even Hub bridge:

```typescript
await bridge.setLocalStorage('key', 'value') // returns boolean
const value = await bridge.getLocalStorage('key') // returns string
```

**This is the only persistent storage available.** Browser `localStorage` does **not** survive app restarts inside the `.ehpk` WebView – it is wiped when the Even Hub app or the glasses restart. Use `bridge.setLocalStorage` / `bridge.getLocalStorage` for anything that must persist across sessions (favourites, preferences, reading positions, etc.).

There is no `removeLocalStorage` method. To delete a key, write an empty string and treat empty strings as "not present" when reading.

### Recommended pattern: in-memory cache wrapper

The bridge storage calls are async, which makes them awkward for synchronous UI reads. The recommended pattern is to pre-load all keys into an in-memory `Map` at startup, then read from the cache synchronously and write through to the bridge in the background:

```typescript
const cache = new Map<string, string>()

// At startup, after bridge connects – before any UI renders
async function initStorage(bridge: EvenAppBridge, keys: string[]): Promise<void> {
  await Promise.all(keys.map(async (key) => {
    const value = await bridge.getLocalStorage(key)
    if (value) cache.set(key, value)
  }))
}

// Synchronous read from cache
function getItem(key: string): string | null {
  return cache.get(key) ?? null
}

// Write-through: update cache immediately, persist in background
function setItem(bridge: EvenAppBridge, key: string, value: string): void {
  cache.set(key, value)
  void bridge.setLocalStorage(key, value).catch(() => {})
}
```

This gives consumers a synchronous API while keeping the bridge as the source of truth across sessions.

## What the SDK does NOT expose

- No direct BLE access
- No camera or GPS **on the glasses** – `captureImageFromCamera` and `getAppLocation` read the phone's hardware, not the glasses'
- No reverse geocoding – coordinates to place name needs a third-party service
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
