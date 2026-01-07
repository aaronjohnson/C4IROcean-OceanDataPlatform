# Proposal: Data Discoverability Gap

**Status:** Discussion
**Author:** Aaron Johnson
**Date:** 2026-01-06
**Updated:** 2026-01-06 (TABULAR discovered in STAC)
**Related:** `tutorials/01_catalog_discovery.ipynb`, `tutorials/02_geospatial_analysis.ipynb`

## Summary

STAC catalog contains **3,774 items** but returns only 10 per page by default, and doesn't indicate access type (TABULAR vs FILE). Users must paginate through all items and probe each via SDK to find queryable data. This creates a significant discovery gap.

## Key Findings

### Finding 1: STAC Has 3,774 Items (Pagination Issue)

The STAC search endpoint returns **3,774 total items** but only **10 per page by default**:

```python
# GET https://api.hubocean.earth/api/stac/search
{
  "numberMatched": 3774,
  "numberReturned": 10,  # Default limit!
  "links": [{"rel": "next", "href": "...?limit=10&offset=10"}]
}
```

Initial investigation only saw first page, missing 99.7% of catalog.

### Finding 2: TABULAR Datasets ARE in STAC

Further probing discovered **50+ TABULAR datasets** in Norwegian waters:

```
Aker BP Metocean Data - Alvheim - Air THP Data - Air Dewpoint Temperature: TABULAR
    Columns: ['timestamp', 'value']
Aker BP Metocean Data - Alvheim - Air THP Data - Air Humidity: TABULAR
    Columns: ['timestamp', 'value']
... (50+ datasets)
```

**Key insight:** TABULAR data exists and IS discoverable via STAC - but requires:
1. Paginating through all 3,774 items
2. Probing each with SDK to determine type

### Finding 3: No Access Type Indicator in STAC

STAC metadata doesn't indicate whether a dataset is:
- **TABULAR** (queryable via `dataset.table.select()`)
- **FILE** (downloadable via `dataset.files.list()`)
- **Empty** (metadata only, no accessible data)

Users must call SDK for each dataset to determine:
```python
ds = client.dataset(collection_id)
if ds.table.schema():
    print("TABULAR")
elif ds.files.list():
    print("FILE")
else:
    print("Empty/No access")
```

### Finding 4: Mixed Results Across Items

| Category | Count | Example |
|----------|-------|---------|
| TABULAR (queryable) | 50+ | Aker BP Metocean time series |
| FILE (0 files) | ~30 | GLODAP, GEBCO |
| TABULAR (not in STAC) | 1+ | PGS Brazil Biota |

### Finding 5: Dual UUIDs Pattern

Some datasets have two UUIDs with different access patterns:

| UUID | STAC | SDK Type | Access |
|------|------|----------|--------|
| `1d801817-742b-4867-82cf-5597673524eb` | 404 | TABULAR | 2,241 rows |
| `b960c80e-7ead-47af-b6c8-e92a9b5ac659` | Yes | FILE | 0 files |

Both refer to PGS Brazil Biota data.

## The Real Gap

```
What STAC Shows              What Users Need to Know
─────────────────            ──────────────────────
Collection metadata     →    Is this TABULAR or FILE?
Title, description      →    Can I query it? How many rows?
Geographic extent       →    What columns are available?
```

**User Experience:**
1. Browse STAC catalog → finds 80+ datasets
2. No indication which are queryable vs file-based
3. Must probe each via SDK to find usable data
4. Time-consuming trial and error

## Impact

1. **Scale hidden** - Default pagination hides 99.7% of catalog (10 of 3,774)
2. **Discovery expensive** - Must paginate 378 pages, then probe each item via SDK
3. **API roundtrips** - 378 + 3,774 = 4,152 requests to fully discover catalog
4. **No filtering** - Can't query "give me all TABULAR items" directly
5. **UX friction** - Users see 10 items and assume that's all there is

## Questions for Discussion

1. **Can pagination default be increased?** Returning 10 of 3,774 items hides the catalog's scale.

2. **Can STAC include access type?** A simple `odp:access_type: "tabular"` property would enable filtering.

3. **Why do some FILE datasets return 0 files?** GLODAP, GEBCO show in STAC but `files.list()` returns empty.

4. **Why are some TABULAR datasets not in STAC?** PGS Biota TABULAR UUID returns 404 from STAC.

5. **Is dual UUID intentional?** Same data with FILE and TABULAR UUIDs seems confusing.

6. **Can SDK provide discovery?** A `client.list_datasets(type="tabular")` would be valuable.

## Observation: TABULAR vs FILE for Cloud-Native Workflows

In our exploration, TABULAR access appears better suited for cloud-native use in ODP Workspaces:

