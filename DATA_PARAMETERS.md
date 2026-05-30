# Meridian — Hidden Cost Data Parameters

> Comprehensive inventory of every data point Meridian uses (or can use) to compute the true 10-year cost of owning residential property in Toronto. Cross-referenced against Toronto Open Data (open.toronto.ca), TRCA ArcGIS, federal/provincial tax rules, and industry benchmarks.
>
> Last updated: 2026-05-30

---

## How to Read This Document

Each parameter includes:

- **Data source** — where the data comes from, with CKAN dataset slug or API endpoint
- **Records** — approximate dataset size
- **Refresh** — how often the source updates
- **Cost impact** — estimated dollar effect on 10-year ownership cost
- **Confidence** — how reliably Meridian can quantify this parameter (0–100%)
- **V1/V2/V3** — which product version includes this parameter

Confidence scoring rules (aligned with Agent 2 retrieval logic):
- 90–100%: deterministic math or direct government data with timestamps
- 75–89%: live API with clean data, minor interpretation needed
- 60–74%: data available but requires spatial inference or proxy logic
- 40–59%: modeled estimate, multiple assumptions
- Below 40%: speculative, macro-dependent

---

## Tier 1: Core Hidden Cost Drivers

**Version:** V1 (Hackathon build)
**Status:** All data sources verified live on CKAN as of May 30, 2026

### 1. Heritage Designation (Part IV / Part V / Listed)

| Field | Value |
|-------|-------|
| Source | `heritage-register` — Toronto Open Data CKAN |
| Format | SHP (Shapefile), WGS84 |
| Records | ~10,000 properties |
| Refresh | Quarterly (last: May 21, 2026) |
| Cost impact | **$40,000–$60,000** renovation premium per exterior project |
| Confidence | **95%** |
| Version | V1 |

**What it captures:** Properties designated or listed under the Ontario Heritage Act. Part IV = individual designation. Part V = within a Heritage Conservation District. Listed = on the register but not yet designated (still triggers 60-day demolition delay).

**Why it's a hidden cost:** Any exterior alteration (windows, facade, roof, additions) requires City Heritage Permit approval. Typical approval timeline: 6–18 months. Typical cost premium: 40–60% over standard renovation. Buyers see the list price but not the permanent renovation constraint.

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

### 5. Development Applications (500m Radius)

| Field | Value |
|-------|-------|
| Source | `development-applications` — Toronto Open Data CKAN |
| Format | CSV, XML, JSON |
| Records | 26,254 |
| Refresh | **Daily** (last: May 30, 2026) |
| Cost impact | Future property tax pressure + construction disruption |
| Confidence | **80%** |
| Version | V1 |

**What it captures:** All active and inactive Community Planning and Committee of Adjustment applications since January 2008. Includes rezoning, site plan, minor variance, consent, and Official Plan amendment applications.

**Why it's a hidden cost:** High density of development applications within 500m signals neighbourhood intensification. Consequences:
- Rising assessed property values = higher property taxes
- Years of construction noise and disruption
- Changed neighbourhood character (shadowing, traffic, density)
- Potential positive: increased transit investment and amenities

**Query logic:** Geocode property → radius query (500m) → count active applications → classify by type (residential intensification, mixed-use, commercial). 10+ active applications within 500m = "development pressure" flag.

---

### 6. TRCA Flood Zone

| Field | Value |
|-------|-------|
| Source | TRCA Conservation Authority ArcGIS REST API |
| Fallback | Local GeoJSON: `data/trca_floodplain_toronto.geojson` |
| Format | ArcGIS Feature Service / GeoJSON |
| Records | Toronto-wide flood plain polygons |
| Refresh | Varies (no extract_date in API response) |
| Cost impact | **$1,800–$3,200/yr** flood insurance premium + basement restrictions |
| Confidence | **75%** |
| Version | V1 |

**What it captures:** TRCA regulatory flood plain boundaries — areas at risk of flooding from rivers (Don, Humber, Rouge, etc.) in a 1-in-100-year storm event.

**Why it's a hidden cost:** Two concrete impacts:
1. Flood insurance riders are effectively mandatory in-zone: $1,800–$3,200/yr additional premium
2. City restricts basement finishing permits in flood-regulated areas, capping renovation upside

Over 10 years, flood insurance alone adds $18,000–$32,000. Basement restriction reduces usable square footage value.

**Query logic:** Point-in-polygon against flood plain boundaries. If property falls within regulatory flood line, flag it.

**Confidence rationale:** 75% because TRCA data lacks an `extract_date` field, so recency cannot be verified programmatically. Direct TRCA confirmation recommended before signing.

---

### 7. RentSafeTO Building Evaluation Scores

| Field | Value |
|-------|-------|
| Source | `apartment-building-evaluation` — Toronto Open Data CKAN |
| Format | CSV, XML, JSON |
| Records | 5,341 (2023–current) + 11,760 (pre-2023) |
| Refresh | **Daily** (last: May 30, 2026) |
| Cost impact | Special assessment risk proxy: **$10,000–$50,000+** |
| Confidence | **85%** |
| Version | V1 (condo/apartment properties only) |

