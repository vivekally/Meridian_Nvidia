# Meridian — Engineering PRD

**Product:** True Cost of Ownership Agent for Toronto Home Buyers
**Version:** V1 (Hackathon) + V1.5 (Post-hackathon polish)
**Deadline:** May 31, 2026 (NVIDIA Spark Hack Toronto — 24-hour build)
**Hardware:** ASUS GX10 (DGX Spark), 128GB unified memory, Blackwell GPU
**Status:** Pre-build. All planning docs, UI prototype, and data research complete. No application code written.

---

## 1. Product Summary

Meridian takes a Toronto address, list price, and buyer profile, then produces a true 10-year cost of ownership report. It queries 29 Toronto Open Data datasets and one external API to surface hidden risks (heritage designations, flood zones, structural permits, development pressure) that list price never shows, then converts each risk into negotiation leverage with dollar ranges.

**One-line pitch:** "The price tag is the least honest number in the room."

**User flow:**
1. Enter address, list price, buyer profile (first-time / investor / downsizer), down payment %, rate, amortization
2. Watch 4-agent reasoning trace execute in real-time
3. Receive: true 10-year cost, hidden flags with confidence, negotiation scripts per flag, cost composition chart, mortgage scenario ladder

---

## 2. Architecture

### 2.1 Three-Layer Signal Model

Every output signal is tagged with one of three types:

| Layer | Signal type | What it means | Examples |
|-------|------------|---------------|----------|
| 1 | `observed` | Direct dataset match or hardcoded constant | Heritage match, flood polygon intersection, permit existence, LTT calculation |
| 2 | `inferred` | Computed from multiple observed signals | Maintenance complexity signal, future tax pressure signal, intensification score |
| 3 | `simulated` | Projection scenario, not a forecast | 10-year property tax, mortgage scenarios (base/bear/bull), transit dividend |

### 2.2 Agent Pipeline

```
┌─────────────────────────────────────────┐
│  USER INPUT                             │
│  address · list_price · buyer_profile   │
│  down_payment · rate · amortization     │
└──────────────┬──────────────────────────┘
               │
┌──────────────▼──────────────────────────┐
│  AGENT 1 — INTAKE (deterministic)       │
│  Resolves address → lat/lon, parcel,    │
│  ward, inferred property type           │
│  NO LLM                                 │
└──────────────┬──────────────────────────┘
               │
┌──────────────▼──────────────────────────┐
│  AGENT 2 — DATA RETRIEVAL (deterministic)│
│  Async parallel httpx fanout to all     │
│  data sources. 5s timeout per source.   │
│  Each result annotated with confidence. │
│  NO LLM                                 │
│                                         │
│  ┌─────────┬──────────┬─────────┐       │
│  │Heritage │RentSafeTO│Dev Apps │       │
│  │ CKAN    │  CKAN    │  CKAN   │       │
│  ├─────────┼──────────┼─────────┤       │
│  │Permits  │  LTT     │TRCA    │       │
│  │ CKAN    │Hardcoded │ArcGIS  │       │
│  └─────────┴──────────┴─────────┘       │
└──────────────┬──────────────────────────┘
               │
┌──────────────▼──────────────────────────┐
│  AGENT 3 — COST COMPUTATION (determ.)   │
│  LTT calc, property tax projection,    │
│  mortgage scenarios, risk-weighted      │
│  loadings, composite inferred signals   │
│  NO LLM                                 │
└──────────────┬──────────────────────────┘
               │
┌──────────────▼──────────────────────────┐
│  AGENT 4 — SYNTHESIS (LLM)              │
│  Mistral Medium 3.5 (256k) or          │
│  Qwen 3 7B — local on GB10 via NIM     │
│  Conditional RAG retrieval              │
│  Plain-language report + negotiation    │
│  leverage per flag                      │
└──────────────┬──────────────────────────┘
               │
┌──────────────▼──────────────────────────┐
│  OUTPUT                                 │
│  True 10-yr cost · hidden flags ·       │
│  confidence per source · negotiation    │
│  scripts · cost composition · verdict   │
└─────────────────────────────────────────┘
```

**Key constraint:** Agents 1–3 are pure Python. No LLM calls. Only Agent 4 uses the LLM. This keeps the cost computation deterministic and auditable.

### 2.3 Standardized Signal Output

