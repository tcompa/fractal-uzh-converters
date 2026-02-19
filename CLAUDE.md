# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Fractal tasks to convert HCS (High-Content Screening) plate data from microscopes into OME-Zarr format. Supports three microscope systems: PerkinElmer Operetta, Olympus ScanR, and Yokogawa CQ3K/CellVoyager.

## Environment & Commands

Uses **pixi** for environment management. Available environments: `default`, `dev`, `test`, `docs`.

```bash
# Run tests
pixi run -e test pytest tests
pixi run -e test pytest tests/test_operetta_task.py       # single test file
pixi run -e test pytest tests/test_operetta_task.py -k "test_name"  # single test

# Lint and format
pixi run -e dev ruff check .
pixi run -e dev ruff check . --fix
pixi run -e dev ruff format .

# Docs
pixi run -e docs mkdocs serve

# Validate Fractal manifest
pixi run -e dev python src/fractal_uzh_converters/dev/task_list.py
```

## Architecture

### Compound Task Pattern

Each converter follows Fractal's **compound task** pattern with two stages:

1. **Init task** (`convert_*_init_task.py`): Parses microscope-specific metadata (XML/CSV), creates OME-Zarr plate structure, returns a `parallelization_list`
2. **Compute task** (`common/image_in_plate_compute_task.py`): Shared across all converters — processes individual FOVs in parallel, writes image tiles to OME-Zarr

### Source Layout (`src/fractal_uzh_converters/`)

- `operetta/`, `olympus_scanr/`, `cq3k/` — Each contains an init task + `utils.py` with metadata parsing and Pydantic models
- `common/` — Shared compute task and utilities (condition table handling, base models)
- `dev/task_list.py` — Generates `__FRACTAL_MANIFEST__.json`

### Key Dependencies

- `ome-zarr-converters-tools` — Core conversion engine (see below)
- `fractal-task-tools` — Fractal task execution framework
- `ngio` — OME-Zarr I/O library (transitive dependency via converters-tools, not imported directly here)
- For local development of dependencies, editable paths can be uncommented in `pyproject.toml` under `[tool.pixi.pypi-dependencies]`

### Conversion Pipeline (ome-zarr-converters-tools)

The conversion follows a data pipeline driven by abstractions from `ome_zarr_converters_tools`:

```
Raw metadata (XML/CSV) → Tiles → TiledImages → OME-Zarr
```

**Key types imported from `ome_zarr_converters_tools`:**
- `Tile` — Atomic unit: a region of image data with coordinates, an `ImageInPlate` collection reference, a `DefaultImageLoader` (TIFF path), and `AcquisitionDetails` (pixel size, channels, axes)
- `TiledImage` — A complete image assembled from multiple tiles (one per well/FOV)
- `ImageInPlate` — Collection type defining output path structure: `plate/row/col/acquisition/suffix.zarr`
- `ConverterOptions`, `AcquisitionOptions`, `OverwriteMode`, `TilingMode`, `WriterMode` — Configuration enums/models
- `ChannelInfo`, `StageCorrections`, `DataTypeEnum` — Metadata models

**Init task flow:**
1. Device-specific parser creates `Tile` objects from raw metadata
2. `tiles_aggregation_pipeline()` groups/filters tiles into `TiledImage` list
3. `setup_images_for_conversion()` creates OME-Zarr plate structure and returns `parallelization_list`

**Compute task flow:**
- `generic_compute_task()` loads a `TiledImage` from JSON, assembles tile data, writes to OME-Zarr, returns `ImageListUpdateDict`

### Model Hierarchy

`BaseAcquisitionModel` (in `common/utils.py`) is extended by each converter's `*AcquisitionModel` in their respective `utils.py`. These models define acquisition paths, plate names, and advanced options (condition tables, channel overrides, stage corrections).

### Testing

Tests use **snapshot-based assertions** — reference YAML files in `tests/data/` store expected image fingerprints (mean, std, min, max, hash). Use `--update-snapshots` pytest flag to regenerate reference data.

## Code Style

- Ruff with 88-char line length, Google-style docstrings
- Pydantic models with `@validate_call` on task functions
- Python 3.10+ type hint syntax (`list[T]`, `str | None`)
- Pre-commit hooks: `validate-pyproject`, `ruff`, `ruff-format`
