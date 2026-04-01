# FloodNav Hanoi — Data Schema & Database Catalogue

**Complete data entity inventory for the 5-model basin-aligned architecture**

Version 1.0 — March 2026

---

## Database Technology Stack

| Database | Technology | Purpose | Location |
|----------|-----------|---------|----------|
| Spatial | PostGIS (PostgreSQL 16+) | Road network, segment depths, institutions, districts, dispatch, community reports | Node B |
| Time-series | TimescaleDB (PostgreSQL extension) | Institution histograms, depth history, pipeline metrics, gauge history | Node B |
| Cache / Message queue | Redis 7+ | Pipeline event fan-out, session cache, rate limiting | Node B |
| File storage | NVMe filesystem + NAS | GeoTIFFs, results_3di.nc, gridadmin.h5, scenario library, DTM rasters | Nodes A-1/A-2 (scratch), Node B (persistent), NAS (archive) |
| User data | PostgreSQL (same instance as PostGIS) | User accounts, saved places, alert preferences, FCM tokens | Node B |

---

## 1. STATIC REFERENCE DATA (Updated infrequently — model build or manual refresh)

### 1.1 road_segments

The core spatial table. Every navigable road segment in Hanoi (OSM-derived), ~50,000+ segments.

```
Table: postgis.road_segments
```

| Column | Type | Description |
|--------|------|-------------|
| segment_id | SERIAL PRIMARY KEY | Unique segment identifier |
| osm_way_id | BIGINT | OpenStreetMap way ID |
| name_vi | TEXT | Vietnamese road name |
| name_en | TEXT | English road name (nullable) |
| district_id | INTEGER FK → districts | District this segment belongs to |
| regional_model_id | INTEGER FK → regional_models | Which of the 5 models covers this segment |
| road_class | TEXT | highway, trunk, primary, secondary, tertiary, residential, service |
| geom | GEOMETRY(LineString, 4326) | Segment geometry (WGS84) |
| geom_vn2000 | GEOMETRY(LineString, 3405) | Segment geometry (VN-2000 UTM 48N — matches GeoTIFF CRS) |
| length_m | DOUBLE PRECISION | Segment length in metres |
| centroid | GEOMETRY(Point, 4326) | Segment centroid for depth sampling |
| centroid_vn2000 | GEOMETRY(Point, 3405) | Centroid in VN-2000 for GeoTIFF pixel lookup |
| is_arterial | BOOLEAN DEFAULT FALSE | TRUE for highway/trunk/primary — affects pre-closure severity ranking |
| near_institution | BOOLEAN DEFAULT FALSE | TRUE if within 200m of a registered institution — affects severity ranking |
| valhalla_edge_id | BIGINT | Valhalla routing graph edge ID for re-weighting |
| updated_at | TIMESTAMPTZ | Last OSM data refresh |

**Indexes:** GIST on geom, GIST on geom_vn2000, BTREE on district_id, BTREE on regional_model_id, BTREE on valhalla_edge_id

**Refresh:** Weekly from OSM PBF extract

---

### 1.2 districts

Hanoi's 30 administrative districts.

```
Table: postgis.districts
```

| Column | Type | Description |
|--------|------|-------------|
| district_id | SERIAL PRIMARY KEY | |
| name_vi | TEXT NOT NULL | Vietnamese district name |
| name_en | TEXT | English name |
| district_type | TEXT | quận / huyện / thị_xã |
| area_km2 | DOUBLE PRECISION | Administrative area |
| regional_model_id | INTEGER FK → regional_models | Primary regional model covering this district |
| geom | GEOMETRY(MultiPolygon, 4326) | District boundary polygon |
| population | INTEGER | Estimated population (for severity weighting) |

---

### 1.3 regional_models

The 5 basin-aligned simulation models.

```
Table: postgis.regional_models
```