Every signal object returned by Agents 2–3 must conform to:

```python
@dataclass
class Signal:
    signal_name: str                                    # e.g. "Heritage Designation"
    signal_type: Literal["observed", "inferred", "simulated"]
    value: str                                          # e.g. "Part IV designated"
    confidence: Literal["High", "Medium", "Low"]
    source: str                                         # e.g. "Toronto Heritage Register"
    data_coverage: str                                  # e.g. "May 2026 snapshot — 12,320 properties"
    message: str                                        # plain-language output shown to user
```

**No-match fallback (apply to EVERY signal):**
```python
Signal(
    value="No data found",
    confidence="Low",
    message=f"No record found for this address in {dataset_name}. Manual verification recommended."
)
```

**Report-level metadata:**
```python
report = {
    "as_of": "2026",
    "disclaimer": "Analysis based on publicly available data current as of 2026. "
                  "Some sources reflect historical snapshots (MPAC assessed values frozen at "
                  "January 1, 2016). This tool surfaces ownership-risk signals and "
                  "due-diligence prompts — it is not financial, legal, or insurance advice."
}
```

---

## 3. User Inputs

| Input | Type | Default | Validation | Used by |
|-------|------|---------|------------|---------|
| `address` | string | — | Must be a Toronto address | Agent 1 (geocoding), all spatial lookups |
| `list_price` | int | — | 100,000 – 10,000,000 | LTT, CMHC, mortgage, assessed value proxy |
| `buyer_profile` | enum | `"first-time"` | `first-time` / `investor` / `downsizer` | Tax rate, rebates, FHSA/HBP flags |
| `down_payment_pct` | float | 20 | 5 – 100 | CMHC logic, mortgage principal |
| `rate` | float | 4.79 | 0.5 – 15.0 | Mortgage scenario base rate |
| `amortization` | int | 25 | 15 – 30 | Payment calc, CMHC surcharge if >25 |

---

## 4. Data Sources — V1 Implementation

### 4.1 CKAN API Configuration

```python
CKAN_BASE = "https://ckan0.cf.opendata.inter.prod-toronto.ca/api/3"
TIMEOUT_SECONDS = 5  # per source — aligns with UNKNOWN confidence rule
```

### 4.2 Source Registry

| # | Signal | Dataset slug | Records | Refresh | Confidence rule |
|---|--------|-------------|---------|---------|-----------------|
| 1 | Heritage designation | `heritage-register` | 12,320 | Quarterly | HIGH if found with date; MEDIUM if status unclear |
| 2 | Heritage conservation districts | `heritage-conservation-districts` | ~30 polygons | Quarterly | HIGH if point-in-polygon match |
| 3 | Active building permits | `building-permits-active-permits` | 228,573 | Daily | HIGH if records returned; MEDIUM if pre-1999 |
| 4 | Cleared building permits | `building-permits-cleared-permits` | 401,265 | Daily | HIGH if records returned |
| 5 | Development applications | `development-applications` | 26,254 | Daily | HIGH if X/Y coords present (97%) |
| 6 | RentSafeTO evaluations | `apartment-building-evaluation` | 5,340 | Daily | MEDIUM (score interpretation) |
| 7 | Flood zone | TRCA ArcGIS (external) | City-wide | Live | HIGH if API responds |
| 8 | Address geocoding | `address-points-municipal-toronto-one-address-repository` | 525,435 | Weekly | HIGH |

**TRCA ArcGIS endpoint:**
```python
TRCA_URL = (
    "https://services1.arcgis.com/pMeXFF5bmbv34Alm/arcgis/rest/"
    "services/Floodline_TRCA_Polygon/FeatureServer/0/query"
)
TRCA_PARAMS = {
    "geometry":        f"{longitude},{latitude}",
    "geometryType":    "esriGeometryPoint",
    "spatialRel":      "esriSpatialRelIntersects",
    "returnCountOnly": "true",
    "f":               "json"
}
in_flood_zone = response.json()["count"] > 0
```

**Fallback:** If ArcGIS fails, load `data/trca_floodplain_toronto.geojson` and do local point-in-polygon via GeoPandas.

### 4.3 Agent 2 Confidence Rules

