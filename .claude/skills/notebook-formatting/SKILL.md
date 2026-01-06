---
name: notebook-formatting
description: Formatting conventions for ODP tutorial Jupyter notebooks
allowed-tools: Read, Write, NotebookEdit
---

## ODP Tutorial Notebook Format

When creating or editing Jupyter notebooks in the `tutorials/` directory, follow this structure:

### Section Layout

```
# Title (no number)

Intro paragraph with:
- **What you'll learn:** bullet points
- **Prerequisites:** requirements

## 1. First Section
[code and markdown cells]

## 2. Second Section
[code and markdown cells]

... numbered sections ...

## Summary (no number)
Brief recap of what was covered

## Next Steps (no number)
- Links to next tutorials in sequence

## Resources (no number)
- External documentation links
```

### Rules

1. **Title cell**: `# Title` with intro, no section number
2. **Numbered sections**: `## 1. Name`, `## 2. Name`, etc.
3. **Ending sections**: `## Summary`, `## Next Steps`, `## Resources` - NO numbers
4. **Code cells**: Follow markdown section headers, not numbered themselves

### Auto-Install Pattern

For optional dependencies, use auto-install in setup cell:

```python
import subprocess
import sys

def ensure_package(package_name, import_name=None):
    """Install package if not available."""
    import_name = import_name or package_name
    try:
        __import__(import_name)
        return True
    except ImportError:
        print(f"Installing {package_name}...")
        subprocess.check_call([sys.executable, "-m", "pip", "install", "-q", package_name])
        return True

# Example usage
ensure_package("folium")
ensure_package("h3")
import folium
import h3
```

### Tutorial Sequence

| Notebook | Topic |
|----------|-------|
| 01_catalog_discovery | STAC API, dataset discovery |
| 02_geospatial_analysis | H3 aggregation, spatial queries |
| 03_data_pipeline | File upload, ingest, metadata |
| 04_multi_dataset_join | Cross-dataset analysis |
| 05_dask_distributed_processing | Dask sidecar (draft) |

### Next Steps Links

Always link to the next notebook(s) in sequence:
```markdown
## Next Steps

- **03_data_pipeline.ipynb**: Ingest files and transform into tabular data
- **04_multi_dataset_join.ipynb**: Combine multiple datasets for analysis
```