| Column | Type | Description |
|--------|------|-------------|
| model_id | SERIAL PRIMARY KEY | 1=Lõi, 2=Tây, 3=Đông, 4=Bắc, 5=Nam |
| name_vi | TEXT | e.g., "Lõi — Tô Lịch + Tả Nhuệ" |
| name_code | TEXT | loi, tay, dong, bac, nam |
| area_km2 | DOUBLE PRECISION | Model domain area |
| comp_grid_m | INTEGER | Computational grid resolution (8, 20, 40) |
| dtm_resolution_m | DOUBLE PRECISION | DTM sub-grid resolution (1.0 or 5.0) |
| estimated_cells | INTEGER | Approximate computational cell count |
| threedimodel_id | TEXT | 3Di API model identifier |
| template_id_2h | TEXT | 3Di simulation template ID for 2h constant_cycle |
| template_id_24h | TEXT | 3Di simulation template ID for on-demand 24h |
| gridadmin_path | TEXT | Filesystem path to gridadmin.h5 |
| geom | GEOMETRY(MultiPolygon, 4326) | Model domain boundary |
| compute_node | TEXT | 'A1' or 'A2' — which cluster node runs this model |
| core_allocation | INTEGER | CPU cores allocated for this model |
| priority | INTEGER | Execution priority (1=highest) |

---

### 1.4 institutions

Registered public institutions (hospitals, schools, government, evacuation points).

```
Table: postgis.institutions
```

| Column | Type | Description |
|--------|------|-------------|
| institution_id | SERIAL PRIMARY KEY | |
| name_vi | TEXT NOT NULL | Vietnamese name |
| name_en | TEXT | English name |
| institution_type | TEXT NOT NULL | hospital / school / government / evacuation_point |
| address_vi | TEXT | Address |
| district_id | INTEGER FK → districts | |
| regional_model_id | INTEGER FK → regional_models | |
| location | GEOMETRY(Point, 4326) | WGS84 location |
| location_vn2000 | GEOMETRY(Point, 3405) | VN-2000 for GeoTIFF lookup |
| is_critical | BOOLEAN DEFAULT FALSE | TRUE for hospitals — never shown as "do not approach" |
| contact_phone | TEXT | Emergency contact |
| capacity | INTEGER | Facility capacity (beds, students, etc.) |
| created_at | TIMESTAMPTZ | |

---

### 1.5 vehicle_types

Vehicle type thresholds for passability determination.

```
Table: postgis.vehicle_types
```

| Column | Type | Description |
|--------|------|-------------|
| vehicle_type_id | SERIAL PRIMARY KEY | |
| code | TEXT UNIQUE NOT NULL | xe_may, sedan, suv, truck |
| name_vi | TEXT | Xe máy, Ô tô con, SUV, Xe tải |
| name_en | TEXT | Motorbike, Sedan, SUV, Truck |
| hard_block_cm | INTEGER NOT NULL | 20, 40, 55, 70 |
| penalty_zone_min_cm | INTEGER NOT NULL | 10, 20, 30, 50 |
| penalty_zone_max_cm | INTEGER NOT NULL | 20, 40, 55, 70 |
| penalty_cost_multiplier | DOUBLE PRECISION | 3.0, 2.0, 2.0, 1.5 |
| display_order | INTEGER | UI display order |

---

### 1.6 gauge_stations

Real-time water level gauge stations used for boundary conditions and validation.

```
Table: postgis.gauge_stations
```

| Column | Type | Description |
|--------|------|-------------|
| station_id | SERIAL PRIMARY KEY | |
| station_code | TEXT UNIQUE NOT NULL | e.g., "HN_NHUE_001" |
| name_vi | TEXT | Station name in Vietnamese |
| river_name | TEXT | River or canal name |
| station_type | TEXT | water_level / discharge / rainfall |
| api_endpoint | TEXT | URL for real-time data polling |
| api_format | TEXT | json / csv / grib2 |
| poll_interval_s | INTEGER | Polling interval in seconds (typically 900 = 15 min) |
| location | GEOMETRY(Point, 4326) | |
| regional_model_id | INTEGER FK → regional_models | Which model uses this gauge for BCs |
| bc_type | TEXT | water_level / discharge / sommerfeld — how this gauge maps to 3Di BC |
| bc_id_in_model | INTEGER | The "id" value in the 3Di BC CSV that corresponds to this gauge |
| calc_node_id | INTEGER | 3Di calculation node ID (for initial waterlevel mapping) |
| is_bc_source | BOOLEAN DEFAULT FALSE | TRUE if used as boundary condition |
| is_validation | BOOLEAN DEFAULT FALSE | TRUE if used for model validation |

---

### 1.7 pump_stations

Drainage pump stations (operational status affects model accuracy).

```
Table: postgis.pump_stations
```

