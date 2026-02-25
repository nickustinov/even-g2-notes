# Even Realities G2 – development reference

> **Disclaimer:** This document is the result of independent research and reverse-engineering of the public SDK. It is not official documentation, may contain inaccuracies, and is updated as new information becomes available. The author is not affiliated with Even Realities.

The G2 are smart glasses with dual micro-LED displays (one per lens), no camera, no speaker. Privacy-focused design. Paired with the R1 control ring for input. Connected to iPhone via BLE 5.x.

## Reference docs

- [Architecture](architecture.md): Overview, BLE connection model, two-way comms, SDK init, hardware specs
- [Display and UI](display.md): Canvas (576x288), container model, text/list/image containers, font and Unicode support
- [Input events](input-events.md): Touch, ring, head gestures, event delivery and quirks
- [Page lifecycle](page-lifecycle.md): createStartUpPageContainer, rebuild, textContainerUpgrade, updateImageRawData, shutdown, callEvenApp
- [Device APIs](device-apis.md): Audio, device info, user info, local storage, SDK limitations
- [Error codes](error-codes.md): Result codes for all operations, SDK JSON compatibility
- [UI patterns](ui-patterns.md): Fake buttons, selection highlights, progress bars, image-based apps, page flipping
- [Browser UI](browser-ui.md): @jappyjan/even-realities-ui React component library for WebView settings pages
- [Simulator](simulator.md): even-dev environment, running apps, app structure, backend servers
- [Packaging](packaging.md): Even Hub CLI, app.json manifest, QR sideloading, .ehpk packaging

## Example apps

| App | Description | Source |
|-----|-------------|--------|
| [chess](https://github.com/dmyster145/EvenChess) | Full app with tests, linting, modular architecture | Complex reference |
| [reddit](https://github.com/fuutott/rdt-even-g2-rddit-client) | Clean app with `app.json` packaging, API proxy, evenhub-cli integration | Packaging reference |
| [weather](https://github.com/nickustinov/weather-even-g2) | Settings UI with even-realities-ui components, plain CSS | Simple reference |
| [tesla](https://github.com/nickustinov/tesla-even-g2) | Tesla vehicle status and controls | Image-based rendering, backend server |
| [pong](https://github.com/nickustinov/pong-even-g2) | Pong game | Canvas-rendered game, image container |
| [snake](https://github.com/nickustinov/snake-even-g2) | Snake game | Canvas-rendered game, image container |
