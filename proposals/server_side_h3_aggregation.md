# Proposal: Server-Side H3 Aggregation for Visualization

**Status:** Discussion
**Author:** Aaron Johnson
**Date:** 2026-01-06
**Related:** `tutorials/02_geospatial_analysis.ipynb`

## Summary

The ODP SDK's `aggregate()` function supports H3 hexagonal grouping via `group_by=f"h3({geometry_col}, {resolution})"`, but the returned values are internal indices rather than H3 cell ID strings. This makes the results incompatible with visualization libraries (Folium, h3-py) that require standard H3 hex strings like `"852b9afffffffff"`.

## Problem Statement

### Current Behavior

```python
h3_agg = dataset.table.aggregate(
    group_by=f"h3(footprintWKT, 5)",
    filter="footprintWKT IS NOT NULL",
    aggr={"occurrenceID": "count"}
)
```

Returns a DataFrame with columns like:
```
   h3_footprintWKT_5   count
0                  7      15
1                  8      23
2                 49      12
...
```

The first column contains small integers (7, 8, 49...) rather than H3 cell ID strings.

### Expected Behavior

For visualization compatibility, the h3 column should return standard H3 hex strings:
```
          h3_cell   count
0  852b9afffffffff      15
1  852b9bfffffffff      23
2  852b9cfffffffff      12
...
```

### Why This Matters

1. **Scale**: Client-side H3 computation works for small datasets (<100k rows) but becomes prohibitive for millions of rows - exactly the scale where cloud environments shine.

2. **Visualization**: The h3-py library's `cell_to_boundary()` function requires H3 hex strings to convert cells to polygon coordinates for mapping.

3. **Interoperability**: Standard H3 cell IDs enable joining with external datasets, caching results, and sharing analysis reproducibly.

## Current Workaround

The tutorial uses client-side aggregation:

```python
import h3

def get_h3_cell(lat, lon, res):
    try:
        return h3.latlng_to_cell(lat, lon, res)  # v4 API
    except AttributeError:
        return h3.geo_to_h3(lat, lon, res)  # v3 API

df['h3_cell'] = df.apply(
    lambda row: get_h3_cell(row['lat'], row['lon'], resolution),
    axis=1
)
h3_agg = df.groupby('h3_cell').size().reset_index(name='count')
```

This works but requires downloading all raw data to the client.

## Investigation

A debug cell is included in `02_geospatial_analysis.ipynb` (cell after H3 aggregation) to investigate the server response format:

```python
# Uncomment to investigate server-side H3 response
h3_server = dataset.table.aggregate(
    group_by=f"h3({GEOMETRY_COL}, {H3_RESOLUTION})",
    filter=f"{GEOMETRY_COL} IS NOT NULL",
    aggr={"occurrenceID": "count"}
)
print(f"Dtypes:\n{h3_server.dtypes}")
print(f"Sample values: {h3_server.iloc[:5, 0].tolist()}")
```

## Questions for Discussion

1. **Index mapping**: Is there a way to map the returned indices back to H3 cell IDs? Perhaps via a separate API call or lookup table?

2. **Format option**: Could an optional parameter return H3 hex strings instead of indices?
   ```python
   aggregate(group_by="h3(...)", h3_format="hex")  # vs "index"
   ```

3. **Performance trade-off**: Are indices used for internal efficiency? Would returning hex strings impact query performance?

4. **Roadmap**: Is native H3 hex string support planned for a future SDK version?

## Proposed Solutions

### Option A: SDK Returns Hex Strings (Preferred)

Modify the SDK to return standard H3 cell IDs by default. This provides immediate compatibility with the H3 ecosystem.

### Option B: Index-to-Hex Lookup

Provide a separate API or SDK method to convert server indices to H3 hex strings:
```python
h3_agg['h3_cell'] = client.h3_index_to_hex(h3_agg['h3_index'], resolution=5)
```

### Option C: Documentation + Utility

If the current behavior is intentional, document it clearly and provide a client-side utility for conversion.

## Impact

Resolving this would enable:
- Efficient aggregation of millions of observations
- Interactive dashboards with live H3 queries
- Scalable spatial analysis workflows in ODP Workspace

## References

- [H3 Documentation](https://h3geo.org/docs/)
- [ODP Python SDK](https://docs.hubocean.earth/python_sdk/intro/)
- [h3-py GitHub](https://github.com/uber/h3-py)
