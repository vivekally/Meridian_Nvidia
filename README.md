# Meridian — The Default Prevention Agent

> Don't just calculate cost. Prevent financial ruin.

Meridian is an AI agent that takes any Toronto address and reveals the true 10-year cost of ownership, using 6 City of Toronto Open Data sources. It flags hidden risks — heritage designations, flood zones, development pressure, building permit red flags — that a list price never shows.

Built for **NVIDIA Spark Hack Toronto** (May 31, 2026). Runs locally on the ASUS GX10 (DGX Spark) via NVIDIA NIM.

---

## Documents in this repo

| File | Description |
|------|-------------|
| [`Meridian_V3_Default_Prevention_-_Final_Idea_Doc.md`](./Meridian_V3_Default_Prevention_-_Final_Idea_Doc.md) | Original product idea doc — full thesis, V1/V2/V3 roadmap, judging criteria analysis |
| [`Office_Hours_Gap_Analysis_and_Build_Strategy.md`](./Office_Hours_Gap_Analysis_and_Build_Strategy.md) | Gap analysis from /office-hours session — architecture changes, prompt skeletons, inter-agent data contracts, 24-hour build schedule |

---

## The Problem

Every existing real estate tool asks: "Can you afford the monthly payment?"

That's the wrong question. The right question is: **"Can you survive owning this property for 10 years, through rate renewal shocks, hidden assessments, and unexpected income disruption, without defaulting?"**

Toronto mortgage arrears climbed 45% year-over-year in 2025. 1.15 million Canadian mortgages are renewing in 2026 at rates well above what borrowers originally signed. Behind every arrears statistic is a family that bought based on list price.

Meridian breaks the default cascade at step 1.

---

## V1 Architecture (Hackathon Build)

```
Agent 1: Planning Agent (LLM-powered)
  → Infers property type, decides which data sources to query, explains why

Agent 2: Data Retrieval (deterministic, parallel)
  → Heritage Register, Building Permits, TRCA Flood Zone,
    Development Applications, RentSafeTO (6 Toronto Open Data sources)

Agent 3: Analysis Agent (LLM-powered)
  → Detects contradictions, rates confidence, computes 10-year costs

Agent 4: Synthesis Agent (LLM-powered)
  → Plain-English report with dollar amounts, risk flags, and verdict
```

The key differentiator: **visible reasoning trace**. Judges see the agent deciding what to check and why — not just results.

---

## Tech Stack

| Component | Technology |
|-----------|-----------|
| Runtime | Python 3.11 on ASUS GX10 (DGX Spark) |
| LLM | Llama 3.1 8B via NVIDIA NIM |
| Spatial queries | GeoPandas + Shapely |
| Data fetch | httpx (async) |
| Frontend | Streamlit |
| Data sources | Toronto Open Data (CKAN) + TRCA ArcGIS |

---

## Roadmap

- **V1 (Hackathon):** Hidden cost report with agent reasoning trace
- **V2 (July 2026):** Personal affordability layer via Flinks (bank-verified income, emergency runway, rate renewal stress test)
- **V3 (Q4 2026):** Full buy vs. rent decision engine with Transit Dividend scoring and neighbourhood recommendations
