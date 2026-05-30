# Design System — Meridian

## Product Context
- **What this is:** AI agent that takes a Toronto property address and produces a true 10-year cost of ownership report with hidden risk flags and negotiation leverage in dollar amounts
- **Who it's for:** First-time home buyers and mortgage renewers in Toronto facing high-stakes purchase decisions
- **Space/industry:** Fintech / real estate risk intelligence
- **Project type:** Web app / dashboard (Streamlit V1, React V3)
- **Memorable thing:** "I have leverage I didn't know existed." — The user feels like they just hired a lawyer, not ran a calculator.

---

## Aesthetic Direction
- **Direction:** Due Diligence Room
- **Decoration level:** Intentional — warm parchment base, document-like brass dividers, purposeful confidence indicators. Not a SaaS dashboard.
- **Mood:** A private law firm memo meets a Bloomberg terminal. Institutional gravity + human warmth. Serious but not cold. The product was prepared FOR the user by someone who knows more than the seller's agent.
- **Anti-patterns:** No 3-column icon grids. No gradient buttons. No cool-gray (#F3F4F6) backgrounds. No skeleton screens. The report appears as a composed artifact — like it was printed and handed to you.
- **EUREKA:** Every real estate tool in this category (Opendoor, Zillow, Redfin) is designed for sellers or browsers, not buyers doing due diligence. Meridian has no direct category precedent. Lean into this: document-first, verdict-first, not app-first.

---

## Typography

- **Display/Hero (verdicts, leverage numbers, section titles):** Fraunces variable
  - Use the `opsz` optical size axis deliberately. Large sizes: `font-variation-settings: 'opsz' 90`, `font-weight: 300`. Small heading sizes: `font-variation-settings: 'opsz' 18`, `font-weight: 600`.
  - The leverage number gets `font-size: 56px`, `font-weight: 300`, `opsz: 90`. Most real estate tools never use a variable font axis — we do.
- **Body / UI labels / nav chrome:** Schibsted Grotesk
  - Institutional grotesque. Reads like a document, not a startup. Used by Norwegian editorial institutions. Square geometry that signals "this is serious information."
  - Replaces Plus Jakarta Sans from prior mockup work.
- **Financial data / reasoning trace / confidence labels:** Geist Mono
  - All dollar figures: `font-variant-numeric: tabular-nums`, minimum `font-size: 18px`, `font-weight: 500` or `600`.
  - Rule: if it's a number that could change someone's life, it must be the most visually weighted thing on the page.
- **Code / reasoning trace text:** Geist Mono (same family, weight 400)

### Loading
```html
<link href="https://fonts.googleapis.com/css2?family=Fraunces:ital,opsz,wght@0,9..144,100..900;1,9..144,100..900&family=Geist+Mono:wght@400;500;600&display=swap" rel="stylesheet">
<link href="https://fonts.googleapis.com/css2?family=Schibsted+Grotesk:ital,wght@0,400;0,500;0,600;0,700;1,400&display=swap" rel="stylesheet">
```

### Scale
| Level | Size | Weight | Font | Usage |
|-------|------|--------|------|-------|
| Hero | 56px | 300 | Fraunces (opsz 90) | Leverage number |
| H1 | 40px | 400 | Fraunces (opsz 60) | Page title |
| H2 | 28px | 600 | Fraunces (opsz 28) | Section title, flag name |
| H3 | 20px | 600 | Fraunces (opsz 20) | Card title |
| Body | 15px | 400 | Schibsted Grotesk | Explanations, descriptions |
| Body SM | 14px | 400 | Schibsted Grotesk | Table cells, secondary content |
| Label | 12px | 600 | Schibsted Grotesk | Field labels, uppercase captions |
| Mono XL | 22–36px | 600 | Geist Mono | Large financial figures |
| Mono MD | 18px | 500 | Geist Mono | Inline cost figures |
| Mono SM | 13px | 400 | Geist Mono | Table rows, trace text |
| Caption | 10–11px | 600 | Geist Mono | Confidence badge text, section divider labels |

---

## Color

- **Approach:** Restrained — one primary accent (green), one urgency color (amber), danger (red), one distinctive detail color (brass). Color is meaningful, never decorative.

```css
:root {
  /* Base — warm parchment document, not cool SaaS gray */
  --bg:              #F5F0E8;
  --surface:         #FAF7F2;
  --surface-raised:  #FFFFFF;
  --border:          #DDD5C4;
  --border-light:    #EDE8DF;

  /* Primary — forest green, richer/darker than typical fintech */
  --green:           #0F6B30;
  --green-mid:       #15803D;  /* hover states */
  --green-hover:     #0A5625;
  --green-wash:      #EAF4EE;  /* tint backgrounds */
  --green-text:      #052E16;  /* text on green surfaces */

  /* Urgency — FHSA deadlines, RRSP warnings, permit issues. Never casual. */
  --amber:           #B45309;
  --amber-wash:      #FEF3C7;

  /* Danger — existential risk flags */
  --red:             #991B1B;
  --red-wash:        #FEE2E2;

  /* Brass — confidence indicators, section dividers, subtle iconography */
  /* Evokes legal nameplates and institutional hardware. Never overuse. */
  --brass:           #A07840;
  --brass-light:     #F0E8D8;

  /* Typography */
  --ink:             #1C1208;  /* warm near-black, not cool #111827 */
  --ink-mid:         #4A3F2F;
  --ink-muted:       #8A7B68;
  --ink-faint:       #C4B89E;
}
```

### Dark mode
Reduce saturation 10–20% on all accent colors. Parchment base inverts to `#1A1610`. Surface to `#231F18`. Brass lightens to `#C89650`.

### Confidence indicator system
| Level | Color | Dot | Meaning |
|-------|-------|-----|---------|
| HIGH | Green (#0F6B30) | Green dot | HTTP 200 + `last_updated` field present |
| MEDIUM | Amber (#B45309) | Amber dot | HTTP 200, no recency metadata |
| LOW | Red (#991B1B) | Red dot | HTTP 200, empty results array |
| UNKNOWN | Brass (#A07840) | Brass dot | Timeout >5s, 4xx, or 5xx |

---

## Spacing

- **Base unit:** 8px
- **Density:** Comfortable — legal-document breathing room, not cramped dashboard
- **Scale:** 2px / 4px / 8px / 12px / 16px / 24px / 32px / 40px / 48px / 64px / 96px

### Border radius
| Token | Value | Usage |
|-------|-------|-------|
| `--r-sm` | 4px | Badges, small inline elements |
| `--r-md` | 8px | Inputs, buttons, small cards |
| `--r-lg` | 12px | Main cards, app shell |
| `--r-full` | 9999px | Pills, confidence badges |

---

## Layout

- **Approach:** Verdict-first, single reading column. NOT a 3-column dashboard grid.
- **Principle:** The product reads like a brief, not a BI tool. The buyer sees the verdict before they see the methodology.
- **Max content width:** 800px centered in the app body
- **Grid:** Single column with optional 280px evidence sidebar after scroll on wide screens
- **Verdict positioning:** The leverage dollar amount (`56px Fraunces`) is the FIRST thing rendered above the fold — before any table rows, before any flags.

### Page structure (verdict screen)
```
NAV BAR (address input + re-analyze button)
├── VERDICT HERO (full-width card)
│   ├── leverage number — 56px Fraunces green
│   ├── tagline — italic Fraunces mid
│   └── cost trio (true cost / list price / hidden premium) — Geist Mono
├── SECTION DIVIDER — brass label + horizontal rule
├── RISK FLAG CARDS (one per finding)
│   ├── flag title — 18px Fraunces
│   ├── body — Schibsted Grotesk
│   └── NEGOTIATION SCRIPT (embedded card)
│       ├── "Say this at the table ↓" — brass mono caption
│       ├── exact script — italic Schibsted
│       └── leverage range — Geist Mono green
├── SECTION DIVIDER — brass label + horizontal rule
├── 10-YEAR COST TABLE
│   └── per-row confidence badge (HIGH/MEDIUM/LOW/UNKNOWN)
└── AGENT REASONING TRACE (collapsed by default)
    └── "How we know this ↓" — secondary action, not hero
```

---

## Motion

- **Approach:** Intentional — transitions aid comprehension. No skeleton screens.
- **Report appearance:** The completed report renders as a composed artifact, not a loading animation. Use a single fade-in (`opacity: 0 → 1`, `200ms ease-out`) on the full report container when analysis completes.
- **Risk flag entrance:** Each flag card enters with `translateY(8px) → translateY(0)` staggered 60ms apart.
- **Easing:** `enter: ease-out` / `exit: ease-in` / `move: ease-in-out`
- **Durations:** micro: 50–100ms / short: 150–250ms / medium: 250–400ms
- **Never:** Progress bars during analysis. Spinning loaders. Animated skeleton screens. Use `st.status()` expanding sections in Streamlit to show live agent progress.

---

## Key Design Decisions

### Negotiation scripts, not findings
Standard risk tools write: "Heritage designation detected. Confidence: HIGH."

Meridian writes: "Say this at the table: 'The 1987 Part IV heritage designation adds 40–60% to any future renovation. I'd like to reflect that in the price.'"

Every RED flag card contains the exact sentence the buyer needs at the negotiation table. The underlying source data is in a secondary expand. This is the product's core differentiator.

### Leverage number as hero
The dollar amount ($55,000–$75,000) is the entire emotional payload of the product. It goes in 56px Fraunces, above the fold, before any evidence. We are selling confidence, not defensiveness.

### Brass as the distinctive material
Blue is the universal "info" color — every fintech tool uses it. Brass evokes legal nameplates, institutional hardware, the material of serious institutions. Used sparingly: confidence indicator dots, section divider labels, the "say this at the table" caption. One distinctive material choice makes the system memorable.

---

## Decisions Log

| Date | Decision | Rationale |
|------|----------|-----------|
| 2026-05-30 | Initial system created by /design-consultation | Competitive research found no visual precedent for buyer's-advocate fintech — leaned into document/legal-brief aesthetic |
| 2026-05-30 | Schibsted Grotesk replaces Plus Jakarta Sans | Institutional grotesque reads "document," not "startup" — aligns with "like hiring a lawyer" memorable thing |
| 2026-05-30 | Warm parchment (#F5F0E8) replaces cool gray (#F3F4F6) | Every competitor is cool-gray SaaS; parchment makes Meridian read as a prepared document |
| 2026-05-30 | Brass accent (#A07840) added | Evokes legal/institutional nameplates; replaces blue for confidence indicators; one memorable distinctive element |
| 2026-05-30 | Leverage number at 56px above fold | Selling confidence — reveal the punchline first, evidence below. No enterprise team would ship it this way. |
| 2026-05-30 | Risk flags as negotiation scripts | Transforms research tool into negotiation preparation room; Agent 4 synthesis already planned to support this |
