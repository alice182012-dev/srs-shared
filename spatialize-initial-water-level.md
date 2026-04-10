If you have no pipes, channels, or manholes in the schematisation yet, your model is **purely 2D** at this stage. The answer is straightforward:

**Use 2D initial water levels.**

There are no 1D nodes in your grid to assign values to. The `initial_waterlevels.csv` (1D file) would be empty and pointless. Your sensor readings need to go into the 2D domain.

---

## How to Do It

You have two options depending on how many sensors you have.

### Option A — Global scalar (1 or few sensors, roughly uniform water level)

Set a single water level value that applies to the entire 2D domain. This lives in the schematisation's `initial_conditions` table — no file needed.

```
initial_water_level = -0.5   # m MSL — your best estimate of pre-storm state
```

Simple, but coarse. Every 2D cell starts at the same level regardless of terrain.

### Option B — Spatially varying raster (multiple sensors, spatially distributed)

You interpolate your sensor readings onto a raster (GeoTIFF, same CRS and extent as your DEM), then upload that raster as the 2D initial water level. 3Di aggregates the raster values onto each computational cell using your chosen method (min / max / average).

```python
import numpy as np
import rasterio
from rasterio.transform import from_bounds
from scipy.interpolate import griddata

# Your sensor readings — (easting, northing, water_level_m_MSL) in EPSG:3405
sensors = np.array([
    [585000, 2337000, 0.43],
    [587500, 2335000, 0.61],
    [590000, 2339000, 0.28],
    # ...
])

# Build interpolation grid matching your DEM extent and resolution
xmin, ymin, xmax, ymax = 583000, 2333000, 595000, 2345000
res = 10  # metres — doesn't need to match DEM exactly, 10–50m is fine

xi = np.arange(xmin, xmax, res)
yi = np.arange(ymin, ymax, res)
grid_x, grid_y = np.meshgrid(xi, yi)

# Interpolate sensor points onto grid (linear, with nearest-neighbour fill)
wl_grid = griddata(
    points=sensors[:, :2],
    values=sensors[:, 2],
    xi=(grid_x, grid_y),
    method='linear',
    fill_value=np.nan
)
# Fill extrapolated edges with nearest
wl_grid_filled = griddata(
    points=sensors[:, :2],
    values=sensors[:, 2],
    xi=(grid_x, grid_y),
    method='nearest'
)
wl_grid = np.where(np.isnan(wl_grid), wl_grid_filled, wl_grid)

# Write GeoTIFF
transform = from_bounds(xmin, ymin, xmax, ymax, wl_grid.shape[1], wl_grid.shape[0])
with rasterio.open(
    'initial_wl_2d.tif', 'w',
    driver='GTiff', height=wl_grid.shape[0], width=wl_grid.shape[1],
    count=1, dtype='float32', crs='EPSG:3405', transform=transform,
    nodata=-9999
) as dst:
    data = wl_grid.astype('float32')
    data[np.isnan(data)] = -9999
    dst.write(data, 1)
```

Then upload via the API:

```python
upload_obj = api_client.simulations_initial2d_water_level_raster_create(
    simulation.id,
    data={"filename": "initial_wl_2d.tif", "aggregation_method": "average"}
)
upload_file(upload_obj.put_url, "initial_wl_2d.tif")
# poll until state = "processed"
```

---

## Which Option to Use

| Situation | Recommendation |
|---|---|
| 1–2 sensors, flat terrain | Option A — global scalar, use the average reading |
| 3+ sensors, varying across domain | Option B — interpolated raster |
| Sensors only on roads/drains, not representative of standing water | Option A with a conservative dry value (e.g. bottom of DEM − 0.1m) |
| Pre-storm dry conditions (no flooding yet) | Option A with a clearly sub-ground value, e.g. `-2.0 m MSL` |

For FloodNav's on-demand trigger architecture — where simulations start because rainfall is *about to happen*, not because flooding has already started — a dry initial condition (Option A, low scalar) is often the most defensible choice. The sensor readings become more valuable once you add 1D drainage objects, because the antecedent pipe fullness is what really determines how quickly the surface floods.
