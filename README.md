# venomcheck

> **Beta** — This project is under active development. APIs, inputs, and output
> formats may change without notice. Use with caution in production workflows
> and pin to a specific commit hash.

A GitHub Action that scans Python dependencies for known vulnerabilities and
traces each finding back through the dependency chain to the direct dependency
that introduced it.

## Origin

This project was originally written for [@populationgenomics](https://github.com/populationgenomics) as [python-package-scanner](https://github.com/populationgenomics/python-package-scanner). It has been copied here to allow independent development — adding features, experimenting with new directions, and diverging from the upstream version's scope. Built with help from [Claude](https://claude.ai).

## Why?

Existing scanners (pip-audit, osv-scanner, grype) tell you *what* is vulnerable
but not *why* it's in your project. When a transitive dependency has a CVE,
you're left guessing which direct dependency pulled it in and whether you can
actually fix it.

This action answers: **who brought this in, and can I upgrade past it?**

## Example output

```
Found 3 vulnerabilities in 2 packages

| Package      | Version | Vulnerability  | Fix    | Dependency Chain                       |
|--------------|---------|----------------|--------|----------------------------------------|
| cryptography | 46.0.3  | CVE-2026-26007 | 46.0.5 | azure-identity > azure-core > cryptography |
| protobuf     | 3.20.2  | CVE-2025-4565  | 4.25.8 | **hail** (pinned, blocked)             |
| protobuf     | 3.20.2  | CVE-2026-0994  | 5.29.6 | **hail** (pinned, blocked)             |

### Summary
- 1 fixable via dependency upgrade
- 2 blocked by upstream constraints
```

## Usage

> **Pin to commit hashes, not tags.** Tags are mutable — a compromised upstream
> can repoint a tag to malicious code. Commit SHAs are immutable.
> Add a `# vX` comment for readability.

### Basic (uv project)

```yaml
- uses: nevoodoo/venomcheck@<COMMIT_SHA> # v0
```

### With options

```yaml
- uses: nevoodoo/venomcheck@<COMMIT_SHA> # v0
  with:
    mode: auto              # auto | uv | pip
    fail-on-vulns: true     # exit 1 if vulnerabilities found
    comment-on-pr: true     # post/update PR comment
    ignore-ids: "CVE-2026-0994,GHSA-xxxx"
    ignore-packages: "protobuf"
```

### pip project

```yaml
- uses: actions/setup-python@a309ff8b426b58ec0e2a45f0f869d46889d02405 # v6.2.0
  with:
    python-version: "3.12"
- run: pip install -r requirements.txt
- uses: nevoodoo/venomcheck@<COMMIT_SHA> # v0
  with:
    mode: pip
```

### Monorepo (scan a subdirectory)

```yaml
- uses: nevoodoo/venomcheck@<COMMIT_SHA> # v0
  with:
    path: services/api
```

### Use the report in a later step

```yaml
- uses: nevoodoo/venomcheck@<COMMIT_SHA> # v0
  id: scan
  with:
    fail-on-vulns: false
- run: echo "Found ${{ steps.scan.outputs.vuln-count }} vulnerabilities"
```

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `mode` | no | `auto` | Detection mode: `auto`, `uv`, or `pip` |
| `path` | no | `.` | Path to directory containing uv.lock or requirements.txt |
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
2. **Builds the dependency graph** — parses `uv.lock` (TOML) or reads installed
   package metadata via `importlib.metadata`
3. **Queries OSV.dev** — checks all packages against the largest open-source
   vulnerability database (same source as pip-audit and osv-scanner)
4. **Traces dependency chains** — for each finding, walks the graph back to the
   direct dependency that introduced it
5. **Classifies findings** — fixable (upgrade path exists), blocked (pinned
   upstream), or ignored
6. **Reports** — markdown table with actionable remediation paths

## Design choices

- **stdlib-only** — no external Python dependencies; uses `tomllib`, `importlib.metadata`, and `urllib.request`
- **Composite action** — no Docker, fast startup, works on all runner OSes
- **OSV.dev** — free, no auth, same DB used by pip-audit and `uv audit`
- **Marker-isolated PR comments** — uses an HTML marker to find/update its own comment without clobbering other bots

## Permissions

For PR comments, the workflow needs:

```yaml
permissions:
  pull-requests: write
```

The `comment-on-pr` feature requires the workflow to trigger on `pull_request`
events. The PR number is read from `github.event.pull_request.number`, which
is only available on `pull_request` triggers — not `push`.

A common pattern for running on both PRs and main:

```yaml
on:
  push:
    branches: [main]
  pull_request:
```

This avoids double runs — `push` fires only for `main`, `pull_request` fires
for PRs.

## Local usage

```bash
python -m venomcheck.cli --mode uv --path /path/to/project
python -m venomcheck.cli --mode pip --format json
python -m venomcheck.cli --ignore-packages protobuf --ignore-ids CVE-2026-0994
```

## Status

This is a **beta release**. Known limitations:

- Duplicate CVE entries may appear when multiple vulnerability databases
  (GHSA, PYSEC) track the same issue
- pip mode depends on packages being installed in the current environment
- Version comparison uses a simple numeric parser that may not handle all
  PEP 440 edge cases (epochs, pre-release ordering)

Contributions and bug reports are welcome.

## License

MIT
