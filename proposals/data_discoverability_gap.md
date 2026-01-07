# Proposal: Data Discoverability Gap

**Status:** Discussion
**Author:** Aaron Johnson
**Date:** 2026-01-06
**Related:** `tutorials/01_catalog_discovery.ipynb`, `tutorials/02_geospatial_analysis.ipynb`

## Summary

The STAC catalog does not reflect actual data accessibility in ODP. Users discover datasets via STAC but cannot access the data - meanwhile, accessible TABULAR datasets exist but are not discoverable via STAC or SDK.

## Key Findings

### Finding 1: STAC Shows FILE, SDK Returns Empty

All 35 STAC collections appear as FILE type via SDK, but with 0 accessible files:

```python
ds = client.dataset(stac_collection_id)
schema = ds.table.schema()  # None
files = ds.files.list()     # []
```

**Affected datasets:**
- GLODAP (`15dac249-4e3d-474b-a246-ba95cffc8807`)
- GEBCO Bathymetry (`5070af58-6d8a-4636-a6a0-8ca9298fb3ab`)
- AkerBP Metocean, and 32 others

### Finding 2: TABULAR Exists But Not Discoverable

Probed all 35 STAC collections - none provide TABULAR access. However, a TABULAR dataset exists that works via SDK:

| Metric | Result |
|--------|--------|
| STAC collections | 35 |
| TABULAR via STAC UUIDs | 0 |
| FILE via STAC UUIDs | 35 (all 0 files) |
| Known TABULAR UUID | 1 confirmed (2,241 rows) |

```python
# This UUID is NOT in STAC (returns 404)
# But works via SDK as TABULAR
ds = client.dataset("1d801817-742b-4867-82cf-5597673524eb")
ds.table.schema()       # Returns schema!
ds.table.stats()        # 2,241 rows
```

**How was this UUID found?** Unknown - it was pre-existing in tutorial code. Possible sources:
- Web UI catalog (app.hubocean.earth/catalog)
- Direct communication with ODP team
- Previous documentation

**Are there more?** Unknown - no programmatic way to discover. If PGS Biota has a TABULAR UUID, other datasets might too.

### Finding 3: Dual UUIDs for Same Data

Same dataset can have two UUIDs with different access patterns:

| UUID | STAC | SDK Type | Access |
|------|------|----------|--------|
| `1d801817-742b-4867-82cf-5597673524eb` | 404 | TABULAR | 2,241 rows ✓ |
| `b960c80e-7ead-47af-b6c8-e92a9b5ac659` | Yes | FILE | 0 files |

Both refer to PGS Brazil Biota data.

## The Gap

```
What STAC Shows          What's Actually Accessible
─────────────────        ─────────────────────────
35 FILE datasets    →    0 accessible files
0 TABULAR datasets  →    1+ TABULAR with data (hidden UUIDs)
```

**User Experience:**
1. Browse STAC catalog → finds 35 datasets
2. Try to access data → all return empty
3. Conclude ODP has no usable public data
4. Miss hidden TABULAR datasets entirely

## Impact

1. **New users blocked** - Cannot discover any queryable data
2. **Tutorials fail** - Dynamic discovery returns nothing usable
3. **Value hidden** - ODP's tabular query capabilities undiscoverable
4. **Documentation fragile** - Must hardcode UUIDs that aren't in STAC

## Questions for Discussion

1. **Why are FILE datasets empty?** Is this permissions, storage backend, or access pattern?

2. **Why aren't TABULAR datasets in STAC?** Is this intentional (private data) or a gap in catalog population?

3. **How many TABULAR datasets exist?** We confirmed one, but there may be more that users can't discover.

4. **What's the relationship between UUIDs?** Can users discover TABULAR UUID from FILE UUID? Is there an internal link?

5. **Is there an API we're missing?** Perhaps `client.list_datasets(type="tabular")`? The web UI catalog seems to show more.

6. **Should STAC reflect accessibility?** Either include TABULAR UUIDs or indicate FILE access requirements.

7. **How was the known UUID originally discovered?** Understanding this helps users find more.

## Proposed Solutions

### Option A: Expose TABULAR in STAC

Add TABULAR datasets to catalog:

```json
{
  "id": "pgs-biota-tabular",
  "title": "PGS Brazil Biota (Tabular)",
  "properties": {
    "odp:access_type": "tabular",
    "odp:row_count": 2241
  }
}
```

### Option B: SDK Discovery Method

```python
# Discover all accessible datasets by type
tabular = client.list_datasets(type="tabular")
files = client.list_datasets(type="file", accessible=True)
```