| Confidence | Condition |
|------------|-----------|
| HIGH | HTTP 200 + `last_updated` / `extract_date` field present in response |
| MEDIUM | HTTP 200, no recency metadata |
| LOW | HTTP 200, empty results array (0 records) |
| UNKNOWN | Timeout (>5s), 4xx, or 5xx response |

### 4.4 Demo Mode

```bash
DEMO_MODE=true python run_pipeline.py --address "401 Richmond St W"
```

When `DEMO_MODE=true`, Agent 2 bypasses live API calls and serves pre-cached JSON from `data/demo_cache/<address_slug>.json`. The cached files match Agent 2's output schema exactly. Three demo properties required by Hour 8.

---

## 5. Signal Specifications

### 5.1 Land Transfer Tax (LTT)

**Signal type:** `observed` | **Confidence:** High

```python
ONTARIO_LTT_BRACKETS = [
    {"up_to": 55_000,     "rate": 0.005},
    {"up_to": 250_000,    "rate": 0.010},
    {"up_to": 400_000,    "rate": 0.015},
    {"up_to": 2_000_000,  "rate": 0.020},
    {"up_to": float("inf"), "rate": 0.025},
]

# Toronto MLTT mirrors Ontario for properties under $3M
TORONTO_MLTT_BRACKETS = ONTARIO_LTT_BRACKETS

ONTARIO_FTB_REBATE = 4_000
TORONTO_FTB_REBATE = 4_475
# Apply both when buyer_profile == "first-time"
# Combined max rebate = $8,475
```

**Outputs:** Ontario LTT, Toronto MLTT, combined total, first-time rebate (if applicable), net LTT.

### 5.2 Property Tax Projection

**Signal type:** `simulated` | **Confidence:** High (rate) / Medium (assessed value)

```python
RATE_RESIDENTIAL = 0.00767311
# 2026: City 0.605295% + Education 0.153000% + City Building Fund 0.009016%

RATE_MULTI_RES = 0.01208792
# Investor / multi-residential 2026

ANNUAL_GROWTH = 0.035
# Based on 9.5% (2024), 6.9% (2025), 2.2% (2026)

# Rate selection
rate = RATE_MULTI_RES if buyer_profile == "investor" else RATE_RESIDENTIAL

# Assessed value proxy (MPAC frozen at 2016)
MULTIPLIERS = {
    "condo":             0.90,
    "condo_townhouse":   0.80,
    "semi_detached":     0.65,
    "detached_urban":    0.70,
    "detached_suburban": 0.55,
}
multiplier = MULTIPLIERS.get(property_type, 0.70)

av_mid  = list_price * multiplier
av_low  = av_mid * 0.85
av_high = av_mid * 1.15

annual_tax = av_mid * rate
ten_year_tax = annual_tax * ((pow(1 + ANNUAL_GROWTH, 10) - 1) / ANNUAL_GROWTH)
```

**UI disclaimer (always show):** "Estimated using property-type proxy. Actual taxes depend on your MPAC assessed value. MPAC values are frozen at January 1, 2016. 10-year projection is an illustrative scenario based on recent rate trends — not a confirmed forecast."

### 5.3 Flood Zone Exposure

**Signal type:** `observed` | **Confidence:** High

**Tiered output (NO dollar figures, NO "insurance loading"):**

| Tier | Condition | Message |
|------|-----------|---------|
| Elevated | Property intersects TRCA polygon | "Flood Exposure: Elevated. This property intersects a TRCA flood-risk area. Insurance implications vary significantly by insurer and property characteristics. Verify coverage and premiums directly with your insurer." |
| Moderate | Property within 100m of boundary | "Flood Exposure: Moderate. This property is near a TRCA flood-risk area. Verify flood risk and insurance implications with your insurer." |
| Low | No intersection | No message shown |

### 5.4 Heritage Designation

**Signal type:** `observed` | **Confidence:** High

| Status | Message |
|--------|---------|
| Part IV | "Elevated review recommended. This property is fully heritage designated. Renovations require heritage permit review — budget for delays and potential restrictions on exterior alterations." |
| Part V | "This property is in a heritage conservation district. Neighbourhood-level heritage restrictions apply." |
| Listed | "This property is on the Heritage Register but not yet formally designated. Lower risk, worth monitoring." |
| No match | No flag |

### 5.5 Building Health (Dual-Path)

**Signal type:** `observed` | **Confidence:** Medium

