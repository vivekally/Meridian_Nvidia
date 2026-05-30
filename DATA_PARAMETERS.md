# Meridian — Hidden Cost Data Parameters

> Comprehensive inventory of every data point Meridian uses (or can use) to compute the true 10-year cost of owning residential property in Toronto. Cross-referenced against Toronto Open Data (open.toronto.ca), TRCA ArcGIS, federal/provincial tax rules, and industry benchmarks.
>
> Last updated: 2026-05-30
>
> Aligned with: `meridian_spec_final.txt` v3 and `Meridian Data Flow.pdf`

---

## How to Read This Document

Each parameter includes:

- **Data source** — where the data comes from, with CKAN dataset slug or API endpoint
- **Signal type** — `observed` (Layer 1: direct dataset match), `inferred` (Layer 2: computed from multiple observed signals), or `simulated` (Layer 3: projection scenario)
- **Records** — approximate dataset size
- **Refresh** — how often the source updates
- **Cost impact** — estimated dollar effect on 10-year ownership cost
- **Confidence** — how reliably Meridian can quantify this parameter (High / Medium / Low per spec)
- **V1/V2/V3** — which product version includes this parameter

Every signal output must follow the spec's standardized structure:
```json
{
  "signal_name":   "string",
  "signal_type":   "observed | inferred | simulated",
  "value":         "string",
  "confidence":    "High | Medium | Low",
  "source":        "string",
  "data_coverage": "string",
  "message":       "string"
}
```

**Wording rules (apply everywhere):** Never use "red flag," "hard red flag," "major issues," or "high special assessment risk." Use "elevated review recommended," "further due diligence recommended," and "elevated maintenance complexity signal" instead. Never include "$3,500" or "insurance loading" in flood zone outputs.

---

## User Input Parameters

These are provided by the buyer, not retrieved from data sources. They drive downstream logic across all tiers.

### I1. Address

| Field | Value |
|-------|-------|
| Type | User input |
| Format | Free-text Toronto address string |
| Used by | Agent 1 (geocoding → lat/lon), all spatial lookups |
| Version | V1 |

Resolved via `address-points-municipal-toronto-one-address-repository` (CKAN, 525,435 records, weekly refresh) for geocoding. Nominatim is fallback only.

---

### I2. List Price

| Field | Value |
|-------|-------|
| Type | User input |
| Format | CAD dollar amount |
| Used by | LTT calculation, CMHC premium, mortgage scenarios, assessed value estimate |
| Version | V1 |

---

### I3. Buyer Profile

| Field | Value |
|-------|-------|
| Type | User input |
| Options | `first-time` / `investor` / `downsizer` |
| Used by | Tax rate selection, LTT rebates, FHSA/HBP flagging, property tax relief programs |
| Version | V1 |

Drives critical branching:
- `first-time` → apply LTT rebates ($8,475), flag FHSA ($40K) + RRSP HBP ($60K), flag assumable mortgage opportunity, use `RATE_RESIDENTIAL`
- `investor` → use `RATE_MULTI_RES` (0.01208792, ~57% higher), flag VHT obligations, no rebates
- `downsizer` → use `RATE_RESIDENTIAL`, flag property tax relief programs

---

### I4. Down Payment Percentage

| Field | Value |
|-------|-------|
| Type | User input |
| Default | 20% |
| Used by | CMHC premium logic (required if <20%), mortgage principal calculation |
| Version | V1 |

---

### I5. Mortgage Rate

| Field | Value |
|-------|-------|
| Type | User input |
| Default | 4.79% (current benchmark) |
| Used by | Mortgage scenario modeling (base/bear/bull) |
| Version | V1 |

---

### I6. Amortization Period

| Field | Value |
|-------|-------|
| Type | User input |
| Default | 25 years |
| Used by | Mortgage payment calculation, CMHC surcharge (>25yr adds 0.20%) |
| Version | V1 |

---

## Tier 1: Core Hidden Cost Drivers

**Version:** V1 (Hackathon build)
**Status:** All data sources verified live on CKAN as of May 30, 2026

### 1. Heritage Designation (Part IV / Part V / Listed)

| Field | Value |
|-------|-------|
| Signal type | `observed` |
| Source | `heritage-register` — Toronto Open Data CKAN |
| Format | SHP (Shapefile), WGS84 |
| Records | ~12,320 properties |
| Refresh | Quarterly (last: May 21, 2026) |
| Cost impact | Renovation delays and restriction premium (no fixed levy in Toronto) |
| Confidence | **High** |
| Data coverage | May 2026 snapshot — 12,320 properties, CKAN API |
| Version | V1 |

**What it captures:** Properties designated or listed under the Ontario Heritage Act. Part IV = individual designation. Part V = within a Heritage Conservation District. Listed = on the register but not yet designated (still triggers 60-day demolition delay).

**Why it's a hidden cost:** Any exterior alteration (windows, facade, roof, additions) requires City Heritage Permit approval. Typical approval timeline: 6–18 months. Buyers see the list price but not the permanent renovation constraint.

**Tiered output (per spec section 4):**
- Part IV → "Elevated review recommended. This property is fully heritage designated. Renovations require heritage permit review — budget for delays and potential restrictions on exterior alterations."
- Part V → "This property is in a heritage conservation district. Neighbourhood-level heritage restrictions apply."
- Listed → "This property is on the Heritage Register but not yet formally designated. Lower risk, worth monitoring."
- No match → no flag

**Query logic:** Spatial point-in-polygon against heritage register shapefile. Match on address or nearest point within 25m tolerance.

**Agent 2 confidence rules:**
- HIGH: Property found in register with designation date
- MEDIUM: Property found but designation status unclear
- LOW: No match within 50m (likely not designated)
- UNKNOWN: API timeout or shapefile load failure

---

### 2. Heritage Conservation Districts (HCDs)

| Field | Value |
|-------|-------|
| Source | `heritage-conservation-districts` — Toronto Open Data CKAN |
| Format | SHP (Shapefile) |
| Records | ~30 district polygons |
| Refresh | Quarterly (last: March 2, 2026) |
| Cost impact | **$20,000–$40,000** renovation premium |
| Confidence | **90%** |
| Version | V1 |

**What it captures:** Geographic boundaries of Heritage Conservation Districts. Properties inside an HCD face design review even if not individually designated.

**Why it's a hidden cost:** An unlisted property inside an HCD still requires heritage review for exterior changes. Buyers often don't know their property is in an HCD until they apply for a building permit. Slower approval + heritage-compliant materials = cost premium.

**Query logic:** Point-in-polygon. If property coordinates fall inside any HCD polygon, flag it.

---

### 3. Active Building Permits

