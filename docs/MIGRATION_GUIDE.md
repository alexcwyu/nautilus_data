# Migration Guide: nautilus_data

**Branch**: `feature/ayu_develop`
**Standard**: See `docs/PYTHON_MODERN_STANDARD.md` in the trading workspace root.

## Overview

This project has been modernized on the `feature/ayu_develop` branch to use the 2026 Python tooling stack. When syncing from upstream (default branch), the following changes must be re-applied if upstream overwrites them.

## What Changed

### 1. Build System (pyproject.toml)
- **Build backend**: `hatchling` (was: `hatchling` -- no change, already modern)
- **PEP 621 metadata**: All project metadata in `[project]` table
- **Dependencies**: Managed by `uv`, lockfile in `uv.lock`

### 2. Removed Legacy Files
No legacy files were removed -- the project already used modern tooling.

### 3. Source Layout
- **Layout**: `src/` layout
- **Package moved**: `nautilus_data/` -> `src/nautilus_data/`
- **Import unchanged**: `import nautilus_data` still works

If upstream adds files to the old location, move them to `src/nautilus_data/`.

### 4. Tooling Configuration (in pyproject.toml)

#### Ruff (linting + formatting)
```toml
[tool.ruff]
line-length = 100
target-version = "py313"

[tool.ruff.lint]
select = ["C4", "E", "F", "W", "C90", "D", "UP", "S", "T10", "ICN", "PIE", "PYI", "Q", "I", "RSE", "TID", "SIM", "B", "PERF", "TC", "PTH", "PD", "NPY", "RUF"]

[tool.ruff.lint.isort]
known-first-party = ["nautilus_data"]
```

Changes from initial migration:
- Added `SIM`, `B`, `PERF`, `TC`, `PTH` to ruff lint select per PYTHON_MODERN_STANDARD
- `SIM` was previously commented out, now enabled

#### Pyright (type checking)
```toml
[tool.pyright]
pythonVersion = "3.13"
typeCheckingMode = "basic"
```

#### Pytest
```toml
[tool.pytest.ini_options]
minversion = "9.0"
addopts = ["-ra", "-q", "--strict-markers", "--import-mode=importlib"]
testpaths = ["tests"]
pythonpath = ["src"]
xfail_strict = true
filterwarnings = ["error"]
```

Changes from initial migration:
- Added `--import-mode=importlib` to addopts
- Added `xfail_strict = true` and `filterwarnings = ["error"]`

### 5. File Reorganization
- `MIGRATION_GUIDE.md` moved from repo root to `docs/MIGRATION_GUIDE.md`

### 6. Python Version
- `.python-version` set to `3.13`
- `requires-python = ">=3.13"` in pyproject.toml

### 6. Hatch Build Target
```toml
[tool.hatch.build.targets.wheel]
packages = ["src/nautilus_data"]
```

## After Upstream Sync Checklist

When merging upstream changes into `feature/ayu_develop`:

1. **Check pyproject.toml**: Upstream may modify `[project]` metadata (version bumps, new deps). Merge those changes but keep `[build-system]`, `[tool.hatch.build]`, `[tool.ruff]`, `[tool.pyright]`, `[tool.pytest]` sections intact.
2. **Check source layout**: If upstream adds new modules to the old `nautilus_data/` path, move them to `src/nautilus_data/`.
3. **Re-lock**: Run `uv lock` to update `uv.lock` with any new/changed dependencies.
4. **Verify**: Run `uv sync && uv run python -c "import nautilus_data" && uv run pytest` (if tests exist).

## Quick Commands

```bash
uv sync                                    # Install all deps
uv run python -c "import nautilus_data"    # Verify import
uv run pytest                              # Run tests
uv run ruff check .                        # Lint
uv run ruff format .                       # Format
uv lock                                    # Re-generate lockfile
```