#### Path A: Rental apartments (3+ storeys, 10+ units) — RentSafeTO

| Score | Message |
|-------|---------|
| Below 70 | "Elevated review recommended." |
| 70–85 | "Further due diligence recommended." |
| 85+ | Pass, no flag |

**Disclaimer (always show):** "No RentSafeTO score available does not indicate lower risk — condos, co-ops, and smaller buildings are not covered by this program."

#### Path B: All other properties — Structural Permit History

```python
# FILTER: only surface structural permits
STRUCTURAL_KEYWORDS = ["structural", "unsafe", "major repair", "emergency"]
# NEVER flag routine, cosmetic, or interior permits

structural_active  = [p for p in active_permits if is_structural(p)]
structural_cleared = [p for p in cleared_permits if is_structural(p)
                      and within_last_10_years(p)]

if len(structural_active) > 0:
    flag("Active structural permit found. Review permit details before purchase.")
elif len(structural_cleared) >= 3:
    flag("Further due diligence recommended — notable structural permit history.")
```

### 5.6 Neighbourhood Intensification Signal

**Signal type:** `inferred` | **Confidence:** Medium

```python
# Filter before counting
ACTIVE_STATUSES = ["Under Review", "Application Received",
                   "Council Approved", "NOAC Issued"]
RELEVANT_TYPES  = ["OZ", "SA"]  # zoning amendments + site plan approvals
# EXCLUDE STATUS == "Closed" (removes ~18,504 records)

# Normalize by ward area
density_per_km2 = active_count / ward_area_km2
```

| Count (500m radius) | Tier | Message |
|---------------------|------|---------|
| 0–4 | Low | No flag |
| 5–9 | Medium | "Neighbourhood showing early intensification signals. This may affect future neighbourhood character, density, and property values." |
| 10+ | High | "High neighbourhood intensification signal detected. Significant redevelopment activity is occurring nearby." |

**Never include:** direct tax increase predictions, MPAC reassessment predictions, property value certainties.

### 5.7 Mortgage Cost Layer

**Signal type:** `simulated` | **Confidence:** High (math) / Medium (rate scenarios)

**Rate renewal scenarios — 3 paths over 10 years (2 renewal cycles):**

| Scenario | Behavior | Purpose |
|----------|----------|---------|
| Base | Rate stays flat | Expected path |
| Bear | +1.5% at each renewal | Stress test |
| Bull | -0.5% at each renewal | Optimistic |

**CMHC premium schedule:**

| LTV | Premium |
|-----|---------|
| ≤65% | 0.60% |
| 65.01–75% | 1.70% |
| 75.01–80% | 2.40% |
| 80.01–85% | 2.80% |
| 85.01–90% | 3.10% |
| 90.01–95% | 4.00% |
| Non-traditional | 4.50% |

Add 0.20% surcharge for amortization >25 years. Ontario PST (8%) on premium payable at closing (not financed).

**Sub-signals:**

| Signal | Trigger | Description |
|--------|---------|-------------|
| Assumable mortgage flag | `buyer_profile == "first-time"` | Surface if seller's locked rate is favorable |
| Prepayment penalty (IRD) | Always | Seller's remaining term + current rate → estimated penalty → negotiation leverage |
| Amortization accelerator | Always | Savings from bi-weekly payments or lump-sum prepayments |
| FHSA flag | `buyer_profile == "first-time"` | Up to $40,000 tax-free |
| RRSP HBP flag | `buyer_profile == "first-time"` | Up to $60,000/person tax-free withdrawal |

### 5.8 Maintenance Complexity Signal (Composite)

**Signal type:** `inferred` | **Confidence:** Low (ALWAYS — never upgrade)

```python
# Signals used (all already in stack):
factors = []
if building_age > 30:              factors.append("Older building age")
if len(structural_active) > 0:     factors.append("Active structural permit")
if len(structural_cleared) >= 3:   factors.append("Multiple structural permits")
if rentsafeto_score < 70:          factors.append("Low RentSafeTO score")
if heritage_designated:            factors.append("Heritage designation")
if in_flood_zone:                  factors.append("Flood zone exposure")

tier = "Low"
if len(factors) >= 3: tier = "Elevated"
elif len(factors) >= 1: tier = "Medium"
```

