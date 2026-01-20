# PyPSA-GB v2.0 Repository Evaluation Report

## Executive Summary

**Recommendation: Stick with PyPSA-FES** for water-constrained hydrogen pathway research. PyPSA-GB lacks the critical myopic optimization capability required for multi-year pathway modeling.

| Criterion | PyPSA-FES | PyPSA-GB | Winner |
|-----------|-----------|----------|--------|
| Myopic Pathways | Yes | **No** | PyPSA-FES |
| Zonal Structure | 16 zones (ESO) | 17 zones (different) | Comparable |
| Hydrogen Detail | Medium-High | Low (copper-plate) | PyPSA-FES |
| Water Constraints | No (add custom) | No (add custom) | Tie |
| FES Integration | High | High | Tie |

---

## Phase 1: Repository Structure & Architecture

### PyPSA-GB Directory Structure

```
PyPSA-GB/
├── Snakefile                 # Main workflow orchestration
├── Snakefile_cutouts         # Weather cutout generation
├── config/
│   ├── config.yaml           # Active run configuration
│   ├── scenarios.yaml        # Scenario definitions
│   ├── defaults.yaml         # Default parameters
│   └── clustering.yaml       # Clustering presets
├── rules/
│   ├── renewables.smk        # Wind/solar profile generation
│   ├── network_build.smk     # Network topology construction
│   ├── network_clustering.smk # Spatial clustering
│   ├── demand.smk            # Demand processing
│   ├── FES.smk               # FES data integration
│   ├── generators.smk        # Generator integration (DUKES/REPD/FES)
│   ├── storage.smk           # Battery, PHS, LAES
│   ├── hydrogen.smk          # H2 system (electrolysis, storage, turbines)
│   ├── interconnectors.smk   # Cross-border links
│   ├── solve.smk             # Network optimization
│   └── analysis.smk          # Post-solve visualization
├── scripts/                  # Python implementation scripts
├── data/
│   ├── network/              # ETYS, Reduced, Zonal topologies
│   ├── FES/                  # Future Energy Scenarios data
│   ├── generators/           # DUKES, TEC, REPD data
│   ├── renewables/           # REPD, offshore sites, marine
│   ├── demand/               # ESPENI, eload, DESSTINEE
│   ├── storage/              # Battery, PHS parameters
│   └── interconnectors/      # Cross-border links
└── tests/                    # Unit and workflow tests
```

### Architectural Approach

- **Snakemake-based workflow** with modular rule files
- **Scenario-driven configuration** via YAML files
- **Automatic historical/future routing**: Years ≤2024 use DUKES; >2024 use FES
- **Three network resolutions**: ETYS (2000+ buses), Reduced (32), Zonal (17)

### GB-Specific Data Handling

| Data Source | Purpose | Integration |
|-------------|---------|-------------|
| ESPENI | Historical demand profiles | Half-hourly GB demand |
| DUKES | Historical generator capacities | Power station data |
| REPD | Renewable projects | Wind/solar installations |
| NESO FES | Future projections | Capacity, demand, scenarios |
| ETYS | Transmission network | Full GB grid topology |

---

## Phase 2: Energy System Component Implementation

### 2.1 Demand/Load

**Source**: ESPENI (historical), FES (future annual totals)
**Disaggregation**: Spatial allocation to buses/zones based on GSP breakdown
**Temporal**: Hourly resolution, full year

```yaml
# config/scenarios.yaml
demand_timeseries: "ESPENI"    # or "eload", "desstinee"
demand_year: 2020              # Year for profile shape
```

**Key finding**: No sector-coupled demand - electricity only.

### 2.2 Generation

**Renewable workflow**:
1. Weather cutouts (Atlite/ERA5)
2. Capacity factors by technology and location
3. FES capacity targets applied for future scenarios

**Data sources**:
- **Historical**: DUKES power stations + REPD renewables
- **Future**: FES capacity projections by GSP

**Carriers**: wind_onshore, wind_offshore, solar_pv, hydro, tidal_stream, wave, geothermal, nuclear, CCGT, OCGT, biomass, coal

### 2.3 Storage

**Implemented**:
- Grid-scale batteries (from REPD/FES)
- Pumped hydro storage (PHS)
- LAES (Liquid Air Energy Storage) - placeholder
- CAES (Compressed Air) - placeholder

**Not implemented**: Behind-meter storage, thermal storage for heat pumps

### 2.4 Network

**Three configurations**:

| Model | Buses | Use Case |
|-------|-------|----------|
| ETYS | 2000+ | Full transmission detail |
| Reduced | 32 | Fast prototyping |
| Zonal | 17 | Aggregated regional analysis |

