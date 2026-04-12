# Display and UI system

## Canvas

576x288 pixels. Coordinate system: origin (0, 0) at top-left, X right, Y down. Green micro-LED display – all colours are converted to 4-bit greyscale (16 levels of green) by the host app before transmission. Border colour ranges vary slightly by container type (0–15 for list, 0–16 for text).

## Container model

The UI is built from **containers** – rectangular regions positioned absolutely on the canvas. There is no CSS, no flexbox, no DOM. You define containers with pixel coordinates, and the glasses firmware renders them.

**Rules:**
- Maximum **4 image containers** and **8 other containers** per page (12 total, mixed types allowed)
- Exactly **one** container must have `isEventCapture: 1` – this is the container that receives input events
- Container count is set via `containerTotalNum` and must match the actual number of containers passed
- Containers can overlap (later containers draw on top)
- No z-index control beyond declaration order

## Shared container properties

All container types share these layout properties:

| Property | Type | Range | Notes |
|---|---|---|---|
| `xPosition` | number | 0–576 | Left edge in pixels |
| `yPosition` | number | 0–288 | Top edge in pixels |
| `width` | number | 0–576 | 20–288 for images |
| `height` | number | 0–288 | 20–144 for images |
| `containerID` | number | any | Unique per page, used for updates |
| `containerName` | string | max 16 chars | Unique per page, used for updates |
| `isEventCapture` | number | 0 or 1 | Exactly one container must be 1 |

## Border and decoration

Available on list and text containers (not images):

| Property | Type | Range | Notes |
|---|---|---|---|
| `borderWidth` | number | 0–5 | 0 = no border |
| `borderColor` | number | 0–15 (list), 0–16 (text) | Greyscale level. Practical values: 5 is a subtle grey, 13 is brighter |
| `borderRadius` | number | 0–10 | Rounded corners |
| `paddingLength` | number | 0–32 | Uniform padding on all sides |

No background colour property. No fill colour. The only visual decoration is the border.

## Text containers (`TextContainerProperty`)

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

## Font and Unicode support

The glasses firmware uses a single LVGL font baked into the firmware. There is no font selection, no font size control, and the font is **not monospaced** – different characters have different widths. Characters outside the font are silently skipped (no placeholder glyph is shown).

Tested on the Even Hub Simulator (`@evenrealities/evenhub-simulator` v0.0.7, Feb 2025). Real hardware may differ.

### Monospace workaround with fullwidth characters

Since the font is proportional, grid-based games and aligned text layouts break with regular ASCII characters. The workaround is to use **CJK fullwidth characters**, which are all guaranteed to be the same width as each other by the Unicode standard (East Asian Width property `F` or `W`).

**Key characters:**

- **Ideographic space** `\u3000` – a fullwidth space, same width as other fullwidth characters. Use instead of regular space for grid alignment.
- **Fullwidth Latin letters** (`Ａ`–`Ｚ`, `ａ`–`ｚ`) – U+FF21–U+FF5A
- **Fullwidth digits** (`０`–`９`) – U+FF10–U+FF19
- **CJK ideographs** (日火水木金土 etc.) and **Bopomofo symbols** (ㄌㄛㄜ etc.)

**Converting ASCII to fullwidth:** add `0xFEE0` to any ASCII character code:

```typescript
// 'A' (U+0041) → 'Ａ' (U+FF21)
function toFullwidth(str: string): string {
  return str.replace(/[\x20-\x7E]/g, (ch) =>
    ch === ' ' ? '\u3000' : String.fromCharCode(ch.charCodeAt(0) + 0xFEE0)
  )
}
```

