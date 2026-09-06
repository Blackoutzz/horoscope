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

A `.topbar` at the top of `.wrap` sits above both views and holds the two square controls — theme and share — at the same 30x30 box (`.tbtn`). `#shmenu` lives there rather than in `.apphead` so the pair line up; it carries `.hidden` on the setup screen and both view switches toggle it, since sharing a chart that does not exist yet is meaningless. Its dropdown still positions against `.shmenu`, which stays `position:relative`.

Two mutually exclusive views toggled via `.hidden`:
- `#setup` — birth data form (name, date, time, city or lat/lon/tz), recent profiles, chart-code loader
- `#app` — full chart display (triad, wheel, numerology, readings, ephemeris tables, synastry)

`#setup` shows one panel at a time via `#setuptabs`, driven by `data-tab` on `#setupcols` — **at every width**, not only on mobile. Keep those rules out of a media query: they lived inside one once, so on desktop the tabs rendered but did nothing while both panels showed at once. Note `.periods` sets `display:flex` further down the sheet and will override an earlier `.setuptabs{display:none}`.

### JS sections (in order, within `<script>`)

Sections carry numbered banner comments. Line numbers drift with every edit — grep the banner text, don't trust the number.

| Section | Line | Purpose |
|---|---|---|
| 1. Astronomie | ~596 | Custom geocentric ecliptic planet positions — pure math, no library. Sun, Moon, 9 planets, lunar node, plus the karmic points (`moonApogee`, `chironLon`). Polynomials are only valid roughly 1900–2100 |
| 2. Vocabulaire | ~797 | Sign/glyph/aspect constants, Placidus house system, aspect orbs |
| 3. Lieux et fuseaux | ~850 | City table (lat/lon/tz) |
| 4. Stockage | ~935 | `store` — see Data persistence below |
| 5. Roue gravée | ~966 | `drawWheel()`, inline SVG ; `mountWheel()` + `wzZoom/wzApply` — zoom et panoramique |
| 5bis. Numérologie | ~1038 | Life path, master numbers (11/22/33), personal year/month cycles, traits, challenges, talents |
| 5quinquies. Encodage + synastrie | ~1323 | `encodeProfile`/`decodeProfile`/`checkProfile`, inter-chart aspect scoring, compatibility text |
| 5quater. Vie amoureuse | ~1666 | Venus/Mars/Moon sign profiles, DC sign, relationship archetype text |
| 5sexies. Axe karmique | ~2380 | `renderKarma()`, node/Lilith/Chiron copy, `karmaContacts()`, `karmaScan()`, drawn glyphs (`KSVG`) |
| 5ter. Croisement | ~1764 | Element/mode balance, dignities, rulerships, planet strength, `polarite()` |
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

### Karmic points

`KBODIES` (`noeud`, `noeudsud`, `lilith`, `chiron`) is deliberately **not** part of `BODIES`. These are markers, not actors: keeping them out means `dominance()`, `polarite()`, the element balance, the planet-to-planet aspect table and the synastry index all keep producing the numbers they always produced. Adding them would silently renumber output users have already seen and shared. `kpositions()` computes them; `CHART.karma` carries them.

- **South node** is the north node at 180°, never a separate calculation. It follows that scanning both ends of the axis reports one fact twice — `KSCAN`/`kFace()` scan the north node only and render its opposition as a south-node conjunction. The `NODE_PAIR` guard keeps the definitional 180° out of the full scan table for the same reason.
- **Lilith is the mean apogee.** The osculating ("true") apogee swings **±26°** around it — measured against JPL lunar vectors, not assumed — so it is a different point, not a finer one, and can sit a whole sign away. It would also need the lunar distance and latitude that `moonLon()` does not compute. A `.note` in the section says so; don't quietly upgrade it, and don't offer a mean/true selector without reckoning with the fact that "true" has at least three published definitions.
- **Chiron** has no Standish elements. A plain Kepler fit drifts 0.7° over 1900–2100 under Saturn/Uranus perturbation — enough to name the wrong sign. `CHIRON_EL` plus three Chebyshev polynomials (`CHIRON_C`: heliocentric longitude, latitude, radius) fitted against JPL Horizons hold max error to **0.045°** over 14 726 test epochs. The coefficients are data, like `PL` and `MOON_T`; refitting needs the scripts, not a build step.
- **`NO_RETRO`** replaced a hardcoded `b!=='noeud'`. ℞ is meaningless for any mean point (constant speed by construction) but Chiron genuinely retrogrades.
- Chiron and Lilith are **drawn** (`KSVG`), never written as U+26B7/U+26B8. Apple Symbols does not reliably carry that range, so on iOS the characters render as empty boxes, and a webfont would mean widening the CSP. ☊/☋ (U+260A/260B) are ancient and safe as characters.
- Without a birth time the section still renders: these points move under 0.2°/day so the signs are never in doubt, only the houses are lost, and a note says so rather than hiding the section or omitting the houses silently.

