# Wedding Invitation — Reference Spec & Build Blueprint

A reusable, content-agnostic blueprint for building a single-page digital wedding
invitation, reverse-engineered from the reference site
**https://sahana-weds-manoj.github.io/**.

Target output: **your** customised invitation — same architecture, your content.
Languages scoped to **English (default) + Telugu**.

> Notation: `[[SLOT: name]]` marks a content slot you fill at build time.
> Everything else is structure/behaviour to keep.

---

## 1. Overview, tech stack & file structure

**What it is:** a single-page, mobile-first invitation that opens with a language
splash, then an "open the invitation" gate, then a vertical scroll of themed
sections with a live countdown, calendar/maps/share deep-links, and background music.

**Tech stack (keep as-is):**
- One self-contained `index.html` (~128 KB in the reference).
- **Vanilla HTML + CSS + JS.** No framework, no bundler, no build step.
- **All artwork is inline SVG** — zero `<img>` tags, zero background images.
  (Reference: 24 inline SVGs, ~919 DOM nodes total.)
- One Google Fonts `<link>`. One audio file. That's every external request.
- Hosts free on GitHub Pages (or any static host).

**Why keep this stack:** zero dependencies, trivially hostable, instant load,
nothing to maintain or patch. It's the laziest approach that fully delivers the
experience.

**Proposed file structure:**
```
/index.html            ← everything: markup, <style>, <script>, inline SVG
/audio/music.mp3       ← [[SLOT: background music track]]
/favicon / og-image    ← optional, for link previews
```
Optionally split `styles.css` and `app.js` out of `index.html` for editing
comfort, but a single file matches the reference and keeps hosting dead simple.

**`<head>` essentials:**
```html
<meta name="viewport" content="width=device-width,initial-scale=1,viewport-fit=cover">
<title>[[SLOT: Partner A]] & [[SLOT: Partner B]] Wedding</title>
<link rel="preconnect" href="https://fonts.googleapis.com">
<link rel="preconnect" href="https://fonts.gstatic.com" crossorigin>
<link rel="stylesheet" href="https://fonts.googleapis.com/css2?family=...&display=swap">
```
`viewport-fit=cover` + safe-area insets handle phone notches.
Add Open Graph tags (`og:title`, `og:description`, `og:image`) so the WhatsApp
share shows a rich preview.

---

## 2. Design tokens

Define all of these as CSS custom properties on `:root`. Values below are the
reference's exact tokens — your starting palette; swap hues to rebrand.

### Colour
```css
:root{
  /* Greens (deep → mid) */
  --emerald:   #0c2a1c;   /* page background, darkest */
  --emerald-2: #123d28;   /* cards, filled buttons */
  --emerald-3: #1b5537;   /* accents, borders */

  /* Gold (shadow → base → highlight) */
  --gold-lo:   #8a6823;
  --gold:      #c8a04a;
  --gold-hi:   #f1d98f;

  /* Festive accents */
  --marigold:  #ef8a1d;   /* toran flowers */
  --maroon:    #8b1d2c;   /* sacred invocation text */

  /* Parchment / ivory (card surfaces) */
  --ivory:     #fcf7ec;
  --ivory-2:   #f3e9d2;

  /* Ink (body text on ivory) */
  --ink:       #2a2214;
  --ink-soft:  #6a5a3b;
}
```
Palette logic: deep green canvas → ivory "paper" card → green+gold for structure
→ maroon reserved for the sacred line → marigold only in the floral toran.
Gold is a **3-stop set** so borders/text can use a gradient for a metallic feel
(`linear-gradient(var(--gold-hi),var(--gold),var(--gold-lo))`).