| Field | Value |
|-------|-------|
| Source | `building-permits-active-permits` — Toronto Open Data CKAN |
| Format | CSV, XML, JSON |
| Records | 228,573 |
| Refresh | **Daily** (last: May 30, 2026) |
| Cost impact | **$15,000–$25,000** liability transfer risk |
| Confidence | **95%** |
| Version | V1 |

**What it captures:** All currently active building permit applications and permits. Includes permit type, work type, application date, and status.

**Why it's a hidden cost:** Unresolved permits legally transfer to the buyer at closing. The buyer inherits responsibility for final inspections, outstanding orders, and potential stop-work orders. Structural, electrical, and plumbing permits are the highest-risk categories.

**Query logic:** Filter by property address. Count active permits. Classify by work type (structural > electrical > plumbing > mechanical > other). Flag any permit open >2 years as elevated risk.

**Agent 2 confidence rules:**
- HIGH: HTTP 200, records returned, `APPLICATION_DATE` present
- MEDIUM: HTTP 200, records returned, `APPLICATION_DATE` before Oct 1, 1999 (data may be incomplete per dataset documentation)
- LOW: HTTP 200, zero records (no active permits)
- UNKNOWN: Timeout or error

---

### 4. Cleared Building Permits (Historical)

| Field | Value |
|-------|-------|
| Source | `building-permits-cleared-permits` — Toronto Open Data CKAN |
| Format | CSV, XML, JSON |
| Records | 401,265 |
| Refresh | **Daily** (last: May 30, 2026) |
| Cost impact | Indirect — building condition signal |
| Confidence | **85%** |
| Version | V1 |

**What it captures:** All completed/closed building permits. Historical record of construction, renovation, and demolition work on a property.

**Why it matters:** Permit history is the best proxy for building condition when no inspection report is available. Frequent structural permits = aging systems. Recent major renovation = likely good condition. No permits in 20+ years on an old building = deferred maintenance risk.

**Query logic:** Filter by address. Count permits by type and decade. Flag buildings with zero permits in 15+ years (deferred maintenance signal). Flag buildings with 5+ structural permits (chronic issues signal).

---

### 5. Neighbourhood Intensification Signal (Development Applications, 500m Radius)

| Field | Value |
|-------|-------|
| Signal type | `inferred` |
| Source | `development-applications` — Toronto Open Data CKAN |
| Format | CSV, XML, JSON |
| Records | 26,254 (2008–2026, X/Y coordinates 97% complete) |
| Refresh | **Daily** (last: May 30, 2026) |
| Cost impact | Future neighbourhood character, density, and property value changes |
| Confidence | **Medium** |
| Data coverage | 2026 (live CKAN API) — 26,254 total applications 2008–2026 |
| Version | V1 |

**What it captures:** All active and inactive Community Planning and Committee of Adjustment applications since January 2008.

**Filter BEFORE counting (per spec section 6):**
```python
active_statuses = ["Under Review", "Application Received",
                   "Council Approved", "NOAC Issued"]
relevant_types  = ["OZ", "SA"]   # zoning amendments + site plan approvals
# EXCLUDE all STATUS == "Closed" — removes 18,504 resolved records
```

**Normalize by ward area** to fix downtown/suburban bias:
```python
density_per_km2 = active_count / ward_area_km2
# Ward boundary areas: open.toronto.ca/dataset/ward-profiles
```

**Tiered output (per spec section 6):**
- Low (0–4 active OZ/SA within 500m) → pass, no flag
- Medium (5–9) → "Neighbourhood showing early intensification signals. This may affect future neighbourhood character, density, and property values."
- High (10+) → "High neighbourhood intensification signal detected. Significant redevelopment activity is occurring nearby. This may affect future neighbourhood character, density, and property values."

**Never include:** direct tax increase predictions, MPAC reassessment predictions, or property value change certainties.

---

### 6. TRCA Flood Zone Exposure

| Field | Value |
|-------|-------|
| Signal type | `observed` |
| Source | TRCA Floodline Dataset — ArcGIS REST API |
| Fallback | Local GeoJSON: `data/trca_floodplain_toronto.geojson` |
| Format | ArcGIS Feature Service / GeoJSON |
| Records | Toronto-wide flood plain polygons |
| Refresh | 2026 (live ArcGIS API) |
| Cost impact | Insurance implications vary by insurer and property (no fixed dollar figure) |
| Confidence | **High** |
| Data coverage | 2026 (live ArcGIS API) |
| Version | V1 |

**What it captures:** TRCA regulatory flood plain boundaries — areas at risk of flooding from rivers (Don, Humber, Rouge, etc.).

**API call (per spec section 3):**
```
url = "https://services1.arcgis.com/pMeXFF5bmbv34Alm/arcgis/rest/
       services/Floodline_TRCA_Polygon/FeatureServer/0/query"
params = {
  "geometry":        "{longitude},{latitude}",
  "geometryType":    "esriGeometryPoint",
  "spatialRel":      "esriSpatialRelIntersects",
  "returnCountOnly": "true",
  "f":               "json"
}
in_flood_zone = response.json()["count"] > 0
```

**Tiered output (per spec — NO dollar figures, NO "insurance loading"):**
- Elevated → property intersects TRCA flood polygon → "Flood Exposure: Elevated. This property intersects a TRCA flood-risk area. Insurance implications vary significantly by insurer and property characteristics. Verify coverage and premiums directly with your insurer."
- Moderate → property within 100m of polygon boundary → "Flood Exposure: Moderate. This property is near a TRCA flood-risk area. Verify flood risk and insurance implications with your insurer."
- Low → no intersection → no flag shown

---

### 7. Building Health (dual-path: RentSafeTO or Permit History)

**This parameter has two paths depending on property type (per spec section 5).**

#### Path A: Rental Apartments (3+ storeys, 10+ units) — RentSafeTO

| Field | Value |
|-------|-------|
| Signal type | `observed` |
| Source | `apartment-building-evaluation` — Toronto Open Data CKAN |
| Format | CSV, XML, JSON |
| Records | 5,340 (2023–current) |
| Refresh | **Daily** (last: May 30, 2026) |
| Confidence | **Medium** |
| Data coverage | 2025 — 5,340 records |
| Version | V1 |

**Score distribution:**
- 170 buildings below 70 (3% of stock — elevated review recommended)
- 1,237 score 70–85 (further due diligence recommended)
- 3,932 score 85+ (pass)

**Score logic:**
- Below 70 → "Elevated review recommended."
- 70–85 → "Further due diligence recommended."
- 85+ → pass, no flag

**UI disclaimer (always show):** "No RentSafeTO score available does not indicate lower risk — condos, co-ops, and smaller buildings are not covered by this program."

