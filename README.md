# Sentinel-2-Cloudless-Download-Memory-Optimized-Export-to-Local

## Summary

This script builds a cloud-free Sentinel-2 median mosaic for a shapefile AOI and date range, and saves the result locally. If a single export would produce a file larger than a specified threshold (default ~48 MB), the script automatically splits the AOI into an *n × n* grid (increasing *n* as needed), exports each tile locally, merges the tiles, clips the merged raster to the shapefile boundary, writes a single final GeoTIFF, and removes intermediate files.

## Required inputs

* `shapefile` — path to input `.shp` defining the AOI (example: `C:/path/to/aoi.shp` or `/home/user/aoi.shp`).
* `folder_path` — local output folder (must exist and be writable) (example: `C:/path/to/output`).
* `base_name` — base name for part files and final output (example: `S2_test_cambodia`).
* `start_date`, `end_date` — date range (format: `YYYY-MM-DD`).
* Optional: `max_size_bytes` — maximum file size in bytes before splitting (default ≈ `48 * 1024 * 1024`).

## Google Earth Engine (GEE) authentication

```python
import ee
ee.Authenticate()               # follow the browser prompt
ee.Initialize(project='your-project-id')
```

Run and follow the browser prompt to authenticate. After successful authentication, the script can access Earth Engine datasets.

## Workflow

1. Query `COPERNICUS/S2_SR_HARMONIZED` and `COPERNICUS/S2_CLOUD_PROBABILITY` for the AOI and date range.
2. Join SR and cloud-probability bands; create cloud & shadow masks; mask cloudy/shadow pixels; scale reflectance (divide by 10000).
3. Build a median mosaic and clip to the AOI.
4. Attempt to export the mosaic as a single GeoTIFF locally.
5. If the export would exceed `max_size_bytes`, delete partial attempts, split the AOI into an `n × n` grid (starting `n = 1`, increment `n` until tile exports are below threshold), export tiles locally, then:

   * Merge tiles using `rasterio.merge`.
   * Clip merged raster to shapefile using `rasterio.mask`.
   * Write final `{base_name}_clipped.tif`.
   * Delete intermediate files.

## Output

* Final GeoTIFF: `{folder_path}/{base_name}_clipped.tif`
* Temporary part files created during tiling are removed after merging.

## Dependencies

* Python 3.8+
* `earthengine-api`
* `geemap`
* `rasterio`
* `geopandas`
* `numpy`

Install:

```bash
pip install earthengine-api geemap rasterio geopandas numpy
```

## Example usage (CLI)

```bash
python export_s2_mosaic.py \
  --shapefile "/home/user/aoi.shp" \
  --output "/home/user/output" \
  --base_name "S2_test_cambodia" \
  --start_date "2023-01-01" \
  --end_date "2023-12-31" \
  --max_size_mb 48
```

## Notes & caveats

* Earth Engine does not provide a native "export directly to local disk" API; common patterns use Export to Google Drive or Cloud Storage and then programmatically download. Tools like `geemap` provide helper functions that streamline exporting directly to the drive and downloading to local — verify your chosen method in the script.
* Set `max_size_bytes` conservatively based on RAM and disk performance of the target machine. 48 MB is a reasonable default for low-memory environments, but you may increase it for faster runs on beefier machines.
* Large AOIs and dense outputs can create many tiles — monitor disk usage during processing.
* Keep CRS / resolution consistent during merging. Use the same transform, dtype, and nodata handling for all tiles to avoid artifacts.
