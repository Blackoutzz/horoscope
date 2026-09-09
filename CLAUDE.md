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

`.synth-grid` is three columns, collapsing to one under 820px. Column-count variants are **classes** (`.synth-grid.two`), never an inline `style` — an inline style beats the media query, which is how the synastry grid ("Ce qui vous lie" / "Ce qui frotte") stayed two columns on a 375px phone. The media query lists every variant explicitly, since `.synth-grid.two` outranks a bare `.synth-grid` inside it.

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
| 5quinquies. Encodage + synastrie | ~1323 | `encodeProfile`/`decodeProfile`/`checkProfile`, inter-chart aspect scoring, compatibility text, `attraction()`/`blocEros()` |
| 5quater. Vie amoureuse | ~1666 | Venus/Mars/Moon sign profiles, DC sign, relationship archetype text, `blocIntime()` |
| 5sexies. Axe karmique | ~2380 | `renderKarma()`, node/Lilith/Chiron copy, `karmaContacts()`, `karmaScan()`, drawn glyphs (`KSVG`) |
| 5septies. Structure du thème | ~2897 | `renderStruct()` — secte, Part de Fortune, étoiles fixes, quadrants, forme du thème, densité et figures d'aspects, amas, dispositeurs, rétrogrades, décans/duades, degrés anarétiques |
| 5octies. Progressions secondaires | ~3500 | `renderProg()` — carte progressée, Lune et Soleil progressés, contacts sur le natal |
| 5nonies. Bazi | ~3607 | `renderBazi()` — four pillars, solar terms, lunar new year, five movements, hidden stems, branch relations |
| 5ter. Croisement | ~1764 | Element/mode balance, dignities, rulerships, planet strength, `polarite()` |
| 6. Rendu | ~2052 | `buildChart()`, `renderChart()`, share menu, recents |
| 7. Lecture du jour | ~2138 | `vocScan()` (Lune hors-course), `askClaude()` POSTs to `api.anthropic.com/v1/messages` (model: `claude-sonnet-4-6`) |
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

### Intimate register & attraction index

Two blocks, deliberately different in kind.

- **Solo (`blocIntime()`, section 5quater)** is qualitative — Mars sign (élan), Venus sign (texture), their element pair (`MV_MIX`, 4×4), Mars mode (tempo), house 8 sign when the birth time is known. **Not** a second 12-sign ranking: a Mars/Venus-weighted ranking reuses the same points as `amour()` with other weights, so the two bar lists come out near-identical and read as padding. Without a birth time the house-8 column says so and the rest still renders.
- **Synastry (`attraction()`, section 5quinquies)** rescans the same inter-chart grid as `synastrie()` but keeps only contacts touching Mars, Venus, Pluto or the ASC, and **counts hard aspects positively** — a Mars/Venus square is the most ordinary attraction contact there is, and the synastry index's penalty gives exactly the wrong sign there. Tension is reported as a separate percentage so "strong" is not read as "comfortable". The two indices therefore diverge on purpose.
- `poidsMax += w*0.124` is the expected value of a random pair (26.7 % chance of hitting one of 5 aspects at 6° orb × 0.745 mean quality × 0.625 mean tightness); scaled by 50 it centres a random pair at 50. Measured over 780 pairs: mean 49.6, p10 36, p90 64.
- `erosDesc()` exists because `pairDesc()` appends "mais en carré : ça accroche" to hard aspects — right for the synastry index, contradictory in a block where the hard aspect is the motor. Same `PAIRTXT` table, own tone strings (`EROS_TON`).
- `EROS_CORE` uses `asc` only, never a separate `dc` — same axis, counted once, as in `synastrie()`.

### Structure du thème

Section 5septies (`renderStruct()`) reports twelve classic *form* measures over positions already computed. Like `KBODIES`, none of them enters `BODIES` or any weighting — `dominance()`, `polarite()`, the element balance, the aspect table and the synastry index all return the numbers they returned before, and the block says so in its own intro. It renders between karma and `amour`.

