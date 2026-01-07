# Ocean Data Platform Workspaces

> **Fork Note:** This is an enhanced fork of [C4IROcean/OceanDataPlatform](https://github.com/C4IROcean/OceanDataPlatform) featuring additional tutorial notebooks for real-world oceanographic analysis workflows, Claude Code skills for AI-assisted development, and a My Data API proposal.

Welcome to Workspaces! A JupyterHub environment where you can directly access the ODP Catalog and datasets using our API. Packages for pulling data and analyzing data are already installed, so no need to spend time setting up the environment!

Please find relevant notebooks to get you started and don't forget to check out our [documentation](https://docs.hubocean.earth/)

## Example Notebooks
We provide a number of example notebooks to get you going with Ocean Data Platform and are always adding new ones as we deploy new features.

The examples cover the two types of data, Table (Tabular) and Files, that you work with in ODP and addresses both read and write scenarios.

### SDK Reference (Root Directory)
| Notebook | Description |
|----------|-------------|
| `sdk_reference_table_read.ipynb` | Query and filter tabular data |
| `sdk_reference_table_write.ipynb` | Create tables, insert and modify data |
| `sdk_reference_files_read.ipynb` | List and download files |
| `sdk_reference_files_write.ipynb` | Upload files and manage metadata |

### Tutorials (`tutorials/` Directory)
| Notebook | Description |
|----------|-------------|
| `01_catalog_discovery.ipynb` | Discover datasets via STAC API |
| `02_geospatial_analysis.ipynb` | H3 hexagonal aggregation and mapping |
| `03_data_pipeline.ipynb` | File ingest workflows |
| `04_multi_dataset_join.ipynb` | Cross-dataset analysis |
| `05_dask_distributed_processing.ipynb` | Dask parallel processing in ODP Workspaces |

## Claude Code Skills

This repository includes [Claude Code](https://claude.ai/code) skills for AI-assisted ODP development:

| Skill | Purpose |
|-------|---------|
| `hub-ocean-odp` | ODP documentation URLs, SDK quick reference |
| `notebook-formatting` | Tutorial notebook conventions and templates |
| `odp-dataset-patterns` | Tabular/file detection, upload, H3 aggregation |
| `stac-api-reference` | STAC endpoints, search params, GeoJSON |
| `odp-troubleshooting` | Common issues and solutions |
| `proposal-writing` | When and how to write discussion proposals |
| `dask-odp-patterns` | Dask distributed processing for ODP cloud workflows |

Location: `.claude/skills/`

<br><br>

Note: this Ocean Data Platform Tutorials directory will be automatically synced with new updates and new notebooks when added to the [github repo](https://github.com/C4IROcean/OceanDataConnector). Therefore we recommend that best practice is:
- use a separate directory for your personal notebooks and work, and,
- if you want to use or edit the examples, create a duplicate first and work on that version, then the original files will stay up to date.
