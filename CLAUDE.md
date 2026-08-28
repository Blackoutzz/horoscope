# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Deployment

Static site hosted on GitHub Pages — no build step, no bundler, no dependencies.

- Live URL: `https://blackoutzz.github.io/horoscope/`
- Deploy: `git push origin main` (Pages auto-deploys from `main` branch, root folder)
- Regenerate PWA icons from `logo.png`: run the Python/Pillow snippet used to create `apple-touch-icon.png`, `icon-192.png`, `icon-512.png`, `favicon.ico`

## Architecture

Everything lives in a single file: `index.html`. CSS, HTML, and JS are all inline — no external scripts, no frameworks, no imports.

### HTML structure

Two mutually exclusive views toggled via `.hidden`:
- `#setup` — birth data form (name, date, time, city or lat/lon/tz), recent profiles, chart-code loader
- `#app` — full chart display (triad, wheel, numerology, readings, ephemeris tables, synastry)

`#setup` shows one panel at a time via `#setuptabs`, driven by `data-tab` on `#setupcols` — **at every width**, not only on mobile. Keep those rules out of a media query: they lived inside one once, so on desktop the tabs rendered but did nothing while both panels showed at once. Note `.periods` sets `display:flex` further down the sheet and will override an earlier `.setuptabs{display:none}`.

### JS sections (in order, within `<script>`)

Sections carry numbered banner comments. Line numbers drift with every edit — grep the banner text, don't trust the number.

| Section | Line | Purpose |
|---|---|---|
| 1. Astronomie | ~596 | Custom geocentric ecliptic planet positions — pure math, no library. Sun, Moon, 9 planets + lunar node. Polynomials are only valid roughly 1900–2100 |
| 2. Vocabulaire | ~797 | Sign/glyph/aspect constants, Placidus house system, aspect orbs |
| 3. Lieux et fuseaux | ~850 | City table (lat/lon/tz) |
| 4. Stockage | ~935 | `store` — see Data persistence below |
| 5. Roue gravée | ~966 | `drawWheel()`, inline SVG ; `mountWheel()` + `wzZoom/wzApply` — zoom et panoramique |
| 5bis. Numérologie | ~1038 | Life path, master numbers (11/22/33), personal year/month cycles, traits, challenges, talents |
| 5quinquies. Encodage + synastrie | ~1323 | `encodeProfile`/`decodeProfile`/`checkProfile`, inter-chart aspect scoring, compatibility text |
| 5quater. Vie amoureuse | ~1666 | Venus/Mars/Moon sign profiles, DC sign, relationship archetype text |
| 5ter. Croisement | ~1764 | Element/mode balance, dignities, rulerships, planet strength |
| 6. Rendu | ~2052 | `buildChart()`, `renderChart()`, share menu, recents |
| 7. Lecture du jour | ~2138 | `askClaude()` POSTs to `api.anthropic.com/v1/messages` (model: `claude-sonnet-4-6`) |
| 7bis. Moteur local | ~2179 | Offline template reading, used when the API call fails |
| 8. Amorçage | ~2627 | Startup: URL token → stored profile → setup screen |

### Wheel zoom

`mountWheel()` wraps the SVG in `.wheelview` (clipping box) → `.wheelpan` (transformed). Zoom is a **CSS `transform`**, not a `viewBox` change: strokes and glyphs grow with the drawing, which is what a manual reading needs — a viewBox zoom would leave hairlines hairline-thin. State lives in the single `wz` object; `wzApply()` re-clamps translation so the drawing never pulls away from the edges, and is also the `resize` handler.

- Zoom 100→800 %: wheel, pinch, double-click, ± buttons, `+`/`-`/`0`/arrows on the focused view.
- `touch-action` is `pan-y` at 100 % (one finger still scrolls the page) and flips to `none` via `.zoomed` once zoomed, so one finger pans instead.
- « Plein écran » is a CSS overlay (`.fsmode`, `position:fixed`), **not** `requestFullscreen()`. The native API is silently useless in embedded webviews — the promise never settles, `fullscreenElement` stays `null`, nothing throws — and `Element.requestFullscreen` doesn't exist on iPhone at all. The overlay also fixes the control bar to the bottom, locks body scroll, and exits on Escape; `mountWheel()` clears both on re-mount so a re-render can't leave the page locked.
- `.wheelview` is `background:transparent`. `.wrap` paints a radial gradient; a flat `var(--vellum)` panel on top of it reads as a brighter rectangle around the wheel.
- The darkened core is two `.disc` circles (R3, R4) at `fill-opacity:.035`, stacked so the centre sits deepest — a deliberate tint. It replaces an accidental one: `.wheel .hair` had no `fill`, so those same circles filled **black** at 35 % opacity and drowned the aspect lines. Every SVG `circle` needs an explicit `fill`.