**Output:**
```json
{
    "signal_name": "Maintenance Complexity Signal",
    "signal_type": "inferred",
    "value": "Elevated",
    "label": "Elevated maintenance complexity signal",
    "factors": ["Older building age", "Multiple structural permits", "Flood zone exposure"],
    "disclaimer": "This is an inferred signal and not evidence of a historical special assessment.",
    "confidence": "Low"
}
```

**Wording:** NEVER output "High special assessment risk." USE: "Elevated maintenance complexity signal."

### 5.9 Future Tax Pressure Signal (Composite)

**Signal type:** `inferred` | **Confidence:** Low (ALWAYS — never upgrade)

```python
# Signals used:
dev_density = count_active_oz_sa_within_500m
permit_activity = len(structural_active) + len(structural_cleared)
has_constraints = heritage_designated or in_ravine_protection

if dev_density >= 10:          tier = "High"
elif dev_density >= 5:         tier = "Medium"
else:                          tier = "Low"
```

**Output:**
```json
{
    "signal_name": "Future Tax Pressure Signal",
    "signal_type": "inferred",
    "value": "High",
    "confidence": "Low",
    "message": "Neighbourhood intensification signal detected. Significant redevelopment activity nearby may affect future neighbourhood character and values.",
    "disclaimer": "This is a neighbourhood signal, not an MPAC reassessment prediction."
}
```

**Wording:** Frame as "neighbourhood reassessment risk signal." NEVER frame as MPAC prediction or tax guarantee.

---

## 6. Wording Rules

Apply these substitutions across ALL agent outputs, UI text, and prompt templates:

| Find | Replace with |
|------|-------------|
| "hard red flag" / "red flag" | "Elevated review recommended" |
| "High special assessment risk" | "Elevated maintenance complexity signal" |
| "Expect rising neighbourhood assessments and potential tax increases" | "Neighbourhood intensification signal detected" |
| "potential tax increases" | "potential neighbourhood change" |
| "Building has major issues" / "major issues" | "Further due diligence recommended" |
| "$3,500" (flood insurance figure) | REMOVE entirely |
| "insurance loading" | REMOVE entirely |
| "hard flag" | "elevated signal" |

---

## 7. First-Time Buyer Flags

When `buyer_profile == "first-time"`, surface ALL:

- LTT rebate: $8,475 combined (Ontario $4,000 + Toronto $4,475)
- FHSA: up to $40,000 tax-free ($8K/yr contribution, tax-deductible in, tax-free out)
- RRSP HBP: up to $60,000 per person
- Assumable mortgage opportunity flag
- Property Tax Relief and Rebates programs link

---

## 8. Agent 4 — LLM Synthesis

### 8.1 Model

**Primary:** Mistral Medium 3.5 (256k context window, local on GB10 via NIM)
**Fallback:** Qwen 3 7B

Decision at Hour 1: benchmark latency. If single call <5s for 300 tokens, use primary. If >15s, merge Agent 3 + Agent 4 into a single prompt.

### 8.2 Prompt Structure

Agent 4 receives the complete structured output from Agents 1–3 as JSON. Its job is synthesis, not computation.

```
SYSTEM: You are a property report synthesis agent. Generate a plain-English
report for a Toronto home buyer. Reference specific data points from the
analysis. Include:
1. One-paragraph verdict summary
2. Hidden costs breakdown with dollar amounts
3. Risk flags with explanations and confidence levels
4. Negotiation leverage per elevated flag (dollar range + script)
5. 10-year cost comparison across 3 mortgage scenarios

Rules:
- Every claim must reference a specific data source finding
- Use dollar amounts, not percentages, for cost impacts
- If confidence is "Medium" or "Low", say so explicitly
- Do NOT fabricate data. Only reference findings in the input.
- For each elevated flag, include one sentence of negotiation
  leverage with a dollar range
- Tone: direct, clear, protective of the buyer's interests
- Never use "red flag" — use "elevated review recommended"

USER: Planning output: {agent_1_json}
      Analysis findings: {agent_3_json}
      Property: {address} at ${list_price}
      Buyer profile: {buyer_profile}
```

### 8.3 RAG Corpus (Conditional Retrieval)

Retrieve from `/rag_corpus/` only when the corresponding flag is active:

