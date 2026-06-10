# Topography Section — Draft Milestone Text (CM)

## Data (topography subsection)

Topographic features are derived from the USGS 3D Elevation Program (3DEP) 1/3 arc-second (~10 m) digital elevation model, accessed through Google Earth Engine (`USGS/3DEP/10m`). Because the DEM is static, no temporal join is required: each fire is joined to topography purely on its ignition coordinates (`incident_latitude`, `incident_longitude`). For each ignition point we extract elevation (m), slope (degrees), aspect (degrees), hillshade, and a 1 km neighborhood standard deviation of elevation as a terrain-ruggedness proxy.

## Data preprocessing (CAL FIRE base table)

The raw CAL FIRE incident export contains 3,519 records spanning 1969–2026. Cleaning proceeded in five steps: (1) restrict to incidents created in 2013 or later, where reporting is consistent (3,516 remain); (2) drop non-fire incident types (floods, earthquakes) tracked in the same feed (3,513); (3) drop records with missing or non-positive burned acreage, since the log-transformed target requires positive values (3,438); (4) drop records whose coordinates fall outside a California bounding box (32.5–42.1°N, 114.1–124.6°W), removing placeholder and out-of-state coordinates (3,419); (5) keep only finalized incidents (`incident_is_final == 'Y'`) so the target reflects final, not in-progress, acreage (3,405). The final modeling table has **3,405 fires**.

The target is heavily right-skewed: the median fire burns 68 acres while the maximum exceeds 1,000,000 acres (mean 3,583; SD 31,680). We therefore model **log10(acres burned)**, which is approximately unimodal with mean ~2.0 (≈100 acres). Aspect is a circular variable (0° and 360° are identical), so it is encoded as sine and cosine components plus a south-facing indicator (112.5°–247.5°), reflecting that south-facing slopes in the northern hemisphere carry drier fuels.

For train/validation/test we use a chronological 70/15/15 split (2,383 / 511 / 511 fires), holding out the most recent fires as test to avoid temporal leakage and to mirror the operational use case (predicting future fires from past data).

## EDA findings (fire table — already runnable)

- **Target skew:** raw acreage spans 7 orders of magnitude; log transform is essential (Fig. 1).
- **Spatial pattern:** large fires cluster in the Sierra Nevada foothills and North Coast ranges; small fires dominate the Central Valley floor and urban-adjacent areas — preliminary evidence that terrain matters (Fig. 2).
- **Seasonality:** ignition-month boxplots show larger median sizes June–September, consistent with the fire season; this motivates the weather features handled by QW (Fig. 3).
- **Reporting-density shift:** 2024–2025 show many more recorded fires with much smaller median size (43 and 30 acres vs ~100 historically), suggesting CAL FIRE now logs more small incidents. This is a data challenge: a chronological split places these in test, so we will assess whether to include an era indicator or restrict the test window.

## Data challenges (topography)

1. **Single-point sampling vs fire footprint:** topography is sampled at the ignition point, but a fire burns across terrain. Mitigation: the 1 km elevation-std-dev band captures neighborhood ruggedness; if needed we can sample within a buffer.
2. **Coordinate precision:** some incidents report coordinates rounded to ~0.001° (~100 m), coarser than the 10 m DEM. Slope/aspect at the exact cell may be noisy; neighborhood statistics mitigate this.
3. **GEE rate limits:** `reduceRegions` on 3,405 points must be batched (500 points/call) and the ruggedness band may require export tasks rather than interactive calls.
4. **Circular aspect encoding:** handled via sin/cos + south-facing indicator as described above.
