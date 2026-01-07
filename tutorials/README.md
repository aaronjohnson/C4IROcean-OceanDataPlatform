# ODP Tutorials

These notebooks demonstrate real-world analysis workflows on the Ocean Data Platform, building on the SDK reference examples.

## Notebooks

| Notebook | Description |
|----------|-------------|
| [01_catalog_discovery.ipynb](01_catalog_discovery.ipynb) | Programmatically discover datasets using the STAC API |
| [02_geospatial_analysis.ipynb](02_geospatial_analysis.ipynb) | Query and visualize data with H3 hexagonal aggregation |
| [03_data_pipeline.ipynb](03_data_pipeline.ipynb) | Ingest files and transform into tabular data |
| [04_multi_dataset_join.ipynb](04_multi_dataset_join.ipynb) | Combine multiple datasets for analysis |
| [05_dask_distributed_processing.ipynb](05_dask_distributed_processing.ipynb) | Parallel processing with Dask in ODP Workspaces |

## Prerequisites

- Running in [ODP Workspace](https://workspace.hubocean.earth/) (recommended) or have an API key
- `odp-sdk` installed (`pip install -U odp-sdk`)

## STAC API Reference

The notebooks use the STAC API for dataset discovery:

- **Base URL**: `https://api.hubocean.earth/api/stac`
- **Endpoints**:
  - `GET /` - Root catalog
  - `GET /collections` - List all datasets
  - `GET /collections/{id}` - Collection metadata
  - `POST /search` - Search with spatial/temporal filters

## Resources

- [ODP Documentation](https://docs.hubocean.earth/)
- [Python SDK Reference](https://docs.hubocean.earth/python_sdk/intro/)
- [STAC API Documentation](https://docs.hubocean.earth/stac-api/)
- [ODP Catalog (Web UI)](https://app.hubocean.earth/catalog)