| Trigger | Retrieval query | Document category |
|---------|----------------|-------------------|
| Heritage flag | "heritage designation renovation restrictions Toronto" | `/rag_corpus/heritage/` |
| Flood flag | "TRCA flood zone insurance implications Toronto" | `/rag_corpus/flood/` |
| `buyer_profile == "first-time"` | "FHSA RRSP home buyers plan eligibility 2026" | `/rag_corpus/programs/` |
| Maintenance signal == "Elevated" | "condo reserve fund special assessment Ontario" | `/rag_corpus/assessment/` |
| Tax pressure == "High" | "Toronto dev application zoning implications" | `/rag_corpus/zoning/` |

**Corpus files:**

| Path | Source |
|------|--------|
| `/rag_corpus/legislation/ontario_ltt_act.txt` | ontario.ca/laws/statute/90l06 |
| `/rag_corpus/legislation/toronto_mltt_ch760.pdf` | toronto.ca/legdocs/municode/toronto-code-760.pdf |
| `/rag_corpus/legislation/ontario_heritage_act.txt` | ontario.ca/laws/statute/90o18 |
| `/rag_corpus/programs/fhsa_cra_guide.txt` | canada.ca/.../first-home-savings-account |
| `/rag_corpus/programs/rrsp_hbp_guide.txt` | canada.ca/.../what-home-buyers-plan |
| `/rag_corpus/programs/ontario_ltt_ftb_refund.txt` | ontario.ca/page/land-transfer-tax-refunds-first-time-homebuyers |
| `/rag_corpus/assessment/mpac_guide.txt` | mpac.ca/en/UnderstandingYourAssessment |
| `/rag_corpus/zoning/toronto_bylaw_569.txt` | toronto.ca/zoning |
| `/rag_corpus/flood/trca_flood_guide.txt` | trca.ca/conservation/flood-risk-management |
| `/rag_corpus/mortgage/osfi_b20.txt` | osfi-bsif.gc.ca/en/guidance/guidance-library/b-20 |
| `/rag_corpus/heritage/toronto_heritage_planning.txt` | toronto.ca/.../heritage-preservation |

---

## 9. Inter-Agent Data Contracts

### 9.1 Agent 1 → Agent 2

```json
{
    "address": "401 Richmond St W, Toronto",
    "coordinates": {"lat": 43.6492, "lon": -79.3925},
    "property_type_inferred": "commercial_condo",
    "ward": "10",
    "query_plan": [
        {"source": "heritage_register", "reason": "Pre-1990 building in heritage district"},
        {"source": "building_permits", "reason": "Condo = assessment risk from structural work"},
        {"source": "trca_flood", "reason": "Ground-floor unit near watercourse"},
        {"source": "dev_applications", "reason": "Standard 500m radius check"},
        {"source": "rentsafeto", "reason": "SKIP — not a residential rental building"}
    ],
    "reasoning_trace": "This appears to be a commercial condo conversion in a heritage area."
}
```

### 9.2 Agent 2 → Agent 3

```json
{
    "results": {
        "heritage_register": {
            "found": true,
            "designation": "Part IV",
            "year": 1987,
            "confidence": "High",
            "data_coverage": "May 2026 snapshot"
        },
        "building_permits": {
            "found": true,
            "active_structural_count": 3,
            "cleared_structural_10yr": 5,
            "types": ["structural", "mechanical", "plumbing"],
            "confidence": "High",
            "data_coverage": "2008-2026 (live API)"
        },
        "trca_flood": {
            "found": false,
            "in_flood_zone": false,
            "confidence": "High",
            "data_coverage": "2026 (live ArcGIS API)"
        },
        "dev_applications": {
            "found": true,
            "active_oz_sa_500m": 12,
            "density_per_km2": 8.4,
            "confidence": "High",
            "data_coverage": "2026 (live CKAN API)"
        },
        "rentsafeto": {
            "skipped": true,
            "reason": "Not in query plan"
        }
    },
    "failures": [],
    "total_latency_ms": 2300
}
```

### 9.3 Agent 3 → Agent 4