### Typography
Latin fonts + Telugu fonts (drop Kannada/Hindi from the reference's set):
```css
:root{
  --f-body:  'Cormorant Garamond', Georgia, 'Times New Roman', serif; /* body */
  --f-label: 'Marcellus SC', 'Cormorant Garamond', serif;            /* caps labels */
  --f-name:  'Pinyon Script', 'Cormorant Garamond', cursive;         /* the names */
  /* Telugu */
  --f-te-body: 'Noto Serif Telugu', serif;
  --f-te-disp: 'Suranna', 'Noto Serif Telugu', serif; /* Telugu display/names */
}
```
Google Fonts request (2-language version):
```
https://fonts.googleapis.com/css2?family=Cormorant+Garamond:ital,wght@0,500;0,600;0,700;1,500;1,600&family=Marcellus+SC&family=Pinyon+Script&family=Noto+Serif+Telugu:wght@400;600;700&family=Suranna&display=swap
```

Roles:
- **Pinyon Script** → the couple's names + ampersand (large, romantic script).
- **Marcellus SC** → section labels, button text, host names, small-caps headings
  (always letter-spaced).
- **Cormorant Garamond** → paragraphs, dates, addresses; italic for blessings.
- Telugu renders in **Suranna** (display) / **Noto Serif Telugu** (body) — see §7.

### Scale & rhythm
```css
:root{
  --name-size: clamp(3.3rem, 16vw, 4.7rem); /* fluid couple names */
  --body-size: 1.2rem;
  --small:     1rem;
  --lh:        1.5;
  --track:     .14em;  /* letter-spacing for caps labels */
}
```
Use `clamp()` for the hero names and major headings so type scales with the
viewport without breakpoints. Labels are UPPERCASE + `letter-spacing:var(--track)`.

---

## 3. Entry flow (two-stage gate)

Three states, controlled by JS; `<body>` carries state classes.

**State A — Language splash** (`#splash`)
- Centred: kalasha emblem (SVG) → invocation `॥ ... ॥` → couple names (gold,
  serif) → "Choose your language" in each offered script → a **grid of language
  buttons** (English, Telugu) → `🎧 Turn your volume up` hint.
- Dark green dotted-texture background (no ivory card yet).
- Clicking a language sets the active language, saves it, and advances to State B.

**State B — Envelope gate** (`#gate`)
- An ivory card with gold border: invocation + couple names + date + a single
  pill button **"Open the invitation" / "ఆహ్వానాన్ని తెరవండి"**.
- Behind it, the full invitation is already rendered but hidden/blurred.

**State C — Invitation** (main scroll)
- Clicking "Open" hides the gate, **unlocks scroll**, and **starts the music**
  (must be triggered by this user gesture — browsers block autoplay otherwise).

**Scroll lock:** while in A/B, `document.body.classList.add('locked')` with:
```css
body.locked{ overflow:hidden; height:100vh; }
```
Remove `locked` on gate open.

**Reveal animation:** reference fades/reveals the card on open; keep a subtle
fade/scale transition (respect `prefers-reduced-motion`).

---

## 4. Section-by-section anatomy

All sections live inside a single centred **"paper" card**: `max-width ≈ 560px`,
ivory background, gold frame, decorative top toran + bottom scalloped border.
On desktop the card centres with the dark-green dotted background showing in the
gutters; on mobile the card fills the width.

### 4.1 Hero
- Decorative **toran** (marigold flowers + mango-leaf festoon) across the top edge — SVG.
- **Kalasha emblem** in a gold-ringed circular badge (pot + coconut + mango leaves + maroon band).
- `॥ [[SLOT: invocation]] ॥` — maroon, Marcellus SC.
- `WEDDING INVITATION` — gold label, letter-spaced.
- `[[SLOT: Partner A]]` / `&` / `[[SLOT: Partner B]]` — Pinyon Script, `--name-size`, green; `&` in gold.
- `[[SLOT: date, e.g. SUNDAY, 29 NOVEMBER 2026]]` — small-caps serif.
- `[[SLOT: venue short line]]`.
- **Countdown card** (see §5) + two pill buttons: **View events** (calendar icon,
  scrolls to §4.3) and **Get directions** (map-pin icon, opens Maps).

### 4.2 The Couple
- `THE COUPLE` gold label.
- `[[SLOT: blessing paragraph]]` — italic Cormorant, centred.
- **Bride block:** `THE BRIDE` label → optional honorific prefix (small-caps, e.g.
  "Chi. Sow.") + `[[SLOT: bride full name]]` in Pinyon Script → `[[SLOT: parents line]]`.
- Gold **`&`** divider.
- **Groom block:** mirror of bride (`THE GROOM` + honorific + name + context).
- Closing line: `[[SLOT: e.g. Kindly grace the occasion and bless the couple.]]` (maroon).

### 4.3 Celebrations (events timeline)
- `[[SLOT: date range]]` gold label + `Celebrations` heading (script/serif).
- A vertical list of **event cards** (§5). Each: `[[SLOT: icon]]`, name, date,
  time (large), location, **Add to calendar** button.
- **Featured-event variant:** the main ceremony (reference: *Muhurtham*) uses the
  **inverted card** — filled `--emerald-2` background with gold text/border —
  to stand out among the ivory cards. Mark one event `featured`.
- Reference events (replace with yours): Haldi · Baraat · **Muhurtham (featured)** · Reception.

### 4.4 Venue
- `WHERE WE GATHER` label → `Venue` heading.
- `[[SLOT: venue name]]` (caps serif) → `[[SLOT: address]]` → `[[SLOT: note, e.g.
  "All events held here."]]`.
- Buttons: **Get directions** (filled green → Google Maps) + **Copy address** (outline → clipboard).
- No embedded map iframe in the reference — just the directions link. (Optional:
  add a static map image or iframe if you want one; costs an external request.)

### 4.5 Share
- `SPREAD THE JOY` label → `Share the invitation` heading.
- **WhatsApp** (filled green, WhatsApp glyph → `wa.me` prefilled) + **Copy link**
  (outline → clipboard).

### 4.6 Footer
- Decorative diamond divider (SVG).
- `[[SLOT: closing line, e.g. "Eagerly awaiting your gracious presence"]]` (italic).
- `[[SLOT: host names]]` (caps serif) + `[[SLOT: "and family"]]`.
- Scalloped gold bottom border closes the card (SVG).

---

## 5. Reusable components

**Buttons** — three variants, all pill-shaped (`border-radius:999px`), Marcellus SC
caps, optional leading icon:
- *Filled*: `--emerald-2` bg, gold text/border, gold-glow on hover.
- *Outline*: transparent bg, gold border + ink text.
- *Pill (hero)*: same as outline/filled, larger padding.

**Event card** — ivory surface, gold hairline border, rounded; left **icon badge**
(circular, gold ring, SVG glyph) + right text column (name / date / time / location
/ Add-to-calendar). `featured` modifier flips to filled green + gold text.

**Countdown** — dark green card, gold border, label (`[[SLOT: "Muhurtham in"]]`),
then four units **Days / Hours / Minutes / Seconds** (large ivory serif numerals +
caps labels). JS ticks every 1 s to a target datetime; see §6.

**Floating controls** (fixed, bottom-right, two circular buttons):
- *Language toggle* — shows the current language's glyph (`A` for English, `అ`
  for Telugu); tap opens a small popover list of languages.
