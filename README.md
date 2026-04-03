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

| Package      | Version | Vulnerability  | Fix    | Dependency Chain                              |
|--------------|---------|----------------|--------|-----------------------------------------------|
| cryptography | 42.0.0  | CVE-2024-26130 | 42.0.4 | boto3 > botocore > urllib3 > cryptography     |
| requests     | 2.31.0  | CVE-2024-35195 | 2.32.0 | flask-admin > requests                        |
| certifi      | 2023.7  | CVE-2024-39689 | 2024.7 | **httpx** (pinned ==0.24.0, blocked)          |

### Summary
- 2 fixable via dependency upgrade
- 1 blocked by upstream constraints
```

## Usage

> [!TIP]
> **Pin to commit hashes, not tags.** Tags are mutable — a compromised upstream
> can repoint a tag to malicious code. Commit SHAs are immutable.

### GitHub Action

#### uv project

```yaml
- uses: nevoodoo/venomcheck@<COMMIT_SHA> # v0
```

#### pip project

```yaml
- uses: actions/setup-python@a309ff8b426b58ec0e2a45f0f869d46889d02405 # v6.2.0
  with:
    python-version: "3.12"
- run: pip install -r requirements.txt
- uses: nevoodoo/venomcheck@<COMMIT_SHA> # v0
  with:
    mode: pip
```

#### With all options

```yaml
- uses: nevoodoo/venomcheck@<COMMIT_SHA> # v0
  with:
    mode: auto              # auto | uv | pip
    fail-on-vulns: true     # exit 1 if vulnerabilities found
    comment-on-pr: true     # post/update PR comment
    ignore-ids: "CVE-2024-39689,GHSA-xxxx"
    ignore-packages: "certifi"
    exclude-dev: false       # set true to skip dev dependencies
```

#### Monorepo (scan a subdirectory)

```yaml
- uses: nevoodoo/venomcheck@<COMMIT_SHA> # v0
  with:
    path: services/api
```

#### Use the report in a later step

```yaml
- uses: nevoodoo/venomcheck@<COMMIT_SHA> # v0
  id: scan
  with:
    fail-on-vulns: false
- run: echo "Found ${{ steps.scan.outputs.vuln-count }} vulnerabilities"
```

### CLI

```bash
# uv project
python -m venomcheck.cli --mode uv --path /path/to/project

# pip project
python -m venomcheck.cli --mode pip --path /path/to/project

# JSON output
python -m venomcheck.cli --format json

# Ignore specific packages or CVEs
python -m venomcheck.cli --ignore-packages certifi --ignore-ids CVE-2024-39689
```

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
