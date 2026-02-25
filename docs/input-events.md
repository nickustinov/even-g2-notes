# Input events

## Event types (`OsEventTypeList`)

| Event | Value | Source |
|---|---|---|
| `CLICK_EVENT` | 0 | Ring tap, temple tap |
| `SCROLL_TOP_EVENT` | 1 | Emitted when internal scroll reaches the top boundary |
| `SCROLL_BOTTOM_EVENT` | 2 | Emitted when internal scroll reaches the bottom boundary |
| `DOUBLE_CLICK_EVENT` | 3 | Ring double-tap, temple double-tap |
| `FOREGROUND_ENTER_EVENT` | 4 | App comes to foreground |
| `FOREGROUND_EXIT_EVENT` | 5 | App goes to background |
| `ABNORMAL_EXIT_EVENT` | 6 | Unexpected disconnect |

## Event delivery

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

## Event quirks (critical for real-world apps)

1. **`CLICK_EVENT = 0` becomes `undefined`:** The SDK's `fromJson` normalises `0` to `undefined` in many cases. Always check `eventType === OsEventTypeList.CLICK_EVENT || eventType === undefined` to catch clicks.

2. **Missing `currentSelectItemIndex`:** The simulator (and sometimes real hardware) omits `currentSelectItemIndex` for the first list item (index 0). Fall back to tracking `selectedIndex` in your app state.

3. **Event routing depends on `isEventCapture`:** The container with `isEventCapture: 1` determines whether you get `listEvent` vs `textEvent`. If a list has capture, scroll gestures are handled natively by the firmware and boundary events come as `listEvent`. If text has capture, the firmware scrolls the text internally and boundary events come as `textEvent`.

4. **Simulator vs real device:** The simulator sends `sysEvent` for button clicks, while real hardware sends `textEvent` or `listEvent` depending on the active container. Handle all three event sources.

5. **Swipe throttling:** Scroll events can fire rapidly. Use a cooldown (e.g. 300ms) to prevent duplicate actions.