**Topology source**: ETYS data with planned upgrades through 2031
**Clustering**: GSP spatial or k-means algorithms available
**Interconnectors**: All GB-Europe links modeled (Norway, France, Belgium, Netherlands, Ireland)

### 2.5 Hydrogen & Gas

**CRITICAL LIMITATION**: Simplified "copper-plate" hydrogen model

```python
# From scripts/hydrogen/add_hydrogen_system.py
"""
The current implementation uses a "copper-plate" hydrogen network:
- Single GB-wide hydrogen bus (no spatial resolution)
- All electrolysers feed into common H2 storage
- All H2 turbines draw from common H2 storage

This is a simplification - future work will model regional H2 networks.
"""
```

**Parameters**:
- Electrolysis efficiency: 70%
- H2 turbine efficiency: 50%
- Storage: 168 hours (1 week) of generation capacity

**No water constraints**: Search confirms no water resource modeling for electrolysis or thermal cooling.

---

## Phase 3: Workflow & Data Flow

### Pipeline Stages

```
1. Weather cutouts (Snakefile_cutouts)
   └── Atlite/ERA5 → renewable profiles

2. Network building
   ├── Base topology (ETYS/Reduced/Zonal)
   ├── Apply upgrades (ETYS through 2031)
   └── Optional clustering (GSP/k-means)

3. Component integration
   ├── Demand → buses
   ├── Renewables → generators with p_max_pu profiles
   ├── Thermal → generators with marginal costs
   ├── Storage → storage_units
   ├── Hydrogen → Links (electrolysis, H2 turbines) + Store
   └── Interconnectors → Links

4. Solve (single year optimization)
   ├── LP or MILP mode
   ├── Optional solve_period (subset of year)
   └── Export results (generation, flows, costs, emissions)

5. Analysis
   ├── Spatial plots
   ├── Dashboards
   └── Jupyter notebooks
```

### Key Difference from PyPSA-FES

**PyPSA-FES workflow**:
```
rules/solve_overnight.smk  → Single year
rules/solve_myopic.smk     → Multi-year pathway with brownfield
```

**PyPSA-GB workflow**:
```
rules/solve.smk            → Single year ONLY
```

**No multi-year pathway optimization capability**.

---

## Phase 4: Parameters vs Variables Taxonomy

### Exogenous Parameters (Fixed Inputs)

| Category | Parameter | Source |
|----------|-----------|--------|
| Temporal | modelled_year, demand_year, renewables_year | scenarios.yaml |
| Demand | Annual totals, profiles | FES + ESPENI |
| Network | Topology, line ratings | ETYS data |
| Technology | Efficiencies, costs | defaults.yaml, costs.csv |
| FES Targets | Capacity by technology | FES projections |
| Weather | Capacity factors | Atlite cutouts |

### Endogenous Variables (Optimized)

| Component | Variable | Constraint |
|-----------|----------|------------|
| Generators | p (dispatch) | p_nom (fixed from FES) |
| Storage | p (charge/discharge), e (state) | e_nom (fixed from FES) |
| Lines | p0, p1 (flows) | s_nom (fixed) |
| Links | p (flows) | p_nom (fixed) |
| **NOT optimized** | p_nom_opt | Capacity expansion |

**Key finding**: PyPSA-GB optimizes **dispatch only**, not capacity investment.

---

## Phase 5: Economic Modeling Capabilities

### Optimization Formulation

**Objective**: Minimize total system cost
```
min Σ (marginal_cost × generation) + VOLL × load_shedding
```

**Constraints**:
- Power balance at each bus
- Generator capacity limits (from FES)
- Line thermal limits
- Storage energy/power limits
- Optional ramp rate constraints (MILP mode)

### Solver Configuration

```yaml
# config/defaults.yaml
solver:
  name: "gurobi"    # or "highs"
  method: 2         # barrier
  threads: 4
solve_mode: "LP"    # or "MILP" for unit commitment
```

### Investment Modeling Capability

| Feature | PyPSA-GB | PyPSA-FES |
|---------|----------|-----------|
| Overnight (single year) | Yes | Yes |
| Myopic pathways | **No** | Yes |
| Brownfield handling | **No** | Yes (add_brownfield.py) |
| Capacity expansion | **No** | Yes (p_nom_extendable) |
| Planning horizons | Single | 2025→2030→2035→2040→2050 |

### Performance

```
# From rules/solve.smk documentation:
- Full year ETYS network: 30 min - 2 hours
- Full year clustered: 5-30 minutes
- One week clustered: 1-5 minutes
- Zonal networks: 1-10 minutes
```

---

## Phase 6: Research Suitability Assessment

### Critical Requirements for Water-Constrained Hydrogen Research

