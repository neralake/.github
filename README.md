# neralake/.github

Centralized GitHub Actions workflows and actions for the neralake organization. Member repositories call these with a small triggering workflow — the logic lives here.

---

## Reusable Workflows

### `shortcut-check` — Enforce Shortcut Ticket

Blocks a PR from merging unless a valid Shortcut ticket ID is found in the branch name, PR title, or PR body. Accepted formats: `123/feature-name`, `sc-123`, `ch-123`, `story/123`, `epic/123`.

**Requires:** `SHORTCUT_API_TOKEN` org secret.

```yaml
# .github/workflows/compliance.yml
name: Compliance Checks
on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  shortcut-check:
    uses: neralake/.github/.github/workflows/shortcut-check.yml@main
    secrets: inherit
```

---

## Reusable Actions

### `trivy-scan` — Security Scanning

Runs two Trivy filesystem scans in sequence: one for exposed secrets, one for vulnerabilities. Results are printed as a table in the job log.

```yaml
# .github/workflows/security-scan.yml
name: Security Scan
on:
  pull_request:
    branches:
      - main
  push:
    branches-ignore:
      - main

jobs:
  trivy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      - uses: neralake/.github/actions/trivy-scan@main
```

**Inputs** (all optional — defaults shown):

| Input | Default | Description |
|---|---|---|
| `severity` | `CRITICAL,HIGH` | Severities to report for the vulnerability scan |
| `exit-code` | `0` | `0` = advisory only; `1` = fail the job on findings |
| `ignore-unfixed` | `true` | Skip vulnerabilities with no available fix |
| `scan-ref` | `.` | Path to scan for secrets (and vulnerabilities if `vuln-scan-ref` is not set) |
| `vuln-scan-ref` | _(scan-ref)_ | Override the path for the vulnerability scan only |

**Example with stricter enforcement:**

```yaml
    steps:
      - uses: actions/checkout@v6
      - uses: neralake/.github/actions/trivy-scan@main
        with:
          exit-code: 1
          severity: 'CRITICAL,HIGH,MEDIUM'
```