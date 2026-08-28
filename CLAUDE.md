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

### JS sections (in order, within `<script>`)

| Section | Line | Purpose |
|---|---|---|
| Astronomie | ~551 | Custom geocentric ecliptic planet positions — pure math, no library. Handles Sun, Moon, and 9 planets + lunar node |
| Astrologie | ~752 | Sign/glyph/aspect constants, Placidus house system, city table (lat/lon/tz), aspect orbs |
| Numérologie | ~981 | Life path, master numbers (11/22/33), personal year/month cycles, traits, challenges, talents |
| Synastrie | ~1266 | Inter-chart aspect scoring with weighted planets, compatibility text generation |
| Amour | ~1585 | Venus/Mars/Moon sign profiles, DC sign, relationship archetype text |
| Éléments | ~1683 | Element/mode balance, dignities, rulerships, planet strength analysis |
| Reading + Claude | ~2467 | `askClaude()` POSTs to `api.anthropic.com/v1/messages` (model: `claude-sonnet-4-6`). Falls back to offline template reading if the call fails |

### Data persistence & sharing

- `localStorage` stores recent birth profiles and last active chart
- Charts are shareable via base64-encoded code (prefix `TW`, `Qm`, `WW`…) or `#c=…` URL hash
- `askClaude()` has **no `x-api-key` header** — an API key must be injected if direct browser calls are intended, or a proxy must be used

### Language

UI and all reading text are in **French**.