**What it captures:** Inspection scores across 50 evaluation categories for Toronto rental buildings (3+ storeys, 10+ units). Categories include mechanical systems, common areas, security, parking, exterior grounds. Each category scored 1–3 (1 = lowest).

**Why it's a hidden cost:** Low evaluation scores are the strongest available proxy for deferred maintenance in apartment and condo buildings. Buildings scoring poorly on mechanical systems and structural categories are statistically likely to face special assessments within 3–5 years. Special assessments can range from $10,000 to $50,000+ per unit for major capital projects (elevator replacement, garage repair, balcony remediation, cladding).

**Query logic:** Match property address to registered buildings. If found, retrieve latest evaluation. Compute average score across all 50 categories. Flag if average <2.0 or if any structural/mechanical category scores 1.

**Applicability:** Only for buildings registered under RentSafeTO. Agent 1 (Planning) skips this source for freehold houses and low-rise buildings.

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
| Source | City of Toronto published rates + MPAC assessment baseline |
| Reference | `current-value-assessment-cva-tax-impact-residential-properties` (CKAN, 2019) |
| Records | N/A (deterministic calculation) |
| Refresh | Annual (rate set by City budget) |
| Cost impact | **$60,000–$100,000+** over 10 years |
| Confidence | **90%** |
| Version | V1 |

**What it captures:** Annual property tax computed from assessed value and the combined residential tax rate (municipal + education + City Building Fund).

**Key parameters:**
- 2025 residential rate: ~0.754% of assessed value
- 2026 projected increase: ~2.2% (per CBC/City budget)
- Annual escalator assumption: 2.5% (Toronto historical average)
- MPAC assessment base: frozen at January 1, 2016 valuations

**Why it's a hidden cost:** Toronto's rate looks low compared to other Ontario cities, but applied to Toronto price points ($800K–$1.5M+), the absolute dollar amount is significant. The real hidden risk is the MPAC reassessment freeze — when reassessments resume, properties that appreciated significantly since 2016 will see a tax shock. This is the single most uncertain long-run cost driver.

---

### 10. Mortgage Cost (3-Scenario Model)

| Field | Value |
|-------|-------|
| Source | Deterministic math |
| Records | N/A |
| Refresh | N/A |
| Cost impact | **$400,000–$750,000** over 10 years (largest single cost) |
| Confidence | **95%** |
| Version | V1 |

**What it captures:** Monthly mortgage payments under three interest rate scenarios applied to a standard 25-year amortization with 20% down payment.

**Three scenarios:**
| Scenario | Rate | Use case |
|----------|------|----------|
| Base | Current 5-year fixed (~5.14%) | Expected path |
| Bear | Current + 1.5% (~6.64%) | Rate shock at renewal |
| Bull | Current - 1.0% (~4.14%) | Rate relief |

**CMHC premium logic:** If down payment <20% on a home ≤$1.5M, add mortgage default insurance premium (0.60–4.50% of mortgage amount) to the loan. Provincial sales tax on the premium is payable at closing.

**Down payment rules (Canada):**
- 5% on first $500,000
- 10% on $500,001–$1,500,000
- 20% minimum for homes over $1,500,000 (no CMHC available)

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

## Tier 3: Transit & Lifestyle Cost Parameters

**Version:** V2/V3 (post-hackathon)

### 21. TTC Transit Proximity (Transit Dividend)

| Field | Value |
|-------|-------|
| Source | `ttc-routes-and-schedules` — CKAN (GTFS format) |
| Refresh | Every 6 weeks (last: May 23, 2026) |
| Cost impact | **$8,000–$12,000/yr** transport cost differential |
| Confidence | **85%** |
| Version | V3 |

**What it captures:** Complete TTC route definitions, stop locations, stop patterns, and schedules in standard GTFS format. Covers subway, streetcar, and bus.

**Why it matters:** The "Transit Dividend" — the dollar savings from living near rapid transit versus a car-dependent location. Two-car suburban household: ~$12,000/yr. TTC-dependent household: ~$3,300/yr. Over 10 years: $87,000 difference. This is often the largest hidden cost differential between two properties at the same list price.

**Query logic:** Compute distance from property to nearest subway station and nearest streetcar/bus stop. Score transit accessibility. Model transport cost based on transit score vs. car-dependent baseline.

---

### 22. TTC Subway Station Ridership

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

### 23. Motor Vehicle Collisions (KSI)

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

### 24. Noise Exemption Permits

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

### 25. Short-Term Rental Activity

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

### 26. CMHC Mortgage Insurance Premium

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

### 27. First Home Savings Account (FHSA)

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

### 28. RRSP Home Buyers' Plan (HBP)

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

### 29. Vacant Home Tax (VHT)

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

### 30. Home Insurance Baseline

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

