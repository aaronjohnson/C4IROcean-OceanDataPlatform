# Proposal: SDK and API Improvements

**Status:** Discussion
**Author:** Aaron Johnson
**Date:** 2026-01-06
**Related:** `tutorials/01_catalog_discovery.ipynb`, `.claude/skills/odp-troubleshooting/`

## Summary

This proposal consolidates several SDK and API behavior observations that affect the developer experience when working with ODP datasets.

## Issue 1: FILE Datasets Return Empty File Lists

### Observed Behavior

Several datasets visible in the STAC catalog return empty lists when calling `dataset.files.list()`:

```python
ds = client.dataset("15dac249-4e3d-474b-a246-ba95cffc8807")  # GLODAP
files = ds.files.list()
print(len(files))  # Returns 0
```

**Affected datasets:**
- GLODAP (`15dac249-4e3d-474b-a246-ba95cffc8807`)
- GEBCO Bathymetry (`5070af58-6d8a-4636-a6a0-8ca9298fb3ab`)
- AkerBP Metocean data sharing

### Expected Behavior

If a dataset is discoverable via STAC and contains files, `files.list()` should return those files (or raise a clear permissions error).

### Possible Causes

1. **Permissions model**: STAC metadata is public, but file access requires additional permissions
2. **External storage**: Files may be stored externally with only references in STAC assets
3. **Access pattern mismatch**: SDK expects one storage model, dataset uses another

### Questions for Discussion

1. What determines whether `files.list()` returns results vs empty?
2. Is there a way to check file access permissions before calling `files.list()`?
3. Should the SDK raise an informative error instead of returning empty list?
4. Can STAC assets provide direct download links as a fallback?

### Proposed Solutions

**Option A**: Return informative error when files exist but aren't accessible
```python
files = ds.files.list()
# If no files but STAC shows assets:
# raise PermissionError("Files exist but require additional access")
```

**Option B**: Document access patterns clearly
- Which datasets support `files.list()` vs STAC asset downloads
- Permission requirements for file access

**Option C**: Unified access method
```python
# Single method that tries SDK files, falls back to STAC assets
files = ds.get_downloadable_files()
```

---

## Issue 2: Inconsistent Dataset Type Detection

### Observed Behavior

Some datasets that appear to contain tabular data show as FILE type:

```python
ds = client.dataset(glodap_id)
schema = ds.table.schema()  # Returns None
files = ds.files.list()     # Returns []
# Dataset appears empty but has data in STAC catalog
```

### Questions for Discussion

1. How is dataset type (TABULAR vs FILE) determined?
2. Can a dataset have both tabular and file access?
3. Should `schema()` return `None` or raise an error for non-tabular datasets?

---

## Issue 3: STAC vs SDK Data Discrepancy

### Observed Behavior

The STAC API shows rich metadata for datasets that appear inaccessible via the Python SDK:

- STAC shows: title, description, bbox, temporal extent, keywords
- SDK shows: empty files, no schema

### Proposed Improvement

Provide SDK methods to access STAC metadata directly:

```python
ds = client.dataset(dataset_id)

# Get STAC metadata even if SDK access is limited
stac_meta = ds.stac_metadata()
print(stac_meta['title'])
print(stac_meta['extent'])
print(stac_meta['assets'])  # Direct download links
```

---

## Related Proposals

- `server_side_h3_aggregation.md` - H3 hex string format and COUNT(*) syntax
- `my_data_api.md` - Programmatic dataset creation

## Impact

Resolving these issues would:
- Reduce confusion when discovering datasets
- Enable clearer error messages for access issues
- Improve the tutorial/onboarding experience
- Help users understand which datasets they can actually use

## References

- [ODP Python SDK](https://docs.hubocean.earth/python_sdk/intro/)
- [STAC Specification](https://stacspec.org/)