| Column | Type | Description |
|--------|------|-------------|
| pump_id | SERIAL PRIMARY KEY | |
| name_vi | TEXT | e.g., "Trạm bơm Yên Sở" |
| capacity_m3s | DOUBLE PRECISION | Design capacity (m³/s) |
| operational_status | TEXT | operational / partial / under_construction / planned |
| basin | TEXT | to_lich, ta_nhue, huu_nhue, long_bien |
| regional_model_id | INTEGER FK → regional_models | |
| location | GEOMETRY(Point, 4326) | |
| calc_node_id | INTEGER | 3Di calculation node ID |
| notes | TEXT | e.g., "La Khê channel incomplete, not at full capacity" |
| is_modelled | BOOLEAN DEFAULT TRUE | FALSE for planned-but-unbuilt stations |

---

### 1.8 scenario_library_index

Index of pre-computed scenario library entries (actual result files on disk).

```
Table: postgis.scenario_library_index
```

| Column | Type | Description |
|--------|------|-------------|
| scenario_id | SERIAL PRIMARY KEY | |
| regional_model_id | INTEGER FK → regional_models | |
| rainfall_intensity_class | TEXT | very_light / light / moderate / heavy / extreme |
| rainfall_duration_hours | DOUBLE PRECISION | 1, 2, 4, 6 |
| spatial_pattern | TEXT | uniform / north_heavy / south_heavy / east_heavy / west_heavy / central |
| initial_sewer_state | TEXT | morning_peak_dwf / midday_low_dwf / evening_peak_dwf |
| result_path | TEXT | Filesystem path to pre-computed GeoTIFF series |
| computed_at | TIMESTAMPTZ | When this scenario was last computed |
| comp_grid_m | INTEGER | Resolution used for this scenario |
| total_rainfall_mm | DOUBLE PRECISION | Total rainfall in mm |
| peak_depth_m | DOUBLE PRECISION | Maximum depth anywhere in domain |

---

## 2. REAL-TIME CYCLE DATA (Written every constant_cycle by pipeline workers)

### 2.1 segment_flood_state

Current and forecast flood state per road segment. **The primary table for routing and pre-closure.**

```
Table: postgis.segment_flood_state
```

| Column | Type | Description |
|--------|------|-------------|
| segment_id | INTEGER FK → road_segments | |
| cycle_id | INTEGER FK → pipeline_cycles | Which cycle produced this data |
| current_depth_m | DOUBLE PRECISION | Depth at T+0 (metres) |
| avg_depth_m | DOUBLE PRECISION | Average depth across segment length |
| max_depth_m | DOUBLE PRECISION | Maximum depth at any node along segment |
| depth_t15m | DOUBLE PRECISION | Forecast depth at T+15 min |
| depth_t30m | DOUBLE PRECISION | Forecast depth at T+30 min |
| depth_t1h | DOUBLE PRECISION | Forecast depth at T+1h |
| depth_t1h30 | DOUBLE PRECISION | Forecast depth at T+1h30 |
| depth_t2h | DOUBLE PRECISION | Forecast depth at T+2h |
| peak_depth_m | DOUBLE PRECISION | Maximum depth across all timesteps |
| peak_time | TIMESTAMPTZ | Wall-clock time of peak depth |
| recession_start | TIMESTAMPTZ | Wall-clock time when depth begins decreasing |
| return_passable_xemay | TIMESTAMPTZ | When depth < 0.20m (nullable if never floods) |
| return_passable_sedan | TIMESTAMPTZ | When depth < 0.40m |
| return_passable_suv | TIMESTAMPTZ | When depth < 0.55m |
| return_passable_truck | TIMESTAMPTZ | When depth < 0.70m |
| preclosure_xemay | BOOLEAN DEFAULT FALSE | T+1h depth ≥ 0.20m |
| preclosure_sedan | BOOLEAN DEFAULT FALSE | T+1h depth ≥ 0.40m |
| preclosure_suv | BOOLEAN DEFAULT FALSE | T+1h depth ≥ 0.55m |
| preclosure_truck | BOOLEAN DEFAULT FALSE | T+1h depth ≥ 0.70m |
| preclosure_eta_minutes | INTEGER | Estimated minutes until hard block (nearest vehicle type) |
| data_source | TEXT | 'realtime' or 'scenario_library' |
| source_model_id | INTEGER FK → regional_models | Which model produced this data |
| pipeline_completion_time | TIMESTAMPTZ | Wall-clock time when this data became available |
| updated_at | TIMESTAMPTZ DEFAULT NOW() | |