### 31. Property Tax Reassessment Risk (MPAC)

| Field | Value |
|-------|-------|
| Source | MPAC policy + market data |
| Cost impact | **20–40% tax increase** when reassessment resumes |
| Confidence | **50%** |
| Version | V2 |

- Current assessed values frozen at January 1, 2016 levels
- Market values in many Toronto neighbourhoods have increased 50–100%+ since 2016
- When MPAC reassessments resume (timing unknown), assessed values will jump
- Properties that appreciated most will see the largest tax increases
- Phase-in over 4 years is typical but not guaranteed

This is the single highest-uncertainty parameter in the model. Meridian should present it as a scenario, not a prediction.

---

### 32. Condo Maintenance Fee Trajectory

| Field | Value |
|-------|-------|
| Source | Building age proxy + RentSafeTO scores |
| Cost impact | **3–8% annual increase** for aging buildings |
| Confidence | **65%** |
| Version | V2 |

No direct Open Data source for condo fees. Meridian uses:
- Building age (from `apartment-building-registration`, #20) as primary predictor
- RentSafeTO evaluation scores (#7) as condition signal
- Cleared permit history (#4) as maintenance pattern indicator

Rule of thumb: buildings 20+ years old with below-average RentSafeTO scores face 6–8% annual fee increases. New buildings: 3–4%.

---

### 33. Utilities Estimate

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

### 34. Maintenance Reserve

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

### 35. Down Payment Opportunity Cost

| Field | Value |
|-------|-------|
| Source | Market return assumptions |
| Cost impact | **$50,000–$150,000** over 10 years |
| Confidence | **50%** |
| Version | V3 |

What the down payment would earn if invested in a diversified portfolio instead of locked in home equity. Assumption range: 6–8% nominal annual return. Critical for buy-vs-rent comparison.

---

### 36. Rent Trajectory (Comparable Unit)

| Field | Value |
|-------|-------|
| Source | CMHC Rental Market Reports + escalator |
| Cost impact | **$300,000–$450,000** over 10 years |
| Confidence | **55%** |
| Version | V3 |

Projected rent for a comparable unit over 10 years with 3% annual increase (Toronto historical average). No reliable per-unit Open Data source. Use CMHC rental market survey data for neighbourhood-level benchmarks.

---

### 37. Interest Rate Renewal Risk

| Field | Value |
|-------|-------|
| Source | Bank of Canada rate path scenarios |
| Cost impact | **$500–$900/mo payment shock** at 5-year renewal |
| Confidence | **40%** |
| Version | V3 |

The biggest single uncertainty for any leveraged buyer. Meridian stress-tests at current rate + 1.5% for the bear scenario. Context: 1.15 million Canadian mortgages renewing in 2026 at rates above original signing.

---

### 38. Neighbourhood Appreciation Trajectory

| Field | Value |
|-------|-------|
| Source | `neighbourhood-profiles` + `development-pipeline` + Census |
| Cost impact | **+/- $100,000+** over 10 years |
| Confidence | **35%** |
| Version | V3 |

Combine neighbourhood demographics (income, education, population growth), development pipeline, intensification projections, and transit investment to model neighbourhood trajectory. Highly speculative. Meridian presents scenarios, not predictions.

---

## Data Source Summary

### Toronto Open Data CKAN Datasets Used

| # | Dataset Slug | Records | Refresh | Tier |
|---|-------------|---------|---------|------|
| 1 | `heritage-register` | ~10,000 | Quarterly | T1 |
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
| 17 | `address-points-municipal-toronto-one-address-repository` | 525,435 | Weekly | Infra |
| 18 | `property-boundaries` | 498,499 | Daily | Infra |
| 19 | `ttc-routes-and-schedules` | All routes | 6 weeks | T3 |
| 20 | `ttc-ridership-subway-scarborough-rt-station-usage` | All stations | Annual | T3 |
| 21 | `motor-vehicle-collisions-involving-killed-or-seriously-injured-persons` | 20,519 | Daily | T3 |
| 22 | `noise-exemption-permits` | 3,685 | Weekly | T3 |
| 23 | `short-term-rental-program-data` | 768 | Monthly | T3 |
| 24 | `neighbourhood-profiles` | 2,383 | Census cycle | T3 |

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

### Must-have (V1 core — 10 parameters)
Parameters 1–10. These ARE Meridian V1. All data sources verified live.

### Should-have (V1.5 cherry-picks — pick 2–3)
- **#11 Building violations** — 30 min build, same CKAN pattern, strong red-flag
- **#2 Heritage Conservation Districts** — 30 min build, spatial join, complements #1
- **#15 Basement flooding areas** — 30 min build, complements TRCA flood data

### Nice-to-have (V2 — post-hackathon)
Parameters 12–20, 26–34. Require spatial joins, proxy models, or cross-source inference.

### Future (V3 — product)
Parameters 21–25, 35–38. Require external data, macro assumptions, or Flinks integration.