### Data persistence & sharing

`store` (section 4) tries three backends in order: **localStorage** → `window.storage` → an in-memory `mem` object.

- `LS` probes localStorage with a real write/remove at load. Private browsing, a full quota, or blocked cookies leave it `null` and the app degrades to `mem` (lost on reload) rather than throwing.
- `window.storage` only exists in the Claude Artifacts sandbox. It is **not** available on GitHub Pages. Do not make it the only backend — that bug shipped once and silently disabled all persistence.
- Keys: `astro:profil` (last active chart), `astro:recents`, `astro:lecture:<YYYY-MM-DD>`.

Sharing:

- Chart code is base64 of `nom|date|heure|kt|lat|lon|tz|lieu` (so codes start `TW`, `Qm`, `WW`… depending on the name).
- Share links use a **fragment** (`#c=…`), never a query string — see Security.
- Parsing accepts `#c=`, `?c=`, or a bare pasted code, so links shared before the fragment switch still work. Keep it that way.
- **Build the chart before writing to storage.** Both `chargerCode()` and the setup form do this, and startup drops a stored profile it can no longer compute. Reversing that order lets one bad code poison `astro:profil` and leave the app on a blank chart every visit.

`askClaude()` has **no `x-api-key` header** — an API key must be injected if direct browser calls are intended, or a proxy must be used.

## Security

A chart token is untrusted input: it arrives from a link or a paste. Three invariants hold the line.

**Escape everything user-supplied before it reaches `innerHTML`.** `esc()` (end of section 7) escapes `& < > " '` — quotes included, because it is also used inside `aria-label` and `style` attributes where one quote breaks out. `nom` and `moi` are escaped **once at the top of `renderSyn`**, because they flow through `pairDesc()` and `temperaments()`, which build sentences that get inserted as HTML. Don't add a second `esc()` downstream — that double-escapes apostrophes, which are everywhere in French.

**Validate at the trust boundary, not at the form.** `checkProfile()` (section 5quinquies) enforces a real calendar date, 1900→today, `HH:MM` in range, lat ±90, lon ±180, resolvable IANA timezone. `decodeProfile()` runs it on every token. Out-of-range coordinates used to produce a confident, *wrong* ascendant rather than an error — silent-wrong is the failure mode this guards against. The setup form is the trusted path and only mirrors the range check.

**Never put birth data in a query string.** The token holds a full name, exact birth date and time, and coordinates, in reversible base64. A fragment never leaves the browser. A query string goes into GitHub's access logs on every load, and any messaging platform building a link preview fetches the URL and receives it too.

**CSP** is a `<meta http-equiv>` in `<head>`. Everything is inline, so `script-src`/`style-src` need `'unsafe-inline'` — the policy cannot stop an injected event handler from running. It stops the payload: no remote scripts, `img-src 'self'`, and `connect-src` limited to `api.anthropic.com`, so injected markup cannot ship stored charts to another server.

- Adding any external asset (font, CDN script, remote image) means widening the policy — prefer inlining instead.
- `connect-src` does **not** include `'self'`. Nothing fetches same-origin today; code that does will be blocked until `'self'` is added.
- A hash-based `script-src` would block inline handlers outright, but the hash needs recomputing on every edit to `index.html`, and a stale hash means a silently blank app. Rejected: it breaks the no-build-step property.
- `frame-ancestors` is ignored in a `<meta>` CSP; it would need a real response header.

### Accessibility

The share menu (`#shlist`) declares `role="menu"`, which is a contract: arrow keys cycle entries and wrap, Home/End jump to the ends, Escape closes and returns focus to `#shtog`, hidden entries are skipped, and entries carry `tabindex="-1"` so only the button sits in the tab order. `#combolist` implements the same pattern. If you add a menu, match it or drop the roles — declaring the role without the keyboard behaviour is worse than no role.

### Language

UI and all reading text are in **French**.
