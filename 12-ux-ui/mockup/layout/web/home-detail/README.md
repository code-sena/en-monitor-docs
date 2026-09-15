# Mockups — Home detail

Everything a user sees once inside a specific home. Currently only `/consumption` is wired as
its own route in `AppRoutes.jsx` — `Devices`, `Thresholds`, `Users`, `ConsumptionHistory`, and
`Home` exist as components under `features/DetailHome/` but are not yet routed independently.
**Confirm with the frontend dev whether these are meant to be tabs inside one detail page, or
separate routes still to be wired**, before finalizing this section's navigation-map.md entry.

## Real components

| Component | Purpose |
|-----------|---------|
| `DetailHome/Home/Home.jsx` | Home overview |
| `DetailHome/Consumption/Consumption.jsx` | Current consumption (routed at `/consumption`) |
| `DetailHome/ConsumptionHistory/ConsumptionHistory.jsx` | Historical consumption chart |
| `DetailHome/Devices/Devices.jsx` | Device list, with `LinkDeviceModal` and `ConfirmDeleteModal` |
| `DetailHome/Thresholds/Thresholds.jsx` | Daily/monthly consumption limits |
| `DetailHome/Users/Users.jsx` | Home members and their `Home Role` |

## Mockups needed

- [ ] Home overview
- [ ] Consumption (current)
- [ ] Consumption history (chart)
- [ ] Devices — list, empty state (`shared/EmptyChart.jsx`), link-device modal, delete confirmation
- [ ] Thresholds — view and edit
- [ ] Users — member list, roles (`OWNER` / `MEMBER`)

## States to cover

- Loading, empty, populated, and error for each of the above (matches the shared
  `LoadingState`/`ErrorState`/`EmptyChart` components already in the codebase)