```json
{
    "signals": [
        {
            "signal_name": "Heritage Designation",
            "signal_type": "observed",
            "value": "Part IV designated (1987)",
            "confidence": "High",
            "source": "Toronto Heritage Register",
            "data_coverage": "May 2026 snapshot",
            "message": "Elevated review recommended. This property is fully heritage designated."
        }
    ],
    "composite_signals": [
        {
            "signal_name": "Maintenance Complexity Signal",
            "signal_type": "inferred",
            "value": "Elevated",
            "confidence": "Low",
            "factors": ["Older building age", "Multiple structural permits"],
            "disclaimer": "Inferred signal, not evidence of historical special assessment."
        }
    ],
    "cost_breakdown": {
        "ltt_ontario": 14475,
        "ltt_toronto": 14475,
        "ltt_combined": 28950,
        "ftb_rebate": -8475,
        "ltt_net": 20475,
        "assessed_value_mid": 595000,
        "annual_property_tax": 4566,
        "ten_year_property_tax": 53821,
        "mortgage_base": 547109,
        "mortgage_bear": 582557,
        "mortgage_bull": 535812,
        "cmhc_premium": 0,
        "transit_dividend_10yr": 87000
    },
    "buyer_flags": ["LTT rebate applied", "FHSA eligible", "RRSP HBP eligible"],
    "overall_risk": "medium"
}
```

---

## 10. Error Handling

| Error | Handler | Behavior |
|-------|---------|----------|
| `json.JSONDecodeError` on CKAN response | Log warning | Set source confidence = UNKNOWN, continue pipeline |
| `FileNotFoundError` on TRCA GeoJSON | Log error | Set flood confidence = UNKNOWN, continue |
| Empty LLM response (Agent 4) | Retry once | If still empty: "Unable to synthesize — check NIM container" |
| `json.JSONDecodeError` on LLM output | Strip markdown fences | Retry JSON parse, then fall back to plain-text extraction |
| `httpx.ConnectError` on NIM | Surface immediately | "NIM container not responding. Start with: nvidia-nim start" |
| `httpx.TimeoutException` on CKAN | Set UNKNOWN | Continue with other sources, note timeout in reasoning trace |

---

## 11. Frontend (Streamlit V1)

### 11.1 Screens

| Screen | Components |
|--------|-----------|
| **Landing** | Address input (autocomplete via Address Points CKAN), list price, buyer profile dropdown, down payment/rate/amortization fields, "Analyze" CTA |
| **Analysis** | Real-time reasoning trace via `st.status()` — 4 agent steps with progress, timing, and status messages |
| **Verdict** | Verdict card (true 10-yr cost, leverage range, key stats), flag cards (expandable, with negotiation scripts), cost breakdown table, agent reasoning trace (collapsible), chat panel for follow-up questions |

### 11.2 Design System

Full specification in [`DESIGN.md`](./DESIGN.md). Key tokens:

| Token | Value |
|-------|-------|
| Background | `#171721` |
| Surface | `#1E1E2A` |
| Border | `rgba(175,178,206,0.36)` |
| Accent/CTA | `#5266EB` (indigo) |
| Green (leverage/positive only) | `#34D399` |
| Amber (urgency only) | `#FBBF24` |
| Red (danger only) | `#F87171` |
| All text | DM Sans variable |
| Financial figures | Geist Mono, `tabular-nums` |
| Button radius | 40px pill |

### 11.3 Prototype

Working 3-screen prototype with chat panel: [`meridian-prototype.html`](./meridian-prototype.html)

---

## 12. Build Schedule (24 Hours)

### Phase 1: Foundation (Hours 0–8)

| Hour | Task | Gate |
|------|------|------|
| 0–1 | NIM latency benchmark | **BLOCKING.** If >15s/300 tokens → merge Agents 3+4 |
| 1–3 | Address Points CKAN autocomplete + geocoding | |
| 3–5 | CKAN API wrapper (heritage, permits, dev apps, RentSafeTO) | Each endpoint tested individually |
| 5–6 | TRCA ArcGIS flood zone + GeoJSON fallback | |
| 6–7 | LTT calculator + property tax + mortgage scenarios | |
| 7–8 | Pre-cache 3 demo properties | **GATE:** 3 cached properties in `data/demo_cache/` |

### Phase 2: Agent Reasoning (Hours 8–14)

| Hour | Task | Gate |
|------|------|------|
| 8–10 | Agent 1 intake (property type classification, query plan) | |
| 10–12 | Agent 3 cost computation + composite signals | |
| 12–14 | Agent 4 synthesis prompt with negotiation leverage | **GATE:** `python run_pipeline.py --demo` passes end-to-end |