**Indexes:** BTREE on segment_id (PRIMARY lookup for routing), BTREE on cycle_id, partial index WHERE preclosure_xemay = TRUE (fast pre-closure feed query)

**Retention:** Only the latest cycle's data is used operationally. Previous cycles archived to TimescaleDB (see depth_history).

**Critical note:** The Valhalla routing graph is re-weighted from this table. `current_depth_m` determines flood cost per edge.

---

### 2.2 district_flood_summary

Pre-computed district-level aggregates (avoids query-time spatial joins).

```
Table: postgis.district_flood_summary
```

| Column | Type | Description |
|--------|------|-------------|
| district_id | INTEGER FK → districts | |
| cycle_id | INTEGER FK → pipeline_cycles | |
| total_segments | INTEGER | Total road segments in district |
| impassable_xemay | INTEGER | Count where current_depth ≥ 0.20m |
| impassable_sedan | INTEGER | Count where current_depth ≥ 0.40m |
| impassable_suv | INTEGER | Count where current_depth ≥ 0.55m |
| impassable_truck | INTEGER | Count where current_depth ≥ 0.70m |
| preclosure_xemay | INTEGER | Count of segments in pre-closure (xe máy) |
| preclosure_sedan | INTEGER | Count in pre-closure (sedan) |
| inaccessible_hospitals | INTEGER | Hospitals with current_depth ≥ 0.20m |
| inaccessible_schools | INTEGER | Schools with current_depth ≥ 0.20m |
| preclosing_institutions | INTEGER | Institutions with pre-closure within 1h |
| dominant_severity | TEXT | none / watch / warning / danger |
| updated_at | TIMESTAMPTZ | |

---

### 2.3 pipeline_cycles

Metadata for each simulation cycle.

```
Table: postgis.pipeline_cycles
```

| Column | Type | Description |
|--------|------|-------------|
| cycle_id | SERIAL PRIMARY KEY | |
| cycle_type | TEXT NOT NULL | 'constant' or 'ondemand_24h' |
| trigger_type | TEXT | 'forecast_threshold' / 'operator_manual' / 'synoptic_auto' |
| cycle_start | TIMESTAMPTZ NOT NULL | Wall-clock start of this cycle |
| cycle_complete | TIMESTAMPTZ | Wall-clock completion (NULL if in progress) |
| duration_seconds | DOUBLE PRECISION | Total wall-clock duration |
| sla_met | BOOLEAN | TRUE if duration ≤ 1800s (30 min) |
| models_activated | INTEGER[] | Array of regional_model_ids that ran this cycle |
| coupling_approach | TEXT | 'A_independent' or 'B_sequential' |
| rainfall_source | TEXT | 'nchmf' / 'gfs' / 'ecmwf' / 'synthetic' |
| sim_start_datetime | TIMESTAMPTZ | Simulation T+0 datetime |
| sim_duration_s | INTEGER | 7200 (2h) or 86400 (24h) |
| output_timestep_s | INTEGER | 900 (15 min) or 3600 (60 min) |
| geotiff_count | INTEGER | Number of depth GeoTIFFs produced |
| scenario_library_match_id | INTEGER FK → scenario_library_index | Which scenario was served as Layer 1 |
| status | TEXT | 'running' / 'completed' / 'failed' / 'timeout' |
| error_message | TEXT | NULL if successful |

---

### 2.4 model_cycle_detail

Per-model timing detail within a pipeline cycle.

```
Table: postgis.model_cycle_detail
```

| Column | Type | Description |
|--------|------|-------------|
| id | SERIAL PRIMARY KEY | |
| cycle_id | INTEGER FK → pipeline_cycles | |
| model_id | INTEGER FK → regional_models | |
| compute_node | TEXT | 'A1' or 'A2' |
| cores_used | INTEGER | |
| ingestion_start | TIMESTAMPTZ | |
| ingestion_complete | TIMESTAMPTZ | |
| bc_upload_start | TIMESTAMPTZ | |
| bc_upload_complete | TIMESTAMPTZ | |
| sim_start | TIMESTAMPTZ | |
| sim_complete | TIMESTAMPTZ | |
| sim_duration_seconds | DOUBLE PRECISION | |
| download_start | TIMESTAMPTZ | |
| download_complete | TIMESTAMPTZ | |
| depth_calc_start | TIMESTAMPTZ | |
| depth_calc_complete | TIMESTAMPTZ | |
| threedi_simulation_id | TEXT | 3Di API simulation UUID |
| api_calls_total | INTEGER | Total 3Di API calls for this model |
| api_calls_failed | INTEGER | Failed API calls |
| status | TEXT | 'completed' / 'failed' / 'timeout' |
| error_message | TEXT | |

