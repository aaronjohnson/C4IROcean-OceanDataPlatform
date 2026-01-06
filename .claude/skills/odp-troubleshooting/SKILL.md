---
name: odp-troubleshooting
description: Common ODP issues, error patterns, and solutions
allowed-tools: Read, Grep, WebFetch
---

## ODP Troubleshooting Guide

### Issue: Collection Returns 404

**Symptom:** STAC API returns 404 for a collection ID

**Causes:**
- Collection ID is incorrect or outdated
- Collection was removed or renamed
- Typo in UUID

**Solution:**
```python
# Verify collection exists before using
try:
    collection = stac_get(f"/collections/{collection_id}")
except requests.HTTPError as e:
    if e.response.status_code == 404:
        print(f"Collection not found: {collection_id}")
        # Fall back to discovery
        collections = stac_get("/collections")['collections']
```

### Issue: Dataset Shows as FILE When Expected TABULAR

**Symptom:** `dataset.table.schema()` returns `None` for datasets that should be tabular

**Causes:**
- Dataset genuinely is file-based
- Permissions issue - user may not have tabular access
- Dataset has files but no ingested table yet

**Solution:**
```python
schema = dataset.table.schema()
if schema is None:
    # Check if it has files
    files = dataset.files.list()
    if files:
        print(f"Dataset has {len(files)} files but no table")
        print("May need to ingest files first, or check permissions")
    else:
        print("Dataset is empty")
```

**Known FILE datasets that look tabular:**
- GLODAP (`15dac249-4e3d-474b-a246-ba95cffc8807`) - shows as FILE
- GEBCO (`5070af58-6d8a-4636-a6a0-8ca9298fb3ab`) - shows as FILE

**Known working TABULAR datasets:**
- PGS Biota (`1d801817-742b-4867-82cf-5597673524eb`) - 2,241 rows

### Issue: Import Errors for Optional Packages

**Symptom:** `ModuleNotFoundError: No module named 'folium'`

**Solution:** Auto-install pattern
```python
import subprocess
import sys

def ensure_package(package_name, import_name=None):
    import_name = import_name or package_name
    try:
        __import__(import_name)
        return True
    except ImportError:
        print(f"Installing {package_name}...")
        subprocess.check_call([sys.executable, "-m", "pip", "install", "-q", package_name])
        return True

ensure_package("folium")
ensure_package("h3")
```

### Issue: h3 Library Version Mismatch (v3 vs v4 API)

**Symptom:** `AttributeError: module 'h3' has no attribute 'latlng_to_cell'`

**Cause:** ODP Workspace has h3 v3.x pre-installed. Even after `pip install h3`, the old version may persist because:
- Pip doesn't upgrade if any version exists
- Python caches the already-imported module
- Need kernel restart to pick up new version

**Solution 1:** Version-agnostic wrapper functions
```python
def get_h3_cell(lat, lon, res):
    try:
        return h3.latlng_to_cell(lat, lon, res)  # v4+ API
    except AttributeError:
        return h3.geo_to_h3(lat, lon, res)  # v3 API

def get_h3_boundary(hex_id):
    try:
        return h3.cell_to_boundary(hex_id)  # v4+ API
    except AttributeError:
        return h3.h3_to_geo_boundary(hex_id, geo_json=False)  # v3 API
```

**Solution 2:** Force upgrade and restart kernel
```python
!pip install --upgrade --force-reinstall h3
# Then: Kernel → Restart Kernel
```

**API differences:**
| v3 API | v4 API |
|--------|--------|
| `h3.geo_to_h3(lat, lon, res)` | `h3.latlng_to_cell(lat, lon, res)` |
| `h3.h3_to_geo_boundary(hex)` | `h3.cell_to_boundary(hex)` |
| `h3.h3_to_parent(hex, res)` | `h3.cell_to_parent(hex, res)` |

### Issue: H3 Server-Side Aggregation Fails

**Symptom:** `aggregate()` with H3 group_by raises error

**Solution:** Fall back to client-side
```python
try:
    h3_agg = dataset.table.aggregate(
        group_by=f"h3({geometry_col}, {resolution})",
        aggr={"*": "count"}
    )
except Exception as e:
    print(f"Server-side H3 failed: {e}")
    print("Using client-side aggregation...")

    import h3
    df['h3_index'] = df.apply(
        lambda row: h3.latlng_to_cell(row['lat'], row['lon'], resolution)
        if pd.notna(row['lat']) else None,
        axis=1
    )
    h3_agg = df.groupby('h3_index').size().reset_index(name='count')
```

### Issue: Query Returns Empty Results

**Symptom:** `select()` returns 0 rows

**Causes:**
- Filter syntax error
- No data matches filter
- Wrong column names

**Solution:**
```python
# First, check schema for correct column names
schema = dataset.table.schema()
print("Columns:", [f.name for f in schema])

# Try without filter first
df_all = dataset.table.select().all(max_rows=10).dataframe()
print(f"Total rows available: {len(df_all)}")

# Then add filter
df_filtered = dataset.table.select(
    "column_name = $value",
    vars={"value": "test"}
).all().dataframe()
```

### Issue: File Upload Succeeds but Ingest Fails

**Symptom:** `files.upload()` works but `files.ingest()` errors

**Causes:**
- CSV format issues (encoding, delimiters)
- Schema mismatch with existing table
- Invalid data types

**Solution:**
```python
# Use drop mode for fresh table
dataset.files.ingest(file_id, opt="drop")

# If appending, ensure schema matches
existing_schema = dataset.table.schema()
if existing_schema:
    print("Existing columns:", [f.name for f in existing_schema])
    # Verify your CSV has matching columns
```

### Issue: Geometry/WKT Errors

**Symptom:** Spatial queries fail with geometry errors

**Solution:** Validate WKT format
```python
# Correct WKT polygon format (note: lon lat order, closed ring)
bbox_wkt = "POLYGON((lon1 lat1, lon2 lat1, lon2 lat2, lon1 lat2, lon1 lat1))"

# Common mistakes:
# - lat/lon reversed (should be lon lat)
# - ring not closed (first point != last point)
# - invalid coordinates
```

### Issue: Authentication Errors

**Symptom:** 401/403 errors

**Solution:**
```python
# In ODP Workspace: should auto-authenticate
client = Client()

# Outside workspace: need API key
# Set ODP_API_KEY environment variable or pass to Client
import os
os.environ['ODP_API_KEY'] = 'your-api-key'
client = Client()
```

### Issue: Rate Limiting

**Symptom:** 429 errors or slow responses

**Solution:**
```python
import time

def safe_request(func, *args, retries=3, **kwargs):
    for attempt in range(retries):
        try:
            return func(*args, **kwargs)
        except Exception as e:
            if '429' in str(e) and attempt < retries - 1:
                time.sleep(2 ** attempt)  # Exponential backoff
            else:
                raise
```

### Debugging Tips

1. **Check dataset exists:** `client.dataset(id)` - will error if not found
2. **Check permissions:** Some datasets are access-controlled
3. **Check type first:** Always probe with `schema()` before assuming tabular
4. **Use max_rows:** Limit queries during development: `.all(max_rows=100)`
5. **Print intermediate results:** Verify data at each step
