# simplyRoba shared GitHub workflows

Reusable GitHub Actions workflows for the `simplyRoba` repositories.

## Versioning

Use `@v1` as the recommended consumer reference. It advances automatically when a compatible stable `v1.x.y` release is published. Use an exact tag such as `@v1.0.0` when an immutable version is required; `v1.x.y` tags never move. Breaking shared-workflow interface changes require a new major tag such as `v2`.

## Rust release workflows

The three workflows below form one shared Rust release system. Release Please configuration (`release-please-config.json`, `.release-please-manifest.json`), the `Dockerfile` and CI stay in the calling repository.

```text
push to main            -> rust-release-please.yml   -> Release Please PR / release
manual dispatch         -> rust-create-prerelease.yml -> GitHub prerelease vX.Y.Z-next.N
release: published      -> rust-publish-release.yml  -> release binaries + GHCR image
```

### `rust-release-please.yml`

Runs `googleapis/release-please-action@v5` against the calling repository using its own Release Please config and manifest.

* **Inputs:** none
* **Secrets:** `RELEASE_BOT_CLIENT_ID`, `RELEASE_BOT_PRIVATE_KEY` (GitHub App with `contents: write` and `pull-requests: write` on the caller; all Release Please API calls use its installation token)
* **Caller permissions:** `contents: read`

```yaml
on:
  push:
    branches: [main]

jobs:
  release-please:
    permissions:
      contents: read
    uses: simplyRoba/.github/.github/workflows/rust-release-please.yml@v1
    secrets: inherit
```

### `rust-create-prerelease.yml`

Creates a `X.Y.Z-next.N` GitHub prerelease from the exact HEAD of the calling repository's pending Release Please pull request. It commits a synthetic Cargo version bump on top of that HEAD, tags it and publishes a GitHub prerelease; the Release Please PR itself is left untouched.

* **Inputs:** none
* **Secrets:** `RELEASE_BOT_CLIENT_ID`, `RELEASE_BOT_PRIVATE_KEY` (GitHub App with `contents: write` and `pull-requests: read`; its installation token creates the tag and release so the resulting `release: published` event can start downstream workflows)
* **Caller permissions:** `contents: read`

```yaml
on:
  workflow_dispatch:

jobs:
  prerelease:
    permissions:
      contents: read
    uses: simplyRoba/.github/.github/workflows/rust-create-prerelease.yml@v1
    secrets: inherit
```

### `rust-publish-release.yml`

Builds the released tag for `linux/amd64` and `linux/arm64`, uploads `<binary-name>-linux-amd64` / `<binary-name>-linux-arm64` to the GitHub release, and pushes a multi-platform image to `ghcr.io/${{ github.repository }}`. The Docker build context expects the binaries at `release-artifacts/linux-amd64/<binary-name>` and `release-artifacts/linux-arm64/<binary-name>`.

Every job checks out `refs/tags/${{ github.event.release.tag_name }}`, so the released source is built rather than the caller's default branch. Stable Docker aliases (`{{major}}`, `{{major}}.{{minor}}`, `latest`) are gated on `github.event.release.prerelease`, so a prerelease only publishes its own version tags.

* **Inputs:** `binary-name` (required, string) — the binary produced by `cargo build --release`
* **Secrets:** none; the caller's `GITHUB_TOKEN` is used for release assets and the GHCR login
* **Caller permissions:** `contents: write`, `packages: write`

```yaml
on:
  release:
    types: [published]

jobs:
  publish-release:
    permissions:
      contents: write
      packages: write
    uses: simplyRoba/.github/.github/workflows/rust-publish-release.yml@v1
    with:
      binary-name: my-binary
```

## License

GPL-3.0. See [LICENSE](LICENSE).
