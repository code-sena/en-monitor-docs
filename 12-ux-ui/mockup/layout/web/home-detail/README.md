# Mockups — Home detail

Everything a user sees after opening a specific home. Unlike `landing`, this area requires a
signed-in session and lives inside a different navigation shell (fixed sidebar, not a public
navbar) that repeats on every screen in this module.

**The 6 tabs share a single screen** — they are not 6 independent pages but 6 views switched
inside one home-detail screen. The mockups reproduce that with one HTML file per tab/state, all
sharing the same shell (sidebar + header + tab row) and linked to each other, so that opening any
file and clicking through the tabs feels like a single screen.

## Screens and mockup files

The screen is also split by the user's **Home Role** (`OWNER` / `MEMBER`, see
`02-domain/entities-and-rules.md`). In the React app the role reaches the screen as `isOwner`
(derived from `home.variant === "owned"`), and it gates individual controls; the tabs and layout
are the same for both roles. Every tab below shares the same intended route — a home's detail
screen — switched by tab state, not by a distinct URL per tab.

### `owner/` — the full, built view (baseline for every tab)

| Tab / state | Mockup file | Stylesheet |
|-------------|-------------|------------|
| Consumption (populated, default tab) | `consumption.html` | `css/consumption-styles.css` |
| History | `consumption-history.html` | `css/consumption-history-styles.css` |
| Devices (populated list) | `devices.html` | `css/devices-styles.css` |
| Devices (no linked devices) | `devices-empty.html` | `css/devices-styles.css` |
| Devices + "Link device" modal | `devices-link-modal.html` | `css/devices-styles.css` |
| Devices + delete-confirmation modal | `devices-delete-confirm.html` | `css/devices-styles.css` |
| Thresholds (view, system defaults on) | `thresholds.html` | `css/thresholds-styles.css` |
| Thresholds (edit, custom limits) | `thresholds-edit.html` | `css/thresholds-styles.css` |
| Users (members + invite + pending invitations) | `users.html` | `css/users-styles.css` |
| Home (information + responsible user) | `home.html` | `css/home-styles.css` |

The 4 Devices states share `devices-styles.css` and the 2 Thresholds states share
`thresholds-styles.css` — they are not new screens, only variants of the same tab.

### `member/` — only the tabs whose controls change

| Tab / state | Mockup file | Stylesheet |
|-------------|-------------|------------|
| Devices (read-only list) | `devices.html` | `css/devices-styles.css` |
| Devices (empty, read-only message) | `devices-empty.html` | `css/devices-styles.css` |
| Users (read-only list) | `users.html` | `css/users-styles.css` |
| Home (leave home, no access code) | `home.html` | `css/home-styles.css` |

Consumption, History and Thresholds are **not duplicated** here — they render identically for
both roles, so the member tabs for them link straight to `../owner/`. See `member/README.md`.

## Content per tab

**Consumption** (`consumption.html`) — 3 KPI cards (current power, today's consumption, online/total
devices), area chart "Global consumption — last 24 hours" (active power, kW), two side-by-side
cards (distribution by appliance type as a donut + current consumption per device as bars), and
two progress bars for the daily/monthly limit (kWh). Charts are drawn as simplified SVG/CSS, not
with a real chart library — a faithful visual placeholder of the composition, not functional.

**History** (`consumption-history.html`) — period filters (Day/Week/Month/Year) with sub-filter
chips and a date range, a single-selection appliance legend above a vertical bar chart (kWh, Y axis
with dashed grid, one bar per day for "Month", horizontal scroll), a consumption ranking table
(#, device, total, % bar coloured per device) and 3 stat cards (period total, average, top
consumer) laid out in two columns (ranking `1fr` + stats `320px`).

**Devices** (`devices.html` / `devices-empty.html` / `devices-link-modal.html` /
`devices-delete-confirm.html`) — device list with type icon, room, connection badge
(online/offline), current consumption in kW and delete button; "Link device" button; empty variant
with illustration and an invitation to link the first one; variant with the link modal open (step 1
of 5: discover nearby modules — appliance, Wi-Fi network, connecting and done steps are not drawn);
variant with the delete-confirmation modal open.

