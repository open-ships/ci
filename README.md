# Open Ships CI

Shared, versioned GitHub Actions policy for Open Ships repositories.

Changes to reusable workflows are linted in this repository before a new
semantic-version tag is published. A caller remains on its reviewed policy
version until its exact tag reference is deliberately updated.

## Go checks

`.github/workflows/check-go.yaml` owns checkout, Go setup, tool installation,
lint and security execution, Bash command execution, and artifact retention.
Callers own triggers, platform and compatibility matrices, dependencies between
jobs, and repository-specific checks. Every check job should call this workflow;
do not add another checkout/setup/install sequence in a caller.

```yaml
jobs:
  test:
    strategy:
      fail-fast: false
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
    uses: open-ships/ci/.github/workflows/check-go.yaml@v1.2.0
    with:
      runner: ${{ matrix.os }}
      command: go test ./...

  lint:
    uses: open-ships/ci/.github/workflows/check-go.yaml@v1.2.0
    with:
      lint: true

  secure:
    uses: open-ships/ci/.github/workflows/check-go.yaml@v1.2.0
    with:
      security: true

  release-gate:
    if: ${{ always() }}
    needs: [test, lint, secure]
    uses: open-ships/ci/.github/workflows/gate.yaml@v1.2.0
    with:
      results: ${{ toJSON(needs) }}
```

Grant check workflows `contents: read`. Keep their workflow name `CI` when a
release workflow listens for successful `CI` runs. `gate.yaml` rejects failed,
cancelled, skipped, and empty dependency sets. List every required job in
`needs`; `if: always()` ensures a skipped dependency cannot bypass the gate.

The check and release workflows default to Go **1.26.8**. Routine callers omit
`go-version`; only explicit compatibility jobs override it. Checks set
`GOTOOLCHAIN=local` so module requirements cannot silently switch toolchains.
The shared check toolset is:

| Tool | Version | Enabled by |
| --- | --- | --- |
| golangci-lint | 2.12.0 | `lint: true` |
| govulncheck | 1.6.0 | `security: true` or `static-analysis: true` |
| gosec | 2.27.1 | `security: true` |
| errcheck | 1.20.0 | `static-analysis: true` |
| staticcheck | 0.7.0 | `static-analysis: true` |
| actionlint | 1.7.12 | `static-analysis: true` |
| just | 1.58.0 | `setup-just: true` |
| Node | 22.23.2 | `browser: true` |

`browser` also runs `npm ci` and installs Playwright Chromium. The repository's
lockfile owns browser-test dependencies. `static-analysis` supplies binaries for
repository scripts; those scripts must use the supplied tools when
`OPEN_SHIPS_CI=true`, rather than downloading their own CI versions. Local
developer recipes can still manage their own installations.

`command` is reviewed Bash code, executed with `-e -o pipefail` on every OS.
Pass dispatch inputs through the `environment` JSON object and quote them in
the command, so user-provided values stay data:

```yaml
with:
  environment: >-
    {"FUZZTIME":${{ toJSON(inputs.fuzztime || '5m') }}}
  command: ./scripts/fuzz.sh
```

Use `artifact-name` and `artifact-path` for evidence retained on success or
failure; a platform matrix can set `artifact-enabled` to select the upload OS.
Use `failure-artifact-name` and `failure-artifact-path` for failing fuzz seeds.
`timeout-minutes` defaults to 30 and can be raised for bounded scheduled runs.
Use `fetch-depth: 0` for release metadata checks and `cache: false` for
repositories without a Go module. Gosec rule exclusions stay explicit in caller
inputs; generated files and `.claude` worktrees are excluded by default.

## Caller migration and rollout

The September 2026 audit found that all five repositories already shared release
automation, while their ordinary and scheduled checks installed tools separately.
The v1.2.0 migration routes every workflow job through this repository:

| Repository | Checks retained through the shared toolset |
| --- | --- |
| n2k | Three platforms, PGN sync, conformance, race/fuzz/soak evidence, lint, security, release metadata, weekly reliability |
| n2k-cli | Three platforms, install/uninstall smoke checks, race, lint, security |
| teleop | Three platforms, CGO-disabled tests, race, coverage, formatting, vet |
| beacon | Three platforms, build, browser, race, lint, security, vessel resource/recovery gates |
| statemachine | Three platforms, static analysis, 90% coverage, SQLite integration, benchmarks, minimum/current Go compatibility, smoke/nightly fuzzing |

This preserves each project's required checks. For example, teleop continues
using the Go built-in checks; the migration does not silently add new golangci
or gosec policy. Statemachine retains `coverage.out` and `benchmark.out` together
in its `source-assurance-<sha>` artifact.

Merge the shared policy **before** merging caller changes:

1. Merge and run this repository's `CI`, including the local reusable workflow
   smoke matrix and aggregate gate. Review the exact commit being released.
2. Publish a new annotated `v1.2.0` tag at that passing commit, or pin callers to
   the exact reviewed commit SHA. Never move an existing version tag.
3. Merge caller updates to that policy version, preserving `CI` names and release
   permissions. Update branch protection if it requires the old job check names;
   reusable workflows introduce nested check names such as `test / check`.
4. Verify caller CI and the existing release follow-up on `main`.

For coordinated pull requests, callers can pin the shared policy PR's exact
commit SHA so their checks run before a version tag exists. Keep those caller
PRs dependent on the shared policy PR, and update all pins if that commit changes.
References to the `v1.2.0` tag only resolve after publication.

## Go releases

`.github/workflows/release-go.yaml` is a reusable release workflow for Go
projects. A caller passes the commit from a successful `CI` workflow run and
selects one of two distribution strategies:

- `source` verifies the module, retains coverage evidence, builds a deterministic
  source archive, and can optionally publish `coverage.out`.
- `goreleaser` builds the repository's `.goreleaser.yaml` configuration without
  publishing, then publishes the resulting archives through the shared policy.

Callers may also set `container-image` to publish a multi-architecture image
tagged with the release version and `latest`. Container publication includes
BuildKit provenance and SBOM attestations plus a GitHub build-provenance
attestation, and must succeed before the GitHub release is published.

Both strategies:

- serialize version selection and ignore superseded untagged commits;
- use `VERSION` as the next baseline and increment patch versions thereafter;
- create an annotated Git tag;
- publish a GitHub release with SHA-256 checksums, an SBOM, and toolchain evidence;
- generate separate build-provenance and SBOM attestations for distributed
  artifacts; and
- resume an annotated tag whose GitHub release was not published.

Callers must grant `contents: write`, `id-token: write`, `attestations: write`,
and `artifact-metadata: write`. They must guard the privileged reusable job so
it accepts only a successful `push` CI run from the caller repository. Reference
an exact semantic-version tag such as `v1.2.0`; do not reference `main` or a
moving major-version tag. Callers that set `container-image` must additionally
grant `packages: write`.
