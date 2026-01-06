# Proposal: My Data API for ODP Python SDK

## Summary

Add programmatic access to personal datasets ("My Data") in the ODP Python SDK, enabling users to list, create, and manage their datasets without requiring the web interface.

## Current Limitations

The ODP Python SDK currently requires users to:

1. **Create datasets via web UI only** - No `client.create_dataset()` method exists
2. **Manually copy UUIDs** - Users must visit `app.hubocean.earth` to find their dataset IDs
3. **No way to list personal datasets** - Cannot programmatically enumerate "My Data" collections

This creates friction for:
- Automated data pipelines
- Notebook tutorials (users must leave the notebook to create datasets)
- Programmatic dataset management
- CI/CD workflows

## Proposed API

### 1. List Personal Datasets

```python
from odp.client import Client

client = Client()

# List all datasets in My Data
my_datasets = client.my_datasets()

for ds in my_datasets:
    print(f"{ds.id}: {ds.name}")
    print(f"  Created: {ds.created_at}")
    print(f"  Type: {'tabular' if ds.has_table else 'files'}")
```

**Returns:** List of `DatasetInfo` objects with:
- `id` (UUID)
- `name` (string)
- `description` (string, optional)
- `created_at` (datetime)
- `updated_at` (datetime)
- `has_table` (bool)
- `file_count` (int)

### 2. Create Dataset

```python
# Create a new personal dataset
dataset = client.create_dataset(
    name="Norwegian Sea Monitoring",
    description="Oceanographic observations from Norwegian coastal stations",
    tags=["norway", "monitoring", "temperature"]
)

print(f"Created dataset: {dataset.id}")

# Immediately usable
dataset.files.upload("data.csv", csv_bytes)
```

**Parameters:**
- `name` (required): Human-readable name
- `description` (optional): Dataset description
- `tags` (optional): List of tags for discoverability
- `public` (optional, default=False): Whether to make publicly visible

**Returns:** `Dataset` object (same as `client.dataset(id)`)

### 3. Delete Dataset

```python
# Delete a dataset (requires confirmation)
client.delete_dataset(dataset_id, confirm=True)
```

### 4. Update Dataset Metadata

```python
dataset = client.dataset(id)
dataset.update_metadata(
    name="Updated Name",
    description="New description",
    tags=["updated", "tags"]
)
```

## Implementation Considerations

### Authentication & Authorization
- Uses existing OAuth/API key authentication
- Only returns datasets where user has owner/editor role
- Respects existing permission model

### API Endpoints (Backend)
Suggested REST endpoints:

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/api/v1/my-data` | List user's datasets |
| POST | `/api/v1/my-data` | Create new dataset |
| DELETE | `/api/v1/my-data/{id}` | Delete dataset |
| PATCH | `/api/v1/my-data/{id}` | Update metadata |

### Pagination
For users with many datasets:
```python
# Paginated listing
page1 = client.my_datasets(limit=20, offset=0)
page2 = client.my_datasets(limit=20, offset=20)
```

## Use Cases

### 1. Tutorial Notebooks
```python
# Self-contained tutorial - no web UI needed
dataset = client.create_dataset(name="Tutorial Data")
# ... run tutorial ...
client.delete_dataset(dataset.id, confirm=True)  # Cleanup
```

### 2. Automated Pipelines
```python
# Daily data ingest pipeline
datasets = {ds.name: ds for ds in client.my_datasets()}

if "Daily Observations" not in datasets:
    dataset = client.create_dataset(name="Daily Observations")
else:
    dataset = datasets["Daily Observations"]

dataset.files.upload(f"obs_{date}.csv", data)
dataset.files.ingest(file_id, opt="append")
```

### 3. Dataset Inventory
```python
# Audit personal datasets
for ds in client.my_datasets():
    stats = client.dataset(ds.id).table.stats()
    print(f"{ds.name}: {stats.num_rows if stats else 0} rows")
```

## Alternatives Considered

### 1. STAC API Extension
Extend STAC `/collections` to include personal datasets with auth filter.
- Pro: Reuses existing infrastructure
- Con: STAC is designed for public catalog, not personal data management

### 2. Web UI Only (Current State)
- Pro: No SDK changes needed
- Con: Breaks notebook workflows, prevents automation

## Priority

**High** - This is a fundamental capability gap that affects:
- New user onboarding (tutorials require web UI detour)
- Automation use cases
- Developer experience

## Backward Compatibility

Fully backward compatible - adds new methods without changing existing API.

## References

- Current SDK docs: https://docs.hubocean.earth/python_sdk/intro/
- My Data web UI: https://app.hubocean.earth/
- Related tutorial: `tutorials/03_data_pipeline.ipynb`