**Example character set** (from [eventrix](https://github.com/dmi3-dev/eventrix)):

```typescript
const MONO_SPACE = '\u3000'
const MONO_CHARS =
  'ㄌㄛㄜㄝㄞㄟㄠㄡㄢㄣㄤㄥㄦ个丫丬中丮丰丱串丳临〆丸' +
  '丹为々主丼丽乂乃久乇么义日火水木金土年月時分秒人大' +
  '小上下左右前後東西南北０１２３４５６７８９ＡＢＣＤ' +
  'ＥＦＧＨＩＪＫＬＭＮＯＰＱＲＳＴＵＶＷＸＹＺａｂｃ' +
  'ｄｅｆｇｈｉｊｋｌｍｎｏｐｑｒｓｔｕｖｗｘｙｚ＄＆'
```

**Practical usage for games:** for empty grid cells, use `\u3000` (ideographic space) instead of regular space or `□` (U+25A1). This ensures all cells are the same width regardless of whether they contain a game element or are empty. Note that non-fullwidth characters like `▦` (U+25A6) or `●` (U+25CF) may not be exactly the same width as fullwidth characters – but in practice the alignment is close enough for game grids on the G2 display.

**Scrollbar fix:** text containers show a scrollbar when content overflows. A common cause is a trailing `\n` after the last row of a grid – it creates an empty line that pushes content past the container height. Remove the trailing newline from the last row to fix this.

### Available glyphs

**ASCII and Latin (U+0020–U+00FF)** – nearly complete (all printable characters except five):

| Char | Code | Description |
|------|------|-------------|
| `!` `"` `#` … `~` | U+0021–U+007E | All printable ASCII |
| `¡` `¢` `£` … `ÿ` | U+00A1–U+00FF | Latin-1 Supplement (accented letters, symbols, etc.) |

**Arrows (U+2190–U+2199, U+21D2, U+21D4)**

| Char | Code | | Char | Code | | Char | Code |
|------|------|-|------|------|-|------|------|
| ← | U+2190 | | ↑ | U+2191 | | → | U+2192 |
| ↓ | U+2193 | | ↔ | U+2194 | | ↕ | U+2195 |
| ↖ | U+2196 | | ↗ | U+2197 | | ↘ | U+2198 |
| ↙ | U+2199 | | ⇒ | U+21D2 | | ⇔ | U+21D4 |

**Box Drawing (U+2500–U+2573)** – single-line and light/heavy characters:

| Char | Code | | Char | Code | | Char | Code | | Char | Code |
|------|------|-|------|------|-|------|------|-|------|------|
| ─ | U+2500 | | ━ | U+2501 | | │ | U+2502 | | ┃ | U+2503 |
| ┌ | U+250C | | ┍ | U+250D | | ┎ | U+250E | | ┏ | U+250F |
| ┐ | U+2510 | | ┑ | U+2511 | | ┒ | U+2512 | | ┓ | U+2513 |
| └ | U+2514 | | ┕ | U+2515 | | ┖ | U+2516 | | ┗ | U+2517 |
| ┘ | U+2518 | | ┙ | U+2519 | | ┚ | U+251A | | ┛ | U+251B |
| ├ | U+251C | | ┝ | U+251D | | ┞ | U+251E | | ┟ | U+251F |
| ┠ | U+2520 | | ┡ | U+2521 | | ┢ | U+2522 | | ┣ | U+2523 |
| ┤ | U+2524 | | ┥ | U+2525 | | ┦ | U+2526 | | ┧ | U+2527 |
| ┨ | U+2528 | | ┩ | U+2529 | | ┪ | U+252A | | ┫ | U+252B |
| ┬ | U+252C | | ┭ | U+252D | | ┮ | U+252E | | ┯ | U+252F |
| ┰ | U+2530 | | ┱ | U+2531 | | ┲ | U+2532 | | ┳ | U+2533 |
| ┴ | U+2534 | | ┵ | U+2535 | | ┶ | U+2536 | | ┷ | U+2537 |
| ┸ | U+2538 | | ┹ | U+2539 | | ┺ | U+253A | | ┻ | U+253B |
| ┼ | U+253C | | ┽ | U+253D | | ┾ | U+253E | | ┿ | U+253F |
| ╀ | U+2540 | | ╁ | U+2541 | | ╂ | U+2542 | | ╃ | U+2543 |
| ╄ | U+2544 | | ╅ | U+2545 | | ╆ | U+2546 | | ╇ | U+2547 |
| ╈ | U+2548 | | ╉ | U+2549 | | ╊ | U+254A | | ╋ | U+254B |
| ═ | U+2550 | | ╞ | U+255E | | ╡ | U+2561 | | ╪ | U+256A |
| ╭ | U+256D | | ╮ | U+256E | | ╯ | U+256F | | ╰ | U+2570 |
| ╱ | U+2571 | | ╲ | U+2572 | | ╳ | U+2573 | | | |

**Block Elements (U+2580–U+2595)** – lower blocks, left blocks, full block, and a few others:

| Char | Code | Name |
|------|------|------|
| ▁ | U+2581 | Lower one eighth block |
| ▂ | U+2582 | Lower one quarter block |
| ▃ | U+2583 | Lower three eighths block |
| ▄ | U+2584 | Lower half block |
| ▅ | U+2585 | Lower five eighths block |
| ▆ | U+2586 | Lower three quarters block |
| ▇ | U+2587 | Lower seven eighths block |
| █ | U+2588 | Full block |
| ▉ | U+2589 | Left seven eighths block |
| ▊ | U+258A | Left three quarters block |
| ▋ | U+258B | Left five eighths block |
| ▌ | U+258C | Left half block |
| ▍ | U+258D | Left three eighths block |
| ▎ | U+258E | Left one quarter block |
| ▏ | U+258F | Left one eighth block |
| ▒ | U+2592 | Medium shade |
| ▔ | U+2594 | Upper one eighth block |
| ▕ | U+2595 | Right one eighth block |

**Geometric Shapes (U+25A0–U+25EF)** – selective support:

| Char | Code | | Char | Code | | Char | Code | | Char | Code |
|------|------|-|------|------|-|------|------|-|------|------|
| ■ | U+25A0 | | □ | U+25A1 | | ▣ | U+25A3 | | ▤ | U+25A4 |
| ▥ | U+25A5 | | ▦ | U+25A6 | | ▧ | U+25A7 | | ▨ | U+25A8 |
| ▩ | U+25A9 | | ▲ | U+25B2 | | △ | U+25B3 | | ▶ | U+25B6 |
| ▷ | U+25B7 | | ▼ | U+25BC | | ▽ | U+25BD | | ◀ | U+25C0 |
| ◁ | U+25C1 | | ◆ | U+25C6 | | ◇ | U+25C7 | | ◈ | U+25C8 |
| ◊ | U+25CA | | ○ | U+25CB | | ◌ | U+25CC | | ◎ | U+25CE |
| ● | U+25CF | | ◐ | U+25D0 | | ◑ | U+25D1 | | ◢ | U+25E2 |
| ◣ | U+25E3 | | ◤ | U+25E4 | | ◥ | U+25E5 | | ◯ | U+25EF |

**Misc Symbols (U+2605–U+2667)** – thirteen characters:

| Char | Code | | Char | Code | | Char | Code |
|------|------|-|------|------|-|------|------|
| ★ | U+2605 | | ☆ | U+2606 | | ☉ | U+2609 |
| ☎ | U+260E | | ☏ | U+260F | | ☜ | U+261C |
| ☞ | U+261E | | ♠ | U+2660 | | ♡ | U+2661 |
| ♣ | U+2663 | | ♤ | U+2664 | | ♥ | U+2665 |
| ♧ | U+2667 | | | | | | |

**Typographic symbols**

| Char | Code | | Char | Code | | Char | Code |
|------|------|-|------|------|-|------|------|
| © | U+00A9 | | ® | U+00AE | | ™ | U+2122 |
| † | U+2020 | | ※ | U+203B | | ° | U+00B0 |
| ∞ | U+221E | | | | | | |

**Superscripts and subscripts**

| Char | Code | | Char | Code | | Char | Code | | Char | Code |
|------|------|-|------|------|-|------|------|-|------|------|
| ⁰ | U+2070 | | ¹ | U+00B9 | | ² | U+00B2 | | ³ | U+00B3 |
| ⁴ | U+2074 | | ⁵ | U+2075 | | ⁶ | U+2076 | | ⁷ | U+2077 |
| ⁸ | U+2078 | | ⁹ | U+2079 | | | | | | |
| ₀ | U+2080 | | ₁ | U+2081 | | ₂ | U+2082 | | ₃ | U+2083 |
| ₄ | U+2084 | | ₅ | U+2085 | | ₆ | U+2086 | | ₇ | U+2087 |
| ₈ | U+2088 | | ₉ | U+2089 | | | | | | |

**Fractions**

| Char | Code | | Char | Code | | Char | Code |
|------|------|-|------|------|-|------|------|
| ¼ | U+00BC | | ½ | U+00BD | | ⅛ | U+215B |

### Useful combinations for apps

Progress bars and UI:
- `━` U+2501, `─` U+2500 – thick/thin horizontal lines
- `█▇▆▅▄▃▂▁` U+2588–U+2581 – full block + lower fractional blocks (8 levels)
- `▉▊▋▌▍▎▏` U+2589–U+258F – left fractional blocks (7 levels)
- `▒` U+2592 – medium shade (empty fill)

Games and diagrams:
- `●○` – filled/empty circles; `■□` – filled/empty squares
- `▲△▶▷▼▽◀◁` – directional triangles
- `◢◣◤◥` – corner triangles
- `─│┌┐└┘├┤┬┴┼` – box drawing
- `╭╮╯╰` – rounded corners
- `★☆` – stars
- `♠♤` `♥♡` `♣♧` – card suits (filled/empty)

### Missing glyphs

**ASCII/Latin (U+00A0–U+00FF)** – five characters missing:

| Char | Code | Name |
|------|------|------|
| ¨ | U+00A8 | Diaeresis |
| ¯ | U+00AF | Macron |
| ´ | U+00B4 | Acute accent |
| µ | U+00B5 | Micro sign |
| ¸ | U+00B8 | Cedilla |

**Arrows** – everything outside the 12 listed above is missing (U+219A–U+21D1, U+21D3, U+21D5–U+21FF), including most double arrows, dashed arrows, and wavy arrows.

**Box Drawing** – missing subsets:
- U+2504–U+250B – dashed/dotted lines (┄ ┅ ┆ ┇ ┈ ┉ ┊ ┋)
- U+254C–U+254F – dash variants (╌ ╍ ╎ ╏)
- U+2551–U+255D (except U+2550) – most double-line characters
- U+255F–U+2560, U+2562–U+2569, U+256B–U+256C – double-line intersections
- U+2574–U+257F – half-line segments (╴ ╵ ╶ ╷ ╸ ╹ ╺ ╻ ╼ ╽ ╾ ╿)

**Block Elements** – missing from the range:

| Char | Code | Name |
|------|------|------|
| ▀ | U+2580 | Upper half block |
| ▐ | U+2590 | Right half block |
| ░ | U+2591 | Light shade |
| ▓ | U+2593 | Dark shade |
| ▖–▟ | U+2596–U+259F | All quadrant characters |

**Geometric Shapes** – missing: small squares (U+25AA–U+25AB), rectangles, small triangles, circle segments, and everything from U+25F0 onward.

**Misc Symbols** – missing: ☀ ☁ ☂ ☃ (weather), ☠ ☢ ☣ (hazard), ☮ ☯ (peace/yin-yang), ☺ ☻ (faces), and most other symbols in U+2600–U+263F.

**Entirely absent ranges:**
- Dingbats (U+2700–U+273F)
- Emoji (U+1F300+)
- Misc weather symbols (U+26C4–U+26C8 – ⛄ ⛅ etc.)
- Snowflake (U+2744 ❄), droplet (U+1F4A7 💧)

## List containers (`ListContainerProperty`)

Native scrollable list widget rendered by the glasses firmware. The glasses handle scroll highlighting natively – you don't need to manually track selection for scroll events.

```typescript
new ListContainerProperty({
  xPosition: 0,
  yPosition: 0,
  width: 576,
  height: 288,
  borderWidth: 1,
  borderColor: 13,
  borderRadius: 6,
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

## Image containers (`ImageContainerProperty`)

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
| Width | 20–288 px |
| Height | 20–144 px |
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
