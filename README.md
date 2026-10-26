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

### `contract-codegen-check` — Contract Codegen Is Current

Regenerates a consumer's contract code from `buf.build/neralake/contracts` and fails if the
result differs from what is committed. Catches stale generated code and generated code that
was hand-edited instead of changing the proto.

**Requires:** `BUF_TOKEN` (a BSR token with read access to the org). Composite actions cannot
read secrets, so it must be passed in. The calling job must already have run `setup-node` and
installed dependencies — this action does neither.

```yaml
- uses: neralake/.github/actions/contract-codegen-check@main
  with:
    gen-dir: src/gen          # functions/shared/gen in serverless
    buf-token: ${{ secrets.BUF_TOKEN }}
```

The module reference in each consumer's `buf.gen.yaml` is unpinned, so this compares against
whatever the BSR holds *now*. A red result is therefore not always the PR author's fault — it
also fires when someone else publishes a contract change. The failure message says so; keep
that wording if you edit it, or the check gets written off as flaky.

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

---

### `codeql-scan` — SAST Security Analysis

Runs CodeQL static analysis using the `security-extended` query suite, tuned for JavaScript/TypeScript back-end services (NestJS, Express, GraphQL). Catches logic-level vulnerabilities that dependency scanners miss: SQL/NoSQL injection, path traversal, SSRF, prototype pollution, ReDoS, and more.

Findings are surfaced as **PR annotations** (visible in the Files Changed tab) and a **job summary table** — no GitHub Advanced Security subscription required.

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
  codeql:
    runs-on: ubuntu-latest
    permissions:
      security-events: write  # required for CodeQL database upload
    steps:
      - uses: actions/checkout@v6
      - uses: neralake/.github/actions/codeql-scan@main
```

**Inputs** (all optional — defaults shown):

| Input | Default | Description |
|---|---|---|
| `exit-code` | `0` | `0` = advisory only; `1` = fail the job on high/critical findings |

**Example with stricter enforcement:**

```yaml
    steps:
      - uses: actions/checkout@v6
      - uses: neralake/.github/actions/codeql-scan@main
        with:
          exit-code: '1'
```