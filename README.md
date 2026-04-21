# Nairobi Healthcare Accessibility — GIS Analysis

**Research Question:** How does healthcare accessibility differ between wealthy neighbourhoods and informal settlements within Nairobi?

**Study Areas:**
| Area | Type | Approx. Population |
|------|------|-------------------|
| Karen | Wealthy residential | ~50,000 |
| Westlands | Wealthy/commercial | ~300,000 |
| Kibera | Informal settlement | 200,000–1,000,000 |
| Mathare | Informal settlement | 200,000–500,000 |

---

## Data Sources

| Dataset | Source | How to Download |
|---------|--------|-----------------|
| Kenya OSM POI (hospitals, clinics) | [Geofabrik](https://download.geofabrik.de/africa/kenya.html) | Download `kenya-latest-free.shp.zip`, extract to `data/raw/` |
| Kenya admin boundaries | [GADM](https://gadm.org/download_country.html) | Download Kenya GeoPackage to `data/raw/` |
| WorldPop Kenya 2020 (100m) | [WorldPop Hub](https://hub.worldpop.org/geodata/summary?id=49694) | Download `ken_ppp_2020_UNadj.tif` to `data/raw/` |

> **Note:** Raw data files are not committed to Git due to size. Download them manually into `data/raw/`.

---

## Project Structure

```
Project/
├── data/
│   ├── raw/                        # Downloaded source data (not versioned)
│   │   ├── gis_osm_pois_free_1.shp # Kenya OSM POI
│   │   ├── ken_ppp_2020_UNadj.tif  # WorldPop population raster
│   │   └── gadm41_KEN.gpkg         # Kenya admin boundaries
│   └── processed/                  # Cleaned / joined outputs
├── notebooks/
│   └── nairobi_healthcare.ipynb    # Main analysis notebook
├── outputs/
│   ├── figures/                    # Generated maps
│   └── tables/                     # Summary statistics (CSV)
├── requirements.txt
└── README.md
```

---

## Analysis Workflow

### Step 1 — Data Loading & Spatial Filtering
Load Kenya OSM POI, filter to Nairobi extent, classify by facility type (hospital / clinic / pharmacy).

### Step 2 — Hexagon Grid (H3 resolution 8, ~0.7 km²/hex)
Generate hexagonal grid over Nairobi. Count facilities per hexagon. Join WorldPop raster to get population per hexagon.

### Step 3 — Facility Density Map
Choropleth: healthcare facilities per 10,000 people per hexagon. Highlight four study areas.

### Step 4 — Walk-Time Accessibility (Multi-source Dijkstra)
Download Nairobi walking network via OSMnx. Using multi-source Dijkstra (super-source trick), compute travel time from every network node to the nearest hospital. Assign walk time to each hexagon centroid.

### Step 5 — Comparison Maps & Charts
- Side-by-side walk-time maps (to hospital vs to any facility)
- Bar charts: population-weighted average walk time by area
- Coverage rate: % population within 15 / 30 / 60 min threshold

---

## Key Metrics

- **Facilities per 10,000 people** — equity metric (population-weighted)
- **Average walk time to nearest hospital** — physical accessibility
- **% population within 15/30/60 min** — threshold-based coverage

---

## Limitations

- OSM data may underrepresent informal healthcare in Kibera/Mathare
- WorldPop may underestimate population in dense informal settlements
- Walk-time assumes 5 km/h on mapped paths; actual conditions in slums may differ
- OSMnx walk network may omit unmapped footpaths in informal areas
