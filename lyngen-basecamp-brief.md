# Brief: Base camp site shortlisting, southern Lyngen Alps

## What this is for

We run a 3-week youth expedition in Arctic Norway (24 July – 13 August 2027) and need a base camp site for about 20 tents plus a mess tent, stores and toilets. A first recce found very few workable sites, so this is a wide-net desk study: find every patch of ground in the southern Lyngen Alps that could plausibly work, rank them, and give us a shortlist to check on imagery and then on foot.

The output is a shortlist for people to verify, not a decision. Where the data is uncertain, keep the candidate and flag the doubt rather than dropping it.

## Study area

The southern Lyngen Alps: the part of the Lyngen peninsula south of the Kjosen fjord / Lyngseidet, between Ullsfjorden–Sørfjorden to the west and Lyngenfjorden–Storfjorden to the east, down to the neck of the peninsula.

Approximate bounding box (WGS84) — check it against a map and adjust before running:

- South 69.20, North 69.60
- West 19.40, East 20.45

Furuflaten (east side, on Lyngenfjorden) is the starting point of interest, but treat the whole box equally.

Work in EPSG:25833 (the Norwegian national standard projection) for all analysis. Export in WGS84.

## Data sources

Verify each endpoint before relying on it; these are the places to look, not guaranteed URLs. If a source can't be reached, say so and use the fallback, don't silently skip the layer.

| Layer | Source | Notes |
|---|---|---|
| Terrain | Kartverket national terrain model (DTM) via hoydedata.no or the Geonorge WCS (`wcs.geonorge.no`, DTM in 25833) | Use 10 m for the whole area. Fallback: Copernicus GLO-30 DEM (open, no login). |
| Fine terrain | Kartverket DTM 1 m, same source | Only for the final candidates (step 6). Coverage may be incomplete; report where missing. |
| Paths, tracks, roads | OpenStreetMap via Overpass API | `highway=*`. Separate drivable (`track` and above) from foot only (`path`, `footway`). |
| Water | OSM `waterway=*`, `natural=water`; cross-check Kartverket N50 if available from Geonorge | |
| Unusable ground | OSM `natural=wetland`, `scree`, `bare_rock`, `glacier`, `water`; `landuse=farmland`, `residential` etc. | Farmland is not excluded outright, see criteria. |
| Buildings | OSM `building=*` | |
| Protected areas | OSM `boundary=protected_area`, or Miljødirektoratet's naturvernområder dataset | Especially Lyngsalpan landskapsvernområde. |
| Hazards | NVE susceptibility maps (aktsomhetskart): rockfall (steinsprang), snow avalanche (snøskred), debris flow (jord- og flomskred), flood (flom) | NVE publishes these as open map services and downloads. |
| Path cross-check | Kartverket N50 paths (sti), if available | OSM path coverage is patchy; use N50 to fill gaps. |

Cache every download in `data/raw/` so reruns don't hit the services again.

## Criteria

Run three tiers. The first recce found little, so the relaxed tiers matter as much as the strict one.

| Criterion | Tier A (ideal) | Tier B (workable) | Tier C (compromise) |
|---|---|---|---|
| Slope | ≤ 3° | ≤ 5° | ≤ 8° |
| Contiguous usable area | ≥ 3,000 m² | ≥ 2,000 m² | ≥ 1,200 m², or two patches ≥ 700 m² within 100 m of each other |
| Distance to running water | ≤ 200 m | ≤ 400 m | ≤ 800 m |
| Walking distance to drivable road or track | ≤ 2 km | ≤ 5 km | ≤ 10 km |
| Distance to a mapped path | ≤ 200 m | ≤ 500 m | any |

Hard exclusions, all tiers:

- Wetland, scree, bare rock, glacier, open water.
- Within 150 m of any building.
- Inside NVE rockfall, debris-flow or flood susceptibility zones.
- Within 20 m of a stream or 30 m of a lake shore (flooding and water protection), and any ground less than 2 m above the nearest stream level where the terrain model allows that check.
- Above 700 m elevation (adjustable parameter).