**Thresholds** (`thresholds.html` / `thresholds-edit.html`) — "use system default thresholds"
toggle with an On/Off pill and read-only daily/monthly limit fields (kWh) while active; edit variant
with the toggle off, the Daily/Monthly selector and the non-edited field calculated automatically
(monthly = daily × 30).

**Users** (`users.html`) — member list with initials avatar, name, email, role badge
(Responsable = `OWNER` / Miembro = `MEMBER`) and remove button; invite-by-email card; pending
invitations card (empty state).

**Home** (`home.html`) — information card (name, type, address, description, creation date, access
code with copy button, "Delete home" button) and the responsible-user card (avatar, name, email,
phone).

## Navigation

Every file includes the same shell:

- **Sidebar**, fixed on the left: logo, "Inicio", an expanded "Hogares" group with 2 sample homes
  (the current one highlighted), "Favoritos", "Notificaciones", "Ajustes", and a profile block at
  the bottom (generic person-icon avatar, bold name, "…" options button — no email line).
- **Header**: a `header-card` with the breadcrumb "Inicio › Casa Principal", followed by the row of
  6 tabs (Consumo, Historial, Usuarios, Dispositivos, Umbrales, Hogar — same order as the React
  `TABS` array), each linking to its file, active tab underlined.

The Devices and Thresholds variants are reached from buttons inside the tab itself ("Vincular
dispositivo" → `devices-link-modal.html`, delete icon → `devices-delete-confirm.html`,
"Personalizar límites" → `thresholds-edit.html`) and return to the base view on confirm/cancel.
In `member/`, tabs whose screen is not duplicated link to the matching file in `../owner/`.

## Sample devices

All screens use the same four example devices:

| Device | Type | Color |
|--------|------|-------|
| Nevera (Fridge) | `fridge` | `#2563EB` |
| Lavadora (Washer) | `washer` | `#7C3AED` |
| Computador (PC) | `pc` | `#CA8A04` |
| Televisor (TV) | `tv` | `#DB2777` |

Names come from `i18n/locales/es/devices.json` (`applianceTypes`); colors come from
`shared/deviceChartConfig.js` (light mode). The first three are the `INITIAL_DEVICES` of the dev
branch; the fourth is the next type in the `APPLIANCE_ICON` catalog. Nevera, Lavadora and Televisor
are online; Computador is offline (as in `INITIAL_DEVICES`).

## Visual identity

Every stylesheet declares the canonical tokens of the real project (`:root` copied from
`src/styles/global.css`: `--color-primary`, `--color-secondary`, `--color-text-primary`,
`--color-background`, `--color-background-left`, `--color-surface`, `--shadow-lg`, `--spacing-*`,
`--radius-*`, …) so the whole mockup system reads as one product with the React app. Sidebar text
and hover colors, and the orange `#f5a623`, have no token and stay as literal values, as in the real
`Sidebar.module.css`. Each stylesheet re-declares the tokens (same self-contained pattern as
`about-styles.css` / `support-styles.css` in `landing/`).

## Bounded context

Unlike `landing` (where no bounded context applied), these screens reflect data from several
bounded contexts defined in `02-domain/domain-map.md`:

| Tab | Bounded context reflected |
|-----|---------------------------|
| Consumption, History | Monitoring and Measurements — readings, aggregated consumption, distribution per appliance |
| Devices | Devices — registration, connection status and linking a device to this home |
| Thresholds, Users, Home | Home Management — limits, home members/roles, and the home's own data |

No tab in this module belongs to the Security context (authentication already happened before
arriving here) nor directly renders Alerts/Recommendations — those live in the `notifications`
module, although the limit bars in Consumption are the visual trigger of those alerts.

## Mockups needed

