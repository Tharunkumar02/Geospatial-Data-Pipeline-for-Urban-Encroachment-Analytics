<img width="1830" height="987" alt="image" src="https://github.com/user-attachments/assets/54f3ba84-6a3e-44bb-84fa-c1aa68a72723" />

# Hyderabad Urban Encroachment Analytics

An end-to-end geospatial pipeline on Databricks (serverless + Apache Sedona) that maps every lake/tank in Greater Hyderabad (GHMC), names it, and detects potential urban encroachment inside its legally protected zones.

## The story this project tells

The GHMC publishes lake boundaries (Full Tank Level / FTL polygons) for 847 water bodies in Greater Hyderabad — but the dataset **contains no lake names**. This project solves that by spatially joining three public gazetteers onto the polygons, then goes further: it identifies buildings standing inside the lake beds and the 30 m protected buffer strips around them, and combines that with GHMC's own survey annotations (e.g. "Mining Inside Tank") into an encroachment score per lake.

## Architecture (medallion)

| Layer | Asset | Contents |
| --- | --- | --- |
| Setup | `00_setup` | Sedona on serverless (`SedonaContext.create`), functional SQL test |
| Bronze | `01_ingest_bronze` | GHMC tanks KML (OpenCity) + GeoJSON conversion, Telangana Irrigation Dept waterbodies, OSM & Wikidata reference data, OSM building batches — cached in a UC volume |
| Silver | `02_process_silver` | 847 cleaned lake polygons as WKT with survey attributes (`silver_hydra_lakes` Delta table) |
| Gold | `03_analytics_gold` | Named, enriched and scored tables (below) |
| Serving | `Hyderabad Lake Encroachment Analytics` dashboard | Map + KPIs for stakeholders |

## Gold tables (catalog `end-to-end-geospatial-data-pipeline-for-urban-encroachment-analytics`, schema `hydraa`)

* `gold_hydra_buffers` — one row per lake: real lake name + name provenance, district, nearest habitation/village, GHMC survey encroachment flag, lake area (km2), centroid coordinates, and the FTL + 30 m buffer geometries (WKT).
* `gold_hydra_encroachment` — per-lake encroachment analytics: buildings inside the buffer ring, structures inside the FTL, GHMC survey flags, encroachment score and a human-readable status ("Confirmed (survey + structures)", "High building pressure", ...).
* `gold_hydra_buffer_buildings` — the individual OSM building footprints found inside protected zones, for drill-down.

## How lake names are resolved

1. **OpenStreetMap** (Overpass API, full polygon geometry) — accepted when the OSM water polygon overlaps >2% of the lake or sits entirely inside it.
2. **Wikidata** gazetteer — lakes/reservoirs/tanks within 45 km of Hyderabad, matched by proximity.
3. **Telangana Irrigation Dept** waterbody inventory — matched by largest polygon overlap (>10%).
4. **GHMC survey attribute** — place-name style values of the `PI_CODE` field (encroachment notes excluded).
5. **Contextual label** — "Unnamed tank near <village>, <district>" from the nearest irrigation inventory habitation.

Every lake carries a readable name, and the `name_source` column records exactly where each name came from.

## Key numbers (2026-10-31)

* 847 lakes; 176 named from public gazetteers; all 847 human-readable names (0 raw feature ids)
* 179 lakes flagged with encroachment in the GHMC survey; 250 with structures inside the FTL; 350 with building pressure inside the 30 m buffer strip
* 20,446 OSM building footprints detected inside protected zones

## Engineering notes

* Apache Sedona spatial SQL on serverless compute (Spark Connect): `SedonaContext.create(spark)` — the legacy `SedonaRegistrator` requires direct JVM access and must not be used.
* External fetches (OpenCity, Overpass, Wikidata) are cached in the bronze volume; Overpass building queries are batched, rotated across two endpoints, retried with backoff and halved on failure.
* Distances/areas: geometries are in EPSG:4326 degrees; the 30 m buffer is approximated as 0.00027° and areas converted at Hyderabad's latitude (~17.4N: 1 deg² ≈ 11,756 km²).

## Reproduction

Run `00_setup` → `01_ingest_bronze` → `02_process_silver` → `03_analytics_gold` top to bottom. First run of the encroachment cell downloads ~43 Overpass batches (~6 minutes); re-runs are cached.
