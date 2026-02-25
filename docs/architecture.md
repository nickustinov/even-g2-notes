# Architecture

## Overview

The G2 are smart glasses with dual micro-LED displays (one per lens), no camera, no speaker. Privacy-focused design. Paired with the R1 control ring for input. Connected to iPhone via BLE 5.x (~28m range).

## Connection model

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