| Requirement | PyPSA-FES | PyPSA-GB | Assessment |
|-------------|-----------|----------|------------|
| **Myopic optimization** | Yes (solve_myopic.smk) | **No** | BLOCKER |
| ESO zonal structure | 16 zones | 17 zones (different) | Compatible |
| Hydrogen production | Regional nodes | Single copper-plate bus | FES better |
| Hydrogen storage | Regional | Single bus | FES better |
| H2 pipeline network | Yes (H2_network) | No | FES better |
| Water constraints | No (add custom) | No (add custom) | Equivalent |
| FES integration | High | High | Equivalent |
| Computational | 16GB feasible | 16GB feasible | Equivalent |

### Detailed Comparison

#### Myopic Pathways (CRITICAL)

**PyPSA-FES** (rules/solve_myopic.smk):
```python
rule add_brownfield:
    """
    Add brownfield capacity to network for myopic optimization.
    Carries forward existing capacity from previous planning horizon.
    """
    input:
        network_p=solved_previous_horizon,  # Previous year's solution
```

**PyPSA-GB**: No equivalent capability. Scenarios are independent single-year optimizations.

#### Hydrogen Infrastructure

**PyPSA-FES** (config/config.default.yaml):
```yaml
sector:
  H2_network: true
  hydrogen_underground_storage: true
  hydrogen_fuel_cell: true
  SMR: true
```

**PyPSA-GB** (scripts/hydrogen/add_hydrogen_system.py):
```python
"""
Single GB-wide hydrogen bus (no spatial resolution)
All electrolysers feed into common H2 storage
All H2 turbines draw from common H2 storage
"""
```

#### Water Constraints

Neither model includes water constraints. Both would require custom additions:
- Cooling water limits for thermal plants
- Water consumption for electrolysis (~9L/kg H2)
- Regional water availability constraints

### Migration Effort Assessment

| If migrating to PyPSA-GB | Effort | Notes |
|--------------------------|--------|-------|
| Add myopic optimization | **HIGH** | Requires new rules, brownfield scripts, planning horizon logic |
| Regionalize H2 network | **MEDIUM** | Add spatial H2 buses, pipelines, regional storage |
| Add water constraints | **MEDIUM** | Similar effort in both models |
| Adapt to 17-zone structure | **LOW** | Minor mapping changes |

**Total estimated effort**: 3-6 months of development

### Why PyPSA-FES is Better Suited

1. **Myopic optimization exists**: The core capability for pathway modeling is already implemented and tested

2. **Better H2 infrastructure**: Regional hydrogen nodes, storage, and network options

3. **Active development trajectory**: Based on PyPSA-Eur with ongoing sector coupling improvements

4. **Brownfield handling**: Essential for realistic transition pathway modeling

5. **Planning horizons**: Built-in support for 2025→2030→2035→2040→2050 sequences

---

## Final Recommendation

### Stick with PyPSA-FES

**Rationale**:
1. **Myopic optimization is essential** for your PhD research on transition pathways - PyPSA-GB doesn't have it
2. Adding myopic capability to PyPSA-GB would require substantial development (3-6 months)
3. Water constraints require custom addition in either model - equivalent effort
4. PyPSA-FES has better hydrogen infrastructure out of the box

### Recommended Approach

1. **Continue with PyPSA-FES** as your base model
2. **Add water constraints** as custom constraints using `extra_functionality`:
   ```python
   def add_water_constraints(network, snapshots):
       # Regional water availability limits
       # Electrolysis water consumption (~9L/kg H2)
       # Thermal cooling water requirements
   ```
3. **Consider PyPSA-GB's zonal data** if you need the 17-zone representation
4. **Reference PyPSA-GB's hydrogen research** (Dergunova & Lyden, 2024) for methodology

### Hybrid Approach (Optional)

If PyPSA-GB's network topology or FES integration is valuable:
- Use PyPSA-GB for network/data preparation
- Export to PyPSA-FES format for optimization
- This preserves myopic capability while gaining data quality

---

## Appendix: PyPSA-GB Strengths (for future reference)

Despite limitations for your research, PyPSA-GB excels at:
- **Detailed ETYS transmission network** (2000+ buses)
- **Comprehensive DUKES integration** for historical validation
- **Well-documented codebase** with tests
- **GSP-level spatial clustering**
- **ETYS network upgrades** through 2031

Consider PyPSA-GB for:
- Historical validation studies
- Single-year dispatch analysis
- Transmission constraint studies
- Network clustering methodology

---

*Report generated: 2026-01-20*
*Repositories analyzed: PyPSA-GB (andrewlyden/PyPSA-GB), PyPSA-FES (koneill-lboro/pypsa-fes)*
