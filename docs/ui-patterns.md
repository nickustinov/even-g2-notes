# UI patterns from real apps

## Fake "buttons" with text cursor

Since there are no button widgets, apps simulate interactive items by appending action labels to text content and tracking a cursor position with `>` prefix:

```
> Return
  Delete note
```

Scroll moves the `>` between items. Click triggers the selected action. Updated via `textContainerUpgrade` to avoid full page flicker.

## Selection highlight with text borders

Without lists, selection can be shown by toggling `borderWidth` (0 = unselected, 1–3 = selected) on individual text containers, each representing a "row".

## Multi-slot text layout

Multiple text containers act as rows (e.g. 3 containers at `height: 96` each = 288px total). Each container shows one "item" with its own border state. This simulates a list without the native list widget's scroll-takeover.

## Progress bars with Unicode

Text containers can display Unicode progress bars:
```typescript
const filled = '━'.repeat(n)
const empty = '─'.repeat(total - n)
const bar = filled + empty
```

## Event capture for image-based apps

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

## Page flipping for long text

Although text containers scroll internally for overflow content, many apps choose to pre-paginate text into ~400–500 char pages at word boundaries. This gives better UX: controlled page breaks, progress indicators, and avoidance of the 1000-char `rebuildPageContainer` content limit. Track `pageIndex`, rebuild page on `SCROLL_BOTTOM_EVENT`/`SCROLL_TOP_EVENT`. Show page indicator in header or footer container.