### Phase 3: UI + Integration (Hours 14–20)

| Hour | Task |
|------|------|
| 14–16 | Streamlit app, address input with autocomplete |
| 16–18 | Reasoning trace display (`st.status()` expanding sections) |
| 18–19 | Confidence badges per source |
| 19–20 | Verdict card + cost table + flag cards |

### Phase 3b: Error Rescues (Hours 19–21)

Wire all 5 error handlers from Section 10.

### Phase 4: Polish + Buffer (Hours 21–24)

| Task | Gate |
|------|------|
| Run 3 demo properties end-to-end (cached + live) | All 3 produce different risk profiles |
| Test 3+ random Toronto addresses | Geocoding robustness |
| Time pitch + demo | **GATE:** Under 4 minutes total |
| OPTIONAL: RAPIDS cuDF spatial acceleration | Only if 2+ hours buffer |

### Cut List (in order)

1. RentSafeTO (least dramatic flag)
2. Live API calls for non-demo properties
3. Agent 3 as separate call (merge into Agent 4)
4. Confidence badges (defer to TODOS)
5. **NEVER cut:** Reasoning trace UI

---

## 13. V1.5 Expansion (Post-Hackathon)

Add these 6 data sources using the same CKAN pattern already built in Agent 2:

| Source | Dataset slug | Records | Effort |
|--------|-------------|---------|--------|
| Heritage Formerly Listed | `heritage-formerly-listed` | ~3,700 | 15 min |
| Residential Fire Inspections | `residential-fire-inspection-results` | 124,414 | 30 min |
| Building Violations | `building-construction-demolition-violations` | 48,035 | 30 min |
| Basement Flooding Areas | `basement-flooding-study-areas` | 67 | 30 min |
| Transit Dividend | `ttc-routes-and-schedules` (GTFS) | All routes | 1 hr |
| Tax Relief Programs | Hardcoded by buyer profile | N/A | 30 min |

---

## 14. Testing

### 14.1 Demo Properties (3 required)

| Property | Risk profile | Key flags expected |
|----------|-------------|-------------------|
| 401 Richmond St W | High risk | Heritage Part IV, active structural permits, high dev density |
| TBD Scarborough semi | Low risk | Clean — demonstrates "no concerns identified" path |
| TBD Eglinton condo | Medium risk | Near flood zone, moderate dev pressure, transit-aligned |

### 14.2 Validation Checks

- [ ] LTT calculation matches ratehub.ca/land-transfer-tax-ontario for 3 price points
- [ ] Property tax estimate within 15% of actual for a known property
- [ ] Flood zone detection matches TRCA interactive map for 3 addresses
- [ ] Heritage lookup matches City of Toronto heritage property search
- [ ] Each demo property produces a meaningfully different report
- [ ] Zero flags property correctly outputs "No concerns identified" per section
- [ ] First-time buyer profile triggers all 5 buyer flags
- [ ] Investor profile uses multi-residential tax rate
- [ ] All wording rules enforced (grep for banned phrases)

---

## 15. Success Criteria

| Criterion | Threshold |
|-----------|-----------|
| Pipeline end-to-end latency | <35 seconds |
| Agent reasoning visible in UI | Yes — real-time status updates |
| 3 properties with different risk profiles | Meaningfully different reports |
| LLM output references specific data points | Dollar amounts, not generic warnings |
| Demo + pitch | Under 4 minutes |
| Live query capability | At least 1 non-cached property works |
| Wording rules compliance | Zero banned phrases in output |

---

## References

| Document | Path |
|----------|------|
| Data parameters inventory (43 params) | [`DATA_PARAMETERS.md`](./DATA_PARAMETERS.md) |
| Spec v3 (authoritative) | `meridian_spec_final.txt` |
| Data flow diagram + UI mockups | `Meridian Data Flow.pdf` |
| Design system | [`DESIGN.md`](./DESIGN.md) |
| CEO build plan | [`CEO_Plan_V1_Hackathon.md`](./CEO_Plan_V1_Hackathon.md) |
| Architecture gap analysis | [`Office_Hours_Gap_Analysis_and_Build_Strategy.md`](./Office_Hours_Gap_Analysis_and_Build_Strategy.md) |
| UI prototype (working) | [`meridian-prototype.html`](./meridian-prototype.html) |