#### Path B: All Other Property Types — Structural Permit History

| Field | Value |
|-------|-------|
| Signal type | `observed` |
| Source | `building-permits-active-permits` + `building-permits-cleared-permits` — CKAN |
| Records | 228,573 active + 401,265 cleared |
| Refresh | **Daily** |
| Confidence | **Medium** |
| Version | V1 |

**Filter BEFORE flagging (per spec) — only surface permits where:**
```python
PERMIT_TYPE contains: "structural", "unsafe", "major repair", "emergency"
# DO NOT flag: routine, cosmetic, or interior permits

structural_active  = [p for p in active_permits  if is_structural(p)]
structural_cleared = [p for p in cleared_permits if is_structural(p)
                      and within_last_10_years(p)]
```

**Logic:**
- `len(structural_active) > 0` → "Active structural permit found. Review permit details before purchase."
- `len(structural_cleared) >= 3` → "Further due diligence recommended — this property has a notable structural permit history."
- Otherwise → pass, no flag

---

### 8. Land Transfer Tax (Ontario LTT + Toronto MLTT)

| Field | Value |
|-------|-------|
| Source | Hardcoded brackets (Ontario Finance + City of Toronto Bylaw) |
| Reference | `municipal-land-transfer-tax-revenue-summary` (CKAN, retired) |
| Records | N/A (deterministic calculation) |
| Refresh | Static — changes by provincial/municipal legislation |
| Cost impact | **$20,000–$80,000+** one-time at closing |
| Confidence | **98%** |
| Version | V1 |

**What it captures:** The combined provincial and municipal land transfer tax payable at closing.

**Why it's a hidden cost:** Toronto is the ONLY municipality in Ontario with a double land transfer tax. Buyers outside Toronto pay only provincial LTT. The Toronto MLTT effectively doubles the tax. Progressive brackets reach 4.40–8.60% on the upper portion of high-value residential purchases (>$3M as of 2026).

**Calculation:**

Ontario LTT brackets:
| Purchase price bracket | Rate |
|----------------------|------|
| First $55,000 | 0.5% |
| $55,001–$250,000 | 1.0% |
| $250,001–$400,000 | 1.5% |
| $400,001–$2,000,000 | 2.0% |
| Over $2,000,000 | 2.5% |

Toronto MLTT brackets (as of April 1, 2026):
| Purchase price bracket | Rate |
|----------------------|------|
| First $55,000 | 0.5% |
| $55,001–$250,000 | 1.0% |
| $250,001–$400,000 | 1.5% |
| $400,001–$2,000,000 | 2.0% |
| $2,000,001–$3,000,000 | 2.5% |
| $3,000,001–$4,000,000 | 3.5% |
| $4,000,001–$5,000,000 | 4.5% |
| Over $5,000,000 | 5.5% |

**First-time buyer rebates:**
- Ontario LTT: up to ~$4,000
- Toronto MLTT: up to ~$4,475
- Combined maximum: ~$8,475

---

### 9. Property Tax (10-Year Projection)

| Field | Value |
|-------|-------|
| Signal type | `simulated` |
| Source | City of Toronto 2026 + MPAC methodology (hardcoded) |
| Reference | toronto.ca/services-payments/property-taxes-utilities/property-tax/property-tax-rates-and-fees |
| Records | N/A (deterministic calculation) |
| Refresh | Annual (rate set by City budget) |
| Cost impact | **$60,000–$100,000+** over 10 years |
| Confidence | **High** (rate) / **Medium** (assessed value estimate) |
| Data coverage | Tax rate confirmed 2026; MPAC values frozen at Jan 1 2016 |
| Version | V1 |

**Constants (per spec section 2):**
```python
RATE_RESIDENTIAL = 0.00767311
# 2026 confirmed: City 0.605295% + Education 0.153000% + City Building Fund 0.009016%

RATE_MULTI_RES   = 0.01208792
# investor / multi-residential 2026 (~57% higher than residential)

ANNUAL_GROWTH    = 0.035
# based on 9.5% (2024), 6.9% (2025), 2.2% (2026)
```

**Rate selection by buyer profile:**
- first-time / downsizer / owner-occupied → `RATE_RESIDENTIAL`
- investor → `RATE_MULTI_RES`

**Assessed value — property-type-specific multipliers (per spec):**
MPAC values are frozen at 2016. Do NOT use list_price directly. Output as range +/-15% and label as Medium confidence.

```python
if property_type == "condo":             multiplier = 0.90
elif property_type == "condo_townhouse": multiplier = 0.80
elif property_type == "semi_detached":   multiplier = 0.65
elif property_type == "detached_urban":  multiplier = 0.70
elif property_type == "detached_suburban": multiplier = 0.55
else:                                    multiplier = 0.70  # fallback

AV_mid  = list_price * multiplier
AV_low  = AV_mid * 0.85
AV_high = AV_mid * 1.15
```

**Calculation:**
```python
annual_property_tax   = AV_mid * RATE
ten_year_property_tax = annual_property_tax * ((pow(1 + ANNUAL_GROWTH, 10) - 1) / ANNUAL_GROWTH)
```

**UI disclaimer (always show):** "Estimated using property-type proxy. Actual taxes depend on your MPAC assessed value. MPAC values are frozen at January 1, 2016. 10-year projection is an illustrative scenario based on recent rate trends — not a confirmed forecast."

---

### 10. Mortgage Cost Layer

| Field | Value |
|-------|-------|
| Signal type | `simulated` |
| Source | Deterministic math — Government of Canada (hardcoded) |
| Records | N/A |
| Refresh | N/A |
| Cost impact | **$400,000–$750,000** over 10 years (largest single cost) |
| Confidence | **High** (math) / **Medium** (rate scenarios) |
| Data coverage | Current as of 2026 |
| Version | V1 |

**Rate renewal scenarios — 3 paths over 10 years (2 renewal cycles):**
| Scenario | Rate change | Use case |
|----------|------------|----------|
| Base | Rate stays flat | Expected path |
| Bear | +1.5% at each renewal | Rate shock |
| Bull | -0.5% at each renewal | Rate relief |

**CMHC premium logic:** If down payment <20% on a home ≤$1.5M, add mortgage default insurance premium (0.60–4.50% of mortgage amount) to the loan. Provincial sales tax on the premium is payable at closing. Surcharge of 0.20% for amortizations >25 years.

**Down payment rules (Canada):**
- 5% on first $500,000
- 10% on $500,001–$1,500,000
- 20% minimum for homes over $1,500,000 (no CMHC available)

**Sub-signals within mortgage layer (per spec section 7):**

**10a. Assumable Mortgage Flag**
Surface when `buyer_profile === "first-time"`. If the seller has a favorable locked-in rate, the buyer may negotiate to assume the mortgage — direct negotiation leverage.

