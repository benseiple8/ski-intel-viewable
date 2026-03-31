# Snow Intel – Data Architecture & Modeling Pipeline

This repository documents the end-to-end data architecture behind **Snow Intel**:  
how raw weather, snow report, terrain, and demand data flow through normalization,  
terrain-bucket simulation, ranking, pattern extraction, and ultimately into skier-facing summaries.

The goal of this system is not to predict weather in isolation, but to translate  
weather uncertainty into **skiable conditions and constraints** that reflect terrain, exposure, durability, crowding, and real-world resort behavior.

At its core, Snow Intel is a **bucket-first inference system**.

It does not begin with mountain-wide summaries or aspect-level narratives.  
Instead, it simulates the full terrain lattice, ranks every terrain bucket, identifies  
patterns across the best and worst buckets, converts those patterns into explicit  
evidence-backed claims, and only then renders skier-facing language.

---

## High-level system flow

```mermaid
flowchart TD

subgraph S["Raw sources"]
  SR["Resort snow reports"]
  NWS["NWS forecast grids"]
  ECMF["ECMWF forecast"]
  GFS["GFS forecast"]
  ICON["ICON forecast"]
  HRRR["HRRR forecast + analysis"]
  ERA["ECMWF analysis / reanalysis"]
  STATIC["Static resort terrain features"]
  DEMAND["Demand & traffic inputs"]
end

subgraph R["Raw landing zone (append-only)"]
  RSR["raw_snow_reports"]
  RFX["raw_forecasts_by_model"]
  RAN["raw_analysis_retro"]
  RST["raw_static_features"]
  RDM["raw_demand_inputs"]
end

SR --> RSR
NWS --> RFX
ECMF --> RFX
GFS --> RFX
ICON --> RFX
HRRR --> RFX
ERA --> RAN
STATIC --> RST
DEMAND --> RDM

subgraph N["Normalization & canonical schemas"]
  NSR["Normalize snow reports"]
  NFX["Normalize forecasts"]
  NAN["Normalize analysis"]
  NRS["Normalize resort dimension / terrain profile"]
  NDM["Normalize demand inputs"]
end

RSR --> NSR --> SRN["snow_reports_norm"]
RFX --> NFX --> FXN["forecasts_norm"]
RAN --> NAN --> ANN["analysis_norm"]
RST --> NRS --> RSN["resorts_dim"]
RDM --> NDM --> DMN["demand_norm"]

subgraph A["Alignment & ski-relevant derivation"]
  ALIGN["Time, timezone, and elevation alignment"]
  BLEND["Forecast blending and bias correction"]
  DERIVE["Derived ski-relevant drivers"]
  JOIN["Join weather, snow report, static, and demand inputs"]
end

SRN --> ALIGN
FXN --> ALIGN
ANN --> ALIGN

ALIGN --> BLEND --> DERIVE --> JOIN
RSN --> JOIN
DMN --> JOIN

JOIN --> INPUTS["bucket_model_inputs"]

subgraph L1["Layer 1 – Terrain state simulation"]
  SIM["Simulate full terrain lattice at bucket grain"]
  TERR["terrain_state"]
end

INPUTS --> SIM --> TERR

subgraph L2["Layer 2 – Bucket ranking"]
  RANK["Rank every terrain bucket by quality and usability"]
  BR["bucket_ranking"]
end

TERR --> RANK --> BR

subgraph L3["Layer 3 – Pattern extraction"]
  PAT["Detect dominant / exception / deterioration patterns"]
  PE["pattern_evidence"]
end

BR --> PAT --> PE

subgraph L4["Layer 4 – Narrative evidence"]
  CLAIMS["Convert ranked buckets + patterns into evidence-backed claims"]
  NE["narrative_evidence"]
end

BR --> CLAIMS
PE --> CLAIMS
CLAIMS --> NE

subgraph L5["Layer 5 – Rendering"]
  RENDER["LLM or template rendering from evidence only"]
  OUT["public_summary_payload"]
end

NE --> RENDER --> OUT

subgraph Q["QA, backtesting, feedback"]
  DQ["Data quality checks"]
  BACKTEST["Backtesting"]
  FEEDBACK["User feedback"]
  RETUNE["Retuning"]
end

RSR --> DQ
RFX --> DQ
RAN --> DQ
TERR --> BACKTEST
OUT --> FEEDBACK
BACKTEST --> RETUNE
FEEDBACK --> RETUNE
RETUNE --> BLEND
RETUNE --> SIM
RETUNE --> RANK
RETUNE --> PAT
```

---

## Core modeling principle

Snow Intel is built around a **full terrain lattice**, not a resort-wide average.

### Canonical grain

- `resort_id`
- `timestamp_local`
- `terrain_type`
- `aspect`
- `elevation_band`

### Key rules

- All terrain buckets must be simulated (no early pruning)
- State is path-dependent (prior conditions influence current state)
- No aggregation or narrative logic at this layer

This terrain-state layer is the ground truth for all downstream logic.

---

## Layer 1 – Terrain state simulation

Simulates the condition of every terrain bucket through time.

### Example fields

- `surface_family`
- `supportability_score`
- `edgeability_score`
- `softness_score`
- `moisture_score`
- `ski_quality_score`
- `is_in_play_flag`
- `degradation_risk_score`
- `trend_score`
- `solar_load_score`
- `wind_effect_score`
- `crowd_effect_score`
- `prior_surface_family`
- `transition_reason`

No narrative logic exists at this layer.

---

## Layer 2 – Bucket ranking

Ranks every terrain bucket without aggregation.

### Principles

- No aspect-level summaries
- No terrain-type grouping
- Ranking happens at full bucket granularity

### Outputs

- `quality_rank_overall`
- `quality_score`
- `coverage_weight`
- `durability_score`
- `time_decay_risk`
- `actionability_flag`
- `composite_rank_score`

---

## Layer 3 – Pattern extraction

Identifies patterns across top and bottom ranked buckets.

### Examples

- Dominant skiable zones  
- Exceptions  
- Deterioration patterns  
- Separation between good and bad terrain  

### Operations

- Identify top/bottom N buckets  
- Cluster by shared attributes  
- Measure support fraction and terrain coverage  
- Detect divergence between strong and weak terrain  

---

## Layer 4 – Narrative evidence

Converts ranked outcomes into explicit, evidence-backed claims.

### Evidence types

- `dominant_story`
- `secondary_pattern`
- `deterioration_pattern`
- `least_bad_option`
- `dead_zone`
- `timing_shift`

### Fields

- `claim_text_seed`
- `evidence_type`
- `support_fraction`
- `coverage_fraction`
- `confidence_score`
- `actionable_flag`
- `suppression_flag`
- `reason_codes`

All claims must be supported by bucket-level evidence.

---

## Layer 5 – Rendering

Transforms narrative evidence into skier-facing output.

### Constraints

- Does not recompute rankings  
- Does not introduce new relationships  
- Only expresses existing evidence clearly  

### Outputs

- `primary_story`
- `mountain_state`
- `surface_intelligence`
- `start_here`
- `avoid_this`
- `day_evolution`

---

## What this architecture optimizes for

Snow Intel is designed to answer:

> **Where should I ski, what should I avoid, what will change, and why?**

### System priorities

- Terrain granularity  
- Path dependence  
- Operational realism  
- Timing sensitivity  
- Evidence-backed summaries  

The system reasons **from terrain upward**, not from summaries downward.