```python
# TABULAR: server-side query, only matching rows transferred
df = dataset.table.select("temperature > 20").all().dataframe()
```

```python
# FILE: currently requires full download
data = b''.join(dataset.files.download(file_id))
```

**Questions this raises:**
- Is TABULAR the intended pattern for cloud-native ODP use?
- For FILE datasets (GLODAP, GEBCO) returning 0 files - is there a different access method we're missing?
- Are there plans for streaming/partial FILE access?

We may be misunderstanding the intended use of FILE vs TABULAR - guidance from the ODP team would be helpful.

## Proposed Solutions

### Option A: STAC Access Type Property (Recommended)

Add access type to STAC metadata:

```json
{
  "id": "aker-bp-metocean-alvheim-temp",
  "title": "Aker BP Metocean - Air Temperature",
  "properties": {
    "odp:access_type": "tabular",
    "odp:row_count": 52560,
    "odp:columns": ["timestamp", "value"]
  }
}
```

Benefits:
- Filter in STAC queries: `properties.odp:access_type=tabular`
- No SDK roundtrip needed for discovery
- Follows STAC extension pattern

### Option B: SDK Discovery Method

```python
# Discover all accessible datasets by type
tabular = client.list_datasets(type="tabular")
files = client.list_datasets(type="file", accessible=True)

# Or with schema preview
for ds in client.list_datasets(type="tabular"):
    print(f"{ds.title}: {ds.columns}")
```

### Option C: STAC Search Extension

Enable filtering by access type in STAC search:

```python
# STAC API search
response = requests.post(f"{STAC_URL}/search", json={
    "filter": {
        "op": "=",
        "args": [{"property": "odp:access_type"}, "tabular"]
    }
})
```

### Option D: Bulk Type Endpoint

New API endpoint returning access types for all collections:

```python
# GET /api/collections/types
{
  "uuid-1": {"type": "tabular", "rows": 52560},
  "uuid-2": {"type": "file", "files": 12},
  "uuid-3": {"type": "empty"}
}
```

## Workaround: Paginated Discovery with Type Detection

Until a solution is implemented, tutorials must handle pagination and probe each item:

```python
import requests
from odp.client import Client

STAC_URL = "https://api.hubocean.earth/api/stac"
client = Client()

def get_all_stac_items(limit=100):
    """Paginate through all STAC items."""
    items = []
    offset = 0
    while True:
        resp = requests.get(f"{STAC_URL}/search", params={"limit": limit, "offset": offset})
        data = resp.json()
        features = data.get("features", [])
        if not features:
            break
        items.extend(features)
        offset += limit
        print(f"Fetched {len(items)} of {data.get('numberMatched', '?')} items...")
    return items

def probe_dataset_type(client, item_id):
    """Probe a dataset to determine its access type."""
    try:
        ds = client.dataset(item_id)
        schema = ds.table.schema()
        if schema:
            stats = ds.table.stats()
            return {
                "type": "tabular",
                "columns": [f.name for f in schema],
                "rows": stats.num_rows if stats else None
            }
        files = ds.files.list()
        if files:
            return {"type": "file", "files": len(files)}
        return {"type": "empty"}
    except Exception as e:
        return {"type": "error", "message": str(e)}

# Full discovery (warning: 4000+ API calls)
items = get_all_stac_items()
print(f"Total items: {len(items)}")

tabular_datasets = []
for item in items:
    info = probe_dataset_type(client, item['id'])
    if info['type'] == 'tabular':
        tabular_datasets.append((item, info))
        print(f"TABULAR: {item['properties'].get('title', item['id'])}")

print(f"Found {len(tabular_datasets)} TABULAR datasets")
```

## Known Working Datasets

### TABULAR (queryable)
| Dataset | Schema |
|---------|--------|
| Aker BP Metocean (50+ sensors) | `['timestamp', 'value']` |
| PGS Brazil Biota* | Marine mammal observations |

*Not in STAC - requires known UUID: `1d801817-742b-4867-82cf-5597673524eb`

### FILE (0 files via SDK)
| Dataset | Notes |
|---------|-------|
| GLODAP | Metadata only? |
| GEBCO Bathymetry | Metadata only? |

## Related

- `proposals/sdk_batch_metadata.md` - Batch count for Dask integration
- `proposals/server_side_h3_aggregation.md` - Server-side aggregation features
- `proposals/my_data_api.md` - Programmatic dataset creation
- `.claude/skills/odp-troubleshooting/` - Workarounds documented

## References

- [ODP Python SDK](https://docs.hubocean.earth/python_sdk/intro/)
- [STAC Specification](https://stacspec.org/)
- [STAC Extensions](https://stac-extensions.github.io/)
