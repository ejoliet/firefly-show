# Contributing to firefly-show

Thanks for helping! This is a small, focused CLI tool — contributions should stay focused too.

-----

## Ways to Contribute

- Bug reports and repro cases (include: file type, server URL, full error output)
- Improved error messages and exit codes
- Platform-specific install fixes (Linux distros, macOS arm64)
- Additional `show_data` usage patterns and CLI options (see Open Questions in README)
- Documentation improvements

-----

## Before You Start

- Check [open issues](https://github.com/<org>/firefly-show/issues) — your idea may already be tracked
- For new features, open an issue first to discuss scope before writing code
- For bugs, open an issue with a minimal repro (file type, server URL pattern, error output)

-----

## Dev Setup

See <DEVELOPER.md> for full setup instructions. Short version:

```bash
git clone https://github.com/ejoliet/firefly-show
cd firefly-show
uv venv && source .venv/bin/activate
uv pip install -e ".[dev]"
make test && make lint
```

Both must pass before submitting a PR.

-----

## PR Checklist

- [ ] `make test` passes (≥ 80% coverage)
- [ ] `make lint` passes (`ruff` + `mypy --strict`)
- [ ] New behaviour is covered by tests (mock `FireflyClient`, no live server required)
- [ ] `CHANGELOG.md` updated under `## Unreleased`
- [ ] PR description explains *what* and *why* (not just *how*)

-----

## What We Won’t Merge (v1 scope)

To keep the tool focused, the following are out of scope for v1:

- Batch / multi-file upload (single `show_data` call only)
- Interactive stretch/zoom from CLI (`set_zoom`, `set_stretch`, etc.)
- 3-color composite — requires `show_fits_3color`, not covered by `show_data`
- Windows native support (WSL2 is fine)
- GUI or web front-end
- Streaming stdin to Firefly (file-like support exists in API but needs integration testing)

Open an issue tagged `v2` if you want to propose these for later.

-----

## Commit Style

Follow [Conventional Commits](https://www.conventionalcommits.org/):

```
feat: add --preview flag for show_data(preview_metadata=True)
fix: handle file-like object input from stdin
docs: clarify --token usage in README
test: assert show_data called with correct preview_metadata flag
refactor: simplify send_to_firefly signature
```

-----

## Reporting Security Issues

Do **not** open a public issue. Email the maintainer directly (see `pyproject.toml` → `[project.maintainers]`).

-----

## License

By submitting a PR you agree that your contribution is licensed under the [MIT License](LICENSE).
