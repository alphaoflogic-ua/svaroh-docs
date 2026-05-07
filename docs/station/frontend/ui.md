---
title: UI
sidebar_position: 3
---

# 🎨 UI Layer

UI primitives, semantic colors, responsive patterns, and the dashboard tile system.

## UI Kit Inventory (`shared/ui/`)

Reusable primitives. **Always check here before creating a new component.**

| Component | Purpose |
|---|---|
| `AnimatedOutlet` | React Router outlet with route transitions |
| `Button` | Primary / secondary / ghost variants |
| `Card` | Surface container with shadow + border |
| `CollapsibleSection` | Expandable section with chevron |
| `DataView` | Generic data list/grid wrapper |
| `Drawer` | Right slide on desktop, bottom sheet on mobile |
| `Input` | Text input with label and error states |
| `LanguageSelector` | i18next locale dropdown |
| `Logo` | Svaroh logo |
| `MiniSelect` | Compact select for inline use |
| `Modal` | Near-fullscreen on mobile, max-w-lg on desktop |
| `Pagination` | Page navigation |
| `RouteTabs` | Tab nav backed by router |
| `Select` | Standard select |
| `Slider` | Range slider |
| `StatusBadge` | Online/offline/error status pill |
| `StickyHeader` | Sticky positioned section header |
| `SvarohSpinner` | Branded loading spinner |
| `Switch` | On/off toggle |
| `ThemeToggle` | Light / dark mode switch |
| `TruncatedText` | Text with ellipsis + tooltip on overflow |
| `ViewModeToggle` | List / grid view selector |

