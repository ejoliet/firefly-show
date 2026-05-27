# firefly-show

> Upload and visualize local astronomical files in [Firefly](https://github.com/Caltech-IPAC/firefly) from your terminal — one command, zero notebook required.

[![PyPI](https://img.shields.io/pypi/v/firefly-show)](https://pypi.org/project/firefly-show/)
[![Python](https://img.shields.io/pypi/pyversions/firefly-show)](https://pypi.org/project/firefly-show/)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

-----

## Why

`firefly_client` is excellent from notebooks — but there is no first-class CLI for quickly sending a local FITS, VOTable, or IPAC table to a Firefly (IRSA Viewer) Viewer without opening Python. `firefly-show` wraps `FireflyClient.show_data()` — the unified file-display method — into a single terminal command, opens a browser tab, and exits. No manual upload step, no type detection: `show_data` handles everything.

-----

## Architecture

```
┌──────────────┐   show_data(file_input)  ┌──────────────────────┐
│  firefly-show│ ──────────────────────▶  │  Firefly HTTP server │
│  (CLI, Python│                          │  (IRSA / local Docker│
│   Typer app) │ ◀──────────────────────  │   / custom URL)      │
└──────────────┘   {'success': True}      └──────────────────────┘
                                                    │
                                                    ▼
                                            Browser tab opens
```

**Flow:**

1. Parse CLI flags → resolve file path
1. Connect `FireflyClient` to `--url` (default: `$FIREFLY_URL` or `http://localhost:8080/firefly`)
1. `fc.show_data(file_input, title=..., preview_metadata=...)` — **handles upload + display in one call**; `get_payload_from_file()` internally calls `upload_file()` for local paths, passes URLs directly
1. `fc.launch_browser()` → opens viewer in default browser
1. Print viewer URL and exit

-----

## Recommended Stack

|Layer          |Chosen                                                    |Stars            |Last Release|Why chosen                                                                |Rejected                               |
|---------------|----------------------------------------------------------|-----------------|------------|--------------------------------------------------------------------------|---------------------------------------|
|CLI framework  |[Typer](https://github.com/tiangolo/typer)                |16 k             |2024-12     |Click-based, auto –help, native enums                                     |argparse (verbose), Click (lower-level)|
|Firefly binding|[firefly-client](https://pypi.org/project/firefly-client/)|— (IPAC-official)|2026-02     |`show_data()` handles all formats natively; no manual type dispatch needed|dnd-firefly (limited scope)            |
|Install manager|[pipx](https://github.com/pypa/pipx)                      |10 k             |2024-11     |Isolated venv per tool, easy upgrade                                      |pip (global pollution), conda (heavy)  |
|Upgrade UX     |[uv](https://github.com/astral-sh/uv)                     |40 k             |2025-03     |Fastest resolver, drop-in pip                                             |pip, poetry                            |

-----

## Repository Layout

```
firefly-show/
├── src/
│   └── firefly_show/
│       ├── __init__.py          # package version
│       ├── cli.py               # Typer app — entry point
│       └── client.py            # FireflyClient wrapper; calls show_data()
├── tests/
│   ├── test_client.py
│   └── conftest.py              # fixtures, mock FireflyClient
├── pyproject.toml
├── Makefile
├── README.md
├── DEVELOPER.md
└── CONTRIBUTING.md
```

-----

## Prerequisites

|Requirement   |Version|Notes                             |
|--------------|-------|----------------------------------|
|Python        |3.11+  |`python3 --version`               |
|pipx          |latest |install below                     |
|Firefly server|any    |IRSA public viewer used by default|

-----

## Installation

### macOS — recommended

```bash
# 1. Install pipx (skip if already installed)
brew install pipx
pipx ensurepath

# 2. Install firefly-send
pipx install firefly-send

# 3. Verify
firefly-send --version
```

### Linux (Debian/Ubuntu)

```bash
# 1. Install pipx
sudo apt install pipx
pipx ensurepath
source ~/.bashrc   # or restart shell

# 2. Install firefly-send
pipx install firefly-send

# 3. Verify
firefly-send --version
```

### Linux (any distro via uv)

```bash
# 1. Install uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# 2. Install firefly-send into isolated tool env
uv tool install firefly-send

# 3. Verify
firefly-send --version
```

### pip (no isolation)

```bash
pip install firefly-send
```

> ⚠️ Use a virtual environment to avoid dependency conflicts with `astropy`, `pyvo`, etc.

-----

## Upgrade

```bash
# pipx
pipx upgrade firefly-send

# uv
uv tool upgrade firefly-send

# pip
pip install --upgrade firefly-send
```

-----

## Quick Start

```bash
# Send a FITS image to the public IRSA Viewer (opens browser)
firefly-send my-image.fits

# Send a catalog table
firefly-send my-catalog.tbl

# Send a VOTable
firefly-send results.xml

# Use a local Docker Firefly server
firefly-send my-image.fits --url http://localhost:8080/firefly

# Use a custom internal server with auth token
firefly-send my-image.fits --url https://irsa.internal/viewer --token $MY_TOKEN

# Suppress browser auto-open, just print the viewer URL
firefly-send my-image.fits --no-browser
```

-----

## Configuration Reference

|Flag / Env var         |Type|Default                                   |Required|Description                                                                                      |
|-----------------------|----|------------------------------------------|--------|-------------------------------------------------------------------------------------------------|
|`FILE`                 |path|—                                         |✅       |Local file to upload (FITS, .tbl, .xml, .csv, .reg)                                              |
|`--url` / `FIREFLY_URL`|str |`https://irsa.ipac.caltech.edu/irsaviewer`|no      |Firefly server base URL                                                                          |
|`--channel`            |str |auto-generated UUID                       |no      |WebSocket channel to use                                                                         |
|`--plot-id`            |str |filename stem                             |no      |Plot ID for later reference                                                                      |
|`--title`              |str |filename                                  |no      |Display title in Firefly viewer                                                                  |
|`--token`              |str |—                                         |no      |Bearer token for authenticated servers                                                           |
|`--no-browser`         |flag|False                                     |no      |Skip `launch_browser()`; print URL only                                                          |
|`--preview`            |flag|False                                     |no      |Pass `preview_metadata=True` to `show_data()`; lets you pick FITS extensions in UI before loading|

-----

## CLI Interface Contract

```
Usage: firefly-send [OPTIONS] FILE

  Upload FILE to a Firefly server and display it in the viewer.

Arguments:
  FILE  Path to the local file to upload.  [required]

Options:
  --url TEXT        Firefly server URL.  [env: FIREFLY_URL]
  --channel TEXT    WebSocket channel ID.
  --plot-id TEXT    Plot ID for the displayed item.
  --title TEXT      Display title.
  --token TEXT      Bearer token for authenticated servers.
  --no-browser      Print viewer URL instead of opening browser.
  --preview         Preview metadata before loading (pick FITS extensions, columns, etc.)
  --version         Show version and exit.
  --help            Show this message and exit.
```

### `show_data` — the real API

There is **no manual dispatch table**. `FireflyClient.show_data()` is the unified method:

```python
fc.show_data(
    file_input,              # local path, URL, server path, or file-like object
    preview_metadata=False,  # True → show extension picker in UI before loading
    title=None               # display title; derived from filename if omitted
)
```

Firefly’s server-side determines the rendering (FITS image, table, region, MOC, DataLink, UWS) from the file content. Supported formats:

- FITS images (single or multi-extension)
- Catalog / table: IPAC, CSV, TSV, VOTable, Parquet, FITS table
- DS9 region files (`.reg`)
- Multi-Order Coverage (MOC) maps in FITS
- DataLink and UWS job files

`get_payload_from_file()` is called internally: local paths are uploaded via `upload_file()`; URLs are passed directly — no pre-upload step needed in CLI code.

-----

## Usage Examples

### Example 1 — Send a local FITS image

```bash
$ firefly-send 2mass-m31-green.fits --title "2MASS M31 H-band"
✔ show_data: 2mass-m31-green.fits
🌐 Viewer URL: https://irsa.ipac.caltech.edu/irsaviewer?__wsch=abc123
   Browser tab opened.
```

### Example 2 — Send a catalog table (auto-overlays on image)

```bash
$ firefly-send m31-2mass-2412-row.tbl --url http://localhost:8080/firefly
✔ show_data: m31-2mass-2412-row.tbl
🌐 Viewer URL: http://localhost:8080/firefly?__wsch=def456
   Browser tab opened.
```

### Example 3 — Preview metadata before loading (pick FITS extensions in UI)

```bash
$ firefly-send roman_wfi_l2.fits --preview --title "Roman WFI L2"
✔ show_data: roman_wfi_l2.fits (preview_metadata=True)
🌐 Viewer URL: https://irsa.ipac.caltech.edu/irsaviewer?__wsch=xyz789
   Open the URL and select extensions, then click Load.
```

-----

## Error Handling

|Error                |Class                           |Behavior                          |
|---------------------|--------------------------------|----------------------------------|
|File not found       |`FileNotFoundError`             |Exit 1, print path                |
|Server unreachable   |`ConnectionError`               |Exit 1, suggest checking `--url`  |
|Upload failure       |`FireflySendUploadError`        |Exit 1, print server response     |
|Unsupported file type|`ValueError` from Firefly server|Exit 1, print server error message|
|Browser open fails   |warning                         |Prints URL, exits 0               |

-----

## Testing

```bash
make test          # pytest with mocked FireflyClient (no live server needed)
make lint          # ruff + mypy --strict
make test-live     # integration test against $FIREFLY_URL (requires server)
```

Tests mock `firefly_client.FireflyClient` via `unittest.mock`. No real server required for `make test`.

-----

## Non-Goals (v1)

- No multi-file batch upload (v2+)
- No interactive stretch / zoom control from CLI
- No 3-color composite (`show_fits_3color`) support
- No HiPS display (`show_hips`)
- No Windows native support (WSL2 works fine)

-----

## Open Questions

- Should `--preview` auto-enable for multi-extension FITS (e.g. detect HDU count before calling `show_data`)?
- For VOTable auto-detection: check XML namespace vs. extension?
- Authenticated servers: should token be read from `~/.netrc` as fallback?

-----

## Next Steps

|#|Task                                                                 |
|-|---------------------------------------------------------------------|
|1|Scaffold repo with `pyproject.toml` (`[project.scripts]` entry point)|
|2|Implement `client.py` — `send_to_firefly()` wrapping `fc.show_data()`|
|3|Implement `cli.py` — Typer app wiring all flags                      |
|4|Write `tests/` with mocked `FireflyClient` (assert `show_data` calls)|
|5|`make lint` + `make test` green                                      |
|6|Publish to PyPI via `uv build` + `uv publish`                        |
|7|Verify install via `pipx install firefly-send` on macOS and Linux    |

-----

## References

- [firefly_client documentation](https://caltech-ipac.github.io/firefly_client/) — official IPAC Python API
- [firefly_client API: FireflyClient](https://caltech-ipac.github.io/firefly_client/api/firefly_client.FireflyClient.html) — `show_data`, `make_client`, `launch_browser`, `get_firefly_url`
- [pipx documentation](https://pipx.pypa.io/) — isolated CLI tool installation
- [Typer documentation](https://typer.tiangolo.com/) — CLI framework
- [IRSA Viewer (public Firefly instance)](https://irsa.ipac.caltech.edu/irsaviewer)
