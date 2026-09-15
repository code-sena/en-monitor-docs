# HU-[N]: [Story Title]

> **ID convention in this project:** `HU[N]` (HU1–HU6 functional, HU7–HU10 non-functional),
> matching `Historia_de_Usuario_Visualización_de_Consumo_Eléctrico.docx`. Each HU is tagged with
> the SRS requirement(s) it covers (`RF`/`ERF` for functional, `RNF`/`ERNF` for non-functional).
>
> **Worked example below** uses a real story from this project (`HU1`) so the template shows a
> fully specified HU, not just bracketed placeholders. Copy this file, replace the content with
> your own story, and delete this note.

---

## Covers

RF1 (User Management) — ERF1.1, ERF1.1.1, ERF1.2, ERF1.3.1, ERF1.3.2, ERF1.3.3

---

## Story

**As** a user of the system (web or mobile)
**I want** to register, authenticate (directly or via social login), edit my profile, recover my
password, and accept the terms and license
**So that** I can securely access the platform and manage my projects and energy consumption

---

## Acceptance criteria

> Format: Gherkin (`Scenario` / `Given` / `When` / `Then`), matching what `user-stories.md`
> actually uses for every HU — this template previously showed a checklist format instead, which
> did not match the real HU1 and would have propagated the mismatch to every new HU copied from
> here.

```gherkin
Scenario 1: Manual registration
  Given a person who is not yet registered, with a name, email, and password
  When they submit the registration form and accept the terms, conditions, and license permissions
  Then a new account is created

Scenario 2: Email already registered
  Given an email that already belongs to an existing account
  When someone tries to register with that same email
  Then registration is rejected

Scenario 3: Account self-service
  Given a logged-in user
  When they update their profile data, request a password reset via secure email, or request account deletion
  Then the requested change is applied
```

---

## Technical notes

> Implementation constraints, performance considerations, required integrations

**Responsible module:** User Management (RF1)
**Depends on:** None directly — this is the base requirement for RF2, RF3, RF4, and RF5
**Required permissions:** None to register; a valid session to edit profile, reset password, or
delete the account

---

## Definition of Done (DoD)

> This HU can only be closed when it meets the team's full DoD.
> See: [`00-governance/definition-of-done.md`](../../00-governance/definition-of-done.md)

**Additional checks specific to this HU:**

- [ ] Password is never stored or logged in plain text (see NFR/RNF3 — Security)
- [ ] Social login credentials are stored as part of the user's configuration, not merged into
      core account credentials

---

## Estimation and priority

| Field        | Value                           |
| ------------ | ------------------------------- |
| Priority     | High                            |
| Dependencies | Base for RF2, RF3, RF4, and RF5 |
| Status       | Backlog                         |
