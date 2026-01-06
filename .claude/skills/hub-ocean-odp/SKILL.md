---
name: hub-ocean-odp
description: Look up HUB Ocean ODP (Ocean Data Platform) documentation. Use when asking about ODP Python SDK, STAC API, workspaces, datasets, authentication, or any docs.hubocean.earth content.
allowed-tools: Read, Glob, Grep, WebFetch
---

# HUB Ocean ODP Documentation Lookup

This skill helps navigate the Ocean Data Platform documentation and provides quick access to common reference material.

## Documentation URL Structure

The ODP documentation is hosted at `docs.hubocean.earth` with the following structure:

| Topic | URL |
|-------|-----|
| Python SDK Introduction | https://docs.hubocean.earth/python_sdk/intro/ |
| Python SDK - Tabular Data | https://docs.hubocean.earth/python_sdk/intro/#tabular |
| Python SDK - Files | https://docs.hubocean.earth/python_sdk/intro/#files |
| STAC API | https://docs.hubocean.earth/stac-api/ |
| Workspaces | https://docs.hubocean.earth/workspaces/info/ |

## STAC API Endpoints

Base URL: `https://api.hubocean.earth/api/stac`

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/` | GET | Root catalog |
| `/collections` | GET | List all datasets |
| `/collections/{id}` | GET | Get collection metadata |
| `/search` | POST | Search with filters (bbox, datetime, intersects) |

## Python SDK Quick Reference

### Installation
```bash
pip install -U odp-sdk
```

### Authentication
```python
from odp.client import Client

# Auto-authenticated in ODP Workspace
client = Client()

# Or with API key
client = Client(api_key="your-api-key")
```

### Dataset Access
```python
dataset = client.dataset("dataset-uuid-here")
```

### Table Operations
```python
# Schema and stats
schema = dataset.table.schema()
stats = dataset.table.stats()

# Query data
df = dataset.table.select().all().dataframe()
df = dataset.table.select("column == 'value'").all().dataframe()

# Streaming for large datasets
for batch in dataset.table.select().dataframes():
    process(batch)

# Geospatial filter
df = dataset.table.select(
    'geometry within $area',
    vars={"area": "POLYGON((...))"}
).all().dataframe()

# H3 aggregation
df = dataset.table.aggregate(
    group_by="h3(geometry, 5)",
    aggr={"value": "mean"}
)
```

### File Operations
```python
# List files
files = dataset.files.list()

# Download
for chunk in dataset.files.download(file_id):
    f.write(chunk)

# Upload
file_id = dataset.files.upload("name.csv", data_bytes)

# Ingest file into table
dataset.files.ingest(file_id, opt="append")  # or "truncate", "drop"
```

## When to Use This Skill

Activate when the user asks about:
- ODP Python SDK methods or classes
- STAC API endpoints or queries
- ODP Workspace capabilities
- Dataset discovery or catalog browsing
- Authentication with ODP
- Geospatial queries (within, intersects, contains)
- H3 hexagonal aggregation
- File upload/download/ingest operations

## Fetching Documentation

When more detail is needed, fetch from these URLs:
- Python SDK: `https://docs.hubocean.earth/python_sdk/intro/`
- STAC API: `https://docs.hubocean.earth/stac-api/`
- Workspaces: `https://docs.hubocean.earth/workspaces/info/`
