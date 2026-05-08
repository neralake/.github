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

### `trivy-scan` — Security Scanning

Runs a Trivy filesystem scan for vulnerabilities, exposed secrets, and IaC misconfigurations. Uploads results as SARIF to the repository's Security tab.

Runs on pull requests to `main` and on every push to feature branches.

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
    uses: neralake/.github/.github/workflows/trivy-scan.yml@main
    permissions:
      actions: read
      contents: read
      security-events: write
```

**Inputs** (all optional — defaults shown):

| Input | Default | Description |
|---|---|---|
| `severity` | `CRITICAL,HIGH` | Severities to report |
| `exit-code` | `0` | `0` = advisory only; `1` = fail the job on findings |
| `ignore-unfixed` | `true` | Skip vulnerabilities with no available fix |
| `scan-ref` | `.` | Subdirectory to scan |
| `trivy-version` | `0.61.0` | Trivy release to use |

**Example with stricter enforcement:**

```yaml
jobs:
  trivy:
    uses: neralake/.github/.github/workflows/trivy-scan.yml@main
    permissions:
      actions: read
      contents: read
      security-events: write
    with:
      exit-code: 1
      severity: 'CRITICAL,HIGH,MEDIUM'
```

> **Note:** SARIF upload to the Security tab requires GitHub Advanced Security on private repositories. Without it, the upload step will fail but the scan itself will still run.

---

## Reusable Actions

### `pr-size-check` — Enforce PR Size

Fails a job if the number of added lines in a pull request exceeds a configurable limit. Use this to encourage smaller, reviewable PRs.

```yaml
# .github/workflows/compliance.yml
name: Compliance Checks
on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  check-pr-size:
    runs-on: ubuntu-latest
    steps:
      - uses: neralake/.github/actions/pr-size-check@main
        with:
          max_lines_changed: 1000   # optional, default: 1200
```

**Inputs:**

| Input | Default | Description |
|---|---|---|
| `max_lines_changed` | `1200` | Maximum added lines before the check fails |
| `github_token` | `github.token` | Token used to fetch PR stats |
