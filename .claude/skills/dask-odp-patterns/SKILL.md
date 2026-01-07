---
name: dask-odp-patterns
description: Dask distributed processing patterns for ODP cloud-native workflows
allowed-tools: Read, Write, Grep, WebFetch
---

## Dask + ODP Integration Patterns

Best practices for distributed processing of ODP datasets using Dask in cloud-native ODP Workspaces.

### 1. Dask Client Setup (Cloud-Native)

```python
import dask
import dask.dataframe as dd
from dask.distributed import Client
import os

# Clean up existing client to prevent memory bloat
try:
    client.close()
except:
    pass

# Auto-select port to avoid conflicts
client = Client(dashboard_address=':0')

# Cloud-friendly dashboard URL (jupyter-server-proxy)
jupyter_prefix = os.environ.get('JUPYTERHUB_SERVICE_PREFIX', '/')
dashboard_port = client.scheduler_info().get('services', {}).get('dashboard', 8787)

if 'JUPYTERHUB_SERVICE_PREFIX' in os.environ:
    dashboard_url = f"https://workspace.hubocean.earth{jupyter_prefix}proxy/{dashboard_port}/status"
else:
    dashboard_url = f"http://127.0.0.1:{dashboard_port}/status"

print(f"Dashboard: {dashboard_url}")
```

### 2. Delayed Data Generation (Synthetic)

Generate data directly into Dask partitions - avoids large graph serialization warning:

```python
from dask import delayed
import numpy as np
import pandas as pd

n_rows = 1_000_000
n_partitions = max(4, n_rows // 100_000)
rows_per_partition = n_rows // n_partitions

@delayed
def generate_partition(partition_id, n_rows, start_offset):
    """Generate one partition of synthetic data."""
    np.random.seed(42 + partition_id)

    return pd.DataFrame({
        "timestamp": pd.date_range("2020-01-01", periods=n_rows, freq="min")
                    + pd.Timedelta(minutes=start_offset),
        "station_id": np.random.choice([f"ST-{i:03d}" for i in range(50)], n_rows),
        "value": np.random.normal(0, 1, n_rows),
    })

# Create delayed partitions
partitions = [
    generate_partition(i, rows_per_partition, i * rows_per_partition)
    for i in range(n_partitions)
]

# Build Dask DataFrame - requires meta schema
meta = pd.DataFrame({
    "timestamp": pd.Series(dtype="datetime64[ns]"),
    "station_id": pd.Series(dtype="str"),
    "value": pd.Series(dtype="float64"),
})

ddf = dd.from_delayed(partitions, meta=meta)
```

### 3. ODP + Dask Integration (Current SDK)

Loading ODP data into Dask using the current SDK `.batches()` API:

```python
from odp.client import Client as ODPClient
from dask import delayed

odp = ODPClient()
dataset = odp.dataset("dataset-uuid")

# Collect batches into list first (SDK doesn't expose batch count)
batches = list(dataset.table.select().batches())

@delayed
def load_batch(batch):
    return batch.to_pandas()

# Create delayed objects for each batch
delayed_dfs = [load_batch(batch) for batch in batches]

# Need meta schema from first batch
if batches:
    meta = batches[0].to_pandas().head(0)
    ddf = dd.from_delayed(delayed_dfs, meta=meta)
```

### 4. Ideal ODP + Dask Pattern (Proposed)

If SDK exposed batch count or iterator metadata, we could avoid materializing the batch list:

```python
# IDEAL: SDK provides batch count or lazy iterator
# This would require SDK enhancement - see proposals/sdk_batch_metadata.md

@delayed
def load_batch_by_index(dataset_id, batch_idx, filter_expr=None):
    """Load a specific batch by index - requires SDK support."""
    odp = ODPClient()
    dataset = odp.dataset(dataset_id)
    # Hypothetical API:
    # batch = dataset.table.select(filter_expr).batch(batch_idx)
    # return batch.to_pandas()
    pass

# With batch count known upfront:
# n_batches = dataset.table.select().batch_count()
# partitions = [load_batch_by_index(dataset_id, i) for i in range(n_batches)]
# ddf = dd.from_delayed(partitions, meta=meta)
```

### 5. Configurable Scale Pattern

```python
SCALE = "medium"  # small, medium, large

scales = {
    "small": 100_000,    # ~10 MB, quick demo
    "medium": 1_000_000, # ~100 MB, recommended
    "large": 10_000_000, # ~1 GB, requires 8GB+ RAM
}

n_rows = scales[SCALE]
n_partitions = max(4, n_rows // 100_000)
```

### 6. Head with Delayed DataFrames

Avoid "insufficient elements" warning when using `head()` on delayed DataFrames:

```python
# Wrong - may warn about insufficient elements
ddf.head()

# Correct - look at all partitions to find enough rows
ddf.head(5, npartitions=-1)
```

### 7. Parallel Aggregations

```python
# Group-by aggregation (parallel across partitions)
result = ddf.groupby('station_id').agg({
    'temperature': ['mean', 'min', 'max'],
    'depth': ['mean', 'count']
}).compute()

# Use specific column for count (not "*")
# Wrong: aggr={"*": "count"}
# Correct: aggr={"station_id": "count"}
```

### 8. Map Partitions for Custom Processing

```python
def process_partition(df):
    """Applied to each partition in parallel."""
    result = df.copy()
    result['year'] = pd.to_datetime(result['timestamp']).dt.year
    result['month'] = pd.to_datetime(result['timestamp']).dt.month
    return result

processed_ddf = ddf.map_partitions(process_partition)
```

### 9. Memory-Efficient ODP Streaming

For very large ODP datasets, process batches progressively:

```python
def process_odp_streaming(dataset, processor_fn):
    """Stream ODP batches without loading all into memory."""
    results = []

    for i, batch in enumerate(dataset.table.select().batches()):
        df = batch.to_pandas()
        result = processor_fn(df)
        results.append(result)
        print(f"Processed batch {i+1}: {len(df)} rows")

    return pd.concat(results) if results else pd.DataFrame()
```

### 10. Cleanup

Always close the Dask client when done:

```python
client.close()
```

## Common Issues

| Issue | Solution |
|-------|----------|
| "Sending large graph" warning | Use `dd.from_delayed()` instead of `dd.from_pandas()` |
| "insufficient elements for head" | Use `ddf.head(n, npartitions=-1)` |
| Port 8787 already in use | Use `Client(dashboard_address=':0')` |
| Dashboard URL shows localhost | Use jupyter-server-proxy pattern above |
| Memory bloat from re-runs | Add `client.close()` cleanup at start |

## Related

- `tutorials/drafts/05_dask_distributed_processing.ipynb` - Full tutorial
- `proposals/sdk_batch_metadata.md` - Proposal for SDK batch count API
- [Dask Best Practices](https://docs.dask.org/en/stable/best-practices.html)
