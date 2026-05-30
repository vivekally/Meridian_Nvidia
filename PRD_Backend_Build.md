# Meridian — Backend Build PRD (FINAL)

**This is the single source of truth for the backend build.** It supersedes
`PRD_Engineering.md` §2–§10 wherever they conflict. The other docs
(`CEO_Plan_V1_Hackathon.md`, `Office_Hours_Gap_Analysis_and_Build_Strategy.md`,
`DATA_PARAMETERS.md`) remain valid as product/strategy context.

Produced by `/plan-eng-review` on 2026-05-30 after resolving the three
architecture-level contradictions and four build-blocking gaps found across the
planning docs. **Read the "Locked Decisions" and "Gaps Resolved" tables first —
they explain every place this doc overrides the older PRD.**

**Target:** Toronto true 10-year cost-of-ownership agent, NVIDIA Spark Hack
Toronto, 24-hour build, ASUS Ascent GX10 / NVIDIA GB10 Grace Blackwell, 128 GB
unified memory.

---

## 0. Locked Decisions

| # | Decision | Choice | Consequence for the build |
|---|----------|--------|---------------------------|
| D1 | Agent topology | **LLM planner + deterministic cost** | Agent 1 (LLM) decides the query plan and Agent 2 **honors SKIP**. Agent 2 + Agent 3 are pure Python. Agent 4 (LLM) synthesizes. **2 LLM calls per report.** |
| D2 | LLM provider | **Hybrid toggle** | Default = local NVIDIA model on the GB10 (`MERIDIAN_LLM_PROVIDER=local`). `=quality` switches generation to Claude. One `LLMClient`, two backends, **provider-agnostic prompts** (no `response_format` json-schema, no Anthropic tool-forcing). |
| D3 | Retrieval | **Cache-first + dual live pattern** | `DEMO_MODE=true` serves cached JSON. Live: GeoJSON point-in-polygon for polygon sources, bbox-SQL + haversine for tabular sources. Geocode via Address Points CKAN only. |
| D4 | Pipeline interface | **Generator + TraceEvent** | `run_pipeline()` yields `TraceEvent`s as agents run, returns a `Report`. CLI and Streamlit consume the same generator. |
| D5 | RAG | **Local, offline** | Build-time corpus ingest → local embeddings (NV-Embed via NIM, `bge-small` fallback) → local vector store. One `retriever.py` used by Agent 4 and the chat endpoint. Runs entirely on the GB10. |

---

## 1. Resolved Architecture

```
┌────────────────────────────────────────────────────────────────┐
│ USER INPUT: address · list_price · buyer_profile ·            │
│             down_payment_pct · rate · amortization             │
└───────────────┬────────────────────────────────────────────────┘
                │
┌───────────────▼────────────────────────────────────────────────┐
│ AGENT 1 — INTAKE + PLANNING                                     │
│   intake.py (deterministic): geocode via Address Points,       │
│     derive property-type HINT from address tokens              │
│   planning.py (LLM call #1): classify property type, emit      │
│     QueryPlan[] with per-source query/skip + reasoning_trace   │
│   yields TraceEvent(stage=planning)                            │
└───────────────┬────────────────────────────────────────────────┘
                │ QueryPlan (sources to query, sources to SKIP)
┌───────────────▼────────────────────────────────────────────────┐
│ AGENT 2 — RETRIEVAL (deterministic, async, NO LLM)             │
│   retrieval.py: asyncio.gather over ONLY the sources Agent 1   │
│     marked query=true. 5s timeout each. Confidence per source. │
│   DEMO_MODE=true → serve data/demo_cache/<slug>.json           │
│   yields TraceEvent(stage=retrieval) per source                │
└───────────────┬────────────────────────────────────────────────┘
                │ RetrievalResult (raw source data + confidence)
┌───────────────▼────────────────────────────────────────────────┐
│ AGENT 3 — ANALYSIS + COST (deterministic, NO LLM)              │
│   analysis.py + cost/*: LTT, property tax, mortgage scenarios, │
│     CMHC, the 10 V1 signals, 2 composite signals.              │
│     ALL DOLLAR MATH LIVES HERE. The LLM never computes money.  │
│   yields TraceEvent(stage=analysis)                            │
└───────────────┬────────────────────────────────────────────────┘
                │ AnalysisResult (Signals + cost_breakdown)
┌───────────────▼────────────────────────────────────────────────┐
│ AGENT 4 — SYNTHESIS (LLM call #2)                              │
│   synthesis.py: plain-English report, negotiation leverage per │
│     elevated flag, contradiction notes. Conditional RAG.       │
│     References numbers from Agent 3 — never invents them.      │
│   yields TraceEvent(stage=synthesis)                          │
└───────────────┬────────────────────────────────────────────────┘
                │ returns Report
┌───────────────▼────────────────────────────────────────────────┐
│ POST-REPORT: chat.py answer_question(report, question, history)│
│   RAG-grounded follow-up Q&A. Reuses retriever + LLMClient.    │
└────────────────────────────────────────────────────────────────┘
```

**Iron rule:** the LLM (Agent 1, Agent 4, chat) never produces a dollar figure.
Every number in the `Report` originates in deterministic `cost/*` code. Agent 4
formats and explains numbers it is handed. This is why a financial tool can trust
its own output.

