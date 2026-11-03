# Mockups — Landing

Public marketing pages, no authentication required.

## Screens and mockup files

| Screen | Intended route | Mockup file | Stylesheet |
|--------|-----------------|-------------|------------|
| Home | `/` | `index.html` | `css/index-styles.css` |
| About | `/about` | `about.html` | `css/about-styles.css` |
| Support / contact | `/support` | `support.html` | `css/support-styles.css` |

Each screen has its own stylesheet, named `<screen>-styles.css`, so every mockup file is
self-contained — the shared foundation (`:root` tokens, reset, navbar, buttons, section title
helpers) is intentionally repeated in all three files rather than factored into a common
stylesheet, so any one screen can be opened, copied, or handed off on its own.

## Content per screen

**Home** (`index.html`)
- Navbar: logo, "Inicio" / "Sobre nosotros" / "Soporte", theme toggle, language selector,
  "Regístrate" / "Iniciar sesión"
- Hero section — headline, CTA, quick feature bullets
- "¿Cómo funciona EnergyMonitor?" — 4-step grid
- "¿Por qué elegir EnergyMonitor?" — 4-item benefits grid

**About** (`about.html`)
- Hero — mission-oriented headline
- "Nuestra historia" — problema / misión / visión (3 cards)
- "Lo que nos guía" — values (4 cards)
- "¿Cómo lo hacemos posible?" — centered explainer block with a link back to Home's steps section
- "Lo que buscamos lograr" — 3 impact stats
- "El equipo detrás de EnergyMonitor" — 2 member cards
- Final CTA — "Crear cuenta" / "Iniciar sesión"

**Support** (`support.html`)
- Hero with a static search box
- "Temas populares" — 6-topic grid
- "Preguntas frecuentes" — accordion (native `<details>`/`<summary>`, no JS needed)
- "Contacto directo" — WhatsApp card and email card

## Navigation

The same navbar (logo, "Inicio" / "Sobre nosotros" / "Soporte", theme toggle, language selector,
"Regístrate" / "Iniciar sesión") is repeated at the top of all three files, and its links point to
the sibling mockup files rather than to in-page anchors:

```
index.html   (#inicio is this same page)
about.html   ← "Sobre nosotros"
support.html ← "Soporte"
register.html / login.html ← "Regístrate" / "Iniciar sesión" (not built yet — see mockup/layout/web/auth/)
```

One in-page link crosses files: About's "Ver cómo funciona" button points to
`index.html#nosotros`, landing back on Home's "¿Cómo funciona?" section. Support's two contact
actions (WhatsApp, email) are kept as inert placeholders (`href="#"`) since they are external
actions, not part of the screen-to-screen navigation this map documents.

## Visual identity

The same `:root` design tokens are defined identically at the top of all three stylesheets:
`--primary-blue`, `--accent-green`, `--text-dark`, `--text-gray`, `--bg-body`, `--bg-white`,
`--shadow-card`, and `Segoe UI` as the base font. Keeping them duplicated per file (instead of one
shared stylesheet) is deliberate for this static-mockup stage; `12-ux-ui/design-system.md` is
still an unfilled template — once it is populated, this palette should be promoted there so every
future mockup (auth, dashboard, etc.) draws from the same documented tokens instead of each
screen — or each module — picking its own.

## Bounded context

**Does not apply.** `02-domain/domain-map.md` identifies six bounded contexts — Security, Home
Management, Devices, Monitoring and Measurements, Alerts, Recommendations — each owning domain
entities and backed by its own microservice. Landing's three screens are static marketing content:
they own no entity, call no API, and hold no state beyond a search box and an FAQ accordion, both
purely visual. Their only connection to the domain model is that they route the visitor toward
Security's public entry points ("register", "login"), which is where the modeled system
actually begins.

## Mockups needed

- [x] Home — hero section, benefits grid, "how it works" section
- [x] About
- [x] Support / contact

## States to cover

- Default (only state — public, static content, no loading/error/empty variants apply)

## Correlations with other sections

| This mockup is fed by... | And feeds into... |
|-----------------------------|------------------------|
| `04-requirements/user-stories.md` — no HU currently covers the landing/marketing flow; it sits outside the documented functional scope, only the auth flow it links into is in scope | — |
| `02-domain/domain-map.md` | Confirms the "bounded context: does not apply" statement above |
| `12-ux-ui/design-system.md` (still an unfilled template) | Should eventually absorb this module's palette (repeated across `index-styles.css`, `about-styles.css`, `support-styles.css`) as the shared design tokens |
| `assets/logos/project_logo.png` | Navbar brand logo, referenced from all three files via `../../../../../assets/logos/project_logo.png` |
| `assets/images/mockup-assets/hero_img.jpeg` and `hero_img2.png` | Home's hero and "how it works" illustrations, via the same relative path; About and Support use no images, only text and inline SVG icons |
