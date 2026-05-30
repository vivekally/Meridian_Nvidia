# Design System — Meridian

## Product Context
- **What this is:** AI agent that takes a Toronto property address and produces a true 10-year cost of ownership report with hidden risk flags and negotiation leverage in dollar amounts
- **Who it's for:** First-time home buyers and mortgage renewers in Toronto facing high-stakes purchase decisions
- **Space/industry:** Fintech / real estate risk intelligence
- **Project type:** Web app / dashboard (Streamlit V1, React V3)
- **Memorable thing:** "I have leverage I didn't know existed." The user feels like they just hired a lawyer, not ran a calculator.

---

## Aesthetic Direction
- **Direction:** Mercury-style dark fintech
- **Primary inspiration:** [mercury.com](https://mercury.com) — exact colors, typography weight, transitions, and surface hierarchy extracted via CSS inspection
- **Mood:** Refined, serious, trustworthy. The product feels like institutional-grade software for personal use. Not playful, not cold. Premium.
- **Decoration level:** Minimal. No gradients, no decorative elements, no brass/gold accents. Color comes from semantic meaning only (green = leverage, amber = caution, red = danger, indigo = interactive).
- **Anti-patterns:** No warm-toned backgrounds. No serif display fonts. No visible card border grids. No skeleton loaders. No decoration for its own sake.

---

## Typography

- **All text (headings, body, UI, nav):** DM Sans variable
  - Closest open-source match to Mercury's proprietary `arcadia` typeface
  - Variable weight axis: 100-1000. Key weights: 360 (body), 420 (nav/buttons), 480 (headings), 500 (labels), 600 (strong emphasis)
  - `-webkit-font-smoothing: antialiased` always
- **Financial figures / data / reasoning trace:** Geist Mono
  - All dollar amounts: `font-variant-numeric: tabular-nums`, `font-weight: 600`
  - The leverage hero number: `font-size: 64px`, `font-weight: 600`, `letter-spacing: -0.03em`
  - Rule: if it's a number that could change someone's life, it gets Geist Mono at maximum visual weight

### Loading
```html
<link href="https://fonts.googleapis.com/css2?family=DM+Sans:ital,opsz,wght@0,9..40,100..1000;1,9..40,100..1000&display=swap" rel="stylesheet">
<link href="https://fonts.googleapis.com/css2?family=Geist+Mono:wght@400;500;600&display=swap" rel="stylesheet">
```

### Scale
| Level | Size | Weight | Font | Usage |
|-------|------|--------|------|-------|
| Hero | 64px | 600 | Geist Mono | Leverage dollar amount |
| H1 | 32px | 480 | DM Sans | Page heading (if needed) |
| H2 | 16px | 480 | DM Sans | Card/flag titles |
| Body | 15-16px | 360 | DM Sans | Descriptions, explanations |
| Body SM | 14px | 360 | DM Sans | Table cells, secondary text |
| UI | 14px | 420 | DM Sans | Buttons, nav items |
| Label | 11px | 500 | DM Sans | Section labels, uppercase captions |
| Mono LG | 20px | 600 | Geist Mono | Cost stat values |
| Mono MD | 13px | 500 | Geist Mono | Table financial cells |
| Mono SM | 12px | 400 | Geist Mono | Reasoning trace text |

---

## Color

- **Approach:** Mercury exact palette. Restrained. Semantic colors only.
- **Default mode:** Dark. Light mode available via toggle.

```css
:root {
  /* Mercury exact background hierarchy */
  --bg:               #171721;   /* rgb(23,23,33) — page background */
  --surface:          #1E1E2A;   /* rgb(30,30,42) — cards, modals */
  --surface-raised:   #272735;   /* rgb(39,39,53) — hover states, section bg */
  --surface-frosted:  rgba(112,115,147,0.22); /* frosted glass overlays */

  /* Mercury exact borders */
  --border:           rgba(175,178,206,0.36); /* card borders */
  --border-faint:     rgba(175,178,206,0.14); /* dividers, separators */
  --border-focus:     rgba(156,180,232,0.5);  /* focus rings */

  /* Mercury exact text hierarchy */
  --text-primary:     #EDEDE3;   /* rgb(237,237,243) — headings, primary content */
  --text-secondary:   #C3C3CC;   /* rgb(195,195,204) — body text, descriptions */
  --text-muted:       #70707D;   /* rgb(112,112,125) — labels, captions */

  /* Mercury exact accent — indigo */
  --accent:           #5266EB;   /* rgb(82,102,235) — CTA buttons */
  --accent-tint:      rgba(156,180,232,0.2);  /* ghost buttons, hover bg */
  --accent-subtle:    rgba(82,102,235,0.12);  /* badge bg */
  --accent-light:     #CDDDFF;   /* rgb(205,221,255) — accent text on dark */

  /* Meridian semantic (tuned for Mercury dark palette) */
  --green:            #34D399;   /* leverage / positive signals */
  --green-tint:       rgba(52,211,153,0.12);
  --amber:            #FBBF24;   /* urgency / medium confidence */
  --amber-tint:       rgba(251,191,36,0.12);
  --red:              #F87171;   /* danger / low confidence */
  --red-tint:         rgba(248,113,113,0.12);
}

/* Light mode */
.light-mode {
  --bg:               #F4F5F9;
  --surface:          #FFFFFF;
  --surface-raised:   #F4F5F9;
  --surface-frosted:  rgba(112,115,147,0.06);
  --border:           rgba(112,115,147,0.22);
  --border-faint:     rgba(112,115,147,0.12);
  --text-primary:     #13131E;
  --text-secondary:   #3D3D52;
  --text-muted:       #70707D;
}
```

### Confidence indicator system
| Level | Color | Badge style | Meaning |
|-------|-------|-------------|---------|
| HIGH | `#34D399` green | Frosted pill, green tint bg | HTTP 200 + `last_updated` field present |
| MEDIUM | `#FBBF24` amber | Frosted pill, amber tint bg | HTTP 200, no recency metadata |
| LOW | `#F87171` red | Frosted pill, red tint bg | HTTP 200, empty results array |
| UNKNOWN | `#5266EB` indigo | Frosted pill, accent tint bg | Timeout >5s, 4xx, or 5xx |

---

## Spacing

- **Base unit:** 8px
- **Density:** Comfortable. Mercury-level breathing room.
- **Scale:** 4px / 8px / 12px / 16px / 20px / 24px / 28px / 32px / 48px / 64px / 96px

### Border radius (Mercury exact)
| Token | Value | Usage |
|-------|-------|-------|
| `--r-sm` | 8px | Small badges |
| `--r-md` | 12px | Inputs, script blocks, nested cards |
| `--r-lg` | 16px | Main cards, flag cards, table wrapper |
| `--r-xl` | 24px | Verdict hero card |
| `--r-pill` | 40px | Buttons, confidence badges (Mercury exact) |

---

## Layout

- **Approach:** Verdict-first, single reading column
- **Max content width:** 860px centered
- **Nav height:** 72px (Mercury exact)
- **Verdict positioning:** The leverage dollar amount (64px Geist Mono, green) is the FIRST thing rendered above the fold

### Page structure (verdict screen)
```
NAV (floating, blur, semi-transparent — Mercury pattern)
├── Logo mark + wordmark
├── Address input + price input (frosted glass inputs)
└── Re-analyze button (ghost) + theme toggle

MAIN (860px centered, 48px top padding)
├── VERDICT CARD (24px radius, surface bg, subtle green radial glow)
│   ├── Risk pill (top-right, red tint bg)
│   ├── Address line (muted, uppercase, 11px)
│   ├── Preamble ("You have negotiation leverage...")
│   ├── $55,000–$75,000 (64px Geist Mono, green)
│   ├── Tagline (italic, secondary text)
│   └── Stats row: True Cost | List Price | Hidden Premium (border-top divider)
│
├── SECTION LABEL ("Risk flags" — muted uppercase + faint line)
├── FLAG CARDS (accordion expand/collapse — Mercury section-link pattern)
│   ├── Colored left stripe (3px, red/amber/green)
│   ├── Title (16px, weight 480) + confidence badge + chevron
│   └── Body (hidden until expanded):
│       ├── Description (15px, secondary text)
│       └── NEGOTIATION SCRIPT block (raised surface bg)
│           ├── "Say this at the table" (accent-light, uppercase 11px)
│           ├── Script text (italic, primary text)
│           └── Leverage amount (Geist Mono, green)
│
├── SECTION LABEL ("10-Year cost breakdown")
├── COST TABLE (surface bg, raised header, hover rows)
│   └── Per-row confidence badge (frosted pill)
│
├── SECTION LABEL ("Agent reasoning")
└── REASONING TRACE (accordion, collapsed by default)
    └── Per-agent entries (Geist Mono, muted text)
```

---

## Motion

- **Easing:** `cubic-bezier(0,0,.6,1)` (Mercury exact)
- **Duration:** `0.5s` for theme/bg transitions, `0.2s` for hover/focus, `0.15s` for micro-interactions
- **Report appearance:** Composed artifact, no loading animation
- **Accordion flags:** `display: none → block` (no height animation — Mercury pattern)
- **Hover on table rows:** Frosted glass bg appears on hover
- **Button hover:** `opacity: 0.88`, `translateY(-1px)`
- **Never:** Skeleton screens, progress bars, spinning loaders

---

## Key Design Decisions

### Mercury as the design source
Mercury.com was chosen because it solves the same trust problem: users are making high-stakes financial decisions and need to feel the software is institutional-grade. Mercury's dark palette, restrained typography, and near-invisible borders create authority without coldness.

### DM Sans replaces Fraunces + Schibsted Grotesk
One variable font family for everything. Mercury uses a single sans-serif (arcadia) across headings and body. DM Sans is the closest open-source equivalent. This eliminates font-loading complexity and creates visual unity. Financial figures remain in Geist Mono for tabular alignment.

### Indigo accent replaces green/brass
Mercury uses `#5266EB` indigo for all interactive elements (CTAs, ghost buttons, focus states). Green is reserved strictly for positive/leverage semantic meaning. Brass has been removed entirely. This is cleaner and more systematic.

### Accordion risk flags
Mercury's section-link pattern (click to expand, chevron rotation) replaces the always-visible card layout. This keeps the page scannable and puts the negotiation scripts one click away instead of forcing a long scroll.

### 40px pill buttons
Mercury uses 40px `border-radius` on all buttons and badges. This is the most distinctive visual element and should be preserved exactly.

---

## Decisions Log

| Date | Decision | Rationale |
|------|----------|-----------|
| 2026-05-30 | Initial system: Fraunces + Schibsted Grotesk + warm parchment | Competitive research showed no visual precedent for buyer's advocate fintech |
| 2026-05-30 | Rebuilt with Mercury.com exact design language | User preference. Mercury's dark fintech palette is a better fit for high-stakes financial decisions |
| 2026-05-30 | DM Sans replaces Fraunces + Schibsted Grotesk | Mercury uses single sans-serif (arcadia). DM Sans is closest open-source match |
| 2026-05-30 | Background: #171721 (Mercury exact rgb 23,23,33) | Replaces warm parchment and near-black attempts |
| 2026-05-30 | Accent: #5266EB indigo (Mercury exact) | Replaces brass (#A07840) for interactive elements |
| 2026-05-30 | Semantic colors: green #34D399, amber #FBBF24, red #F87171 | Tuned for contrast on Mercury dark palette |
| 2026-05-30 | Button radius: 40px pill (Mercury exact) | Most distinctive Mercury visual element |
| 2026-05-30 | Risk flags: accordion pattern | Mercury section-link expand/collapse — keeps page scannable |
| 2026-05-30 | Leverage number: 64px Geist Mono (not serif) | Mono tabular-nums at large scale reads as financial authority, not editorial |
