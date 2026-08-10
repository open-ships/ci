# Open Ships CI

Shared, versioned GitHub Actions policy for Open Ships repositories.

Changes to reusable workflows are linted in this repository before a new
semantic-version tag is published. A caller remains on its reviewed policy
version until its exact tag reference is deliberately updated.

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
an exact semantic-version tag such as `v1.1.0`; do not reference `main` or a
moving major-version tag. Callers that set `container-image` must additionally
grant `packages: write`.