**Agentic claim is real, not theatrical:** Agent 1's `QueryPlan` materially
changes what Agent 2 fetches (it honors `query=false`). If you ever stub Agent 1,
keep the contract: Agent 2 must still branch on the plan. A planner whose output
is ignored is the failure mode called out in the project learnings.

---

## 2. Project Structure

```
meridian/
  __init__.py
  config.py              # env vars, paths, feature flags
  models.py              # dataclasses (single source of every contract)
  wording.py             # banned-phrase substitution + validator
  pipeline.py            # run_pipeline() generator -> Report
  chat.py                # answer_question() RAG-grounded follow-up
  llm/
    client.py            # LLMClient: provider-agnostic complete()/complete_json()
    nim_backend.py       # OpenAI-compatible NIM (local default)
    claude_backend.py    # Anthropic (quality toggle)
    parsing.py           # robust JSON extraction (fence strip, brace match, retry)
  agents/
    intake.py            # deterministic geocode + property-type hint
    planning.py          # Agent 1 (LLM) -> QueryPlan
    retrieval.py         # Agent 2 (deterministic async) -> RetrievalResult
    analysis.py          # Agent 3 (deterministic) -> AnalysisResult
    synthesis.py         # Agent 4 (LLM) -> report fields
  sources/
    registry.py          # dataset slugs + RESOURCE_IDS (UUIDs) + ward constants
    ckan.py              # resource_id resolution, datastore_search_sql, bbox+haversine
    geo.py               # GeoJSON load + shapely point-in-polygon
    arcgis.py            # TRCA ArcGIS live + GeoJSON fallback
    geocode.py           # Address Points geocoding + autocomplete
  cost/
    ltt.py               # Ontario + Toronto MLTT brackets, FTB rebates
    property_tax.py      # assessed-value proxy + 10-year projection
    mortgage.py          # scenarios (base/bear/bull) + CMHC
    composites.py        # maintenance complexity, future tax pressure
  rag/
    ingest.py            # build-time: download corpus, chunk, embed, persist
    store.py             # vector store wrapper (Chroma persisted to disk)
    retriever.py         # retrieve(query, category, k) — shared interface
run_pipeline.py          # CLI: --address --list-price --profile --demo --provider
app.py                   # Streamlit (frontend, out of backend scope; consumes pipeline)
data/
  demo_cache/<slug>.json # 3 demo properties (Agent 2 output schema)
  geo/                   # heritage.geojson, hcd.geojson, trca_flood.geojson
  rag_corpus/            # ~11 .txt/.pdf legislation + program docs
  rag_index/             # persisted Chroma store
tests/
  test_ltt.py test_property_tax.py test_mortgage.py test_composites.py
  test_wording.py test_parsing.py test_geocode.py test_pipeline_demo.py
  fixtures/
requirements.txt
.env.example
```

**Module count is ~20 but cleanly layered.** `cost/*` and `sources/*` are small
single-responsibility files. This is the right size for a 4-agent pipeline; it is
not over-engineering.

---

## 3. Dependencies

```
# requirements.txt
httpx>=0.27            # async CKAN + ArcGIS + NIM HTTP
shapely>=2.0           # point-in-polygon (NO geopandas — avoids GDAL/Fiona install pain on fresh GB10)
openai>=1.40           # NIM is OpenAI-compatible (local default backend)
anthropic>=0.39        # Claude quality-toggle backend
chromadb>=0.5          # local persisted vector store (offline)
sentence-transformers>=3.0  # bge-small embedding fallback when NV-Embed NIM absent
pyproj>=3.6            # WGS84 -> UTM 17N (EPSG:26717) for dev-application distance
streamlit>=1.38        # frontend (separate workstream)
python-dotenv>=1.0     # .env loading
pytest>=8.0            # tests
pytest-asyncio>=0.23   # async retrieval tests
```

**Deliberate choice: `shapely` instead of `geopandas`.** Point-in-polygon only
needs `shapely.geometry.Point` + `shape(geojson)`. GeoPandas drags in GDAL/Fiona,
which is the classic "lost 3 hours to a build error" trap on fresh hardware.
Load GeoJSON with stdlib `json`, build shapely polygons, test containment.

---

## 4. Config & Environment

```python
# config.py — all via os.environ, defaults shown
DEMO_MODE            = bool   # default False; True serves data/demo_cache/
MERIDIAN_LLM_PROVIDER= str    # "local" (default) | "quality"
MERIDIAN_MODEL       = str    # default "meta/llama-3.1-8b-instruct"
NIM_BASE_URL         = str    # default "http://localhost:8000/v1"
NIM_API_KEY          = str    # default "not-needed" (local NIM)
ANTHROPIC_API_KEY    = str    # required only when provider=quality
MERIDIAN_CLAUDE_MODEL= str    # default "claude-sonnet-4-6"
EMBED_PROVIDER       = str    # "nim" (default) | "sentence-transformers"
EMBED_MODEL          = str    # default "nvidia/nv-embedqa-e5-v5"
MERIDIAN_RAG         = str    # "off" (default) | "on" — RAG is flagged, see §12
CKAN_BASE            = "https://ckan0.cf.opendata.inter.prod-toronto.ca/api/3"
SOURCE_TIMEOUT_S     = 5
```

`.env.example` ships with the local defaults so a fresh clone runs against NIM
with zero edits. `provider=quality` is the only path that needs a secret.

---

## 5. Data Contracts (single source: `models.py`)

All inter-agent data is typed dataclasses. **This section overrides the divergent
schemas in PRD_Engineering §9 and Office-Hours §"Inter-Agent Data Contracts".**

