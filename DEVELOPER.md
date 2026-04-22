# Developer Guide — firefly-send

This guide covers local dev setup, architecture decisions, code patterns, and release process.

-----

## Dev Setup

### Requirements

|Tool  |Version|Install                                          |
|------|-------|-------------------------------------------------|
|Python|3.11+  |`brew install python` / `apt install python3.11` |
|uv    |latest |`curl -LsSf https://astral.sh/uv/install.sh | sh`|
|Docker|20+    |optional, for local Firefly server               |

### Clone and install

```bash
git clone https://github.com/<org>/firefly-send
cd firefly-send

# Install with dev extras into an isolated venv
uv venv
source .venv/bin/activate
uv pip install -e ".[dev]"
```

### Dev extras (`pyproject.toml`)

```toml
[project.optional-dependencies]
dev = [
  "pytest>=8",
  "pytest-cov",
  "ruff",
  "mypy",
  "types-requests",
]
```

-----

## Project Structure

```
src/firefly_send/
├── __init__.py     # version = "0.1.0"
├── cli.py          # Typer app; all CLI flags defined here
└── client.py       # FireflyClient wrapper; calls show_data()
```

> No `detect.py` needed — `show_data()` and its internal `get_payload_from_file()` handle
> type detection, upload, and dispatch server-side.

### `client.py`

Exports one function. **No manual type detection needed** — `show_data()` delegates file
handling and format dispatch to Firefly server-side via `get_payload_from_file()`:

```python
def send_to_firefly(
    file_input: str,           # local path, URL, or server ref
    url: str,
    channel: str | None,
    title: str | None,
    token: str | None,
    preview_metadata: bool,
    open_browser: bool,
) -> str:                      # returns viewer URL
    fc = FireflyClient.make_client(url=url, channel=channel, token=token)

    # show_data handles upload + display in one call:
    # - local path  → internally calls upload_file(), then ShowAnyData action
    # - URL         → passed directly as 'url' payload key
    # - file-like   → uploaded via upload_data()
    fc.show_data(
        file_input,
        preview_metadata=preview_metadata,
        title=title,
    )

    if open_browser:
        fc.launch_browser()

    return fc.get_firefly_url()
```

Supported `file_input` types (from `get_payload_from_file()` source):

- Local file path (`str`) → `upload_file(path)` called automatically
- HTTP/HTTPS URL (`str`) → passed as `url` payload key, no upload
- Server reference starting with `${` → passed as `fileOnServer` directly
- File-like object (`io.IOBase`) → `upload_data(stream, 'UNKNOWN')` called

### `cli.py`

Typer app; no business logic — delegates entirely to `client.show_data()`.

```python
app = typer.Typer()

@app.command()
def send(
    file: Path = typer.Argument(...),
    url: str = typer.Option(None, envvar="FIREFLY_URL"),
    ...
):
    ...
```

-----

## Makefile Targets

|Target          |What it does                                    |
|----------------|------------------------------------------------|
|`make install`  |`uv pip install -e ".[dev]"`                    |
|`make test`     |`pytest tests/ -v --cov=firefly_send`           |
|`make lint`     |`ruff check src/ tests/` + `mypy src/`          |
|`make fmt`      |`ruff format src/ tests/`                       |
|`make test-live`|integration test against `$FIREFLY_URL`         |
|`make build`    |`uv build` → `dist/`                            |
|`make publish`  |`uv publish` (requires `$PYPI_TOKEN`)           |
|`make clean`    |remove `dist/`, `.mypy_cache/`, `.pytest_cache/`|

-----

## Testing

Tests live in `tests/`. The `FireflyClient` is always mocked — no live server needed for `make test`.

### Mock pattern

Mock `make_client` and assert `show_data` is called with the right arguments.
No need to mock `upload_file` — it’s called internally by `show_data` and never
exposed in `client.py`.

```python
# conftest.py
import pytest
from unittest.mock import MagicMock

@pytest.fixture
def mock_fc(monkeypatch):
    fc = MagicMock()
    fc.show_data.return_value = {"success": True}
    fc.get_firefly_url.return_value = "https://irsa.ipac.caltech.edu/irsaviewer?__wsch=test"
    monkeypatch.setattr("firefly_send.client.FireflyClient.make_client", lambda **kw: fc)
    return fc
```

### Example test

```python
# test_client.py
from pathlib import Path
from firefly_send.client import send_to_firefly

def test_local_fits_calls_show_data(mock_fc, tmp_path):
    f = tmp_path / "img.fits"
    f.write_bytes(b"SIMPLE  = T")  # minimal FITS magic bytes

    url = send_to_firefly(
        file_input=str(f),
        url="http://localhost:8080/firefly",
        channel=None,
        title="Test image",
        token=None,
        preview_metadata=False,
        open_browser=False,
    )

    mock_fc.show_data.assert_called_once_with(
        str(f), preview_metadata=False, title="Test image"
    )
    assert "irsaviewer" in url

def test_preview_metadata_flag(mock_fc, tmp_path):
    f = tmp_path / "cube.fits"
    f.write_bytes(b"SIMPLE  = T")

    send_to_firefly(str(f), url="http://localhost:8080/firefly",
                    channel=None, title=None, token=None,
                    preview_metadata=True, open_browser=False)

    mock_fc.show_data.assert_called_once_with(str(f), preview_metadata=True, title=None)

def test_url_input_passed_directly(mock_fc):
    remote = "https://irsa.ipac.caltech.edu/data/2MASS/fits/m31.fits"

    send_to_firefly(remote, url="http://localhost:8080/firefly",
                    channel=None, title=None, token=None,
                    preview_metadata=False, open_browser=False)

    # show_data receives the URL string — get_payload_from_file handles it internally
    mock_fc.show_data.assert_called_once_with(remote, preview_metadata=False, title=None)
```

### Coverage target

`make test` fails if coverage drops below 80%.

-----

## Code Style

- **Type hints everywhere** — `mypy --strict` must pass
- **No sync calls from async context** — not currently async, keep it that way unless needed
- **Secrets via env vars only** — never log `--token` value
- `ruff` line length: 100

-----

## Local Firefly Server (Docker)

Useful for integration tests without hitting the public IRSA viewer:

```bash
docker run --rm -p 8080:8080 ipac/firefly
```

Then:

```bash
FIREFLY_URL=http://localhost:8080/firefly make test-live
```

Image reference: [`ipac/firefly` on Docker Hub](https://hub.docker.com/r/ipac/firefly)

-----

## Release Process

1. Bump version in `src/firefly_send/__init__.py`
1. Update `CHANGELOG.md`
1. `git tag vX.Y.Z && git push --tags`
1. `make build`
1. `make publish` (or let GitHub Actions do it on tag push)

### GitHub Actions (`.github/workflows/publish.yml`)

Trigger: `push` to tag `v*`. Steps: checkout → uv build → uv publish.

-----

## References

- [firefly_client API reference](https://caltech-ipac.github.io/firefly_client/api/firefly_client.FireflyClient.html)
- [Typer docs](https://typer.tiangolo.com/)
- [uv docs](https://docs.astral.sh/uv/)
- [ipac/firefly Docker image](https://hub.docker.com/r/ipac/firefly)
