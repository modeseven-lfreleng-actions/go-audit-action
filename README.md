<!--
# SPDX-License-Identifier: Apache-2.0
# SPDX-FileCopyrightText: 2026 The Linux Foundation
-->

# 🛡️ Go Audit Action

<!-- prettier-ignore-start -->
<!-- markdownlint-disable-next-line MD013 -->
[![Linux Foundation](https://img.shields.io/badge/Linux-Foundation-blue)](https://linuxfoundation.org/) [![Source Code](https://img.shields.io/badge/GitHub-100000?logo=github&logoColor=white&color=blue)](https://github.com/lfreleng-actions/go-audit-action) [![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0) [![pre-commit.ci status badge]][pre-commit.ci results page] [![OpenSSF Scorecard](https://api.scorecard.dev/projects/github.com/lfreleng-actions/go-audit-action/badge)](https://scorecard.dev/viewer/?uri=github.com/lfreleng-actions/go-audit-action)
<!-- prettier-ignore-end -->

Runs govulncheck, gosec and staticcheck security/static audits for Go
projects.

## go-audit-action

This action wraps the three standard Go audit tools behind one
validated interface:

- [govulncheck](https://github.com/golang/vuln) — known-vulnerability
  scanning against the Go vulnerability database, reporting the
  vulnerabilities your code actually calls
- [gosec](https://github.com/securego/gosec) — security-focused
  static analysis with SARIF report output
- [staticcheck](https://staticcheck.dev/) — correctness and
  simplification static analysis

Each tool toggles individually and installs at a pinned version via
`go install pkg@version` — never `@latest`. Every enabled tool runs
to completion before an aggregation step evaluates the combined
verdict, so one failing tool never masks another tool's report.

## Usage Example

<!-- markdownlint-disable MD046 -->

```yaml
steps:
  - name: "Audit project"
    id: audit
    uses: lfreleng-actions/go-audit-action@main
    with:
      path_prefix: '.'
```

Full audit in non-blocking mode with collected reports:

```yaml
steps:
  - name: "Audit project"
    id: audit
    uses: lfreleng-actions/go-audit-action@main
    with:
      gosec: 'true'
      staticcheck: 'true'
      permit_fail: 'true'
      output_directory: audit-reports
```

<!-- markdownlint-enable MD046 -->

## Requirements

The action needs `realpath` (GNU coreutils, including `-m`
support), `mktemp` and `tr` on the runner, plus `jq` when gosec
runs and `tee` when govulncheck or staticcheck runs; the
preflight check enforces the tools the enabled toggles need.
GitHub-hosted Ubuntu runners
include these tools; minimal self-hosted or non-Linux runners must
provide them. The action installs the Go toolchain via the pinned
`actions/setup-go` (caching off per organisation security policy) and
the audit tools via `go install`, so runners need egress to the Go
distribution endpoints, `proxy.golang.org`, `sum.golang.org` and, for
govulncheck, `vuln.go.dev`.

## Inputs

<!-- markdownlint-disable MD013 -->

| Name                | Required | Default   | Description                                                           |
| ------------------- | -------- | --------- | --------------------------------------------------------------------- |
| go_version          | False    | `''`      | Explicit Go version to install; takes precedence over go_version_file |
| go_version_file     | False    | `go.mod`  | File to read the Go version from, relative to path_prefix             |
| path_prefix         | False    | `.`       | Project directory; must resolve within the workspace                  |
| govulncheck         | False    | `true`    | Run govulncheck                                                       |
| gosec               | False    | `false`   | Run gosec                                                             |
| staticcheck         | False    | `false`   | Run staticcheck                                                       |
| govulncheck_version | False    | `v1.5.0`  | golang.org/x/vuln version to install                                  |
| gosec_version       | False    | `v2.27.1` | securego/gosec version to install                                     |
| staticcheck_version | False    | `v0.7.0`  | honnef.co/go/tools version to install                                 |
| permit_fail         | False    | `false`   | Report tool findings without failing the action                       |
| output_directory    | False    | `.`       | Audit report directory, within workspace or runner temp               |

<!-- markdownlint-enable MD013 -->

## Outputs

<!-- markdownlint-disable MD013 -->

| Name               | Description                                            |
| ------------------ | ------------------------------------------------------ |
| govulncheck_passed | govulncheck result: `true`, `false` or `skipped`       |
| gosec_passed       | gosec result: `true`, `false` or `skipped`             |
| staticcheck_passed | staticcheck result: `true`, `false` or `skipped`       |
| reports_directory  | Absolute path to the directory holding audit reports   |

<!-- markdownlint-enable MD013 -->

## Reports

The action writes reports for the enabled tools into
`reports_directory`:

<!-- markdownlint-disable MD013 -->

| File               | Tool        | Format                                    |
| ------------------ | ----------- | ----------------------------------------- |
| `govulncheck.txt`  | govulncheck | Human-readable text (also on the console) |
| `govulncheck.json` | govulncheck | Machine-readable JSON findings stream     |
| `gosec.sarif`      | gosec       | SARIF, suitable for code-scanning upload  |
| `staticcheck.txt`  | staticcheck | Text diagnostics (also on the console)    |

<!-- markdownlint-enable MD013 -->

A step summary table shows the per-tool verdicts. To surface gosec
findings in the GitHub Security tab, upload `gosec.sarif` with
`github/codeql-action/upload-sarif` from the calling workflow.

## Input Constraints

The action validates every input before use and fails closed on
anything outside the expected form:

- Tool version inputs must start with `v` and accept the characters
  `A-Z a-z 0-9 . -` and nothing else; `latest` is not a valid value
- Booleans accept `true` or `false`; at least one tool must stay
  enabled
- `go_version` accepts `A-Z a-z 0-9 . -`

## Path Constraints

Relative values for `path_prefix` and `output_directory` resolve
against `GITHUB_WORKSPACE`, not the current working directory, so
behaviour stays deterministic when a calling workflow sets a custom
working directory. `path_prefix` must resolve within
`GITHUB_WORKSPACE`, `go_version_file` must resolve within the
project directory (`path_prefix`), and `output_directory` must
resolve within `GITHUB_WORKSPACE` or `RUNNER_TEMP`. Paths that
escape these locations fail the action, preventing scans or writes
against arbitrary runner filesystem locations.

## Verdict and Soft-Fail Semantics

Each tool's verdict comes from its documented exit codes
(govulncheck: `3` means findings; staticcheck: `1` means findings;
gosec: non-zero with a SARIF report produced). With other exit codes
the scan itself broke, and the action fails regardless of
`permit_fail`. With `permit_fail: true`, tool findings report through
the outputs and step summary while the action exits with a zero
status; with the default `permit_fail: false`, findings fail the
action after every enabled tool has run and written its reports.

## Toolchain Selection

With `go_version` set, the action installs that exact version. With
`go_version` empty, it reads the version from `go_version_file`
(default `go.mod`) under `path_prefix`. The pinned govulncheck
release requires a recent Go toolchain: the golang.org/x/vuln
v1.5.0 `go.mod` declares `go 1.25.0`, so `go install` refuses to
build it on older toolchains. For legacy modules with ancient `go`
directives (for example Jenkins-era Gerrit projects on `go 1.16`),
pass an explicit modern `go_version`:

<!-- markdownlint-disable MD046 -->

```yaml
steps:
  - name: "Audit nested legacy module"
    uses: lfreleng-actions/go-audit-action@main
    with:
      path_prefix: 'src/k8splugin'
      go_version: '1.25.11'
      gosec: 'true'
      staticcheck: 'true'
      permit_fail: 'true'
```

<!-- markdownlint-enable MD046 -->

## Implementation Details

<!-- markdownlint-disable MD013 -->

1. **Input Validation**: Validates booleans, pinned tool version strings and path boundaries before use; requires at least one enabled tool
2. **Toolchain Setup**: Installs Go via the pinned `actions/setup-go` with `cache: false` (organisation security stance on cache poisoning)
3. **Per-Tool Steps**: Each enabled tool installs into a scratch directory under `RUNNER_TEMP` (removed on step exit) and scans `./...` in the project directory, writing reports without failing the action on findings
4. **Aggregation**: A final step normalises per-tool verdicts (`true`/`false`/`skipped`), writes the step summary table and applies the `permit_fail` policy

<!-- markdownlint-enable MD013 -->

## Notes

- The tools analyse the module's dependency graph, so runs on large
  projects download modules through the proxy; govulncheck runs
  twice (text and JSON formats) with the second run served from the
  build cache
- The `go install` builds happen outside the project directory, so
  the audited module's own `go.mod` never influences which tool
  version installs

[pre-commit.ci results page]: https://results.pre-commit.ci/latest/github/lfreleng-actions/go-audit-action/main
[pre-commit.ci status badge]: https://results.pre-commit.ci/badge/github/lfreleng-actions/go-audit-action/main.svg