- [x] Home overview
- [x] Consumption (current)
- [x] Consumption history (chart)
- [x] Devices — list, empty state, link-device modal, delete confirmation
- [x] Thresholds — view and edit
- [x] Users — member list, roles (Responsable / Miembro)
- [x] Member (read-only) variants — Users, Devices, Home

## States to cover

- Loading and Error were not mocked file by file: in practice they are a centered placeholder block
  (a "Loading…" text, or an error message + "Retry" button) with no visual composition of their own
  that would justify 12 almost-identical files. If needed as a reference, a centered block inside
  `.main-content` replacing the content of any of these screens is enough. See "Known gaps" — in
  `dev` the shared `LoadingState` / `ErrorState` components do not exist yet.
- Empty, populated, and the modal/edit variants are covered as separate files (see the tables above)
  for Devices and Thresholds, the two tabs where the state changes the visual composition
  significantly.
- Role variants (OWNER / MEMBER) are covered by the `owner/` and `member/` folders.

## Known gaps / no clear correspondence

- `design-system.md` is still an unfilled template. It still needs the extended sidebar and status
  color tokens this module added (sidebar text/hover, success/warning/danger soft backgrounds),
  along with those of `landing/`.
- `LoadingState` / `ErrorState` do not exist in `dev`, so there are no loading/error states to mock
  beyond a centered placeholder block. `EmptyChart` exists but is not mocked.
- Home deletion/leaving uses `window.confirm` in `dev`, not a custom modal; no confirmation is drawn.
- Two real bugs found in `dev` that the mockup intentionally shows fixed, not as-is: (1) the literal
  untranslated string `t("kpi.currentLimit")` printed in `Consumption.jsx`'s second KPI card (the
  mockup shows "Límite actual: 30 kWh"); (2) the "Todos operativos" device-status card, which never
  reflects an actual offline device (it always shows that text; the mockup shows it next to 3 / 4).
- `kW` is used for both instantaneous power and per-device consumption in `dev` (device rows and the
  "consumption per device" bars), inconsistent with kWh elsewhere; the mockup keeps `kW` in both to
  match `dev` as-is.
- The real seed names in `Users.jsx` (Carlos García, Ana Martínez, Luis Pérez, Sofía Ramos) differ
  from the placeholder names still used in the mockup (Camila Restrepo, Andrés Beltrán, Laura
  Ramírez, Daniel Torres).
- The role reaches `DetailHome` as `isOwner` from `home.variant === "owned"`; there is no
  `routes/HomeMemberRoute.jsx` in `dev`, so no route-level restriction for members exists yet — only
  the per-control conditions described in `member/README.md`.
- `Thresholds.jsx` also changes for a member (lock notice, disabled controls, no Save button) but,
  per the current decision, no `member/` mockup was drawn for it.
- The link-device modal is drawn only for step 1 (discover); the domain's `api_key` is not part of
  the `dev` flow.
- The Devices empty state uses an illustration in the mockup and a `WifiOff` icon in React.
- The device-name tooltip in the Consumption bar chart partially overlaps the first bar.

## Correlations with other sections

| This mockup is fed by... | And feeds into... |
|---------------------------|------------------------|
| `02-domain/domain-map.md` | The bounded-context table above |
| `02-domain/entities-and-rules.md` | The data fields shown (limits in kWh, 8-character access code, 6-character device code, roles `OWNER` / `MEMBER` shown as Responsable / Miembro) |
| `assets/logos/project_logo.png` | Sidebar logo, referenced as in `landing/` |
| `assets/images/mockup-assets/empty_img.png` | Devices empty-state illustration (`devices-empty.html`) |
| `12-ux-ui/design-system.md` (still an unfilled template) | Should absorb this extended palette (sidebar and status colors) together with that of `landing/` |
| — | The future frontend implementation of these 6 tabs — this mockup is the design reference, not the other way around |
| `mockup/layout/web/dashboard/` (no HTML yet) | The real entry point to this module — from there the user would navigate to the home these mockups represent |