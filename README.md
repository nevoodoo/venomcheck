<div align="center">

# venomcheck

**Scan Python dependencies for vulnerabilities with dependency chain tracing.**

[![CI](https://github.com/nevoodoo/venomcheck/actions/workflows/ci.yml/badge.svg)](https://github.com/nevoodoo/venomcheck/actions/workflows/ci.yml)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)
[![Python 3.13+](https://img.shields.io/badge/python-3.13%2B-blue.svg)](https://python.org)

</div>

> [!WARNING]
> **Beta** — APIs, inputs, and output formats may change without notice.
> Pin to a specific commit hash in production workflows.

## Why venomcheck?

Existing scanners (pip-audit, osv-scanner, grype) tell you *what* is vulnerable but not *why* it's in your project. When a transitive dependency has a CVE, you're left guessing which direct dependency pulled it in and whether you can actually fix it.

**venomcheck answers: who brought this in, and can I upgrade past it?**

## Example output

```
Found 3 vulnerabilities in 2 packages

| Package      | Version | Vulnerability  | Fix    | Dependency Chain                           |
|--------------|---------|----------------|--------|--------------------------------------------|
| cryptography | 46.0.3  | CVE-2026-26007 | 46.0.5 | azure-identity > azure-core > cryptography |
| protobuf     | 3.20.2  | CVE-2025-4565  | 4.25.8 | **hail** (pinned, blocked)                 |
| protobuf     | 3.20.2  | CVE-2026-0994  | 5.29.6 | **hail** (pinned, blocked)                 |

### Summary
- 1 fixable via dependency upgrade
- 2 blocked by upstream constraints
```

## Quick start

### GitHub Action

```yaml
- uses: nevoodoo/venomcheck@<COMMIT_SHA> # v0
```

### CLI

```bash
python -m venomcheck.cli --mode uv --path /path/to/project
```

## Usage

> [!TIP]
> **Pin to commit hashes, not tags.** Tags are mutable — a compromised upstream
> can repoint a tag to malicious code. Commit SHAs are immutable.

<details>
<summary><strong>With options</strong></summary>

```yaml
- uses: nevoodoo/venomcheck@<COMMIT_SHA> # v0
  with:
    mode: auto              # auto | uv | pip
    fail-on-vulns: true     # exit 1 if vulnerabilities found
    comment-on-pr: true     # post/update PR comment
    ignore-ids: "CVE-2026-0994,GHSA-xxxx"
    ignore-packages: "protobuf"
```

</details>

<details>
<summary><strong>pip project</strong></summary>

```yaml
- uses: actions/setup-python@a309ff8b426b58ec0e2a45f0f869d46889d02405 # v6.2.0
  with:
    python-version: "3.12"
- run: pip install -r requirements.txt
- uses: nevoodoo/venomcheck@<COMMIT_SHA> # v0
  with:
    mode: pip
```

</details>

<details>
<summary><strong>Monorepo (scan a subdirectory)</strong></summary>

```yaml
- uses: nevoodoo/venomcheck@<COMMIT_SHA> # v0
  with:
    path: services/api
```

</details>

<details>
<summary><strong>Use the report in a later step</strong></summary>

```yaml
- uses: nevoodoo/venomcheck@<COMMIT_SHA> # v0
  id: scan
  with:
    fail-on-vulns: false
- run: echo "Found ${{ steps.scan.outputs.vuln-count }} vulnerabilities"
```

</details>

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `mode` | no | `auto` | Detection mode: `auto`, `uv`, or `pip` |
| `path` | no | `.` | Path to directory containing `uv.lock` or `requirements.txt` |
| `fail-on-vulns` | no | `true` | Exit with failure if vulnerabilities found |
| `comment-on-pr` | no | `true` | Post/update PR comment with report |
| `ignore-ids` | no | `""` | Comma-separated vulnerability IDs to ignore |
| `ignore-packages` | no | `""` | Comma-separated package names to skip |

## Outputs

| Output | Description |
|--------|-------------|
| `vuln-count` | Number of vulnerabilities found |
| `report` | Markdown report content |

## How it works

1. **Detects project type** — `uv.lock` present → uv mode, otherwise pip mode
2. **Builds the dependency graph** — parses `uv.lock` (TOML) or reads installed package metadata via `importlib.metadata`
3. **Queries OSV.dev** — checks all packages against the largest open-source vulnerability database
4. **Traces dependency chains** — walks the graph back to the direct dependency that introduced each finding
5. **Classifies findings** — fixable (upgrade path exists), blocked (pinned upstream), or ignored
6. **Reports** — markdown table with actionable remediation paths

## Design choices

- **Minimal dependencies** — pip mode uses only the stdlib (`importlib.metadata`, `urllib.request`); uv mode additionally requires `uv` for lock file resolution
- **Composite action** — no Docker, fast startup, works on all runner OSes
- **OSV.dev** — free, no auth, same DB used by pip-audit and `uv audit`
- **Marker-isolated PR comments** — uses an HTML marker to find/update its own comment without clobbering other bots

## Permissions

For PR comments, the workflow needs:

```yaml
permissions:
  pull-requests: write
```

<details>
<summary><strong>Avoiding double runs</strong></summary>

The `comment-on-pr` feature requires `pull_request` triggers. A common pattern:

```yaml
on:
  push:
    branches: [main]
  pull_request:
```

`push` fires only for `main`, `pull_request` fires for PRs — no double runs.

</details>

## Local usage

```bash
python -m venomcheck.cli --mode uv --path /path/to/project
python -m venomcheck.cli --mode pip --format json
python -m venomcheck.cli --ignore-packages protobuf --ignore-ids CVE-2026-0994
```

## Status

> [!NOTE]
> This is a **beta release**. Known limitations:
> - Duplicate CVE entries may appear when multiple vulnerability databases (GHSA, PYSEC) track the same issue
> - pip mode depends on packages being installed in the current environment
> - Version comparison uses a simple numeric parser that may not handle all PEP 440 edge cases

Contributions and bug reports are welcome.

## Origin

This project was originally written for [@populationgenomics](https://github.com/populationgenomics) as [python-package-scanner](https://github.com/populationgenomics/python-package-scanner). It has been copied here to allow independent development — adding features, experimenting with new directions, and diverging from the upstream version's scope. Built with help from [Claude](https://claude.ai).

## License

[MIT](LICENSE)