---

## 3. TIME-SERIES DATA (TimescaleDB hypertables)

### 3.1 institution_depth_forecast

Institution flood depth histogram — the "Google popular times" equivalent.

```
Hypertable: timescaledb.institution_depth_forecast
Time column: forecast_valid_time
```

| Column | Type | Description |
|--------|------|-------------|
| institution_id | INTEGER FK → institutions | |
| forecast_valid_time | TIMESTAMPTZ NOT NULL | The wall-clock hour this bar represents |
| depth_m | DOUBLE PRECISION | Predicted depth at institution location (metres) |
| data_source | TEXT | 'realtime' (bars 1–4) or 'scenario_library' (bars 5–24) or 'ondemand_24h' |
| cycle_id | INTEGER FK → pipeline_cycles | Which cycle produced this bar |
| created_at | TIMESTAMPTZ DEFAULT NOW() | |

**Chunking:** By forecast_valid_time, 1-day chunks

**Retention:** Keep 30 days of history for post-event analysis, then compress to weekly summaries

---

### 3.2 depth_history

Historical depth per segment per cycle — for trend analysis, model calibration, and ML training.

```
Hypertable: timescaledb.depth_history
Time column: measured_at
```

| Column | Type | Description |
|--------|------|-------------|
| segment_id | INTEGER | |
| measured_at | TIMESTAMPTZ NOT NULL | Pipeline completion time |
| current_depth_m | DOUBLE PRECISION | |
| peak_depth_m | DOUBLE PRECISION | |
| data_source | TEXT | 'realtime' / 'scenario_library' |
| cycle_id | INTEGER | |

**Chunking:** By measured_at, 7-day chunks

**Retention:** Keep 5 years (the full system lifetime). Compressed after 90 days.

**Note:** This is a write-heavy table. ~50,000 segments × every cycle during active flood events. TimescaleDB compression is essential.

---

### 3.3 gauge_readings

Historical gauge water level / discharge readings.

```
Hypertable: timescaledb.gauge_readings
Time column: reading_time
```

| Column | Type | Description |
|--------|------|-------------|
| station_id | INTEGER FK → gauge_stations | |
| reading_time | TIMESTAMPTZ NOT NULL | |
| value | DOUBLE PRECISION | Water level (m) or discharge (m³/s) |
| reading_type | TEXT | 'water_level' / 'discharge' / 'rainfall_mm' |
| quality_flag | TEXT | 'valid' / 'suspect' / 'missing' |
| source | TEXT | 'api_poll' / 'manual_entry' / 'interpolated' |

**Chunking:** 7-day chunks

**Retention:** 5 years (essential for model calibration)

---

### 3.4 pipeline_metrics

Prometheus-compatible pipeline performance metrics.

```
Hypertable: timescaledb.pipeline_metrics
Time column: recorded_at
```

| Column | Type | Description |
|--------|------|-------------|
| metric_name | TEXT | e.g., 'cycle_duration_seconds', 'sim_duration_seconds', 'api_call_count' |
| model_id | INTEGER | Nullable — NULL for system-wide metrics |
| recorded_at | TIMESTAMPTZ NOT NULL | |
| value | DOUBLE PRECISION | |
| labels | JSONB | Additional key-value labels for Prometheus export |

---

## 4. ALERT DATA

### 4.1 flood_alerts

Active and historical flood alerts.

```
Table: postgis.flood_alerts
```

| Column | Type | Description |
|--------|------|-------------|
| alert_id | SERIAL PRIMARY KEY | |
| alert_level | TEXT NOT NULL | 'danger' / 'warning' / 'watch' |
| alert_source | TEXT NOT NULL | 'user_alert_engine' / 'segment_alert_engine' / 'forecast_model' |
| target_type | TEXT | 'user_location' / 'segment' / 'area' |
| target_id | TEXT | user_id, segment_id, or district_id depending on target_type |
| location | GEOMETRY(Point, 4326) | Alert location |
| current_depth_m | DOUBLE PRECISION | Depth at alert location at time of alert |
| forecast_depth_m | DOUBLE PRECISION | Forecast depth triggering this alert |
| forecast_horizon | TEXT | 'current' / 't1h' / 't2h' / 't24h' |
| vehicle_type_code | TEXT | The vehicle type threshold that was exceeded |
| message_vi | TEXT | Vietnamese alert message text |
| cycle_id | INTEGER FK → pipeline_cycles | |
| created_at | TIMESTAMPTZ NOT NULL | |
| expires_at | TIMESTAMPTZ | Alert expiry (current cycle + staleness buffer) |
| delivered_via | TEXT | 'fcm' / 'apns' / 'api_poll' |
| delivery_status | TEXT | 'pending' / 'delivered' / 'failed' |
| fcm_message_id | TEXT | FCM delivery receipt ID |