- **Sect** (`secteDe()`) — Sun above the horizon means houses 7–12, the same hemisphere convention as `polarite()`, not a separate altitude calculation. Without a birth time it returns `null` and the block says the question has no answer: the Sun crosses the axis twice a day, so a guess would be wrong half the time. `fortuneDe()` takes the sect as an argument and returns `null` with it, which is why the two share a grid.
- **Part of Fortune** — ASC + Moon − Sun by day, **inverted at night**. The inversion is the ancient form and the only one that keeps the point on the Moon's side of the horizon; the un-inverted "modern" formula is a different point, not a simplification. Needs the time, since the ASC is a term.
- **Retrogrades** (`retrosDe()`) — the flag was already set per body in `positions()` and never totalled. The denominator is `RETROABLE` (8 bodies): Sun and Moon never retrograde, and the node is a mean point (`NO_RETRO`). `RETRO_BAND` states plainly that two retrogrades is the statistical floor, not a signature — the outer three are retrograde about five months a year each.
- **Amas** (`amasDe()`) — 3+ of the ten planets in one sign, and in one house when the time is known. The two counts are independent and both are listed: a sign stellium routinely straddles two houses.
- **Anaretic degrees** (`anaretiques()`) — last degree (≥29°) of a sign, angles included when timed. `MC` is written as the letters, not a glyph, for the reason recorded under Karmic points.
- **Aspect figures** (`figures()`) — grand cross, grand trine, T-square, yod. It runs its **own** aspect scan rather than reusing `CHART.asp`, for two reasons: figures want tighter orbs than the display table's 8°/6° (`FIG_ANG` uses 6° / 4° on sextiles / 3° on quincunxes), and the yod needs the **quincunx**, which is deliberately absent from `ASPECTS` — adding it there would renumber the natal aspect table and the synastry index. Ten planets, no angles, so figures survive a missing birth time. T-squares wholly contained in a detected grand cross are dropped: same figure, described smaller.
- **Aspect density** (`densiteDe()`) — counted on `CHART.asp`, so the numbers match the table the user can scroll to. Conjunctions count as neither hard nor soft; their register depends on the two planets, not the angle. The tension percentage is `dur/(dur+doux)`, and the copy states its own limit — it weighs a Moon–Mars square the same as a Uranus–Neptune one.
- **Chart shape** (`formeDe()`) — Jones patterns from the spread of the ten longitudes alone, no time needed. Order matters: faisceau (≤125°) → seau (a handle with ≥60° on both sides and the remaining nine inside 190°) → bol (≤190°) → locomotive (largest gap ≥110° and no second gap ≥60°) → balançoire (two gaps ≥60°) → éclaboussure. The bucket test must precede the bowl test, or every bucket reads as something else.
- **Dispositors** (`dispositeurs()`) — each planet walks to the ruler of its sign via `MAITRE` until it reaches a fixed point (a planet in its own domicile) or a cycle. `cpt` counts how many of the ten bodies land on each terminal, cycles included: without it, planets feeding *into* a loop are invisible in the output, and the counts don't sum to ten. `MAITRE` is **modern** rulership, which gives fewer fixed points and more loops than traditional rulership — a `.note` says the chains are a property of the system chosen, not of the chart.
- **Quadrants and hemispheres** (`quadrants()`) — `polarite()` already computed the above/below split and threw it away undisplayed; the east/west axis (houses 10–3 vs 4–9) existed nowhere. Same weights as `dominance()`, ASC excluded — it sits on a cusp and belongs to no quadrant. Needs the time; the block says this is the measure that loses most without it. Its hemisphere figure and `polarite()`'s are the same number by construction, so a divergence between the two blocks is a bug.
- **Decans and duads** — decans by **triplicity** (1st = the sign's ruler via `MAITRE`, 2nd and 3rd = the rulers of the other two signs of the same element), duads at 2°30 walking the zodiac from the sign itself. Shown for the same triple point as `#triad`. A `.note` names the Chaldean decan system as the other live convention rather than implying triplicity is the only one; neither changes the sign.

- **Fixed stars** (`STARS`, `starHits()`) — 37 traditional stars, conjunction by **longitude only** at 1°. The catalogue's J2000 longitudes were derived from J2000 RA/Dec and then cross-checked against published tropical positions: 35 of 37 agreed within 1′, worst case 4.2′ (Toliman). Precession to date reuses `precession(T)`, the same function that puts the planets in the of-date frame — a star precessed differently from the planets would make the conjunction meaningless. Proper motion is ignored (under 6′/century for all of them, a tenth of the orb over two centuries). The `.note` states that ecliptic latitude is deliberately not tested, which is why Vega (β +61.7°) can be reported "conjunct" a planet.

Nothing here reaches `buildPrompt()` — the daily reading and its cache keys are untouched. The whole section renders in about 1 ms.

### Secondary progressions

Section 5octies (`renderProg()`), rendered in `#prog` after `#struct`. One day after birth per year of life: `progJD()` divides the elapsed days by `AN_TROP` and feeds the result straight to `positions()`. No new astronomy — only the evaluation date changes.

- **Five bodies only** (`PROG_CORPS`). At a day per year Jupiter advances 0.08° per year of life and Pluto 0.004°: their "progressions" are their natal positions, and showing them would dress a birth aspect up as an event.
- **No progressed angles.** At least three incompatible conventions exist (ASC recomputed for the place at the progressed date, solar arc, Naibod), and they separate the progressed MC by degrees over a lifetime. Picking one silently would yield a wrong progressed house with the confidence of a right one. The houses shown are the **natal** cusps, which progressed planets travel through.
- **No "écart" column.** Over 40 years the progressed Moon laps the zodiac more than once, so any displayed difference modulo 360° reads as the planet moving backwards. The natal sign shown alongside says the same thing without lying.
- Progressed→natal aspects use a **1° orb**: the progressed Moon covers a degree in a year, so a wider orb would smear each contact across three years and stop dating anything. `findAspects`'s `exact` flag means "under 1°" and is therefore useless at this orb — the render tags `orb < 0.1` instead.

### Bazi (four pillars)

Section 5nonies (`renderBazi()`), rendered in `#bazi` between `#prog` and `#amour`. A **different tradition**, not another measure of the same thing — so like `KBODIES` and `renderStruct()`, nothing here enters `BODIES`, `dominance()`, `polarite()`, the element balance, the aspect table or the synastry index, and nothing reaches `buildPrompt()`. The block says so in its own intro.

No new astronomy. Bazi boundaries are **solar terms** — the Sun at a multiple of 15° apparent longitude — and `sunLon()` already gives that; the day pillar is a modulo on the Julian day. Only the lunar new year, shown for comparison, needs `moonLon()`.

- **Two live year conventions, both displayed.** The bazi year turns at **Lichun** (Sun = 315°, Feb 3–5); the popular zodiac turns at the **lunar new year** (Jan 21 – Feb 21). They disagree for any birth between the two dates, which is why the page shows both and names which one the rest of the block uses. Picking one silently would hand out a wrong animal for a month of every year. The Lichun and new-year dates cited are the ones that **opened the subject's year**, not those of the Gregorian birth year — a January birth belongs to the previous Lichun.
- **The lunar calendar reasons in Chinese civil days (UTC+8), not instants.** Month 11 is the month containing the **day** of the winter solstice. Comparing instants picks the wrong month whenever solstice and new moon land on the same CST day — that alone put 1985 and 2015 a full lunation early. `jourCN()`/`minuitCN()` exist for this.
- The leap-month rule is implemented in full (first month with no major solar term; if it lands in position 11 or 12, month 1 slides one lunation later) rather than approximated as "always the second new moon". The exception is rare but moves the date by a month. Verified: **35/35** against known new-year dates 1985–2033 including the leap-11 case 2033, and 1900–2100 all land in the Jan 21 – Feb 21 window.
- **Day pillar anchor**: `(JDN + 49) mod 60`, i.e. JD 2451545 (2000-01-01) = 戊午, index 54. The whole day column rests on that one constant — change it and every chart is wrong by a fixed offset, silently.
- **No hour pillar without a birth time.** One column in four would be invented with the confidence of the other three. When the time is missing *and* the birth falls within 12 h of a solar term, a `.note` says the month (and possibly the year) pillar turns during that day.
- **Chinese characters are written, not drawn** — unlike Chiron and Lilith. CJK is carried by system fonts on every target platform, so `.han` names a fallback stack and adds no webfont, hence no CSP widening.
- **Hidden stems** (`CACHES`) are weighted 1 / 0.35 / 0.2. That is one convention among several — others count them equally or not at all — and the `<details>` says so, because the five-movement percentages move with it.
- The ten gods are shown as **five families** (peers / expression / resources / authority / support) relative to the day master's element, and there is **one bar list, not two**. Each element falls in exactly one family, so a second list by family is the first list reordered with other labels — the same five numbers read twice, the padding `blocIntime()` is warned about above. The family, why the element lands in it (`FAM_LIEN`), and the characters that produced the number go in a per-row drawer instead, reusing `.erow`/`.edraw`/`.epts` from `#elements` — hence `wireDrawers(id)`, which `wireElements()` now calls for both sections. A flat legend under two mirrored lists made the reader decode; the drawer answers on demand and carries what the bar cannot show.
- **No luck pillars (大運)**: their direction depends on sex, which the form does not collect. **No true solar time**: pillars sit on the birth zone's clock, while traditional practice uses local apparent time — it moves the hour pillar only near a double-hour boundary. The day turns at **midnight**, not 23:00; schools differ and the note says which one is used.
- Solar-term instants ignore ΔT and are good to a few minutes (Lichun 2024 computed 16:21 CST vs 16:27 actual) — visible only for a birth sitting exactly on a boundary. Stated in the note.
- French copy needs **gendered articles per element**: `LE_WX`/`AUCUN_WX`/`IL_WX` exist because `WX[i].toLowerCase()` behind a fixed article or pronoun writes « le terre », « le eau », « il porte » of la terre.

### Void-of-course Moon

`vocScan()` (section 7, before `buildPrompt`). The Moon is void when it forms no further major aspect before leaving its sign — the one fact about the day that the reading could not name, since transits say what connects and this says when nothing more will.

- **Classical bodies only** (`VOC_CORPS`, Sun through Saturn). Adding the outers shortens the windows so much the measure stops distinguishing anything. The choice is stated in the page, not hidden in the code.
- `VOC_ANG` holds **signed** angles (`0, ±60, ±90, ±120, 180`). Testing only the five nominal values misses every aspect formed on the other side — separations of 240°, 270°, 300° — which silently halved the aspect count and reported the Moon void when it was not. The bug shipped in the first draft of this function; the fix is verified by an independent sweep.
- The scan covers the Moon's **whole stay in the sign**, ingress included, because the void's start is the last aspect — which has usually already happened. A 0.05-day step then bisection; `|g| < 10` guards the `sgn180` discontinuity, which lands on the conjunction when testing the opposition.
- Costs about 9 ms, so it is computed once in `loadReading()` and passed to both `localReading(tr,hits,voc)` and `buildPrompt(tr,hits,jd,voc)`. It is computed **only for `PERIOD==='jour'`** — over a week or a month the window opens and closes a dozen times and the fact is meaningless.
- Windows with the classical set are long: median 13 h over a 60-day sample, max 37 h. That maximum was brute-force checked (no separation comes within 0.02° of an aspect anywhere in the claimed window) rather than assumed to be a bug.

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

- The button shows the theme it switches **to**, not the current one: a crescent means "go dark". The icons are drawn inline (`TICO`), not ☀/☾, for the reason recorded under Karmic points — and an icon-only button has no text to fall back on if the glyph turns into an empty box. A crescent beats a half-disc at 15 px: the terminator of a half moon is a straight line that reads as a plain bisected circle once the rim stroke thickens against it.
- The preference is applied by a **second inline `<script>` in `<head>`**, read from `localStorage` directly rather than through `store`: the main script runs at the end of the body, so going through it would show the light palette for a frame before switching. That is the only reason the app has two script tags. It sets nothing when no preference is stored, which is what lets the media query paint the right palette on a first visit.
- **Every colour must be a token.** A literal anywhere outside `:root` survives the switch and will be wrong in one theme. Tints compose as `rgb(var(--v-rgb) / .13)` from raw triplets, and `--shadow` / `--ring` are separate tokens because a shadow built from `--ink` glows once the ink turns pale. `--disc` is the wheel's core tint, raised in dark where 3.5 % is invisible. The JS colours in `ASPECTS`, the agenda and the gauge are `var(--…)` strings for the same reason — inline SVG resolves them.
- The dark palette is not an inversion: `#EFE9DA` on `#1B2A44` vibrates, so ink and vellum are re-picked, and verdigris and minium are lightened to hold contrast.

### Accessibility

The share menu (`#shlist`) declares `role="menu"`, which is a contract: arrow keys cycle entries and wrap, Home/End jump to the ends, Escape closes and returns focus to `#shtog`, hidden entries are skipped, and entries carry `tabindex="-1"` so only the button sits in the tab order. `#combolist` implements the same pattern. If you add a menu, match it or drop the roles — declaring the role without the keyboard behaviour is worse than no role.

### Language

UI and all reading text are in **French**.
