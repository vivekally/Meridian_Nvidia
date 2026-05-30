# Meridian — True Cost of Ownership Agent

> The price tag is the least honest number in the room.

Meridian turns a Toronto listing into the true 10-year cost of ownership by combining public city data, deterministic financial logic, and a buyer-facing summary generated on local NVIDIA hardware. It surfaces 43 hidden cost parameters across 29 Toronto Open Data datasets, flags risks that list price never shows, and gives buyers negotiation leverage backed by documented data.

Built for **NVIDIA Spark Hack Toronto** (May 31, 2026). Runs locally on the ASUS GX10 (DGX Spark) via NVIDIA NIM.

---

## Documents in this repo

| File | Description |
|------|-------------|
| [`DATA_PARAMETERS.md`](./DATA_PARAMETERS.md) | Complete inventory of 43 hidden cost parameters across 5 tiers, mapped to 29 CKAN datasets with confidence scores and spec alignment |
| [`CEO_Plan_V1_Hackathon.md`](./CEO_Plan_V1_Hackathon.md) | CEO-reviewed build plan — scope decisions, build schedule, phase gates, pitch strategy |
| [`Meridian_V3_Default_Prevention_-_Final_Idea_Doc.md`](./Meridian_V3_Default_Prevention_-_Final_Idea_Doc.md) | Full product thesis — V1/V2/V3 roadmap, judging criteria analysis, default cascade framing |
| [`Office_Hours_Gap_Analysis_and_Build_Strategy.md`](./Office_Hours_Gap_Analysis_and_Build_Strategy.md) | Gap analysis — architecture changes, prompt skeletons, inter-agent data contracts, build schedule |
| [`DESIGN.md`](./DESIGN.md) | Design system — Mercury-inspired dark theme, typography, color palette, spacing |
| [`TODOS.md`](./TODOS.md) | Deferred items — RAPIDS cuDF, model upgrade, data expansion |

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
Agent 1 — Intake (deterministic)
  → Resolves address → parcel, ward, zoning code
    │
Agent 2 — Data Retrieval (deterministic, parallel async)
  → Heritage (CKAN) | RentSafeTO (CKAN) | Dev Apps (CKAN)
  → Permits (CKAN) | LTT (hardcoded)  | TRCA Flood (ArcGIS)
    │
Agent 3 — Cost Computation (deterministic)
  → LTT, property tax, mortgage scenarios, risk-weighted signals
    │
Agent 4 — Synthesis (LLM — local on GB10)
  → Plain-English report with confidence scores and negotiation leverage
    │
Output: True 10-year cost, hidden flags, confidence per source
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
| LLM (Agent 4) | Mistral Medium 3.5 or Qwen 3 7B via NVIDIA NIM |
| Spatial queries | GeoPandas + Shapely |
| Data fetch | httpx (async) |
| Frontend | Streamlit |
| Data sources | 29 Toronto Open Data (CKAN) + TRCA ArcGIS |

---

## Roadmap

- **V1 (Hackathon):** 10 core parameters + 2 composite signals + visible reasoning trace
- **V1.5 (Post-hackathon polish):** Transit Dividend, fire inspections, heritage formerly listed, tax relief programs
- **V2 (July 2026):** Personal affordability layer via Flinks (bank-verified income, emergency runway, rate renewal stress test)
- **V3 (Q4 2026):** Full buy vs. rent decision engine with neighbourhood recommendations