**Indexes:** BTREE on alert_level, BTREE on target_id, BTREE on expires_at (for cleanup)

---

## 5. USER DATA

### 5.1 users

Mobile app user accounts.

```
Table: postgis.users
```

| Column | Type | Description |
|--------|------|-------------|
| user_id | UUID PRIMARY KEY DEFAULT gen_random_uuid() | |
| phone_hash | TEXT | Hashed phone number (for dedup, not stored plaintext) |
| display_name | TEXT | Optional user display name |
| default_vehicle_type | TEXT DEFAULT 'xe_may' FK → vehicle_types.code | |
| fcm_token | TEXT | Firebase Cloud Messaging token for push |
| apns_token | TEXT | Apple Push token |
| location_consent | TEXT DEFAULT 'app_open' | 'always' / 'app_open' / 'never' |
| last_known_lat | DOUBLE PRECISION | |
| last_known_lng | DOUBLE PRECISION | |
| last_known_location | GEOMETRY(Point, 4326) | PostGIS point from lat/lng |
| last_location_update | TIMESTAMPTZ | |
| created_at | TIMESTAMPTZ DEFAULT NOW() | |
| updated_at | TIMESTAMPTZ | |

**Privacy:** Location data not stored beyond session unless location_consent = 'always'. Nightly cleanup job deletes stale locations for 'app_open' users who haven't opened app in 24h.

---

### 5.2 saved_places

User's saved locations (home, work, school, etc.).

```
Table: postgis.saved_places
```

| Column | Type | Description |
|--------|------|-------------|
| place_id | SERIAL PRIMARY KEY | |
| user_id | UUID FK → users | |
| name | TEXT NOT NULL | "Nhà", "Cơ quan", "Trường con" etc. |
| location | GEOMETRY(Point, 4326) | |
| location_vn2000 | GEOMETRY(Point, 3405) | For GeoTIFF depth sampling |
| address_vi | TEXT | |
| place_type | TEXT | 'home' / 'work' / 'school' / 'custom' |
| notify_on_flood | BOOLEAN DEFAULT TRUE | Send push when this location is forecast to flood |
| created_at | TIMESTAMPTZ | |

---

## 6. COMMUNITY DATA

### 6.1 community_reports

Citizen-submitted flood reports. **Informational only — does NOT feed routing or official depth.**

```
Table: postgis.community_reports
```

| Column | Type | Description |
|--------|------|-------------|
| report_id | SERIAL PRIMARY KEY | |
| user_id | UUID FK → users | |
| location | GEOMETRY(Point, 4326) | GPS location at time of report |
| depth_estimate_cm | INTEGER | User-estimated depth |
| passability_vote | TEXT | 'can_pass' / 'difficult' / 'cannot_pass' |
| vehicle_type_code | TEXT FK → vehicle_types.code | Vehicle type for passability vote |
| photo_url | TEXT | S3/storage URL of uploaded photo |
| photo_moderated | BOOLEAN DEFAULT FALSE | Must be TRUE before public display |
| moderation_status | TEXT | 'pending' / 'approved' / 'rejected' |
| created_at | TIMESTAMPTZ NOT NULL | |
| expires_at | TIMESTAMPTZ | created_at + 2 hours (unless confirmed by subsequent reports) |
| confirmed_by_count | INTEGER DEFAULT 0 | Number of subsequent reports within 100m confirming this |

**Indexes:** GIST on location (spatial queries), BTREE on expires_at (cleanup)

---

## 7. GOVERNMENT / DASHBOARD DATA

### 7.1 dispatch_log

City council dispatch actions during flood events.

```
Table: postgis.dispatch_log
```