- *Music toggle* — animated equalizer bars when playing; tap to pause/resume.

**Icon badge** — reused for emblem, event icons, and floating buttons: circle +
gold ring + inline-SVG glyph.

**Section label + heading pair** — gold letter-spaced caps label above a larger
serif/script heading; used to open every section.

**Dividers** — gold `&`, diamond-dot rule, scalloped border (all SVG).

---

## 6. Interactions & integrations

**Countdown (vanilla JS):**
```js
const target = new Date('[[SLOT: 2026-11-29T08:50:00+05:30]]').getTime();
setInterval(()=>{ /* diff → d/h/m/s → write into spans; show "Begun" at <=0 */ }, 1000);
```

**Add to calendar — Google Calendar `TEMPLATE` URL** (one per event):
```
https://calendar.google.com/calendar/render?action=TEMPLATE
  &text=<Event · Couple>
  &dates=YYYYMMDDTHHMMSS/YYYYMMDDTHHMMSS   (local wall-clock)
  &ctz=Asia/Kolkata                        ([[SLOT: timezone]])
  &location=<venue>
  &details=<multiline summary + maps link>
```
(URL-encode every field.) Optionally also offer a downloadable `.ics` for Apple/Outlook.

**Get directions — Google Maps search URL:**
```
https://www.google.com/maps/search/?api=1&query=<url-encoded venue + address>
```

**WhatsApp share — `wa.me`:**
```
https://wa.me/?text=<url-encoded message + site URL>
```

**Copy to clipboard** (link / address):
```js
await navigator.clipboard.writeText(text); // then show a "Copied" toast/state
```

**Music:** `<audio src="audio/music.mp3">`, `volume ≈ 0.85`. Start on the gate-open
gesture (not `autoplay`). Toggle via the floating button; animate the equalizer only
while playing.

**Smooth scroll:** "View events" scrolls to the Celebrations section
(`scrollIntoView({behavior:'smooth'})`).

---

## 7. Bilingual implementation (English + Telugu)

Mechanism used by the reference — keep it; it's simple and framework-free:

1. **Dictionary** — an inline JS object keyed by language, each holding every UI
   string by key:
   ```js
   const I18N = {
     en: { glyph:'A', open:'Open the invitation', invocation:'॥ ... ॥',
           days:'Days', hours:'Hours', /* ...all strings... */ },
     te: { glyph:'అ', open:'ఆహ్వానాన్ని తెరవండి', invocation:'॥ శ్రీ గణేశాయ నమః ॥',
           days:'[[SLOT]]', hours:'[[SLOT]]', /* ... */ }
   };
   ```