[Source ↗](https://github.com/alphaoflogic-ua/smart-home/tree/develop/packages/frontend/src/shared/ui)

## Semantic Colors

Defined in [`colors.css` ↗](https://github.com/alphaoflogic-ua/smart-home/blob/develop/packages/frontend/src/colors.css) as CSS custom properties via Tailwind 4's `@theme {}`. Light/dark variants toggle on `.dark` class on root.

### Semantic Tokens

```css
--color-bg-base           /* page background */
--color-bg-surface        /* card/panel background */
--color-text-primary      /* main text */
--color-text-secondary    /* muted text */
--color-border-base       /* borders */
```

In Tailwind: `bg-bg-base`, `text-text-primary`, `border-border-base`.

### Named Colors

`text-honolulu-blue`, `bg-ghost-white`, `text-spanish-gray`, etc. — used as accent colors.

### Hard rule

**No hardcoded `#hex` or `rgb()`** — only semantic tokens or named colors. Apply via Tailwind classes. **No inline `style={{}}`** for color.

### `cn()` Helper

```typescript
import { cn } from '@/shared/ui/cn';

<div className={cn('px-4 py-2', isActive && 'bg-bg-surface text-text-primary')} />
```

`cn()` = `twMerge(clsx(inputs))` — handles conditional classes and merges conflicting Tailwind utilities.

## Responsive Patterns

Breakpoint: `md` (768px). Mobile-first — base styles are mobile, `md:` for desktop.

### Navigation

- **Desktop** (`md+`): sidebar (`hidden md:block`) — Dashboard, Home, Devices, collapsible "Advanced"
- **Mobile** (`< md`): bottom tab bar (`md:hidden`) — Dashboard, Devices, Home, More
- `/more` route — secondary nav (Automations, User Management, Settings, Station)

### Header

- **Desktop**: breadcrumbs left, avatar + WS status right
- **Mobile**: logo or back arrow left (via `getParentRoute()`), avatar + WS status right

### Layout Conventions

| Element | Mobile | Desktop (`md+`) |
|---|---|---|
| Page padding | `p-4 pb-20` (clears tab bar) | `p-6` |
| Card padding | `p-4` | `p-6` |
| Headings | `text-xl` | `text-2xl` |
| Buttons (with text) | icon-only + `aria-label` | icon + text |
| Touch targets | `min-h-[44px]` (Apple HIG) | default |
| Modal | `max-w-none` | `max-w-lg` |
| Drawer | bottom sheet | right slide |

Pattern for buttons:

```tsx
<Button>
  <PencilIcon />
  <span className="hidden md:inline">{t('common.edit')}</span>
</Button>
```

### Safe Areas

`viewport-fit=cover` in `<meta name="viewport">`, `.safe-bottom` class for `env(safe-area-inset-bottom)`.

## Dashboard Tile System {#dashboard}

The dashboard is the entry surface — a grid of tiles for the most-used devices and zones.

### Hierarchy

**Station → Zone → Tile + flat "Favorites" overlay.** Without favorites, the dashboard breaks past ~20 devices. Stations are singletons; zones group rooms; favorites are `is_favorite` per `dashboard_layout_item` (per-user, not per-device — different users can favorite different devices).

### Tile Sizes

Sizes are an **enum tied to capability**, not free resize:

| Size | Grid | Used for |
|---|---|---|
| `compact` | 1×1 | Binary toggle (switch, lock, motion sensor) |
| `wide` | 2×1 | Has value or slider (dimmer, thermostat, climate sensor) |
| `large` | 2×2 | Preview content (camera, energy chart, weather) |

The user picks "collapse / expand" from sizes valid for that device type. Free grid (e.g. `react-grid-layout`) gives flexibility but home users get lost in it.

### Layout Persistence

Layout is stored on the **backend, not localStorage** — otherwise sync between phone/tablet/web breaks.

```sql
CREATE TABLE dashboard_layouts (
  user_id     uuid PRIMARY KEY,
  items       jsonb,   -- [{deviceId, size, position, isFavorite}]
  updated_at  timestamptz
);
```

Read/written atomically as one JSON — no per-device normalization needed.

### Tile Interactions (3 gesture levels)

1. **Tap on main area** → primary action (toggle light, run scene, open camera live view). **Never** opens settings.
2. **Tap on inner control** (brightness slider, ±temperature) → secondary action without opening details.
3. **Long-press / tap on `⋯`** → detail Drawer (all params, history, settings, delete).

### Edit Mode

Edit mode is a **separate page state**, not always-visible drag handles. "Edit" button in header → tiles get drag handle in corner + slight wobble → "Done" saves layout in one PUT request.

Drag is implemented with `@dnd-kit/core` + `@dnd-kit/sortable`:

- Activation: distance ~8px OR delay 250ms — avoids tap conflicts
- While dragging: `scale: 1.05` + `shadow-lg` + `z-50`
- Other tiles **rearrange with FLIP animation** showing drop zones
- Auto-scroll near edges (built into dnd-kit)
- Drop outside valid zone → spring-back to origin

Edit mode is a **Redux state** in `dashboardSlice`, not local `useState` — F5 in edit mode shouldn't lose context.

### Reading State at a Glance

- **Background color** = on/off (`honolulu-blue` accent for active, neutral `bg-surface` for off)
- **Corner indicator**: offline (gray X), updating (spinner), error (red dot)
- **Big numeric value** (22°C, 65%, 1.2 kW) — main tile content
- **No "Status: ON" text** — admin pattern, not home UI

### Optimistic Updates

Mandatory. Tap → reducer mutates state immediately, command goes out via thunk. WS event `device_state_changed` confirms. If no confirmation in 3s OR error → roll back state + `toast.error`.

## Tile Registry

Capability → renderer mapping (similar idea to backend's device-type-registry, but for UI):

```typescript
// components/dashboard/tileRegistry.ts
type TileRenderer = {
  sizes: TileSize[];           // allowed sizes
  defaultSize: TileSize;
  Component: ComponentType<TileProps>;
};

const tileRegistry: Record<DeviceCapability, TileRenderer> = {
  switch: { sizes: ['compact', 'wide'], defaultSize: 'compact', Component: SwitchTile },
  dimmer: { sizes: ['wide'], defaultSize: 'wide', Component: DimmerTile },
  climate: { sizes: ['wide', 'large'], defaultSize: 'wide', Component: ClimateTile },
  // ...
};
```

A new device type → one entry → tile appears automatically.

### One Device = One Tile (default)

A climate sensor with 4 capabilities (temp + humidity + pressure + battery) does **not** create 4 tiles — that turns 30 devices into 100+ tiles. Resolver picks one renderer per device by capability priority:

```typescript
const resolveTileRenderer = (device: Device): TileRenderer => {
  const caps = new Set(device.capabilities);

  // Richer capability overrides simpler ones
  if (caps.has('dimmer')) return tileRegistry.dimmer;        // dimmer subsumes switch
  if (caps.has('thermostat')) return tileRegistry.thermostat;
  if (caps.has('temperatureMeasurement') && caps.has('humidityMeasurement')) {
    return tileRegistry.climateCombo;                        // single tile, two values
  }
  if (caps.has('switch')) return tileRegistry.switch;
  return tileRegistry.generic;
};
```

Per-capability sub-tiles are **opt-in via favorites** — user picks "pin humidity as separate tile" in the Drawer.

## i18n

`useTranslation()` for components, `i18n.t('key')` for thunks (outside React):

```tsx
const { t } = useTranslation();
return <p>{t('zones.delete_confirm', { name })}</p>;
```

```typescript
// in actions.ts
import i18n from '@/i18n';
toast.success(i18n.t('devices.updated'));
```

Locale files: `ua` (fallback) + `en`. Both must always have the same keys.

## Reference

- [Frontend conventions ↗](https://github.com/alphaoflogic-ua/smart-home/blob/develop/.claude/rules/frontend.md)
- [Dashboard design notes ↗](https://github.com/alphaoflogic-ua/smart-home/blob/develop/docs/dashboard-design.md)
- [colors.css ↗](https://github.com/alphaoflogic-ua/smart-home/blob/develop/packages/frontend/src/colors.css)
- [shared/ui ↗](https://github.com/alphaoflogic-ua/smart-home/tree/develop/packages/frontend/src/shared/ui)
