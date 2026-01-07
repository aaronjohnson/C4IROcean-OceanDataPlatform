# Proposal: Data Discoverability Gap

**Status:** Discussion
**Author:** Aaron Johnson
**Date:** 2026-01-06
**Updated:** 2026-01-06 (TABULAR discovered in STAC)
**Related:** `tutorials/01_catalog_discovery.ipynb`, `tutorials/02_geospatial_analysis.ipynb`

## Summary

STAC catalog contains datasets but doesn't indicate access type (TABULAR vs FILE). Users must probe each dataset via SDK to determine if it's queryable. This creates a discovery gap where users see metadata but can't efficiently find usable data.

## Key Findings

### Finding 1: TABULAR Datasets ARE in STAC (Updated)

Initial investigation found 35 collections returning empty. Further probing discovered **50+ TABULAR datasets** in Norwegian waters:

```
Aker BP Metocean Data - Alvheim - Air THP Data - Air Dewpoint Temperature: TABULAR
    Columns: ['timestamp', 'value']
Aker BP Metocean Data - Alvheim - Air THP Data - Air Humidity: TABULAR
    Columns: ['timestamp', 'value']
... (50+ datasets)
```

**Key insight:** TABULAR data exists and IS discoverable via STAC - but requires probing each dataset with SDK to determine type.

### Finding 2: No Access Type Indicator in STAC

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

### Finding 3: Mixed Results Across Collections

| Category | Count | Example |
|----------|-------|---------|
| TABULAR (queryable) | 50+ | Aker BP Metocean time series |
| FILE (0 files) | ~30 | GLODAP, GEBCO |
| TABULAR (not in STAC) | 1+ | PGS Brazil Biota |

### Finding 4: Dual UUIDs Pattern

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

1. **Discovery inefficient** - Must probe each dataset individually
2. **Tutorials complex** - Need to include type-detection logic
3. **API roundtrips** - N+1 queries to discover N usable datasets
4. **UX friction** - Users can't filter by "datasets I can query"

## Questions for Discussion

1. **Can STAC include access type?** A simple `odp:access_type: "tabular"` property would solve this.

2. **Why do some FILE datasets return 0 files?** GLODAP, GEBCO show in STAC but `files.list()` returns empty.

3. **Why are some TABULAR datasets not in STAC?** PGS Biota TABULAR UUID returns 404 from STAC.

4. **Is dual UUID intentional?** Same data with FILE and TABULAR UUIDs seems confusing.

5. **Can SDK provide discovery?** A `client.list_datasets(type="tabular")` would be valuable.

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

## Workaround: Client-Side Type Detection

Until a solution is implemented, tutorials use this pattern:

```python
def probe_dataset_type(client, collection_id):
    """Probe a dataset to determine its access type."""
    try:
        ds = client.dataset(collection_id)
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

# Probe all STAC collections
for coll in collections:
    info = probe_dataset_type(client, coll['id'])
    if info['type'] == 'tabular':
        print(f"{coll['title']}: {info['rows']} rows")
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

- `proposals/server_side_h3_aggregation.md` - Server-side aggregation features
- `proposals/my_data_api.md` - Programmatic dataset creation
- `.claude/skills/odp-troubleshooting/` - Workarounds documented

## References

- [ODP Python SDK](https://docs.hubocean.earth/python_sdk/intro/)
- [STAC Specification](https://stacspec.org/)
- [STAC Extensions](https://stac-extensions.github.io/)