| Column | Type | Description |
|--------|------|-------------|
| dispatch_id | SERIAL PRIMARY KEY | |
| segment_id | INTEGER FK → road_segments | Nullable (can target institution instead) |
| institution_id | INTEGER FK → institutions | Nullable |
| action_type | TEXT NOT NULL | 'traffic_coordination' / 'emergency_access' / 'evacuation_corridor' / 'heavy_vehicle_diversion' |
| team_identifier | TEXT NOT NULL | e.g., "Đội CSGT Hoàn Kiếm #3" |
| operator_name | TEXT NOT NULL | Dashboard operator who dispatched |
| notes | TEXT | Free-text operational notes |
| segment_depth_at_dispatch | DOUBLE PRECISION | Depth when dispatch was made |
| dispatched_at | TIMESTAMPTZ NOT NULL | |
| resolved_at | TIMESTAMPTZ | When team reported completion |
| status | TEXT DEFAULT 'active' | 'active' / 'resolved' / 'cancelled' |
| cycle_id | INTEGER FK → pipeline_cycles | Pipeline cycle at time of dispatch |

---

### 7.2 public_announcements

Draft public announcements generated from institution access data.

```
Table: postgis.public_announcements
```

| Column | Type | Description |
|--------|------|-------------|
| announcement_id | SERIAL PRIMARY KEY | |
| institution_id | INTEGER FK → institutions | |
| draft_text_vi | TEXT NOT NULL | Auto-generated Vietnamese draft |
| current_status | TEXT | 'accessible' / 'difficult' / 'inaccessible' |
| estimated_resolution | TIMESTAMPTZ | |
| operator_name | TEXT | Who reviewed/edited |
| approved | BOOLEAN DEFAULT FALSE | Operator approval before publishing |
| published_channels | TEXT[] | Array: {'zalo_oa', 'city_website', 'broadcast'} |
| created_at | TIMESTAMPTZ | |
| published_at | TIMESTAMPTZ | |

---

## 8. FILE STORAGE (Filesystem, not database)

### 8.1 Simulation outputs (ephemeral — scratch storage on compute nodes)

| Path Pattern | Description | Lifecycle |
|---|---|---|
| `/scratch/cycle_{id}/model_{id}/results_3di.nc` | Raw 3Di simulation output | Deleted after depth calculation |
| `/scratch/cycle_{id}/model_{id}/depth_*.tif` | Per-timestep depth GeoTIFFs | Moved to persistent after postprocessing |
| `/scratch/cycle_{id}/model_{id}/rain.nc` | Rainfall input NetCDF | Deleted after simulation |
| `/scratch/cycle_{id}/model_{id}/initial_waterlevels.csv` | Gauge-derived initial conditions | Deleted after simulation |
| `/scratch/cycle_{id}/model_{id}/boundary_conditions_1d.csv` | 1D BCs (river crossings) | Deleted after simulation |
| `/scratch/cycle_{id}/model_{id}/boundary_conditions_2d.csv` | 2D BCs (surface edges) | Deleted after simulation |
| `/scratch/cycle_{id}/model_{id}/laterals_1d.csv` | 1D laterals (channel inflows) | Deleted after simulation |

### 8.2 Persistent outputs (Node B storage)

| Path Pattern | Description | Lifecycle |
|---|---|---|
| `/data/tiles/cycle_{id}/{z}/{x}/{y}.png` | XYZ flood depth tiles | Pushed to CDN, kept 30 days |
| `/data/geotiffs/cycle_{id}/merged_depth_*.tif` | City-wide merged depth GeoTIFFs | Kept 90 days, archived to NAS |
| `/data/models/{model_id}/gridadmin.h5` | 3Di model file (versioned) | Permanent |
| `/data/dtm/{model_id}/dtm_1m.tif` | DTM rasters per model | Permanent (updated annually) |

### 8.3 Scenario library (Node B + NAS)

| Path Pattern | Description | Size Estimate |
|---|---|---|
| `/data/scenario_library/{model_id}/{scenario_id}/depth_*.tif` | Pre-computed depth GeoTIFFs per scenario | ~500 MB per scenario |
| `/data/scenario_library/{model_id}/{scenario_id}/metadata.json` | Scenario parameters + summary stats | ~1 KB |
| Total Year 1 | 5 models × ~360 scenarios × ~500 MB | ~900 GB |
| Total Year 5 | Growth + higher resolution | ~2–3 TB |

### 8.4 Archive (NAS)