**10b. Prepayment Penalty Signal (IRD Formula)**
Standard Interest Rate Differential formula: input seller's remaining term + current rate → estimated penalty → negotiation leverage signal. If the seller faces a large penalty to break their mortgage, they have less room to negotiate on price.

**10c. Amortization Accelerator**
Show the buyer how much they save by increasing payment frequency (monthly → bi-weekly → weekly) or making lump-sum payments. Quantifies the value of prepayment privileges.

**10d. FHSA Flag** (first-time buyers only)
Up to $40,000 tax-free. $8,000/yr contribution limit, tax-deductible in, tax-free out. Surface when `buyer_profile === "first-time"`.

**10e. RRSP Home Buyers' Plan Flag** (first-time buyers only)
Up to $60,000/person ($120,000/couple) tax-free RRSP withdrawal. Repayable over 15 years. Surface when `buyer_profile === "first-time"`.

---

## Tier 2: High-Value Expansion Parameters

**Version:** V1.5 (hackathon polish) or V2
**Status:** All datasets verified on Toronto Open Data CKAN

### 11. Building Construction/Demolition Violations

| Field | Value |
|-------|-------|
| Source | `building-construction-demolition-violations` — CKAN |
| Records | 48,035 |
| Refresh | **Daily** (last: May 30, 2026) |
| Cost impact | **$5,000–$30,000** remediation |
| Confidence | **85%** |
| Version | V1.5 |

**What it captures:** Open and closed building violation folders. Tracks compliance issues related to construction and demolition activities.

**Why it matters:** An open violation folder on a property means outstanding compliance issues. The buyer may inherit remediation obligations, fines, or stop-work orders. Even closed violations reveal a history of non-compliance that signals construction quality concerns.

---

### 12. Zoning By-law Restrictions

| Field | Value |
|-------|-------|
| Source | `zoning-by-law` — CKAN |
| Format | CSV, SHP, GeoJSON, GPKG |
| Records | 11,719 base zones + overlay layers |
| Refresh | Semi-annual (last: Feb 20, 2026) |
| Cost impact | Caps future value, restricts use |
| Confidence | **80%** |
| Version | V2 |

**What it captures:** Comprehensive zoning data including:
- Land use zones (residential, commercial, mixed)
- Height restrictions (2,528 overlay zones)
- Lot coverage limits (1,242 zones)
- Setback requirements
- Parking zone designations (913 zones)
- Rooming house overlays (558 zones)
- Policy areas (352 special zones)
- Retail street protections (643 streets)

**Why it matters:** Zoning determines what you can and cannot do with your property. Height limits cap addition potential. Lot coverage limits restrict building footprint expansion. Rooming house overlays affect rental income potential. Buyers often discover zoning constraints only after purchasing, when they apply for a building permit and get rejected.

---

### 13. Ravine & Natural Feature Protection Areas

| Field | Value |
|-------|-------|
| Source | `ravine-natural-feature-protection-area` — CKAN |
| Format | SHP |
| Records | City-wide polygons |
| Refresh | ~10–20 years (last: July 2019, data current May 2018) |
| Cost impact | **$10,000+** per landscaping/grading project |
| Confidence | **85%** |
| Version | V2 |

**What it captures:** Protection areas under Municipal Code Chapter 658. Covers ravine ecosystems and natural landscape features.

**Why it matters:** Properties within protection areas face restrictions on tree removal, grading changes, and construction near natural features. An addition, pool, or even significant landscaping may require environmental review and potentially be denied. Adds $10K+ to any project that touches the protected zone.

---

### 14. Environmentally Significant Areas (ESAs)

| Field | Value |
|-------|-------|
| Source | `environmentally-significant-areas` — CKAN |
| Records | 89 ESAs |
| Refresh | Annual (last: March 16, 2026) |
| Cost impact | Development restrictions, enhanced review requirements |
| Confidence | **80%** |
| Version | V2 |

**What it captures:** 89 areas designated on Map 12A of Toronto's Official Plan as requiring additional environmental protection.

**Why it matters:** Properties adjacent to or within ESAs face enhanced conservation measures. Additions, drainage changes, and grading may require environmental impact assessment. Can delay or prevent planned improvements.

---

### 15. Basement Flooding Study Areas

| Field | Value |
|-------|-------|
| Source | `basement-flooding-study-areas` — CKAN |
| Format | CSV, SHP, GeoJSON, GPKG |
| Records | 67 subsewersheds |
| Refresh | Annual (last: Feb 20, 2026) |
| Cost impact | Flood insurance premium + sewer backup risk |
| Confidence | **70%** |
| Version | V1.5 |

**What it captures:** Sanitary subsewershed boundaries — areas grouped by shared local sewer infrastructure, specifically those studied for basement flooding risk.

**Why it matters:** Complements TRCA flood data (river flooding) with sewer-related flooding risk. Properties in identified study areas have documented history of sewer backup during heavy rainfall. Insurance implications: sewer backup coverage riders cost $200–$500/year extra. Over 10 years: $2,000–$5,000.

**Note:** City explicitly warns this data is not for engineering applications. Use as a risk signal, not a definitive assessment.

---

### 16. Neighbourhood Crime Rates

| Field | Value |
|-------|-------|
| Source | `neighbourhood-crime-rates` — CKAN |
| Format | CSV, SHP, GeoJSON, GPKG |
| Records | 158 neighbourhoods |
| Refresh | Annual (last: Feb 20, 2026) |
| Cost impact | Insurance premium factor (10–30%) + resale trajectory |
| Confidence | **75%** |
| Version | V2 |

**What it captures:** Crime counts and rates per 100,000 population across 8 categories: assault, auto theft, break-and-enter, robbery, theft over, homicide, shootings/firearm discharges, theft from motor vehicle.

**Why it matters:** Break-and-enter and auto theft rates directly affect home insurance premiums. High-crime neighbourhoods see 10–30% higher insurance costs. Long-run, crime rates affect neighbourhood desirability and resale values. Over 10 years, insurance premium differences alone can total $5,000–$15,000.

---

### 17. Fire Incidents

| Field | Value |
|-------|-------|
| Source | `fire-incidents` — CKAN |
| Records | 36,564 |
| Refresh | Annual (last: Feb 20, 2026) |
| Cost impact | Insurance premium factor |
| Confidence | **65%** |
| Version | V2 |

**What it captures:** Fire incidents reported to Toronto Fire Services, aggregated to nearest intersection (not exact address) for privacy.

**Why it matters:** High-incident areas correlate with higher insurance premiums. Intersection-level aggregation is sufficient for neighbourhood-level risk assessment.

