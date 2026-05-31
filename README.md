# Meridian — True Cost of Ownership Agent

> The price tag is the least honest number in the room.

Meridian turns a Toronto listing into the true 10-year cost of ownership by combining public city data, deterministic financial logic, and a buyer-facing summary generated on local NVIDIA hardware. It surfaces 43 hidden cost parameters across 29 Toronto Open Data datasets, flags risks that list price never shows, and gives buyers negotiation leverage backed by documented data.

Built for **NVIDIA Spark Hack Toronto** (May 31, 2026). Runs locally on the ASUS GX10 (DGX Spark) via NVIDIA NIM.

---

## Documents in this repo

| File | Description |
|------|-------------|
| [`PRD_Backend_Build.md`](./PRD_Backend_Build.md) | **Single source of truth** — locked architecture, data contracts, agent specs, 24-hour build sequence |
| [`DATA_PARAMETERS.md`](./DATA_PARAMETERS.md) | 43-parameter inventory across 5 tiers, mapped to 29 CKAN datasets with confidence scores |
| [`DESIGN.md`](./DESIGN.md) | Mercury dark-theme design system — typography, color palette, spacing |
| [`TODOS.md`](./TODOS.md) | V1 UI details (P0), V1.5 data expansion, V2 backlog |
| [`meridian-prototype.html`](./meridian-prototype.html) | Working 3-screen prototype with chat panel |

---

## The Problem

Every existing real estate tool asks: "Can you afford the monthly payment?"

That's the wrong question. The right question is: **"Can you survive owning this property for 10 years, through rate renewal shocks, hidden assessments, and unexpected income disruption, without defaulting?"**

Toronto mortgage arrears climbed 45% year-over-year in 2025. 1.15 million Canadian mortgages are renewing in 2026 at rates well above what borrowers originally signed. Behind every arrears statistic is a family that bought based on list price.

Meridian breaks the default cascade at step 1.

---

## Three-Layer Architecture

Every signal Meridian produces is tagged with its type:

```
LAYER 1 — OBSERVED (direct dataset matches)
  Heritage designation, flood zone intersection, permit existence,
  RentSafeTO score, LTT calculation, fire inspections

LAYER 2 — INFERRED (computed from multiple observed signals)
  Maintenance complexity signal, future tax pressure signal,
  neighbourhood intensification score

LAYER 3 — SIMULATED (projection scenarios, not forecasts)
  10-year property tax projection, mortgage scenario modeling (base/bear/bull),
  rate shock scenarios, transit dividend
```

### Agent Pipeline

```
User Input (address, list price, buyer profile)
    │
Agent 1 — Intake + Planning
  → deterministic: geocode via Address Points CKAN → lat/lon
  → LLM call #1: classify property type, emit QueryPlan (query/skip per source)
    │ QueryPlan
Agent 2 — Data Retrieval (deterministic, async, NO LLM)
  → Parallel httpx fanout to sources marked query=true. 5s timeout each.
  → Heritage/HCD/Flood: local GeoJSON + shapely point-in-polygon
  → Permits/Dev Apps/RentSafeTO: CKAN bbox-SQL + haversine
    │
Agent 3 — Analysis + Cost (deterministic, NO LLM)
  → LTT, property tax, mortgage scenarios, 10 signals, 2 composite signals
  → ALL dollar math lives here. LLM never computes money.
    │
Agent 4 — Synthesis (LLM call #2 — Llama 3.1 8B via NIM, or Claude toggle)
  → Plain-English report + negotiation leverage per elevated flag
  → Conditional RAG from local offline Chroma index
    │
Output: True 10-year cost, hidden flags, confidence per source, negotiation scripts
```

---

## Data Sources (29 Toronto Open Data + 1 External)

| Category | Datasets | Signal type |
|----------|----------|-------------|
| Heritage | Heritage Register (12,320), HCDs, Formerly Listed | Observed |
| Building health | Active Permits (228K), Cleared Permits (401K), Violations (48K), Fire Inspections (124K), RentSafeTO (5,340) | Observed |
| Development pressure | Dev Applications (26K), Dev Pipeline (2,411), CoA Applications (36K), Zoning Reviews (220K) | Observed/Inferred |
| Environmental | TRCA Flood (ArcGIS), Basement Flooding (67 subsewersheds), Ravine Protection, ESAs (89) | Observed |
| Financial | LTT brackets, Property tax rates, CMHC premiums, FHSA/HBP rules | Observed/Simulated |
| Transit | TTC GTFS routes/stops, Subway ridership | Simulated |
| Safety | Crime rates (158 neighbourhoods), KSI collisions (20K), Noise permits (3,685) | Observed |

Full parameter inventory with confidence scores: [`DATA_PARAMETERS.md`](./DATA_PARAMETERS.md)

---

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Runtime | Python 3.11 on ASUS GX10 (DGX Spark) |
| LLM default | `meta/llama-3.1-8b-instruct` via NVIDIA NIM (OpenAI-compatible) |
| LLM quality toggle | Claude (`MERIDIAN_LLM_PROVIDER=quality`) |
| Spatial queries | shapely (point-in-polygon) — no GeoPandas/GDAL dependency |
| Data fetch | httpx (async) |
| Vector store | Chroma (local, offline) + NV-Embed / bge-small embeddings |
| Frontend | Streamlit |
| Data sources | 29 Toronto Open Data (CKAN) + TRCA ArcGIS |

---

## Roadmap

- **V1 (Hackathon):** 10 core parameters + 2 composite signals + visible reasoning trace
- **V1.5 (Post-hackathon polish):** Transit Dividend, fire inspections, heritage formerly listed, tax relief programs
- **V2 (July 2026):** Personal affordability layer via Flinks (bank-verified income, emergency runway, rate renewal stress test)
- **V3 (Q4 2026):** Full buy vs. rent decision engine with neighbourhood recommendations