2. **Markup** — every translatable node gets a key:
   `<span data-t="days">Days</span>`. A `setLang(l)` walks `[data-t]` and writes
   `el.textContent = I18N[l][el.dataset.t]`.
3. **`setLang(l)`** also:
   - sets `document.documentElement.lang = (l==='en' ? 'en-IN' : l)`,
   - updates the floating button glyph (`I18N[l].glyph`),
   - persists: `localStorage.setItem('lang', l)` (reference key: `sm-lang`),
   - swaps fonts (below).
4. **Per-script fonts via CSS** — target `[lang]` so Telugu picks Telugu faces:
   ```css
   html[lang="te"] { --f-body:var(--f-te-body); }
   html[lang="te"] .name { font-family:var(--f-te-disp); }
   ```
5. **Default & persistence** — default to **English**; on load read
   `localStorage['lang']` to pre-highlight. The splash still shows each load
   (choosing is part of the experience); language is also switchable anytime via
   the floating control.

Content to translate per event: invocation, "wedding invitation", blessing, labels
(THE BRIDE/GROOM, THE COUPLE, countdown units, section labels, button text),
closing line, "and family". Names/dates render in the Telugu script too.

---

## 8. Responsive behaviour

- **Mobile-first, single column.** The invitation card is the canonical layout
  (reference tested at 375 px wide).
- **Desktop:** the card is width-capped (~560 px) and centred; the dark-green
  **dotted-texture background** fills the gutters. No separate desktop layout —
  the same card, just centred.
- Fluid type via `clamp()` means almost no breakpoints are needed.
- Buttons stack / wrap on narrow widths; two-up button rows collapse gracefully.
- Honour **safe-area insets** (notches) with `viewport-fit=cover` +
  `env(safe-area-inset-*)` padding on fixed elements (floating controls, splash).
- Respect `prefers-reduced-motion` for the reveal, countdown, and equalizer.

---

## 9. SVG artwork inventory

All hand-coded inline SVG — recreate or replace these (keep them inline to stay
single-file and image-request-free):

| Asset | Where | Notes |
|---|---|---|
| Kalasha emblem | splash, gate, hero | pot + coconut + mango leaves + maroon band, in gold ring |
| Toran (festoon) | top of card | marigold flowers + green mango-leaf strands |
| Mandala patterns | card background ("carve" motifs) | faint gold line-art |
| Event icons | each event card | one glyph per ceremony [[SLOT: pick per event]] |
| Floating-button glyphs | controls | language letter + equalizer bars |
| Dividers | between sections | gold `&`, diamond-dot rule |
| Scalloped border | card bottom | gold zig-zag frame close |
| Button icons | buttons | calendar, map-pin, WhatsApp, copy, link |

If you'd rather not hand-draw, swap in a consistent SVG icon set + a decorative
border set — but keep them inline and in the gold/green palette.

---

## 10. Content-slot checklist (fill for your event)

- [ ] Partner A & Partner B names (+ honorific prefixes if used)
- [ ] Invocation line (sacred/opening text) + emblem choice
- [ ] Wedding date + short venue line (hero)
- [ ] Countdown target datetime + timezone + label
- [ ] Blessing paragraph; bride & groom parents/context lines; closing line
- [ ] Events: name, icon, date, time, location, which one is `featured` (+ calendar fields each)
- [ ] Venue: full name, address, note; Maps query string
- [ ] Share message text + canonical site URL (for WhatsApp + Copy link)
- [ ] Host names + "and family"
- [ ] Background music file (`audio/music.mp3`) + volume
- [ ] Telugu translations for every `data-t` key (and Telugu rendering of names/dates)
- [ ] OG tags + favicon for link previews
- [ ] Palette/font swaps if rebranding away from emerald-gold

---

### Customisation notes
- **Rebrand:** change the colour tokens in §2 and the toran/emblem SVGs; the rest
  of the system (labels, buttons, cards) inherits automatically.
- **Add/remove a language later:** add a key to `I18N`, a button to the splash, and
  a `[lang]` font rule — no structural change.
- **Drop a feature cleanly:** music, countdown, and the language switcher are each
  isolated modules; removing one touches only its own markup + a small JS block.
- **Hosting:** push to a `<name>.github.io` repo (or any static host). No build.
