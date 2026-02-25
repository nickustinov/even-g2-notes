# Browser UI component library

The companion settings/config page that runs in the WebView (not the glasses display) can use **`@jappyjan/even-realities-ui`** – a React component library aligned to Even Realities design guidelines.

NPM: <https://www.npmjs.com/package/@jappyjan/even-realities-ui>

```bash
npm install @jappyjan/even-realities-ui
```

- **Peer dependencies:** React 19, ReactDOM 19
- **Styling:** Tailwind-based. Uses `clsx` + `tailwind-merge` internally. Import `@jappyjan/even-realities-ui/styles.css` once in your app.
- **License:** Proprietary
- **No public source repo or docs** – the package includes Storybook config but no hosted instance is available.

## Entry points

| Import path | Description |
|---|---|
| `@jappyjan/even-realities-ui` | Re-exports everything (components + icons + tokens + utils) |
| `@jappyjan/even-realities-ui/components` | UI components only |
| `@jappyjan/even-realities-ui/icons` | Icon components only |
| `@jappyjan/even-realities-ui/tokens` | Design tokens (currently empty) |
| `@jappyjan/even-realities-ui/styles.css` | CSS stylesheet |

## Components

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

## Icons (90+)

All icons extend `IconBase` (SVG `ForwardRef` with `size` and `title` props). Notable categories:

- **Hardware:** `GlassesIcon`, `GlassesBatteryIcon`, `GlassesChargingIcon`, `CaseIcon`, `CaseBatteryIcon`, `BluetoothIcon`, `BluetoothDisconnectedIcon`, `WearDetectIcon`, `ScreenOffIcon`
- **Battery:** `BatteryFullIcon`, `Battery75PercentIcon`, `Battery50PercentIcon`, `BatteryLowIcon`, `BatteryDyingIcon`
- **Navigation:** `BackIcon`, `ChevronBackIcon`, `ChevronDrillDownIcon`, `ChevronDrillInIcon`, `HomeMenuIcon`, `EvenHubMenuIcon`
- **Actions:** `AddIcon`, `EditIcon`, `CopyIcon`, `TrashIcon`, `UndoIcon`, `RedoIcon`, `ShareIcon`, `PlayIcon`, `PauseIcon`
- **Features:** `TranslateIcon`, `TranscribeIcon`, `TelepromptIcon`, `QuickNoteIcon`, `NavigateIcon`, `CalendarIcon`, `WeatherIcon`, `EvenAiIcon`
- **Settings:** `SettingsIcon`, `BrightnessIcon`, `LanguagesIcon`, `InterfaceSettingsIcon`, `ThemeIcon`
- **General:** `InfoIcon`, `HintIcon`, `NotificationIcon`, `CrossIcon`, `CompleteIcon`, `DotIcon`

## Design tokens (CSS custom properties)

- **Backgrounds:** `--color-bc-1` through `--color-bc-4`, `--color-bc-accent`, `--color-bc-highlight`
- **Surfaces:** `--color-sc-*`
- **Text:** `--color-tc-1`, `--color-tc-2`, `--color-tc-accent`, `--color-tc-green`, `--color-tc-red`
- **Typography:** `--font-size-app-title-xl` through `--font-size-app-detail`
- **Spacing:** `--space-0`, `--space-8`, `--space-12`, `--space-16`, `--space-20`, `--space-24`, `--space-28`, `--space-32`
- **Layout:** `--layout-card-margin`, `--layout-page-padding`, `--layout-section-gap`
- **Borders/Radii:** `--radius-sm`, `--radius-md`

## Utility

- **`cn(...inputs): string`** – `clsx` + `tailwind-merge` for composing class names.
