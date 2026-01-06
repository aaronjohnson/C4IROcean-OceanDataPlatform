---
name: odp-dataset-patterns
description: Common patterns for working with ODP datasets (tabular vs file-based, error handling)
allowed-tools: Read, Write, Edit, NotebookEdit
---

## ODP Dataset Patterns

### Detect Dataset Type

```python
from odp.client import Client

client = Client()
dataset = client.dataset(DATASET_ID)

# Check if tabular
schema = dataset.table.schema()
is_tabular = schema is not None

if is_tabular:
    stats = dataset.table.stats()
    print(f"Tabular: {len(schema)} columns, {stats.num_rows:,} rows")
else:
    files = dataset.files.list()
    print(f"File-based: {len(files)} files")
```

### Branch Logic by Type

```python
if is_tabular:
    # Tabular operations
    df = dataset.table.select().all(max_rows=1000).dataframe()
    schema = dataset.table.schema()
    stats = dataset.table.stats()
else:
    # File operations
    files = dataset.files.list()
    for chunk in dataset.files.download(file_id):
        # process chunk
```

### Known Working Datasets

| Dataset | UUID | Type | Description |
|---------|------|------|-------------|
| PGS Biota | `1d801817-742b-4867-82cf-5597673524eb` | TABULAR | Marine mammal/turtle observations |
| GLODAP | `15dac249-4e3d-474b-a246-ba95cffc8807` | FILE | Global ocean chemistry |
| GEBCO | `5070af58-6d8a-4636-a6a0-8ca9298fb3ab` | FILE | Bathymetry data |

### Auto-Install Dependencies

```python
import subprocess
import sys

def ensure_package(package_name, import_name=None):
    """Install package if not available."""
    import_name = import_name or package_name
    try:
        __import__(import_name)
        return True
    except ImportError:
        print(f"Installing {package_name}...")
        subprocess.check_call([sys.executable, "-m", "pip", "install", "-q", package_name])
        return True

# Usage
ensure_package("folium")
ensure_package("h3")
```

### Safe Dataset Probe

```python
def get_dataset_info(dataset_id):
    """Safely probe dataset type and info."""
    try:
        ds = client.dataset(dataset_id)
        schema = ds.table.schema()

        if schema:
            stats = ds.table.stats()
            return {
                'id': dataset_id,
                'type': 'TABULAR',
                'columns': [f.name for f in schema],
                'rows': stats.num_rows if stats else 0
            }
        else:
            files = ds.files.list()
            return {
                'id': dataset_id,
                'type': 'FILE',
                'file_count': len(files) if files else 0
            }
    except Exception as e:
        return {'id': dataset_id, 'error': str(e)}
```

### File Upload and Ingest Pattern

```python
import io

# Upload CSV
csv_buffer = io.BytesIO()
df.to_csv(csv_buffer, index=False)
file_id = dataset.files.upload("data.csv", csv_buffer.getvalue())

# Update metadata with geometry
bbox_wkt = f"POLYGON(({min_lon} {min_lat}, {max_lon} {min_lat}, {max_lon} {max_lat}, {min_lon} {max_lat}, {min_lon} {min_lat}))"
dataset.files.update_meta(file_id, {
    "name": "data.csv",
    "mime-type": "text/csv",
    "geometry": bbox_wkt
})

# Ingest to table
dataset.files.ingest(file_id, opt="drop")  # or "append" or "truncate"
```

### Geospatial Query Pattern

```python
# WKT polygon filter
region = "POLYGON((lon1 lat1, lon2 lat1, lon2 lat2, lon1 lat2, lon1 lat1))"

df = dataset.table.select(
    f"{geometry_col} within $area",
    vars={"area": region}
).all(max_rows=10000).dataframe()
```

### H3 Aggregation Pattern

```python
# Server-side H3 aggregation
h3_agg = dataset.table.aggregate(
    group_by=f"h3({geometry_col}, {resolution})",
    filter=f"{geometry_col} IS NOT NULL",
    aggr={"*": "count"}
)

# Client-side fallback
import h3
df['h3_index'] = df.apply(
    lambda row: h3.latlng_to_cell(row['lat'], row['lon'], resolution),
    axis=1
)
```