| Path Pattern | Description | Retention |
|---|---|---|
| `/archive/cycles/{year}/{month}/cycle_{id}/` | Complete cycle outputs for post-event analysis | 5 years |
| `/archive/gauge/{year}/` | Historical gauge readings export | 5 years |
| `/archive/community_photos/{year}/{month}/` | Approved community flood photos | 5 years |
| `/archive/validation/{event_id}/` | Sentinel-2 imagery + crowd-sourced data for model validation | Permanent |

---

## 9. REDIS DATA STRUCTURES

### 9.1 Pipeline event channels

| Key / Channel | Type | Description |
|---|---|---|
| `pipeline:cycle:{cycle_id}:status` | STRING | Current cycle status JSON |
| `pipeline:model:{model_id}:stage` | STRING | Current stage: 'ingestion' / 'simulating' / 'postprocessing' |
| `pipeline:metric_extraction_complete` | PUB/SUB channel | Tier 0 → Tier 1 fan-out signal |
| `pipeline:segment_extraction_complete` | PUB/SUB channel | Tier 1 → Tier 2 gate signal |
| `pipeline:last_completed` | HASH | {constant_cycle: timestamp, ondemand_24h: timestamp} |

### 9.2 Cache

| Key Pattern | Type | TTL | Description |
|---|---|---|---|
| `cache:segment:{segment_id}:depth` | STRING (JSON) | 1800s (30 min) | Cached segment depth for API fast-path |
| `cache:district:{district_id}:summary` | STRING (JSON) | 1800s | Cached district summary |
| `cache:institution:{id}:histogram` | STRING (JSON) | 1800s | Cached 24-bar histogram |
| `rate_limit:user:{user_id}` | STRING (counter) | 60s | API rate limiting per user |

---

## 10. DATABASE SIZING ESTIMATES

### 10.1 PostGIS

| Table | Row Count (Year 1) | Row Size | Total | Growth |
|-------|---------------------|----------|-------|--------|
| road_segments | ~50,000 | ~500 bytes + geom | ~100 MB | +10%/year (OSM updates) |
| segment_flood_state | ~50,000 (single cycle active) | ~300 bytes | ~15 MB (active set) | Stable |
| districts | 30 | ~1 KB + geom | <1 MB | Stable |
| institutions | ~2,000 | ~500 bytes | ~1 MB | +20%/year |
| community_reports | ~10,000/monsoon season | ~500 bytes + photo ref | ~5 MB/year | Growing |
| dispatch_log | ~500/monsoon season | ~300 bytes | <1 MB/year | Growing |
| scenario_library_index | ~1,800 (5 × 360) | ~200 bytes | <1 MB | Stable |
| **PostGIS total (operational)** | | | **~200 MB** | |

### 10.2 TimescaleDB

| Hypertable | Write Rate | Row Size | Annual Volume | 5-Year |
|---|---|---|---|---|
| depth_history | 50,000 rows/cycle × ~200 cycles/monsoon season | ~50 bytes | ~500 MB/year | ~2.5 GB |
| institution_depth_forecast | 2,000 × 24 bars × ~200 cycles/season | ~50 bytes | ~500 MB/year | ~2.5 GB |
| gauge_readings | ~20 stations × 96 readings/day × 365 days | ~50 bytes | ~35 MB/year | ~175 MB |
| pipeline_metrics | ~100 metrics/cycle × ~200 cycles/season | ~100 bytes | ~2 MB/year | ~10 MB |
| **TimescaleDB total** | | | **~1 GB/year** | **~5 GB** |

### 10.3 Redis

| Data | Size |
|---|---|
| Active cache (all keys) | ~50–100 MB |
| PUB/SUB overhead | Negligible |
| **Redis total** | **~128 MB allocation sufficient** |

### 10.4 Total Database Footprint

| Component | Year 1 | Year 5 |
|-----------|--------|--------|
| PostGIS | ~200 MB | ~500 MB |
| TimescaleDB | ~1 GB | ~5 GB |
| Redis | ~128 MB | ~128 MB |
| Scenario library (disk) | ~900 GB | ~3 TB |
| GeoTIFF archive (NAS) | ~500 GB | ~3 TB |
| **Total (all storage)** | **~1.5 TB** | **~6.5 TB** |

Well within the Node B storage capacity (15 TB usable RAID 10) + NAS (80 TB).

---

*End of Document — FloodNav Hanoi Data Schema v1.0*