### Option C: Unified Dataset View

```python
ds = client.dataset(any_uuid)
print(ds.access_types)      # ['tabular', 'file']
print(ds.tabular_uuid)      # For tabular access
print(ds.file_uuid)         # For file access
print(ds.is_accessible)     # True/False
```

### Option D: STAC Access Metadata

Add accessibility info to STAC:

```json
{
  "properties": {
    "odp:tabular_accessible": true,
    "odp:tabular_uuid": "1d801817-...",
    "odp:file_accessible": false,
    "odp:access_requirements": "Contact data owner"
  }
}
```

### Option E: Documentation

If current behavior is intentional:
- Document which UUIDs provide TABULAR access
- Explain FILE vs TABULAR UUID relationship
- Clarify access requirements for each dataset

## SDK Investigation

The SDK was explored for discovery methods:

```python
# SDK client has minimal methods
dir(client)  # → ['base_url', 'dataset', 'request']

# Only method is dataset(id) - requires knowing UUID
client.dataset(id: str) -> Dataset

# Tried probing API endpoints via request()
endpoints = ['/catalog', '/datasets', '/data/catalogue', '/']
# All return 404
```

**SDK Documentation confirms this is by design:**

> "Normally you will use the web UI [here](https://app.hubocean.earth/catalog) to find the dataset you want to work with, and then likely use the dataset ID to access it in the SDK"

Source: https://docs.hubocean.earth/python_sdk/intro/

**Conclusion:** No programmatic discovery via SDK (by design). Users must:
1. Use web UI (https://app.hubocean.earth/catalog) - **recommended by docs**
2. Use STAC API (but TABULAR not included, FILE shows 0 files)
3. Know UUID in advance

**Gap:** The docs point to web UI, but many users expect STAC to be the discovery mechanism. STAC doesn't reflect what's actually accessible.

## Known UUIDs to Try

These UUIDs can be used to reproduce findings:

### Working TABULAR (not in STAC)
| UUID | Dataset | Rows |
|------|---------|------|
| `1d801817-742b-4867-82cf-5597673524eb` | PGS Brazil Biota | 2,241 |

### FILE via STAC (all return 0 files)
| UUID | Dataset |
|------|---------|
| `b960c80e-7ead-47af-b6c8-e92a9b5ac659` | PGS Brazil Biota (FILE view) |
| `15dac249-4e3d-474b-a246-ba95cffc8807` | GLODAP |
| `5070af58-6d8a-4636-a6a0-8ca9298fb3ab` | GEBCO Bathymetry |

```python
# Quick test
from odp.client import Client
client = Client()

# TABULAR access (works)
tabular = client.dataset("1d801817-742b-4867-82cf-5597673524eb")
print(f"TABULAR schema: {tabular.table.schema() is not None}")
print(f"TABULAR rows: {tabular.table.stats().num_rows}")

# FILE access (returns empty)
file_ds = client.dataset("b960c80e-7ead-47af-b6c8-e92a9b5ac659")
print(f"FILE schema: {file_ds.table.schema()}")  # None
print(f"FILE files: {len(file_ds.files.list())}")  # 0
```

## Reproduction Steps

```python
from odp.client import Client
import requests

client = Client()
STAC_URL = "https://api.hubocean.earth/api/stac"

# 1. Get all STAC collections
collections = requests.get(f"{STAC_URL}/collections").json()['collections']
print(f"STAC collections: {len(collections)}")

# 2. Probe each for actual access
tabular, file_with_data, empty = 0, 0, 0
for coll in collections:
    ds = client.dataset(coll['id'])
    if ds.table.schema():
        tabular += 1
    elif ds.files.list():
        file_with_data += 1
    else:
        empty += 1

print(f"TABULAR: {tabular}, FILE w/data: {file_with_data}, Empty: {empty}")
# Result: TABULAR: 0, FILE w/data: 0, Empty: 35

# 3. But hidden TABULAR exists
ds = client.dataset("1d801817-742b-4867-82cf-5597673524eb")
print(f"Hidden TABULAR: {ds.table.stats().num_rows} rows")  # 2241
```

## Related

- `proposals/server_side_h3_aggregation.md` - Server-side aggregation features
- `proposals/my_data_api.md` - Programmatic dataset creation
- `.claude/skills/odp-troubleshooting/` - Workarounds documented

## References

- [ODP Python SDK](https://docs.hubocean.earth/python_sdk/intro/)
- [STAC Specification](https://stacspec.org/)
- [STAC Best Practices](https://github.com/radiantearth/stac-spec/blob/master/best-practices.md)
