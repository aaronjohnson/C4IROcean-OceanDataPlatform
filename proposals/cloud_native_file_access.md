# Proposal: Cloud-Native Data Access in ODP Workspace

**Status:** Discussion
**Author:** Aaron Johnson
**Date:** 2026-01-06
**Related:** `tutorials/01_catalog_discovery.ipynb`, `.claude/skills/odp-troubleshooting/`

## Summary

This proposal addresses cloud-native data access patterns in ODP Workspace. The goal is seamless in-notebook data access without explicit downloads - similar to how TABULAR datasets already work.

## The Ideal: TABULAR Access Pattern

ODP's tabular access is excellent - truly cloud-native:

```python
# No download, no local files - direct cloud query
df = dataset.table.select(
    "temperature > 20",
    vars={"region": bbox_wkt}
).all(max_rows=10000).dataframe()
```

This is the gold standard: **query in cloud → get DataFrame**.

## The Gap: FILE Dataset Access

For FILE datasets, the equivalent cloud-native pattern is unclear:

```python
ds = client.dataset("15dac249-4e3d-474b-a246-ba95cffc8807")  # GLODAP
files = ds.files.list()  # Returns [] - empty
# Now what?
```

**Affected datasets:**
- GLODAP (`15dac249-4e3d-474b-a246-ba95cffc8807`)
- GEBCO Bathymetry (`5070af58-6d8a-4636-a6a0-8ca9298fb3ab`)
- AkerBP Metocean data sharing

These are visible in STAC with rich metadata, but SDK shows no accessible data.

---

## Cloud-Native Access Patterns

### What STAC Spec Recommends

From the [STAC Best Practices](https://github.com/radiantearth/stac-spec/blob/master/best-practices.md):

| Format | Cloud-Native Access | Python Library |
|--------|---------------------|----------------|
| Cloud Optimized GeoTIFF (COG) | HTTP range requests | `rasterio`, `xarray` |
| Zarr | Direct chunk access | `xarray`, `dask` |
| GeoParquet | Direct read | `geopandas` |
| NetCDF on S3 | fsspec streaming | `xarray` |

STAC assets provide `href` URLs that support streaming without full download.

### Desired ODP Pattern for FILE Datasets

**Option A: Streaming file handles**
```python
# Return file-like object for streaming
with dataset.files.open(file_id) as f:
    df = pd.read_csv(f)
    # or
    ds = xr.open_dataset(f)
```

**Option B: Direct integration with data libraries**
```python
# SDK returns URL/path that works with standard libraries
href = dataset.files.get_href(file_id)
ds = xr.open_dataset(href, engine='zarr')
gdf = gpd.read_parquet(href)
```

**Option C: Cloud-optimized format detection**
```python
# SDK detects format and returns appropriate accessor
accessor = dataset.files.open_cloud_native(file_id)
# Returns xarray.Dataset for NetCDF/Zarr, GeoDataFrame for GeoParquet, etc.
```

**Option D: Expose STAC assets directly**
```python
# Access STAC asset hrefs when SDK files unavailable
assets = dataset.stac_assets()
for name, asset in assets.items():
    print(f"{name}: {asset['href']} ({asset['type']})")

# Use href directly with fsspec/xarray
ds = xr.open_dataset(assets['data']['href'])
```

---

## Questions for Discussion

1. **What's the intended access pattern for FILE datasets in ODP Workspace?**
   - Should users use `files.download()` and work locally?
   - Or is there a streaming/cloud-native method we're missing?

2. **Why do some datasets return empty file lists?**
   - Permissions model (STAC public, files restricted)?
   - Different storage backend?
   - Data not yet ingested into ODP file system?

3. **Could STAC assets bridge the gap?**
   - If `files.list()` is empty, can we fall back to STAC asset hrefs?
   - Are those hrefs accessible from within ODP Workspace?

4. **What cloud-optimized formats does ODP support?**
   - COG, Zarr, GeoParquet?
   - Does ODP convert uploaded data to cloud-optimized formats?

---

## Related Issues

### Discovery: Dual UUIDs for Same Data (TABULAR vs FILE)

**Finding:** The same dataset can have two UUIDs with different access patterns:

| UUID | STAC Visible | SDK Access | Type |
|------|--------------|------------|------|
| `1d801817-742b-4867-82cf-5597673524eb` | No (404) | Yes | TABULAR (2,241 rows) |
| `b960c80e-7ead-47af-b6c8-e92a9b5ac659` | Yes | Yes | FILE (0 files) |

**Implication:**
- STAC catalog discovery may not find all accessible datasets
- Some TABULAR datasets are "hidden" from STAC but work via SDK
- Need to know the UUID to access tabular data directly

**Question:** Is this intentional? Should STAC expose both UUIDs, or link them?

### Inconsistent Dataset Type Detection

Some datasets appear empty via SDK but have data in STAC:

```python
ds = client.dataset(glodap_id)
schema = ds.table.schema()  # Returns None
files = ds.files.list()     # Returns []
# But STAC shows: title, bbox, temporal extent, keywords
```

**Question:** How is TABULAR vs FILE determined? Can a dataset support both?

### Suggested SDK Enhancement

```python
# Unified access check
access = dataset.access_info()
# Returns:
# {
#   'tabular': True/False,
#   'files': {'count': N, 'accessible': True/False},
#   'stac_assets': {'count': N, 'hrefs': [...]},
#   'recommended_access': 'table.select()' | 'files.open()' | 'stac_assets'
# }
```

---

## Related Proposals

- `server_side_h3_aggregation.md` - H3 hex string format and COUNT(*) syntax
- `my_data_api.md` - Programmatic dataset creation

## Impact

Clarifying cloud-native FILE access would:
- Enable tutorials covering all ODP data types
- Reduce confusion about empty file lists
- Align with modern cloud data access patterns (no downloads)
- Support large-scale analysis without local storage limits

## References

- [ODP Python SDK](https://docs.hubocean.earth/python_sdk/intro/)
- [STAC Specification](https://stacspec.org/)
- [STAC Best Practices](https://github.com/radiantearth/stac-spec/blob/master/best-practices.md)
- [fsspec - Filesystem interfaces for Python](https://filesystem-spec.readthedocs.io/)
- [Cloud-Optimized GeoTIFF](https://www.cogeo.org/)