```python
from dataclasses import dataclass, field
from typing import Literal, Optional

Confidence = Literal["High", "Medium", "Low", "Unknown"]   # NOTE: Unknown added
SignalType = Literal["observed", "inferred", "simulated"]

@dataclass
class Signal:
    signal_name: str
    signal_type: SignalType
    value: str
    confidence: Confidence
    source: str
    data_coverage: str
    message: str                      # plain-language, wording-rule compliant

@dataclass
class CompositeSignal:
    signal_name: str
    signal_type: SignalType            # always "inferred"
    value: str                         # "Low" | "Medium" | "Elevated"/"High"
    confidence: Confidence             # ALWAYS "Low" — never upgrade
    factors: list[str]
    disclaimer: str
    message: str = ""

@dataclass
class QueryPlanItem:
    source: str                        # key in SOURCE_REGISTRY
    query: bool                        # False == SKIP (Agent 2 MUST honor)
    reason: str

@dataclass
class IntakeResult:
    address: str
    lat: float
    lon: float
    ward: Optional[str]
    property_type_hint: str            # deterministic token guess (condo/detached/...)
    geocode_confidence: Confidence

@dataclass
class QueryPlan:
    property_type_inferred: str
    items: list[QueryPlanItem]
    reasoning_trace: str

@dataclass
class SourceResult:
    source: str
    found: bool
    data: dict
    confidence: Confidence
    data_coverage: str
    skipped: bool = False
    error: Optional[str] = None

@dataclass
class RetrievalResult:
    results: dict[str, SourceResult]
    failures: list[str]
    total_latency_ms: int

@dataclass
class CostBreakdown:
    ltt_ontario: int
    ltt_toronto: int
    ltt_combined: int
    ftb_rebate: int                    # negative or 0
    ltt_net: int
    assessed_value_low: int
    assessed_value_mid: int
    assessed_value_high: int
    annual_property_tax: int
    ten_year_property_tax: int
    mortgage_principal: int
    cmhc_premium: int
    cmhc_pst: int
    mortgage_10yr_base: int
    mortgage_10yr_bear: int
    mortgage_10yr_bull: int
    transit_dividend_10yr: int         # V1: hardcoded per demo property, else 0
    true_10yr_cost: int                # sum of the cost components below

@dataclass
class NegotiationLeverage:
    flag: str
    dollar_low: int
    dollar_high: int
    script: str

@dataclass
class AnalysisResult:
    signals: list[Signal]
    composite_signals: list[CompositeSignal]
    cost_breakdown: CostBreakdown
    buyer_flags: list[str]
    overall_risk: Literal["low", "medium", "high"]

@dataclass
class TraceEvent:
    stage: Literal["intake","planning","retrieval","analysis","synthesis"]
    status: Literal["start","progress","done","error"]
    message: str
    confidence: Optional[Confidence] = None
    data: dict = field(default_factory=dict)

@dataclass
class Verdict:
    level: Literal["GREEN","YELLOW","RED"]
    headline: str
    true_10yr_cost: int
    leverage_low: int
    leverage_high: int

@dataclass
class Report:
    address: str
    list_price: int
    buyer_profile: str
    verdict: Verdict
    signals: list[Signal]
    composite_signals: list[CompositeSignal]
    cost_breakdown: CostBreakdown
    negotiation_leverage: list[NegotiationLeverage]
    buyer_flags: list[str]
    reasoning_trace: list[TraceEvent]
    metadata: dict                     # as_of, disclaimer, latency_ms,
                                       # llm_provider, sources_queried, sources_failed
```

`Report` is the **backend → frontend contract**. The Streamlit app renders
exactly these fields. This is the schema PRD_Engineering §9.3 never specified.

---

## 6. Pipeline Interface (`pipeline.py`)

```python
from typing import Iterator, Union

def run_pipeline(
    address: str,
    list_price: int,
    buyer_profile: str = "first-time",
    down_payment_pct: float = 20.0,
    rate: float = 4.79,
    amortization: int = 25,
    *,
    demo_mode: bool | None = None,     # None -> read DEMO_MODE env
) -> Iterator[Union[TraceEvent, Report]]:
    """
    Generator. Yields TraceEvent objects as each agent progresses, then yields
    the final Report as the last item. Both the CLI and Streamlit consume this
    one function:

      *events, report = list(run_pipeline(...))   # simple collect
      # or stream:
      for item in run_pipeline(...):
          if isinstance(item, Report): final = item
          else: render_trace(item)
    """
```

This is what makes the reasoning trace real-time in Streamlit (`st.status()`
updates on each yielded `TraceEvent`) without the backend and UI duplicating
logic. The CLI prints each event then pretty-prints the `Report`.

**Degradation:** any agent that fails yields `TraceEvent(status="error")` and the
pipeline continues with reduced data rather than raising. Only an Agent-4 hard
failure (empty after one retry) produces a minimal `Report` with the error
surfaced in `metadata`.

---

## 7. Agent 1 — Intake + Planning

### 7.1 Intake (deterministic, `agents/intake.py`)

1. **Geocode** via Address Points CKAN (`sources/geocode.py`). Returns lat/lon +
   `geocode_confidence`. **Nominatim is NOT used** (public-endpoint policy bans
   demo use; this also removes a dependency).
