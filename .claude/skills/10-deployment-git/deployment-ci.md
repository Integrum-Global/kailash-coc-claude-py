# CI/CD Infrastructure for Python Projects

Patterns and principles for CI/CD pipelines in Python projects. Covers GitHub Actions workflows, test matrices, documentation deployment, and release automation.

## GitHub Actions Workflow Patterns

### Test Workflow (on every push/PR)

```yaml
name: Tests
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    strategy:
      matrix:
        python-version: ["3.10", "3.11", "3.12"]
        os: [ubuntu-latest, macos-latest, windows-latest]
    runs-on: ${{ matrix.os }}
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ matrix.python-version }}
      - name: Install dependencies
        run: pip install -e ".[dev]"
      - name: Lint
        run: ruff check . && ruff format --check .
      - name: Test
        run: pytest --tb=short
```

### Pure Python Package Build

```yaml
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install build
      - run: python -m build
      - uses: actions/upload-artifact@v4
        with:
          name: dist
          path: dist/
```

### Tag-Triggered Publishing

```yaml
name: Publish
on:
  push:
    tags: ["v*"]

jobs:
  # ... build jobs above ...

  publish-testpypi:
    needs: [build]
    runs-on: ubuntu-latest
    environment: testpypi
    permissions:
      id-token: write  # for trusted publisher (OIDC)
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: dist
          path: dist/
      - uses: pypa/gh-action-pypi-publish@release/v1
        with:
          repository-url: https://test.pypi.org/legacy/

  publish-pypi:
    needs: [publish-testpypi]
    runs-on: ubuntu-latest
    environment: pypi
    permissions:
      id-token: write
    steps:
      - uses: actions/download-artifact@v4
        with:
          name: dist
          path: dist/
      - uses: pypa/gh-action-pypi-publish@release/v1

  github-release:
    needs: [publish-pypi]
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/checkout@v4
      - name: Create GitHub Release
        run: gh release create ${{ github.ref_name }} --generate-notes
        env:
          GH_TOKEN: ${{ github.token }}
```

## Test Matrix Design

### Python Version Strategy

| Python Version | Support Level | Notes |
| -------------- | ------------- | ----- |
| 3.10 | Minimum supported | Test on CI |
| 3.11 | Supported | Test on CI |
| 3.12 | Primary / Latest | Test on CI, build docs |
| 3.13+ | Future | Add when stable |

### OS Strategy

| OS | When to Include | Notes |
| -- | --------------- | ----- |
| Linux (ubuntu-latest) | Always | Primary platform |
| macOS (macos-latest) | If platform-specific code | ARM (M1+) |
| Windows (windows-latest) | If platform-specific code | MSVC toolchain |

### Matrix Optimization

- Use `fail-fast: false` to see all failures, not just the first
- Run linting only on one Python version (fastest feedback)
- Run full test suite on all matrix combinations
- Cache pip dependencies for faster runs

## Documentation Deployment

### ReadTheDocs

```yaml
# .readthedocs.yaml
version: 2
build:
  os: ubuntu-22.04
  tools:
    python: "3.12"
sphinx:
  configuration: docs/conf.py
python:
  install:
    - method: pip
      path: .
      extra_requirements:
        - docs
```

### GitHub Pages (via GitHub Actions)

```yaml
name: Deploy Docs
on:
  push:
    branches: [main]

jobs:
  deploy-docs:
    runs-on: ubuntu-latest
    permissions:
      pages: write
      id-token: write
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: "3.12"
      - run: pip install -e ".[docs]"
      - run: cd docs && make html  # or: mkdocs build
      - uses: actions/upload-pages-artifact@v3
        with:
          path: docs/_build/html  # or: site/
      - uses: actions/deploy-pages@v4
```

## Release Automation Patterns

### Preferred: Tag-Triggered Pipeline

The recommended pattern:

1. Developer bumps version and updates CHANGELOG
2. Developer pushes a version tag: `git tag v1.2.3 && git push origin v1.2.3`
3. CI automatically:
   - Builds packages
   - Runs tests on built packages
   - Publishes to TestPyPI
   - Publishes to production PyPI
   - Creates GitHub Release with auto-generated notes

### Alternative: Manual Workflow Dispatch

```yaml
on:
  workflow_dispatch:
    inputs:
      version:
        description: "Version to release (e.g., 1.2.3)"
        required: true
      skip_testpypi:
        description: "Skip TestPyPI (patch releases only)"
        type: boolean
        default: false
```

## CI Debugging

### Common Failure Patterns

| Symptom | Likely Cause | Fix |
| ------- | ------------ | --- |
| Build fails on Linux | Missing system deps | Add `apt-get install` step |
| Build fails on macOS | Wrong SDK version | Pin macOS runner version |
| Tests pass locally, fail on CI | Environment difference | Check Python version, OS, env vars |
| Publishing fails | Auth misconfigured | Check trusted publisher or token setup |

### Debugging Commands

```bash
# List recent CI runs
gh run list --limit 10

# Watch a specific run
gh run watch <run-id>

# Download CI logs
gh run download <run-id> --name logs

# Re-run failed jobs
gh run rerun <run-id> --failed
```

## What to Research Live

The agent should always research these before configuring -- they change frequently:

- Current GitHub Actions action versions (@v4 vs @v5, etc.)
- Current `gh-action-pypi-publish` version and OIDC setup
- Current ReadTheDocs build configuration format
- Current best practices for trusted publisher (OIDC) setup on PyPI

Use `web search` and CLI `--help` rather than relying on trained knowledge.
