# Browser UI component library

The companion settings/config page that runs in the WebView (not the glasses display) needs a UI component library. Use **[even-toolkit](https://github.com/fabioglimb/even-toolkit)** – a design system and component library built specifically for Even Realities G2 apps. It includes 55+ React components, 191 pixel-art icons, design tokens, and a glasses SDK bridge.

NPM: <https://www.npmjs.com/package/even-toolkit>
Demo: <https://even-demo.vercel.app>
License: MIT

## Setup

```bash
npm install even-toolkit
```

Import the theme CSS in your app entry point:

```css
@import "even-toolkit/web/theme-light.css";
@import "even-toolkit/web/typography.css";
@import "even-toolkit/web/utilities.css";
```

A dark theme is also available via `even-toolkit/web/theme-dark.css`.

## Entry points

### Web components

```typescript
import { Button, Card, NavBar, ListItem, Toggle, AppShell } from 'even-toolkit/web';
```

Deep imports are also available for individual components (e.g. `even-toolkit/web/button`).

### Icons

```typescript
import { IcChevronBack, IcTrash, IcSettings } from 'even-toolkit/web/icons/svg-icons';
```

The icon registry (`even-toolkit/web/icons`) provides `Icon`, `registerIcon`, `registerIcons`, `registerAllIcons`, and `getIconNames` for name-based icon rendering.

### Glasses SDK bridge

```typescript
import { useGlasses } from 'even-toolkit/useGlasses';
import { line, separator } from 'even-toolkit/types';
import { buildActionBar } from 'even-toolkit/action-bar';
```

All glasses entry points:

| Import path | Exports |
|---|---|
| `even-toolkit` | Re-exports everything from types, action-bar, text-utils, timer-display, gestures, text-clean, paginate-text |
| `even-toolkit/types` | `LineStyle`, `DisplayLine`, `DisplayData`, `line()`, `separator()`, `ColumnData`, `ImageTileData`, `PageMode`, `GlassActionType`, `GlassAction`, `GlassNavState` |
| `even-toolkit/bridge` | `EvenHubBridge` class, `ColumnConfig` type |
| `even-toolkit/useGlasses` | `useGlasses<S>()` hook, `UseGlassesConfig<S>` type |
| `even-toolkit/useFlashPhase` | `useFlashPhase(active)` hook |
| `even-toolkit/action-bar` | `buildActionBar()`, `buildStaticActionBar()` |
| `even-toolkit/action-map` | `mapGlassEvent(event)` – maps `EvenHubEvent` to `GlassAction \| null` |
| `even-toolkit/gestures` | `tryConsumeTap()`, `isScrollSuppressed()`, `notifyTextUpdate()`, `isScrollDebounced()` |
| `even-toolkit/keyboard` | `bindKeyboard(dispatch)` – returns cleanup function |
| `even-toolkit/timer-display` | `TimerState`, `renderTimerLines()`, `renderTimerCompact()` |
| `even-toolkit/text-utils` | `truncate()`, `SCROLL_UP`, `SCROLL_DOWN`, `buildHeaderLine()`, `applyScrollIndicators()` |
| `even-toolkit/layout` | `DISPLAY_W` (576), `DISPLAY_H` (288), tile/slot/container constants, `dummySlot()` |
| `even-toolkit/canvas-renderer` | `getCanvas()`, `drawToCanvas()`, `renderToImage()` |
| `even-toolkit/composer` | `composeStartupPage()`, `composeRebuildPage()` |
| `even-toolkit/png-utils` | `encodeTilesBatch()`, `resetTileCache()`, `canvasToPngBytes()` |
| `even-toolkit/keep-alive` | `activateKeepAlive()`, `deactivateKeepAlive()` |
| `even-toolkit/splash` | `createSplash()`, `TILE_PRESETS`, `SplashConfig`, `SplashHandle` |
| `even-toolkit/text-clean` | `cleanForG2()`, `normalizeWhitespace()` |
| `even-toolkit/paginate-text` | `wordWrap()`, `paginateText()`, `pageIndicator()` |

### CSS

| Import path | Purpose |
|---|---|
| `even-toolkit/web/theme-light.css` | Light theme design tokens |
| `even-toolkit/web/theme-dark.css` | Dark theme design tokens |
| `even-toolkit/web/typography.css` | FK Grotesk Neue text style classes |
| `even-toolkit/web/utilities.css` | Scrollbar-hide, animations |

## Components

### Primitives

`Button`, `Card`, `Badge`, `Input`, `Textarea`, `Select`, `Checkbox`, `RadioGroup`, `Slider`, `InputGroup`, `Skeleton`, `Progress`, `StatusDot`, `Pill`, `Toggle`, `SegmentedControl`, `Table` (with `TableHeader`, `TableBody`, `TableRow`, `TableHead`, `TableCell`), `Kbd`, `Divider`

### Layout

`AppShell`, `Page`, `NavBar`, `NavHeader`, `ScreenHeader`, `SectionHeader`, `SettingsGroup`, `CategoryFilter`, `ListItem` (swipe-to-delete), `SearchBar`, `Tag`, `TagCarousel`, `TagCard`, `SliderIndicator`, `PageIndicator`, `StepIndicator`, `Timeline`, `StatGrid`, `StatusProgress`

### Feedback

`TimerRing`, `Dialog`, `ConfirmDialog`, `Toast`, `EmptyState`, `Loading`, `BottomSheet`, `CTAGroup`, `ScrollPicker`, `DatePicker`, `TimePicker`, `SelectionPicker`

### Charts

`Sparkline`, `LineChart`, `BarChart`, `PieChart`, `StatCard` (via recharts)

### Media

`ChatContainer`, `ChatBubble`, `ChatInput`, `ChatThinking`, `ChatCodeBlock`, `ChatDiff`, `ChatToolCall`, `ChatCommand`, `ChatError`, `Calendar`, `FileUpload`, `VoiceInput`, `WaveformVisualizer`, `ImageGrid`, `ImageViewer`, `AudioPlayer`

## Icons (191)

Pixel-art icons organised into seven categories. Each icon has a PascalCase React component (e.g. `IcEditTrash`, `IcFeatCalendar`, `IcStatusBluetooth`). Eleven convenience aliases are also exported: `IcTrash`, `IcChevronBack`, `IcSettings`, `IcSearch`, `IcPlus`, `IcCross`, `IcEdit`, `IcShare`, `IcCopy`, `IcCheck`, `IcMore`.

| Category | Count |
|---|---|
| Edit & Settings | 32 |
| Features | 40 |
| Guide | 20 |
| Health | 12 |
| Menu | 8 |
| Navigation | 23 |
| Status | 56 |

## Design tokens (CSS custom properties)

Light theme values shown; dark theme defines the same properties with dark-appropriate values.

| Token | Light value | Usage |
|---|---|---|
| `--color-text` | #232323 | Primary text |
| `--color-text-dim` | #7B7B7B | Secondary text |
| `--color-text-muted` | #7B7B7B | Alias for dim |
| `--color-text-highlight` | #FFFFFF | Highlighted text |
| `--color-bg` | #EEEEEE | Page background |
| `--color-surface` | #FFFFFF | Card/container surface |
| `--color-surface-light` | #F6F6F6 | Secondary surface |
| `--color-surface-lighter` | #E4E4E4 | Tertiary surface |
| `--color-border` | #E4E4E4 | Primary border |
| `--color-border-light` | #EEEEEE | Subtle border |
| `--color-accent` | #232323 | Primary accent |
| `--color-accent-alpha` | rgba(35,35,35,0.08) | Accent with alpha |
| `--color-accent-warning` | #FEF991 | Warning accent |
| `--color-positive` | #4BB956 | Success/connected |
| `--color-positive-alpha` | (with alpha) | Success with alpha |
| `--color-negative` | #FF453A | Error/warning |
| `--color-negative-alpha` | (with alpha) | Error with alpha |
| `--color-overlay` | | Modal overlay |
| `--color-input-bg` | | Input background |
| `--radius-default` | 6px | Border radius |
| `--spacing-margin` | 12px | Outer margin |
| `--spacing-card-margin` | 16px | Card margin |
| `--spacing-same` | 6px | Same-group spacing |
| `--spacing-cross` | 12px | Cross-group spacing |
| `--spacing-section` | 24px | Section spacing |
| `--font-display` | FK Grotesk Neue, -apple-system, ... | Display/body font |
| `--font-mono` | SF Mono, Cascadia Code, ... | Monospace font |

## Typography

CSS classes from `typography.css`:

| Class | Size | Weight | Tracking |
|---|---|---|---|
| `.text-vlarge-title` | 24px | 400 | -0.72px |
| `.text-large-title` | 20px | 400 | -0.6px |
| `.text-medium-title` | 17px | 400 | -0.17px |
| `.text-medium-body` | 17px | 300 | -0.17px |
| `.text-normal-title` | 15px | 400 | -0.15px |
| `.text-normal-body` | 15px | 300 | -0.15px |
| `.text-subtitle` | 13px | 400 | -0.13px |
| `.text-detail` | 11px | 400 | -0.11px |
