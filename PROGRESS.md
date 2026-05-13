# Project Progress Log — Nairobi Healthcare Accessibility

This file records all key decisions, fixes, and findings across working sessions.
It is maintained by Claude (AI assistant) and updated after each session.

---

## Session 1 — 2026-05-11

### Starting State
- Notebook code was already scaffolded (6 steps) but **no data files** existed in `Data/raw/`
- README had outdated paths (`data/` lowercase vs actual `Data/` uppercase)
- No output figures had been generated

### Data Download
Manually downloaded two required files:

| File | Source | Notes |
|------|--------|-------|
| `kenya-260510-free/gis_osm_pois_free_1.shp` | [Geofabrik Kenya](https://download.geofabrik.de/africa/kenya.html) | POI in subfolder (not directly in raw/) |
| `ken_pop_2020_CN_100m_R2025A_v1.tif` | [WorldPop Hub](https://hub.worldpop.org/geodata/summary?id=49694) | 66 MB, constrained 100m 2020 raster |

**Why 100m constrained (not 1km or unconstrained):**
- H3 resolution 9 cells are ~0.105 km² — 100m raster gives meaningful sub-cell variation
- Constrained model excludes non-residential areas (forests, water), better for urban population
- WorldPop 2030 exists but uses projected data; 2020 matches OSM snapshot date

### Path Fixes (cell-1)
Original notebook had:
```python
POI_PATH   = '../data/raw/gis_osm_pois_free_1.shp'       # wrong: lowercase 'd', missing subfolder
POP_RASTER = '../data/raw/ken_ppp_2020_UNadj.tif'         # wrong: old filename
```
Fixed to:
```python
POI_PATH    = '../Data/raw/kenya-260510-free/gis_osm_pois_free_1.shp'
POP_RASTER  = '../Data/raw/ken_pop_2020_CN_100m_R2025A_v1.tif'
WALK_GRAPH  = '../Data/processed/nairobi_walk.graphml'
```

### Kibera Geocoding Fix (cell-4)
Nominatim does not return a polygon boundary for "Kibera" — only a point.
Tried formal admin format `'Kibera Constituency, Nairobi County, Kenya'` — also failed.

**Fix:** Manual bounding box using known geographic extent:
```python
from shapely.geometry import box
if name == 'Kibera':
    areas[name] = gpd.GeoDataFrame(
        {'geometry': [box(36.760, -1.325, 36.815, -1.290)]},
        crs='EPSG:4326'
    )
```

### Facility Count
After filtering Kenya OSM POI to Nairobi:
- **Total: 980 facilities**
- pharmacy: 475, clinic: 351, hospital: 146, doctors: 8

### Population Threshold (cell-11)
Initial per_10k values had extreme outliers in peri-urban hexagons with very low population.
**Fix:** Apply population > 500 filter before computing per_10k:
```python
gdf_hex['per_10k'] = np.where(
    gdf_hex['population'] > 500,
    gdf_hex['facility_cnt'] / gdf_hex['population'] * 10_000,
    np.nan
)
```

### Legend Fix (cell-13)
`scheme='quantiles'` with `k=5` is incompatible with colorbar `legend_kwds` (causes TypeError).
**Fix:** Remove `scheme`/`k`, use continuous `vmin=0, vmax=5` scale instead.

### Walk Network
Downloaded via OSMnx, cached to `Data/processed/nairobi_walk.graphml`:
- **72,265 nodes, 176,850 edges**
- Travel time added per edge: `length / (5 km/h in m/s)`
- Multi-source Dijkstra: super-source trick (virtual node → all facility nodes at cost 0)

### Eastern Anomaly Investigation
Right-hand side of density map showed unexpectedly high per-capita values.

**Initial wrong guess:** Industrial Area (36.855, -1.325) — incorrect, do not re-add this label.

**Correct identification (verified by web search):**
- Industrial Area is near Upper Hill (~36.833, -1.300), not eastern Nairobi
- The eastern cluster is **Eastleigh / Embakasi** (36.89, -1.29)
- 95 facilities in bounding box (36.88–36.96, -1.25 to -1.32), including 23 hospitals
- High per-capita values explained by WorldPop underestimating daytime workers near JKIA and commercial zones

**Resolution:**
1. Population >500 filter reduces noise
2. vmax lowered from 10 → 5 for better contrast
3. Eastleigh/Embakasi labelled with dashed circle
4. Limitation added in notebook: "WorldPop likely undercounts daytime workers in Eastleigh/Embakasi"

### Landmark Annotations (cell-13, final state)

**Dashed circles (district highlights):**
| Name | Center (lon, lat) | Radius | Notes |
|------|-------------------|--------|-------|
| Upper Hill (Medical Hub) | (36.817, -1.300) | 0.018 | Core medical cluster — TODO reduce to 0.012 |
| Eastleigh / Embakasi | (36.890, -1.290) | 0.025 | High per-capita anomaly area |

**Star markers (point landmarks):**
| Name | Coordinates |
|------|-------------|
| UN / Gigiri | (36.803, -1.220) |
| JKIA ✈ | (36.923, -1.318) — verified |

### Final Results

**Summary table** (`outputs/tables/area_summary.csv`):

| Area | Area km² | Population | All Facilities | Hospitals | Facilities/10k | Avg Walk to Hospital | % <15 min | % <30 min | % <60 min |
|------|---------|-----------|--------------|---------|----------------|---------------------|----------|----------|----------|
| Karen | 71.3 | 101,925 | 4 | 1 | 0.39 | 56.3 min | 5.3% | 16.4% | 54.6% |
| Westlands | 97.5 | 324,665 | 75 | 12 | 2.31 | 38.9 min | 23.5% | 43.7% | 78.9% |
| Kibera | 23.7 | 222,286 | 281 | 38 | 12.64 | 11.5 min | 73.5% | 97.3% | 100.0% |
| Mathare | 3.0 | 193,786 | 129 | 18 | 6.66 | 7.2 min | 100.0% | 100.0% | 100.0% |

**Key finding:** Karen (wealthy, car-dependent) has the worst walking accessibility despite high income. Kibera/Mathare high facility counts reflect NGO clinic provision + compact geography.

---

## Session 2 — 2026-05-13

### Changes
- Re-ran full notebook; all 6 steps executed cleanly
- Updated results (minor numerical shifts from re-run):
  - Karen: avg walk 56.3 → 55.7 min; %<15 min 5.3 → 3.8%
  - Westlands: avg walk 38.9 → 38.3 min
  - Kibera: avg walk 11.5 → 10.6 min; %<15 min 73.5 → 75.2%
  - Mathare: avg walk 7.2 → 6.2 min; %<15 min 100.0 → 99.1%
- Exported notebook as HTML (`nairobi_healthcare.html`, ~2.8 MB, self-contained)
- Updated `outputs/tables/area_summary.csv` and `outputs/figures/` with fresh run

---

## Pending Tasks (as of 2026-05-13)

- [ ] Fix pink dot artifact on Westlands bar in `04_area_comparison.png`
- [ ] Shrink Upper Hill circle radius from 0.018 → 0.012 on density map (cell `9d2c235e`)
- [ ] Re-run density map cell after radius fix and verify output
- [x] Export notebook as HTML for submission → `nairobi_healthcare.html`
- [ ] Create 6-slide executive summary presentation
  - Slide 1: Problem statement + study areas map
  - Slide 2: Data sources + methodology overview
  - Slide 3: Facility density map (Map 2)
  - Slide 4: Walk-time accessibility maps (Map 3)
  - Slide 5: Area comparison charts (Map 4)
  - Slide 6: Key findings + limitations

---

## Design Decisions & Rationale

| Decision | Rationale |
|----------|-----------|
| H3 resolution 9 (~0.105 km²) | Fine city-scale granularity; sub-block level detail suitable for intra-neighbourhood variation |
| Population filter > 500 | Removes peri-urban hexagons where per_10k is mathematically extreme but meaningless |
| vmax = 5 for density map | 75th percentile of per_10k is ~0.56; vmax=10 washed out all contrast |
| Walk speed 5 km/h | Standard pedestrian assumption; conservative for informal settlement paths |
| Multi-source Dijkstra with super-source | O(E log V) single pass; scales to 72k-node graph without per-facility iteration |
| Kibera manual bbox | Nominatim returns a point not a polygon for informal settlements |
| WorldPop constrained 100m | Compatible with H3 res-9 cell size (~0.105 km²); excludes non-residential land use |
