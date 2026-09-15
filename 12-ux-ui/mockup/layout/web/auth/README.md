# Mockups — Auth

Public authentication flows.

## Real routes and components

| Route | Component | Sub-components |
|-------|-----------|-----------------|
| `/login` | `auth/pages/Login.jsx` | `LoginForm`, `AuthLayout`, `BackButton` |
| `/register` | `auth/pages/Register.jsx` | `RegisterForm`, `LegalModal` |
| `/VerifyAccount` | `auth/pages/VerifyAccount.jsx` | — |
| `/recover-password` | `auth/pages/RecoverPassword.jsx` | `RPasswordForm/RvPassword.jsx` |
| `/verifyRecoverPassword` | `auth/pages/VerifyRecoverPassword.jsx` | `RPasswordForm/VRPassword.jsx` |

## Mockups needed

- [ ] Login — with and without a validation error
- [ ] Register — with the legal terms modal open
- [ ] Verify account (code entry)
- [ ] Recover password (request step)
- [ ] Recover password (code/reset step)

## States to cover

- Default / empty form
- Field-level validation error
- Submitting (loading)
- Server error (e.g. invalid credentials, account blocked — `INV-005`)