**Confidence rationale:** 65% due to geographic aggregation (intersection, not address) and annual-only refresh.

---

### 18. Development Pipeline

| Field | Value |
|-------|-------|
| Source | `development-pipeline` — CKAN |
| Records | 2,411 projects |
| Refresh | Semi-annual (last: May 21, 2026) |
| Cost impact | Construction disruption + future density signal |
| Confidence | **75%** |
| Version | V2 |

**What it captures:** Approved but not-yet-built development projects. Tracks how and where the city is growing.

**Why it matters:** A 500+ unit tower approved next door means 3–5 years of construction noise, crane shadows, increased traffic, and permanent density change. Also signals future infrastructure investment (positive) and property tax base expansion.

**Difference from Development Applications (#5):** Development Applications tracks the approval process. Development Pipeline tracks what's been approved and is coming. Pipeline = confirmed future; Applications = possible future.

---

### 19. Neighbourhood Intensification Projections (to 2051)

| Field | Value |
|-------|-------|
| Source | `neighbourhood-intensification-estimates-to-2051` — CKAN |
| Records | Ward-level |
| Refresh | One-time publication (last: July 3, 2025) |
| Cost impact | Future property tax trajectory |
| Confidence | **60%** |
| Version | V2 |

**What it captures:** City Planning's own projections for housing growth by ward under the Expanding Housing Options in Neighbourhoods (EHON) initiative, from 2021 baseline to 2051.

**Why it matters:** High-growth wards will see more construction activity, infrastructure pressure, and eventually higher property tax assessments as the neighbourhood densifies. This is the City's own estimate of where change is coming.

**Confidence rationale:** 60% because these are 25-year projections subject to policy changes, market conditions, and political decisions.

---

### 20. Apartment Building Registration

| Field | Value |
|-------|-------|
| Source | `apartment-building-registration` — CKAN |
| Records | 3,605 buildings |
| Refresh | Annual (last: May 5, 2026) |
| Cost impact | Building age = special assessment risk calibration |
| Confidence | **80%** |
| Version | V1.5 |

**What it captures:** Registration data for Toronto rental buildings with 3+ storeys and 10+ units. Includes building characteristics from owner/manager registration.

**Why it matters:** Building age is the single strongest predictor of special assessment risk in condos and apartment buildings. Registration data provides this when no other source does. Cross-reference with RentSafeTO evaluation scores (#7) for a complete building health picture.

---

### 21. Heritage Formerly Listed Properties

| Field | Value |
|-------|-------|
| Signal type | `observed` |
| Source | `heritage-formerly-listed` — Toronto Open Data CKAN |
| Format | SHP |
| Records | ~3,700 properties |
| Refresh | Annual (last: July 3, 2025) |
| Cost impact | Neighbourhood character change signal |
| Confidence | **Medium** |
| Version | V1.5 |

**What it captures:** Properties REMOVED from the Heritage Register due to Bill 23 (Ontario Heritage Act amendment imposing a 2-year listing deadline). Properties not designated within the deadline are delisted and cannot be re-listed for 5 years.

**Why it matters:** A formerly-listed property near yours could now be demolished or significantly altered — changing neighbourhood character. Also: if YOUR property was formerly listed, it means heritage protection was considered but not applied, which is useful context for renovation planning.

---

### 22. Residential Fire Inspection Results

| Field | Value |
|-------|-------|
| Signal type | `observed` |
| Source | `residential-fire-inspection-results` — Toronto Open Data CKAN |
| Format | CSV, SHP, GeoJSON, GPKG |
| Records | 124,414 |
| Refresh | **Daily** (last: May 2, 2026) |
| Cost impact | Fire code violations = building safety signal |
| Confidence | **High** |
| Version | V1.5 |

**What it captures:** Fire Prevention inspection results for high-rise residential properties since January 2017. Building-level data showing compliance status with Ontario Fire Code. Includes both properties with violations and those passing.

**Why it matters:** Far more precise than `fire-incidents` (which is aggregated to intersection). A building with fire code violations signals infrastructure neglect, higher insurance premiums, and potential remediation costs. For condo buyers, violations may indicate management/board issues.

---

### 23. Committee of Adjustment Applications

| Field | Value |
|-------|-------|
| Signal type | `observed` |
| Source | `committee-of-adjustment-applications` — Toronto Open Data CKAN |
| Format | CSV, XML, JSON |
| Records | 2,954 active + 33,030 closed (since 2017) |
| Refresh | **Daily** (last: May 30, 2026) |
| Cost impact | Zoning change signal — affects property and neighbours |
| Confidence | **Medium** |
| Version | V2 |

**What it captures:** Applications for minor variances to zoning by-laws and consents. Covers all four regional panels (Etobicoke York, North York, Toronto & East York, Scarborough).

**Why it matters:** Active CoA applications on the property itself mean pending zoning changes. Active applications nearby signal that neighbours are seeking variances (adding units, changing use, exceeding height) that could affect your property. Complements development applications (#5) with finer-grained zoning variance data.

---

### 24. Preliminary Zoning Reviews

| Field | Value |
|-------|-------|
| Signal type | `observed` |
| Source | `preliminary-zoning-reviews` — Toronto Open Data CKAN |
| Format | CSV, XML, JSON |
| Records | 220,052 |
| Refresh | **Daily** (last: May 30, 2026) |
| Cost impact | Use change signal |
| Confidence | **Medium** |
| Version | V2 |

**What it captures:** Zoning certificate requests, zoning use reviews, business license compliance, and liquor license reviews. Tracks whether specific locations are being evaluated for new or changed uses.

**Why it matters:** High volume of zoning reviews near a property signals potential use changes (commercial, retail, licensed establishments) that could affect livability and property values.

---

### 25. Property Tax Relief and Rebate Programs

| Field | Value |
|-------|-------|
| Signal type | `observed` |
| Source | City of Toronto — hardcoded program rules |
| Reference | toronto.ca/services-payments/property-taxes-utilities/property-tax/ |
| Cost impact | Potential tax savings (varies by program) |
| Confidence | **High** |
| Version | V1.5 |

**What it captures:** Tax relief programs beyond the LTT first-time buyer rebate. Surfaced based on buyer profile.

**Programs to flag:**
- First-time buyer LTT rebates (already in #8): $8,475 combined
- FHSA: up to $40,000 tax-free (already in #10d)
- RRSP HBP: up to $60,000 (already in #10e)
- Property tax deferrals (seniors, low-income)
- Tax reductions for new multi-residential builds (15% reduction under special subclass)
- Small business property tax subclass (15% reduction — `small-business-property-tax-subclass-eligible-properties`, 28,222 records)

**Surface ALL of the following when `buyer_profile === "first-time"` (per spec):**
- LTT rebate: $8,475 combined
- FHSA: up to $40,000 tax-free
- RRSP HBP: up to $60,000
- Assumable mortgage flag
- Property Tax Relief and Rebates page link

---

## Tier 3: Transit & Lifestyle Cost Parameters

**Version:** V1.5 (Transit Dividend) / V2/V3 (remaining)

### 26. TTC Transit Proximity (Transit Dividend)

| Field | Value |
|-------|-------|
| Signal type | `simulated` |
| Source | `ttc-routes-and-schedules` — CKAN (GTFS format) |
| Refresh | Every 6 weeks (last: May 23, 2026) |
| Cost impact | **$8,000–$12,000/yr** transport cost differential |
| Confidence | **High** |
| Version | **V1.5** (promoted from V3 — already shown in PDF frontend) |

**What it captures:** Complete TTC route definitions, stop locations, stop patterns, and schedules in standard GTFS format. Covers subway, streetcar, and bus.

**Why it matters:** The "Transit Dividend" — the dollar savings from living near rapid transit versus a car-dependent location. Two-car suburban household: ~$12,000/yr. TTC-dependent household: ~$3,300/yr. Over 10 years: $87,000 difference. This is often the largest hidden cost differential between two properties at the same list price.

**Query logic:** Compute distance from property to nearest subway station and nearest streetcar/bus stop. Score transit accessibility. Model transport cost based on transit score vs. car-dependent baseline.

---

### 27. TTC Subway Station Ridership

| Field | Value |
|-------|-------|
| Source | `ttc-ridership-subway-scarborough-rt-station-usage` — CKAN |
| Refresh | Annual |
| Cost impact | Convenience premium / resale signal |
| Confidence | **70%** |
| Version | V3 |

**What it captures:** Station-level ridership data for all TTC subway stations.

**Why it matters:** High-ridership stations indicate strong transit connectivity. Proximity to high-ridership stations correlates with property value resilience and rental demand. A secondary validation for the Transit Dividend calculation.

---

### 28. Motor Vehicle Collisions (KSI)

| Field | Value |
|-------|-------|
| Source | `motor-vehicle-collisions-involving-killed-or-seriously-injured-persons` — CKAN |
| Records | 20,519 |
| Refresh | **Daily** (last: May 30, 2026) |
| Cost impact | Safety signal / family livability |
| Confidence | **75%** |
| Version | V3 |

**What it captures:** All collisions where at least one person was killed or seriously injured, since 2006. Location-specific within Toronto's right-of-way.

**Why it matters:** Collision hotspots near the property affect family safety perception, walkability, and long-run neighbourhood desirability. Particularly relevant for families with children evaluating school-route safety.

---

### 29. Noise Exemption Permits

| Field | Value |
|-------|-------|
| Source | `noise-exemption-permits` — CKAN |
| Records | 3,685 |
| Refresh | Weekly (last: May 26, 2026) |
| Cost impact | Quality of life / construction disruption |
| Confidence | **65%** |
| Version | V3 |

**What it captures:** Approved permits for activities that exceed Toronto's Noise Bylaw decibel limits and time restrictions.

**Why it matters:** High density of noise permits in an area = active construction zone. Construction disruption can persist for years and materially affect livability. Cross-reference with Development Pipeline (#18) for a complete disruption picture.

---

### 30. Short-Term Rental Activity

| Field | Value |
|-------|-------|
| Source | `short-term-rental-program-data` — CKAN |
| Records | 768 |
| Refresh | Monthly (last: May 26, 2026) |
| Cost impact | Condo neighbour quality / board dispute risk |
| Confidence | **55%** |
| Version | V3 |

**What it captures:** Registration and enforcement data for Toronto's short-term rental program (Municipal Code Ch. 547).

**Why it matters:** High STR density in a condo building means transient neighbours, accelerated wear on common elements, and potential bylaw disputes. Can signal future special assessments for increased maintenance. Low confidence due to limited records and enforcement lag.

---

## Tier 4: Financial & Tax Structure Parameters

**Version:** V1 (LTT + mortgage), V2 (FHSA/HBP), V3 (full optimizer)
**Source:** Federal/provincial government rules, CMHC, industry benchmarks

### 31. CMHC Mortgage Insurance Premium

| Field | Value |
|-------|-------|
| Source | CMHC published premium schedule |
| Cost impact | **$2,000–$30,000+** added to mortgage |
| Confidence | **98%** |
| Version | V1 |

Premium schedule:

| Loan-to-Value | Premium (% of mortgage) |
|---------------|------------------------|
| Up to 65% | 0.60% |
| 65.01–75% | 1.70% |
| 75.01–80% | 2.40% |
| 80.01–85% | 2.80% |
| 85.01–90% | 3.10% |
| 90.01–95% | 4.00% |
| Non-traditional down payment | 4.50% |

Plus: Ontario PST (8%) on the premium, payable at closing (not financed).

Surcharge for amortizations >25 years: additional 0.20%.

---

### 32. First Home Savings Account (FHSA)

| Field | Value |
|-------|-------|
| Source | CRA published rules |
| Cost impact | Up to **$40,000** tax-sheltered down payment |
| Confidence | **95%** |
| Version | V3 |

- Annual contribution limit: $8,000
- Lifetime limit: $40,000
- Unused room carry-forward: up to $8,000
- Tax treatment: deductible in, tax-free out (for qualifying home purchase)
- Overcontribution penalty: 1%/month
- Effective benefit: reduces down payment cost by marginal tax rate (20–53% depending on income)

---

### 33. RRSP Home Buyers' Plan (HBP)

| Field | Value |
|-------|-------|
| Source | CRA published rules |
| Cost impact | Up to **$60,000/person** ($120,000/couple) tax-free RRSP withdrawal |
| Confidence | **95%** |
| Version | V3 |

- Withdrawal limit: $60,000 per person (increased from $35,000 in 2024)
- Repayment: over 15 years, starting year 2 after withdrawal
- Minimum annual repayment: 1/15 of total withdrawn
- Missed repayment: shortfall added to taxable income
- Opportunity cost: lost RRSP compounding over repayment period

---

### 34. Vacant Home Tax (VHT)

| Field | Value |
|-------|-------|
| Source | City of Toronto bylaw |
| Cost impact | **3% of assessed value/year** if vacant |
| Confidence | **90%** |
| Version | V2 |

- Annual occupancy declaration required
- Failure to file = deemed vacant
- Relevant for: investment properties, pied-a-terre, properties in transition
- On an $800K property: $24,000/year if triggered

---

### 35. Home Insurance Baseline

| Field | Value |
|-------|-------|
| Source | Industry benchmarks + Open Data inputs |
| Cost impact | **$1,800–$4,800/yr** |
| Confidence | **70%** |
| Version | V2 |

Insurance premium is driven by:
- Building age and construction type
- Distance to fire station (`fire-station-locations` — CKAN)
- Flood zone status (TRCA data, #6)
- Neighbourhood crime rates (#16)
- Fire incident density (#17)
- Claims history in building/neighbourhood
- Presence of risk factors (knob-and-tube wiring, flat roof, basement apartment)

Meridian can model a range using Open Data inputs but cannot compute exact premiums (those require insurer-specific underwriting).

---

### 36. Maintenance Complexity Signal (Composite — per spec section 8)

| Field | Value |
|-------|-------|
| Signal type | `inferred` |
| Source | Composite of permit history + RentSafeTO + flood + heritage |
| Cost impact | Risk inference — not a dollar estimate |
| Confidence | **Low** (ALWAYS display Low — never upgrade) |
| Version | V1 |

No public dataset exists for condo special assessment history (condo corporation records, status certificates, and reserve fund studies are private documents). This is risk inference, not factual history. Never claim otherwise.

**Signals used (all already in stack):**
- Building age → earliest permit date from Building Permits dataset
- Active structural permits → Building Permits active (filtered)
- Permit frequency → Building Permits cleared, structural count over 10 years
- RentSafeTO score → section 7 Path A (if applicable)
- Heritage designation → section 1
- Flood zone exposure → section 6

**Output tiers:** Low / Medium / Elevated

**Output format (per spec):**
```json
{
  "signal_name": "Maintenance Complexity Signal",
  "signal_type": "inferred",
  "value":       "Elevated",
  "label":       "Elevated maintenance complexity signal",
  "factors":     ["Older building age", "Multiple structural permits",
                  "Flood zone exposure"],
  "disclaimer":  "This is an inferred signal and not evidence of a
                  historical special assessment.",
  "confidence":  "Low"
}
```

**Wording rules:** NEVER output "High special assessment risk." USE INSTEAD: "Elevated maintenance complexity signal." Always show disclaimer. Always show confidence as Low.

---

### 37. Future Tax Pressure Signal (Composite — per spec section 9)

| Field | Value |
|-------|-------|
| Signal type | `inferred` |
| Source | Composite of development density + permit activity + heritage/zoning |
| Cost impact | Neighbourhood change signal — not a tax prediction |
| Confidence | **Low** (ALWAYS display Low — never upgrade) |
| Version | V1 |

NOT labelled "projected reassessment." MPAC reassessments are policy-driven, frozen at 2016 values, and unpredictable.

**Signals used:**
- Development application density within 500m → parameter #5
- Recent permit activity → Building Permits active + cleared
- Heritage and zoning constraints → parameters #1 + intake agent

**Output tiers:** Low / Medium / High

**Output format (per spec):**
```json
{
  "signal_name":   "Future Tax Pressure Signal",
  "signal_type":   "inferred",
  "value":         "High",
  "confidence":    "Low",
  "message":       "Neighbourhood intensification signal detected.
                    Significant redevelopment activity nearby may
                    affect future neighbourhood character and values.",
  "disclaimer":    "This is a neighbourhood signal, not an MPAC
                    reassessment prediction."
}
```

**Wording rules:** Frame as "neighbourhood reassessment risk signal." NEVER frame as MPAC prediction or tax guarantee.

---

### 38. Utilities Estimate

| Field | Value |
|-------|-------|
| Source | Property size + building age model |
| Cost impact | **$3,000–$8,000/yr** |
| Confidence | **60%** |
| Version | V2 |

No per-property utility data available on Toronto Open Data. Model from:
- Square footage (estimated from property boundaries + building type)
- Building age (older = less energy efficient)
- Heating system type (gas vs. electric, inferred from building era)
- Condo vs. freehold (utilities may be included in condo fees)

---

### 39. Maintenance Reserve

| Field | Value |
|-------|-------|
| Source | Industry rule + building-specific adjustments |
| Cost impact | **$8,000–$35,000/yr** on $850K+ homes |
| Confidence | **55%** |
| Version | V2 |

Industry guideline: 1–3% of property value per year. Adjust using:
- Building age (older = higher end of range)
- Permit history (frequent structural permits = chronic issues)
- RentSafeTO scores (low scores = deferred maintenance)
- Construction type (wood frame vs. concrete vs. masonry)

---

## Tier 5: Opportunity Cost & Macro Parameters

**Version:** V3 (Decision Engine)

### 40. Down Payment Opportunity Cost

| Field | Value |
|-------|-------|
| Source | Market return assumptions |
| Cost impact | **$50,000–$150,000** over 10 years |
| Confidence | **50%** |
| Version | V3 |

What the down payment would earn if invested in a diversified portfolio instead of locked in home equity. Assumption range: 6–8% nominal annual return. Critical for buy-vs-rent comparison.

---

### 41. Rent Trajectory (Comparable Unit)

| Field | Value |
|-------|-------|
| Source | CMHC Rental Market Reports + escalator |
| Cost impact | **$300,000–$450,000** over 10 years |
| Confidence | **55%** |
| Version | V3 |

Projected rent for a comparable unit over 10 years with 3% annual increase (Toronto historical average). No reliable per-unit Open Data source. Use CMHC rental market survey data for neighbourhood-level benchmarks.

---

### 42. Interest Rate Renewal Risk

| Field | Value |
|-------|-------|
| Source | Bank of Canada rate path scenarios |
| Cost impact | **$500–$900/mo payment shock** at 5-year renewal |
| Confidence | **40%** |
| Version | V3 |

The biggest single uncertainty for any leveraged buyer. Meridian stress-tests at current rate + 1.5% for the bear scenario. Context: 1.15 million Canadian mortgages renewing in 2026 at rates above original signing.

---

### 43. Neighbourhood Appreciation Trajectory

| Field | Value |
|-------|-------|
| Source | `neighbourhood-profiles` + `development-pipeline` + Census |
| Cost impact | **+/- $100,000+** over 10 years |
| Confidence | **35%** |
| Version | V3 |

Combine neighbourhood demographics (income, education, population growth), development pipeline, intensification projections, and transit investment to model neighbourhood trajectory. Highly speculative. Meridian presents scenarios, not predictions.

---

## RAG Corpus (Agent 4 Retrieval Documents — per spec)

Agent 4 (synthesis) retrieves from these documents conditionally based on which flags are active:

| Trigger | Retrieval query | Document |
|---------|----------------|----------|
| Heritage flag active | "heritage designation renovation restrictions Toronto" | Ontario Heritage Act (ontario.ca/laws/statute/90o18) |
| Flood flag active | "TRCA flood zone insurance implications Toronto" | TRCA Living with Flooding guide (trca.ca/conservation/flood-risk-management) |
| `buyer_profile === "first-time"` | "FHSA RRSP home buyers plan eligibility 2026" | CRA FHSA guide + CRA HBP guide |
| Maintenance signal === "Elevated" | "condo reserve fund special assessment Ontario" | MPAC Understanding Your Assessment guide |
| Tax pressure === "High" | "Toronto development application zoning amendment implications" | Toronto Zoning By-law 569-2013 |

Full corpus paths defined in spec under `/rag_corpus/` with legislation, programs, assessment, zoning, flood, mortgage (OSFI B-20), and heritage categories.

---

## V2 Roadmap Parameters (per spec — out of scope for hackathon)

These are mentioned in the spec's V2 roadmap and should be tracked for future implementation:

| Parameter | Source | Notes |
|-----------|--------|-------|
| evidence_strength float (0.0–1.0) | Computed | Replaces High/Medium/Low with continuous score |
| MPAC-calibrated regression | MPAC + postal prefix | Property-type-specific assessed value model |
| Permit classification engine | Building permits | Proactive vs. distress repair classification |
| Address normalization layer | Address Points | Condo unit matching, geocoding confidence score |
| Z-score normalization for dev density | Development apps | City-mean and std computed from full dataset |
| TSSA elevator violations | TSSA portal (external) | For condo buildings — infrastructure neglect signal |
| Utility rate forecasts | Energy data | `annual-energy-consumption` (CKAN, city buildings only) |
| Condo litigation signals | Court records (external) | No public dataset; would need CanLII scraping |
| Insurance market signals | Industry data (external) | No public dataset |
| Rent vs. buy comparison | CMHC rental survey | Requires rental price data source |

---

## Data Source Summary

### Toronto Open Data CKAN Datasets Used

| # | Dataset Slug | Records | Refresh | Tier |
|---|-------------|---------|---------|------|
| 1 | `heritage-register` | ~12,320 | Quarterly | T1 |
| 2 | `heritage-conservation-districts` | ~30 | Quarterly | T1 |
| 3 | `building-permits-active-permits` | 228,573 | Daily | T1 |
| 4 | `building-permits-cleared-permits` | 401,265 | Daily | T1 |
| 5 | `development-applications` | 26,254 | Daily | T1 |
| 6 | `apartment-building-evaluation` | 5,341+ | Daily | T1 |
| 7 | `apartment-building-registration` | 3,605 | Annual | T2 |
| 8 | `building-construction-demolition-violations` | 48,035 | Daily | T2 |
| 9 | `zoning-by-law` | 11,719+ | Semi-annual | T2 |
| 10 | `ravine-natural-feature-protection-area` | City-wide | ~10-20yr | T2 |
| 11 | `environmentally-significant-areas` | 89 | Annual | T2 |
| 12 | `basement-flooding-study-areas` | 67 | Annual | T2 |
| 13 | `neighbourhood-crime-rates` | 158 | Annual | T2 |
| 14 | `fire-incidents` | 36,564 | Annual | T2 |
| 15 | `development-pipeline` | 2,411 | Semi-annual | T2 |
| 16 | `neighbourhood-intensification-estimates-to-2051` | Ward-level | One-time | T2 |
| 17 | `heritage-formerly-listed` | ~3,700 | Annual | T2 |
| 18 | `residential-fire-inspection-results` | 124,414 | Daily | T2 |
| 19 | `committee-of-adjustment-applications` | 35,984 | Daily | T2 |
| 20 | `preliminary-zoning-reviews` | 220,052 | Daily | T2 |
| 21 | `small-business-property-tax-subclass-eligible-properties` | 28,222 | Annual | T2 |
| 22 | `address-points-municipal-toronto-one-address-repository` | 525,435 | Weekly | Infra |
| 23 | `property-boundaries` | 498,499 | Daily | Infra |
| 24 | `ttc-routes-and-schedules` | All routes | 6 weeks | T1.5 |
| 25 | `ttc-ridership-subway-scarborough-rt-station-usage` | All stations | Annual | T3 |
| 26 | `motor-vehicle-collisions-involving-killed-or-seriously-injured-persons` | 20,519 | Daily | T3 |
| 27 | `noise-exemption-permits` | 3,685 | Weekly | T3 |
| 28 | `short-term-rental-program-data` | 768 | Monthly | T3 |
| 29 | `neighbourhood-profiles` | 2,383 | Census cycle | T3 |

### External Data Sources

| Source | Type | Tier |
|--------|------|------|
| TRCA ArcGIS (flood plain) | Public API | T1 |
| Ontario LTT brackets | Government publication | T1 |
| Toronto MLTT brackets | City bylaw | T1 |
| CMHC premium schedule | Government publication | T1 |
| CRA FHSA rules | Government publication | T4 |
| CRA RRSP HBP rules | Government publication | T4 |
| City of Toronto VHT bylaw | City bylaw | T4 |
| CMHC Rental Market Survey | Government publication | T5 |
| Bank of Canada rate data | Government publication | T5 |

### CKAN API Base URL

```
https://ckan0.cf.opendata.inter.prod-toronto.ca/api/3/
```

All datasets queryable via `action/package_show?id=<slug>` for metadata and `action/datastore_search?resource_id=<id>&q=<query>` for data.

---

## Build Priority for Hackathon

### Must-have (V1 core — 10 parameters + 6 inputs + 2 composite signals)
- Input parameters I1–I6 (address, list price, buyer profile, down payment, rate, amortization)
- Parameters 1–10 (heritage, HCDs, active permits, cleared permits, dev apps, flood, building health, LTT, property tax, mortgage layer including sub-signals 10a–10e)
- Parameters 36–37 (maintenance complexity signal, future tax pressure signal — composite inferred signals)

### Should-have (V1.5 cherry-picks)
- **#21 Heritage Formerly Listed** — 15 min build, same heritage shapefile pattern
- **#22 Residential Fire Inspections** — 30 min build, 124K records, building-level precision
- **#25 Property Tax Relief Programs** — 30 min build, hardcoded rules by buyer profile
- **#26 Transit Dividend** — 1 hr build, GTFS data ready, already shown in PDF frontend
- **#11 Building violations** — 30 min build, same CKAN pattern
- **#15 Basement flooding areas** — 30 min build, complements TRCA flood data

### Nice-to-have (V2 — post-hackathon)
Parameters 12–20, 23–24, 31–39. Require spatial joins, proxy models, or cross-source inference. Plus V2 roadmap items (TSSA, condo litigation, permit classification engine).

### Future (V3 — product)
Parameters 27–30, 40–43. Require external data, macro assumptions, or Flinks integration.
