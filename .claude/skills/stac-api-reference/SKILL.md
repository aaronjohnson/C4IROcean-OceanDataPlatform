---
name: stac-api-reference
description: Quick reference for ODP STAC API endpoints, search parameters, and GeoJSON
allowed-tools: Read, WebFetch
---

## ODP STAC API Reference

### Base URL

```
https://api.hubocean.earth/api/stac
```

### Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/` | Root catalog with links |
| GET | `/collections` | List all collections |
| GET | `/collections/{id}` | Single collection metadata |
| POST | `/search` | Search with filters |

### Helper Functions

```python
import requests

STAC_BASE_URL = "https://api.hubocean.earth/api/stac"

def stac_get(endpoint):
    """GET request to STAC API."""
    response = requests.get(f"{STAC_BASE_URL}{endpoint}")
    response.raise_for_status()
    return response.json()

def stac_post(endpoint, payload):
    """POST request to STAC API."""
    response = requests.post(f"{STAC_BASE_URL}{endpoint}", json=payload)
    response.raise_for_status()
    return response.json()
```

### Search Parameters

```python
search_payload = {
    # Bounding box [west, south, east, north]
    "bbox": [-5, 55, 30, 75],

    # Or GeoJSON geometry
    "intersects": {
        "type": "Polygon",
        "coordinates": [[
            [-5, 51], [15, 51], [15, 72], [-5, 72], [-5, 51]
        ]]
    },

    # Time range (ISO 8601)
    "datetime": "2020-01-01T00:00:00Z/2024-12-31T23:59:59Z",

    # Limit results
    "limit": 100,

    # Filter by collection IDs
    "collections": ["collection-id-1", "collection-id-2"]
}

results = stac_post("/search", search_payload)
features = results.get("features", [])
```

### Common Bounding Boxes

| Region | bbox [west, south, east, north] |
|--------|--------------------------------|
| Norwegian Sea | `[-5, 55, 30, 75]` |
| North Sea | `[-5, 51, 15, 62]` |
| Barents Sea | `[15, 70, 60, 82]` |
| Global | `[-180, -90, 180, 90]` |

### GeoJSON Polygon Template

```python
polygon = {
    "type": "Polygon",
    "coordinates": [[
        [lon1, lat1],  # SW corner
        [lon2, lat1],  # SE corner
        [lon2, lat2],  # NE corner
        [lon1, lat2],  # NW corner
        [lon1, lat1]   # Close polygon
    ]]
}
```

### Collection Metadata Structure

```python
collection = stac_get(f"/collections/{collection_id}")

# Key fields
collection['id']           # UUID
collection['title']        # Human-readable name
collection['description']  # Description
collection['license']      # License type
collection['keywords']     # List of tags

# Extent
collection['extent']['spatial']['bbox']      # [[west, south, east, north]]
collection['extent']['temporal']['interval'] # [[start, end]]
```

### Search Result Structure

```python
for feature in results['features']:
    feature['id']                    # Item ID
    feature['collection']            # Parent collection ID
    feature['geometry']              # GeoJSON geometry
    feature['properties']['title']   # Title
    feature['properties']['datetime'] # Timestamp
```

### Pagination

```python
# First page
results = stac_post("/search", {"limit": 100})

# Check for more
next_link = next(
    (link for link in results.get('links', []) if link['rel'] == 'next'),
    None
)

if next_link:
    # Fetch next page
    next_results = requests.get(next_link['href']).json()
```

### Error Handling

```python
try:
    collection = stac_get(f"/collections/{collection_id}")
except requests.HTTPError as e:
    if e.response.status_code == 404:
        print(f"Collection not found: {collection_id}")
    else:
        raise
```
