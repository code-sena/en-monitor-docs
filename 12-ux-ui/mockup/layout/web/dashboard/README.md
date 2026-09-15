# Mockups — Dashboard

Authenticated landing page after login. Lists the user's homes.

## Real route and components

| Route | Component | Sub-components |
|-------|-----------|-----------------|
| `/dashboard` | `dashboard/pages/DashboardPage.jsx` | `HomeCard`, `ModalCreateHome`, `ModalJoinHome`, `AddressInput`, `EmptyState` |

## Mockups needed

- [ ] Dashboard with one or more homes (grid of `HomeCard`)
- [ ] Empty state — user belongs to no home yet (`EmptyState`)
- [ ] "Create home" modal, including the address input step
- [ ] "Join home" modal (access code entry)

## States to cover

- Loading (`LoadingState`, shared component)
- Empty (no homes yet)
- Populated (one home, several homes)
- Error (`ErrorState`, shared component)
