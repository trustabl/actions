# Trustabl Actions

Reusable GitHub Actions that runs [trustabl](https://github.com/trustabl/trustabl),
the static reliability/safety analyzer for agent-SDK repos (Claude Agent SDK,
OpenAI Agents SDK, Google ADK, MCP). The action:

- Downloads the official `trustabl` release binary (counts toward the
  upstream repo's release download stats).
- Scans the caller's checkout for tools, agents, subagents, and MCP servers.
- Emits **SARIF** and uploads it to GitHub Code Scanning.
- Emits the **full JSON `ScanResult`** and uploads it as a downloadable
  workflow artifact.
- Optionally fails the job on a **risk-score** or **severity** threshold.
- Prints a colored pass/fail line in the log and a result table in the
  run's Step Summary.

## Quick start

```yaml
name: Trustabl
on: [push, pull_request]

permissions:
  contents: read
  security-events: write  # required when upload-sarif is on (default)

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: trustabl/actions@main
```

That's it. With zero config the action scans the checkout, uploads the SARIF
to Code Scanning, attaches `trustabl.json` + `trustabl.sarif` as an artifact
named `trustabl-scan-results`, and only fails the job if `trustabl` itself
flags a medium-or-higher finding.

## Pinned + gated

```yaml
- uses: trustabl/actions@main
  with:
    version: v0.5.0
    detectors: claude_sdk,openai_sdk
    severity-threshold: high       # fail on any high or critical finding
    risk-score-threshold: 70       # fail if risk (100 - score) >= 70
    artifact-retention-days: "30"
```

## Inputs

| Name | Default | Description |
|---|---|---|
| `target` | `.` | Path or GitHub URL to scan. |
| `version` | `latest` | trustabl release tag (e.g. `v0.5.0`) or `latest`. |
| `detectors` | _(all)_ | Comma-separated subset: `claude_sdk,openai_sdk,google_adk`. |
| `strict` | `false` | Pass `--strict` to trustabl (fail on any finding). |
| `rules-ref` | _(default)_ | Pin a `trustabl-rules` git ref. |
| `rules-repo` | _(default)_ | Override `trustabl-rules` source repo. |
| `upload-sarif` | `true` | Call `github/codeql-action/upload-sarif`. |
| `sarif-file` | `trustabl.sarif` | SARIF output path. |
| `json-file` | `trustabl.json` | JSON ScanResult output path. |
| `upload-artifact` | `true` | Upload JSON + SARIF as a workflow artifact. |
| `artifact-name` | `trustabl-scan-results` | Artifact name. |
| `artifact-retention-days` | _(repo default)_ | Days to keep the artifact (1-90). |
| `risk-score-threshold` | `0` | Fail when `risk >= N` (1-100). `0` disables. |
| `severity-threshold` | `none` | Fail when any finding `>= severity`. One of `none`, `low`, `medium`, `high`, `critical`. |
| `branch` | _(auto)_ | SARIF category label. Auto-detects `main` → `master` → HEAD. |

## Outputs

| Name | Description |
|---|---|
| `exit-code` | trustabl's native exit code (0 / 1 / 2). |
| `overall-score` | Readiness score (0-100, higher = better). |
| `risk-score` | `100 - overall-score`. |
| `max-severity` | Highest severity among findings, or `none`. |
| `findings-count` | Total finding count. |
| `sarif-file` | Path to the emitted SARIF file. |
| `json-file` | Path to the emitted JSON file. |
| `artifact-name` | Artifact name used for the upload. |

## Downloading the scan result

After a workflow run, open the run page and find the **`trustabl-scan-results`**
artifact in the "Artifacts" section. It contains:

- `trustabl.json` — full machine-readable `ScanResult` (findings, inventory,
  readiness, rules version, scan ID).
- `trustabl.sarif` — SARIF 2.1.0 for any downstream SARIF consumer.

You can also pull it from another job in the same run:

```yaml
- uses: actions/download-artifact@v4
  with:
    name: trustabl-scan-results
```

## Notes

- Runs on `ubuntu-*`, `macos-*`, and `windows-*` runners.
- Requires `permissions: security-events: write` when `upload-sarif: true`.
- Binary downloads via `gh release download`, which increments the
  `trustabl/trustabl` release asset download counter.