### Precession

`planetLon()` adds `precession(T)`. Standish's elements are referred to the **J2000** ecliptic; `sunLon()`, `moonLon()` and `moonNode()` are all of-date, and the tropical zodiac is of-date. Without the term the nine planets were off by exactly general precession — 1.4° in 1900, 0.7° in 1950, 0 in 2000, −0.7° in 2050 — so roughly one 1950 chart in five had at least one planet in the wrong sign. Verified against JPL Horizons: worst case fell from 1.399° to 0.017° (Mercury), 1.386° to 0.011° (Pluto). Any new body added to section 1 must land in the same of-date frame.

### Polarity (intro/extraversion)

`polarite()` averages three traditional readings of the same in/out axis, each weighted like `dominance()` (Sun/Moon/ASC triple): sign polarity (yang = fire + air), hemisphere (above the horizon = houses 7–12), angularity (angular 1, succedent 0.5, cadent 0). The last two need a birth time; without one the function returns the sign-polarity axis alone — and drops the ASC weight with it, so the score legitimately differs from the timed one.

The wording is deliberate and should stay that way. Astrology does not predict personality — controlled tests (Carlson, *Nature* 1985; Dean & Kelly 2003) find no effect — so the block reports **two** sentences per band: one describing the chart, one addressing the reader, plus a `.polnote` line naming the three axes and stating plainly that the extraversion correspondence is convention, not a study result. Don't quietly promote the number to a psychometric claim.

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

Reading cache keys are `astro:lecture:<period>:<date>:<chartKey>`, where `chartKey()` is a 32-bit FNV hash of `encodeProfile()` in base 36. The chart segment is not optional: without it, opening a second chart the same day replayed the first one's reading — solar return and transit agenda included. Two charts colliding on the hash would resurrect that bug, but at ~1 in 4 billion per pair it is not worth a longer key.

`pruneReadings()` runs after each write and drops every `astro:lecture:` key whose window segment isn't today's, legacy four-segment keys included — there is now one entry per chart per period, so nothing would ever be reclaimed otherwise. It needs `store.keys()`, which enumerates localStorage and `mem`; `window.storage` has no listing API, so in the sandbox stale keys simply survive. Ménage failing is harmless; serving a wrong reading is not.

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

### Theme

Two states, toggled by `#themebtn` and stored in `astro:theme`. With nothing stored — a first visit — `prefers-color-scheme` decides, so the toggle starts from the system setting and the first click pins an explicit choice.

- The button shows the theme it switches **to**, not the current one: a moon means "go dark". The icons are drawn inline (`TICO`), not ☀/☾, for the reason recorded under Karmic points — and an icon-only button has no text to fall back on if the glyph turns into an empty box.
- The preference is applied by a **second inline `<script>` in `<head>`**, read from `localStorage` directly rather than through `store`: the main script runs at the end of the body, so going through it would show the light palette for a frame before switching. That is the only reason the app has two script tags. It sets nothing when no preference is stored, which is what lets the media query paint the right palette on a first visit.
- **Every colour must be a token.** A literal anywhere outside `:root` survives the switch and will be wrong in one theme. Tints compose as `rgb(var(--v-rgb) / .13)` from raw triplets, and `--shadow` / `--ring` are separate tokens because a shadow built from `--ink` glows once the ink turns pale. `--disc` is the wheel's core tint, raised in dark where 3.5 % is invisible. The JS colours in `ASPECTS`, the agenda and the gauge are `var(--…)` strings for the same reason — inline SVG resolves them.
- The dark palette is not an inversion: `#EFE9DA` on `#1B2A44` vibrates, so ink and vellum are re-picked, and verdigris and minium are lightened to hold contrast.

### Accessibility

The share menu (`#shlist`) declares `role="menu"`, which is a contract: arrow keys cycle entries and wrap, Home/End jump to the ends, Escape closes and returns focus to `#shtog`, hidden entries are skipped, and entries carry `tabindex="-1"` so only the button sits in the tab order. `#combolist` implements the same pattern. If you add a menu, match it or drop the roles — declaring the role without the keyboard behaviour is worse than no role.

### Language

UI and all reading text are in **French**.