Flags, not exclusions (record as attributes on each candidate):

- Inside a protected area, and which one.
- On or next to farmland or other clearly private managed land.
- Inside an NVE snow-avalanche susceptibility zone (little direct relevance in July–August, but indicates steep ground above).
- Stream fed by a glacier (likely silty, high flow on warm afternoons). Infer from whether the upstream catchment contains glacier polygons.
- Aspect and a simple terrain-shelter measure (e.g. topographic position index).
- Elevation.

Slope must hold across the patch, not just on average: after thresholding, apply a small morphological opening so thin slivers and single-cell bridges don't count, and report the patch's 90th-percentile slope.

Distance to road means walking distance, not straight line. Use a least-cost path over the terrain (a Tobler-style hiking function is fine), following mapped paths where they exist, and report both distance and estimated time carrying a load. Treat water bodies, glaciers and slopes over 30° as impassable.

Put every threshold in one config file so we can change numbers and rerun.

## Method

1. Fetch and cache all layers for the bounding box. Report what was obtained, from where, and anything missing.
2. Build slope and aspect rasters from the 10 m terrain model.
3. Build the exclusion mask from the hard exclusions.
4. For each tier, threshold slope, apply the mask, clean up, and extract contiguous patches meeting the area rule.
5. For each patch, compute the attributes above and a score (weights in config; default: access 30%, water 25%, area 20%, flatness 15%, shelter 10%).
6. For the top 30 candidates overall, re-run the slope and area check on the 1 m terrain model where available, plus a surface-roughness measure as a rough proxy for boulders and hummocks. Report how each candidate's usable area changes.
7. Produce a short "why so little" summary: for the whole study area, how much ground passes slope alone, and how much each further criterion removes. This tells us which constraint is costing the most options.

## Outputs

Put everything in `output/`.

- `candidates.geojson` and `candidates.kml` — all candidate patches with attributes, tier and rank. The KML is for loading into phones and GPS units.
- `candidates.csv` — one row per candidate: ID, tier, rank, score, centre lat/lon, area, mean and p90 slope, elevation, aspect, distance to water, glacier-fed flag, walking distance and time to road, nearest road point lat/lon, distance to path, protected-area flag, farmland flag, hazard flags, 1 m check result.
- `map.html` — a single self-contained interactive map: candidates coloured by tier, with roads, paths, water, protected-area boundary and hazard zones as toggleable layers, a topographic basemap, and a popup per candidate showing its attributes plus a link to that location on norgeibilder.no (orthophotos) and norgeskart.no.
- `report.md` — one page: data obtained and gaps, number of candidates per tier, the top 10 with a sentence each, the "why so little" summary, and caveats.

## Caveats to state in the report

- The data cannot show ground surface (boulders, tussocks, waterlogging, vegetation height). Every candidate needs an orthophoto check and then a visit.
- Whether streams are running and drinkable in late July is not in the data.
- Nothing here establishes permission. A camp this size and duration needs landowner consent, and inside the protected landscape possibly the management board's too. The flags only show where that question arises.
- OSM completeness varies; an absent path or building in the data doesn't prove absence on the ground.

## Working notes

- Python with geopandas, rasterio, shapely, numpy, scipy/scikit-image, folium (or similar). Set up a virtual environment and a `requirements.txt`.
- Structure it as a rerunnable pipeline (`fetch`, `analyse`, `export`) reading from the config file.
- Check the result for sanity before finishing: open the map, confirm candidates sit on plausibly flat valley-floor or bench terrain and not on lakes, roofs or fjord, and spot-check three candidates' slope and distances by hand.
- If a decision is needed that changes the results materially (e.g. a data source is unavailable, or the bounding box clips something obvious), stop and ask.