2. **Property-type hint** from address tokens (deterministic, cheap):
   - contains `unit`/`suite`/`#`/`ph` → `condo`
   - contains `townhouse`/`twnhse` → `condo_townhouse`
   - else `detached_urban` (safe default; Agent 1 LLM refines)
3. **Ward** is best-effort from Address Points; may be `None` (not used for math
   after dropping ward normalization).

If geocoding fails → `geocode_confidence="Unknown"`, lat/lon `None`. Pipeline
yields an error trace and produces a degraded report ("Could not resolve this
address; spatial checks skipped").

### 7.2 Planning (LLM call #1, `agents/planning.py`)

Agent 1 classifies property type and emits a `QueryPlan`. **Its output changes
Agent 2's behavior** — Agent 2 skips every source with `query=false`.

Property types: `detached_urban`, `detached_suburban`, `semi_detached`,
`condo`, `condo_townhouse`, `commercial_mixed`.

Source decision rules baked into the system prompt:
- `heritage_register`, `heritage_conservation_districts`: query if pre-2000 era
  likely or address in a known heritage area; else low priority but still query
  (cheap polygon check).
- `building_permits_active`, `building_permits_cleared`: always query.
- `development_applications`: always query (standard 500 m radius).
- `trca_flood`: always query (polygon check is cheap).
- `rentsafeto`: query **only** if `property_type in {condo, commercial_mixed}` or
  high-rise residential. SKIP for detached/semi/townhouse.

```
SYSTEM: You are a property-risk planning agent for Toronto real estate. Given an
address and a deterministic property-type hint, decide the property type and
which data sources to query. Output ONLY valid JSON matching this schema:
{"property_type_inferred": "...",
 "items": [{"source": "<key>", "query": true|false, "reason": "..."}],
 "reasoning_trace": "2-3 sentences explaining your plan"}
Available sources: heritage_register, heritage_conservation_districts,
building_permits_active, building_permits_cleared, development_applications,
trca_flood, rentsafeto.
Rules: <the rules above>. Be specific about WHY you skip a source.

USER: Address: {address}. Deterministic hint: {property_type_hint}.
Coordinates: {lat},{lon}.
```

Output parsed by `llm/parsing.py` (fence-strip + brace-match + one retry). On
parse failure twice → fall back to a default plan (query all except RentSafeTO)
and keep the reasoning_trace as `"Planner output unparseable; queried all core
sources as a safe default."` (degraded but honest, never crashes).

---

## 8. Agent 2 — Retrieval (deterministic, async, `agents/retrieval.py`)

`asyncio.gather` over the sources where `query=true`. 5 s timeout each. Each
returns a `SourceResult` with confidence. **Mixing async HTTP with sync shapely:**
run point-in-polygon via `asyncio.to_thread(...)` so it never blocks the loop.

### 8.1 Confidence rules (apply to every source)

| Confidence | Condition |
|------------|-----------|
| High | HTTP 200 + recency field (`last_updated`/`extract_date`/`APPLICATION_DATE`) present |
| Medium | HTTP 200, no recency metadata |
| Low | HTTP 200, empty result set (0 records / no polygon hit) |
| Unknown | timeout (>5 s), 4xx, 5xx, or local file load failure |

### 8.2 Retrieval mechanics by source type — **THE gap the old PRD left open**

**Polygon sources (heritage, HCD, TRCA flood): local GeoJSON + shapely PIP.**
Download once to `data/geo/` (build-time, see §15). No live CKAN call; these
refresh quarterly. `sources/geo.py::point_in_polygon(lat, lon, geojson_path)`
returns the matched feature properties or `None`. TRCA also has a live ArcGIS
path (`sources/arcgis.py`) with the GeoJSON as fallback (per the API call in §9.5).

**Tabular sources (permits, dev apps, RentSafeTO): resource_id + bbox-SQL + haversine.**
CKAN `datastore_search` needs a `resource_id` UUID, **not** the dataset slug.
1. Resolve UUIDs once at build time: `package_show?id=<slug>` → pick the
   datastore-active CSV resource → store in `sources/registry.py::RESOURCE_IDS`.
   (Hardcoding them avoids a runtime round-trip; re-resolve if a 404 appears.)
2. Spatial filter: CKAN has **no native radius query**. Use
   `datastore_search_sql` with a lat/lon bounding box prefilter:
   ```sql
   SELECT * FROM "<resource_id>"
   WHERE "<lat_col>" BETWEEN {lat-0.0045} AND {lat+0.0045}
     AND "<lon_col>" BETWEEN {lon-0.0060} AND {lon+0.0060}
   ```
   (~500 m box at Toronto's latitude; lon delta wider because cos(43.7°)≈0.72.)
3. In Python, exact-filter the small candidate set by haversine ≤ 500 m
   (`sources/ckan.py::within_radius`).

**Address-level match (permits) by point proximity, NOT string match.** Match a
permit to the subject property when the permit's geocoded point is within ~30 m of
the subject point. This kills the silent "no permits found" bug from messy
address strings (ranges, units, directionals). If a permit row lacks coordinates,
fall back to a normalized address `q=` full-text match.

### 8.3 DEMO_MODE

`DEMO_MODE=true` → Agent 2 returns `data/demo_cache/<slug>.json` (a serialized
`RetrievalResult`) and skips all network calls. `<slug>` = `slugify(address)`
defined in §15.1. **Agent 1 and Agent 4 still run live** — the demo's star is the
LLM reasoning, and the cache only insulates you from CKAN/ArcGIS flakiness, not
from NIM. (If you also want NIM-independence, cache the full `Report`; out of V1
scope.)

---

## 9. Agent 3 — Analysis + Cost (deterministic, `agents/analysis.py` + `cost/*`)

**All dollar math. No LLM. Fully unit-tested (§13).** Constants below are the
canonical V1 values; they override any conflicting copy elsewhere.

### 9.1 Land Transfer Tax (`cost/ltt.py`) — `observed`, High

Marginal-bracket calculation. **Toronto MLTT uses the full high-value schedule
(correct above $2M), fixing the PRD §5.1 "mirror Ontario" bug.**

```python
ONTARIO_LTT = [(55_000,0.005),(250_000,0.010),(400_000,0.015),
               (2_000_000,0.020),(float("inf"),0.025)]
TORONTO_MLTT = [(55_000,0.005),(250_000,0.010),(400_000,0.015),
                (2_000_000,0.020),(3_000_000,0.025),(4_000_000,0.035),
                (5_000_000,0.045),(float("inf"),0.055)]
ONTARIO_FTB_REBATE = 4_000
TORONTO_FTB_REBATE = 4_475     # combined cap 8_475, first-time only

def marginal_ltt(price, brackets) -> int: ...   # sum marginal rate * band width
```

Apply both rebates only when `buyer_profile == "first-time"`. `ltt_net =
max(0, ltt_combined - rebate)`.

**Test oracles (verify on ratehub.ca/land-transfer-tax-ontario):**
| Price | Profile | ltt_combined | ltt_net |
|-------|---------|--------------|---------|
| $500,000 | not-first-time | 12,950 | 12,950 |
| $750,000 | first-time | 22,950 | 14,475 |
| $1,200,000 | first-time | 40,950 | 32,475 |

### 9.2 Property Tax (`cost/property_tax.py`) — `simulated`, High rate / Medium AV

```python
RATE_RESIDENTIAL = 0.00767311
RATE_MULTI_RES   = 0.01208792          # buyer_profile == "investor"
ANNUAL_GROWTH    = 0.035
MULTIPLIERS = {"condo":0.90,"condo_townhouse":0.80,"semi_detached":0.65,
               "detached_urban":0.70,"detached_suburban":0.55,
               "commercial_mixed":0.70}
rate  = RATE_MULTI_RES if profile=="investor" else RATE_RESIDENTIAL
av_mid = list_price * MULTIPLIERS.get(property_type, 0.70)
av_low, av_high = av_mid*0.85, av_mid*1.15
annual = av_mid * rate
ten_year = annual * ((1+ANNUAL_GROWTH)**10 - 1) / ANNUAL_GROWTH   # 10-term sum
```
Always attach the MPAC-frozen-at-2016 disclaimer (§11) to the signal `message`.

### 9.3 Mortgage (`cost/mortgage.py`) — `simulated`, High math / Medium scenarios

```python
# Minimum down payment (validate, warn if below):
#   5% on first 500k; 10% on 500k-1.5M; 20% (no CMHC) over 1.5M
# principal = list_price - down_payment_amount
# CMHC premium added to principal when down<20% AND price<=1.5M
CMHC = [(0.65,0.0060),(0.75,0.0170),(0.80,0.0240),(0.85,0.0280),
        (0.90,0.0310),(0.95,0.0400)]            # by LTV; non-traditional 0.0450
# +0.20% premium surcharge if amortization > 25
# Ontario PST 8% on premium, paid at CLOSING (not financed) -> cmhc_pst
```

Scenario model: 5-year term, **two 5-year blocks** inside the 10-year window.
Block 1 at `rate`; block 2 at `rate + delta`, recomputed on the remaining balance
with remaining amortization.

| Scenario | delta on block 2 |
|----------|------------------|
| base | 0.0 |
| bear | +1.5 |
| bull | -0.5 |

`mortgage_10yr_<scenario>` = total of monthly payments over 120 months (principal
+ interest paid). Standard payment `M = P*r/(1-(1+r)**-n)`, `r=annual/12/100`,
`n=amort*12`.

### 9.4 The 10 V1 Signals (logic, all in `analysis.py` over Agent 2 data)

| # | Signal | Type | Logic → message |
|---|--------|------|-----------------|
| 1 | Heritage | observed | Part IV / Part V / Listed / none → tiered message (§ below) |
| 2 | HCD | observed | point-in-polygon hit → "in a heritage conservation district" |
| 3 | Active permits | observed | structural-keyword filter; ≥1 active structural → flag |
| 4 | Cleared permits | observed | structural + within 10 yr; ≥3 → "further due diligence" |
| 5 | Dev intensification | inferred | active OZ/SA within 500 m: 0-4 none, 5-9 Medium, 10+ High |
| 6 | Flood | observed | TRCA intersect → Elevated; within 100 m → Moderate; else none |
| 7 | Building health | observed | Path A RentSafeTO score; Path B structural permit history |
| 8 | LTT | observed | §9.1 |
| 9 | Property tax | simulated | §9.2 |
| 10 | Mortgage | simulated | §9.3 + sub-signals 10a-10e (first-time flags) |

**Structural permit filter (shared by #3, #4, #7B, composites):**
```python
STRUCTURAL_KEYWORDS = ["structural","unsafe","major repair","emergency"]
# NEVER flag routine/cosmetic/interior permits
```

**Dev intensification filter (#5):** keep only
`status in {"Under Review","Application Received","Council Approved","NOAC Issued"}`
and `type in {"OZ","SA"}`; exclude `Closed`. **No ward normalization** — raw
500 m count drives the tiers (the radius already controls for area).

### 9.5 Composite signals (`cost/composites.py`) — `inferred`, **always Low**

Maintenance Complexity and Future Tax Pressure exactly per PRD §5.8–5.9 /
DATA_PARAMETERS #36–37. `confidence` is hardcoded `"Low"` and the disclaimer is
mandatory. Maintenance: ≥3 factors → "Elevated", ≥1 → "Medium". Tax pressure:
dev_density ≥10 → "High", ≥5 → "Medium". Wording: never "special assessment
risk"; use "Elevated maintenance complexity signal".

### 9.6 Transit dividend (V1)

`transit_dividend_10yr`: hardcoded per demo property in the cache (the $87K figure
already in the prototype). For live non-demo addresses → `0` and omit from the
report. Real GTFS distance scoring is V1.5 (TODOS).

### 9.7 Verdict derivation (deterministic)

```python
# overall_risk from elevated signals; verdict.level mirrors it
RED    if any elevated observed flag (heritage Part IV, active structural, flood Elevated)
YELLOW if any Medium signal or composite Elevated/High
GREEN  otherwise → headline "No concerns identified in available city data."
leverage_low/high = sum of negotiation dollar ranges across elevated flags
true_10yr_cost = ltt_net + ten_year_property_tax + mortgage_10yr_base
                 + (utilities/maintenance reserves are V2; not in V1 sum)
```

---

## 10. Agent 4 — Synthesis (LLM call #2, `agents/synthesis.py`)

Receives the full `AnalysisResult` JSON. Produces: one-paragraph verdict,
hidden-cost narrative, per-flag explanation, **negotiation leverage per elevated
flag (dollar range from `cost_breakdown`, never a point estimate)**, and a
3-scenario mortgage comparison. It formats numbers it is given; it must not
compute or alter them.

```
SYSTEM: You are a property report synthesis agent for a Toronto home buyer.
Use ONLY the provided analysis JSON. Rules:
- Every claim references a specific finding in the input.
- Use the exact dollar figures provided. Never invent or recompute numbers.
- If a confidence is Medium or Low, say so.
- For each elevated flag, give ONE sentence of negotiation leverage with a
  dollar RANGE drawn from cost_breakdown.
- Never use "red flag" — say "elevated review recommended".
- If a category has no data, say "No concerns identified" for it. Never speculate.
- Tone: direct, clear, protective of the buyer.
Output JSON: {"verdict_paragraph","flag_explanations":[...],
"negotiation_leverage":[{"flag","dollar_low","dollar_high","script"}]}

USER: Analysis: {analysis_json}. Property: {address} at ${list_price}.
Buyer profile: {buyer_profile}. RAG context: {retrieved_chunks_or_empty}.
```

**Conditional RAG (§12):** before the call, for each active flag retrieve from the
matching corpus category and inject as `RAG context`. Triggers per PRD §8.3:
heritage→heritage docs, flood→flood docs, first-time→FHSA/HBP docs, maintenance
Elevated→assessment docs, tax-pressure High→zoning docs.

**Wording enforcement:** run `wording.apply()` over every output string before it
enters the `Report`. `wording.validate()` (used in tests) greps for banned phrases.

**Parse + retry:** `parsing.py` strips ```json fences, brace-matches, retries once;
on second failure, fall back to a templated report built from `AnalysisResult`
(deterministic, ugly-but-correct) so the demo never shows a crash.

---

## 11. Wording Rules (`wording.py`)

Substitution map applied to all agent/UI strings; `validate()` asserts none of the
LHS phrases survive (a test fails the build if they do).

| Find | Replace |
|------|---------|
| "red flag" / "hard red flag" / "hard flag" | "elevated review recommended" / "elevated signal" |
| "high special assessment risk" | "elevated maintenance complexity signal" |
| "potential tax increases" | "potential neighbourhood change" |
| "major issues" | "further due diligence recommended" |
| "$3,500" (flood) / "insurance loading" | (removed) |

Mandatory disclaimers (attach to the relevant signal `message`):
- Property tax: MPAC-frozen-at-2016 proxy disclaimer.
- RentSafeTO: "No score available does not indicate lower risk — condos, co-ops,
  and smaller buildings are not covered."
- Composites: "inferred signal, not evidence of a historical special assessment."
- Report metadata `disclaimer`: not financial/legal/insurance advice.

---

## 12. RAG Subsystem (`rag/`, local + offline)

**Build-time ingest (`rag/ingest.py`, run before hackathon / once):**
1. Download the ~11 corpus docs (PRD §8.3 table) to `data/rag_corpus/`.
2. Chunk (~800 tokens, 100 overlap), embed via `EMBED_PROVIDER`:
   - `nim`: NV-Embed (`nvidia/nv-embedqa-e5-v5`) at `NIM_BASE_URL` (on GB10).
   - `sentence-transformers`: `BAAI/bge-small-en-v1.5` fallback (also local).
3. Persist to Chroma at `data/rag_index/` (on-disk, offline).

**Retriever (`rag/retriever.py`):** `retrieve(query, category, k=4) -> list[str]`.
Category filters to a corpus subdir (heritage/flood/programs/assessment/zoning).
Used by **both** Agent 4 (report) and `chat.py` (follow-up Q&A).

**Embeddings stay on the GB10 regardless of D2** (Anthropic serves no embeddings),
which is why the privacy/hardware story holds even in `quality` mode.

---

## 13. Chatbot Endpoint (`chat.py`)

```python
def answer_question(report: Report, question: str,
                    history: list[dict]) -> Iterator[str]:
    """RAG-grounded follow-up. Retrieves corpus chunks relevant to `question`,
    grounds in the existing Report + chunks, streams the answer via LLMClient.
    Same provider toggle. Refuses to invent numbers not in the Report."""
```

System prompt: "Answer ONLY from the provided report and retrieved context. If
the report does not contain the answer, say so and suggest professional
verification. Never invent dollar figures."

---

## 14. Error Handling

| Error | Handler | User-visible behavior |
|-------|---------|----------------------|
| CKAN `JSONDecodeError` | log; source confidence=Unknown | trace: "Heritage data unavailable this session" |
| TRCA GeoJSON `FileNotFoundError` | log; flood confidence=Unknown | trace notes flood unverified |
| Agent 1 parse fail ×2 | default plan (query all but RentSafeTO) | honest reasoning_trace, pipeline continues |
| Agent 4 empty / parse fail ×2 | templated report from AnalysisResult | correct numbers, plainer prose |
| `httpx.ConnectError` on NIM | surface immediately | "Local model not responding. Start the NIM container." |
| `httpx.TimeoutException` on source | confidence=Unknown, continue gather | other sources unaffected |
| Geocode miss | confidence=Unknown, lat/lon None | degraded report, spatial checks skipped |

**Critical-gap check:** every failure path above is either tested, surfaced in the
trace, or both. None fail silently. The one to watch: a permit point-proximity
miss that returns "no permits" when permits exist but lack coordinates — mitigated
by the address `q=` fallback (§8.2) and called out as a known V1 limitation.

---

## 15. Demo Data & Build-Time Assets

### 15.1 Slugify (shared by cache write + read)

```python
import re
def slugify(address: str) -> str:
    return re.sub(r"[^a-z0-9]+","-", address.lower()).strip("-")
# "401 Richmond St W" -> "401-richmond-st-w"
```

### 15.2 Three demo properties (different risk profiles)

| Property | Profile | Expected flags |
|----------|---------|----------------|
| 401 Richmond St W | RED | Heritage Part IV, active structural permits, high dev density |
| (Scarborough semi, TBD) | GREEN | clean — exercises "No concerns identified" |
| (Eglinton condo, TBD) | YELLOW | near flood, moderate dev pressure, transit-aligned |

Each cached as `data/demo_cache/<slug>.json` (serialized `RetrievalResult`) by
**Hour 8 gate**. Pick the two TBD addresses early.

### 15.3 Pre-hack asset downloads (before clock starts)

- `data/geo/heritage.geojson`, `hcd.geojson`, `trca_flood.geojson`
- `data/rag_corpus/*` + build `data/rag_index/`
- `RESOURCE_IDS` resolved into `sources/registry.py`

---

## 16. Test Plan (`tests/`)

Coverage target: **100% on deterministic cost math** (highest ROI, zero LLM
flakiness). LLM agents get contract/shape tests + one demo end-to-end.

```
DETERMINISTIC (must be ★★★ — behavior + edges):
  test_ltt.py          → 3 oracle rows (§9.1) + rebate boundary + >$2M brackets + $0 floor
  test_property_tax.py → each multiplier, investor rate, AV range, 10yr growth sum
  test_mortgage.py     → payment formula vs known value, CMHC each LTV band,
                         amort>25 surcharge, base/bear/bull ordering (bear>base>bull),
                         down<20% insured vs >=20% uninsured, >$1.5M no-CMHC path
  test_composites.py   → factor thresholds, confidence ALWAYS "Low", disclaimer present
  test_wording.py      → validate() catches every banned phrase; substitution idempotent
  test_parsing.py      → fenced JSON, trailing prose, double-brace, retry path

CONTRACT / INTEGRATION:
  test_geocode.py      → Address Points hit returns lat/lon; miss → Unknown
  test_pipeline_demo.py→ run_pipeline(DEMO_MODE) for all 3 properties:
                         yields TraceEvents then a Report; verdict levels differ
                         (RED/GREEN/YELLOW); no banned phrases in any string;
                         Agent 2 honored at least one SKIP for the detached property

E2E / EVAL (manual, demo day):
  [→EVAL] Agent 4 prompt change → re-check 3 reports reference real numbers, non-generic
  [→E2E]  one live (non-cached) address completes < 35 s
```

**Regression guard:** `test_pipeline_demo.py` is the safety net — if any refactor
breaks the demo path or reintroduces a banned phrase, it fails before you demo.

`pytest -q` is the single command. Wire it in CLAUDE.md as the project test cmd.

---

## 17. Revised 24-Hour Build Sequence

| Hours | Task | Gate |
|-------|------|------|
| 0–1 | NIM latency benchmark (local model, 300 tok). **>15 s → keep 2 LLM calls but cap Agent 4 max_tokens=800; consider Nemotron only if memory + latency allow.** Wire `LLMClient` both backends. | BLOCKING |
| 1–2 | `models.py`, `config.py`, `cost/*` + their unit tests (oracle-driven) | `pytest cost/` green |
| 2–4 | `sources/*`: geocode, CKAN resource_id + bbox/haversine, geo PIP, ArcGIS+fallback | each source tested solo |
| 4–5 | `agents/retrieval.py` (async gather, confidence, DEMO_MODE) | live single-source fetch works |
| 5–6 | `agents/intake.py` + `agents/planning.py` (Agent 1 LLM, plan honored) | plan SKIP changes Agent 2 |
| 6–7 | `agents/analysis.py` (signals + composites + verdict) | 3 demo profiles differ |
| 7–8 | Pre-cache 3 demo properties; `slugify` | **3 caches in data/demo_cache/** |
| 8–10 | `rag/ingest.py` + `retriever.py` (offline index) | retrieve returns chunks |
| 10–12 | `agents/synthesis.py` (Agent 4 + conditional RAG + wording) | report references real numbers |
| 12–13 | `pipeline.py` generator + `run_pipeline.py` CLI | `python run_pipeline.py --demo` passes E2E |
| 13–14 | `chat.py` follow-up endpoint | grounded answer, no invented numbers |
| 14–20 | Streamlit (separate workstream — consumes `run_pipeline` + `Report`) | trace streams live |
| 20–21 | Error rescues wired + `test_pipeline_demo` green | all 3 demos clean |
| 21–24 | Rehearse, 3 random live addresses, time pitch < 4 min | demo < 35 s, pitch < 4 min |

**Cut list (unchanged):** RentSafeTO → live non-demo queries → merge Agent 3 logic
into Agent 4 prompt (last resort; loses auditability) → confidence badges. **Never
cut the reasoning trace.**

---

## 18. Parallelization (for worktrees / multiple builders)

| Lane | Modules | Depends on |
|------|---------|------------|
| A | `cost/*` + tests | models.py |
| B | `sources/*` + tests | models.py |
| C | `rag/*` | models.py |
| D | `agents/*` + `pipeline.py` | A, B (and C for Agent 4) |
| E | `app.py` Streamlit | D (or the `Report` schema alone) |

**Launch A + B + C in parallel after `models.py` lands.** Merge, then D. E can
start against the `Report` schema (§5) before D finishes, using a stub pipeline.
Conflict risk: A and B both import `models.py` only (read-only) — no overlap.

---

## 19. Gaps Resolved (audit trail)

| Finding | Old state | Resolution in this PRD |
|---------|-----------|------------------------|
| Agent topology contradiction | PRD: Agents 1/3 deterministic; CEO/OH: all LLM | §1 D1: Agent 1 LLM planner, Agent 3 deterministic, Agent 4 LLM |
| Model contradiction | "Mistral Medium 3.5" (not local) vs Llama 3.1 8B | §4 D2 hybrid: local Llama default, Claude toggle, one client |
| Signal schema fork | `observed/inferred` vs `severity:red/green` | §5: one `Signal` dataclass; severity-colors dropped (also violated wording rules) |
| Confidence enum | dataclass had 3 values, rules needed Unknown | §5: `Confidence` includes `"Unknown"` |
| CKAN spatial unspecified | slugs given, no resource_id, no radius mechanic | §8.2: resource_id resolution + bbox-SQL + haversine |
| Heritage = shapefile vs query | two mechanics | §8.2: GeoJSON + shapely PIP for all polygon sources |
| Address match fragile | exact string → false "no permits" | §8.2: 30 m point proximity, `q=` fallback |
| Geocoding contradiction | accepted-scope vs build-schedule disagree | §7.1: Address Points only, Nominatim removed |
| Ward normalization | tiers vs density mismatch, no ward source | §9.4: dropped; raw 500 m count tiers |
| LTT >$2M wrong | PRD mirrored Ontario | §9.1: full Toronto MLTT 8-bracket schedule |
| §9.3 example math wrong | numbers didn't match brackets | §9.1: real oracle table replaces the bad example |
| Report schema missing | only Agent 3→4 specified | §5: full `Report` contract |
| Trace not streamable | batch CLI vs live UI | §6: generator yielding `TraceEvent` |
| Transit dividend tier | V1.5 field in V1 contract | §9.6: hardcoded V1, real GTFS deferred |
| NIM endpoint shape | unspecified | §4: OpenAI-compatible `localhost:8000/v1` |
| RAG underspecified | conditional retrieval, no ingest/store | §12: offline ingest + Chroma + shared retriever |
| Missing referenced files | spec/PDF/geojson/cache absent | §15.3 pre-hack downloads; this PRD is now the spec |

---

## 20. NOT in Scope (V1)

- Streamlit frontend internals (separate workstream; consumes §5 `Report`).
- Live GTFS transit scoring (V1.5 — hardcoded dividend for demo).
- V1.5 sources: Heritage Formerly Listed, Residential Fire Inspections, Building
  Violations, Basement Flooding, Property Tax Relief programs.
- V2: zoning spatial joins, Committee of Adjustment, crime, VHT, utilities,
  insurance modeling, RAPIDS cuDF GPU spatial.
- Full `Report` caching for NIM-independence (only `RetrievalResult` cached).
- Multi-city, auth, persistence, streaming token output in the report path.

---

## 21. Known V1 Limitations (state honestly if asked)

- Property-type inference is an LLM guess from an address; the assessed-value
  multiplier it selects swings the tax estimate. Output is a ±15% range labelled
  Medium confidence — by design, not precision.
- Permit point-proximity can miss permits lacking coordinates (mitigated by `q=`
  fallback). A "no permits found" is "none found in available data", not "none".
- Composite signals are inferences over public data, never condo-corporation
  facts. Always Low confidence, always disclaimed.
```
