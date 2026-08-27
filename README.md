# simplyRoba shared GitHub workflows

Reusable GitHub Actions workflows for the `simplyRoba` repositories.

## Versioning

Consumers should use `@v1`. Use `@v1.0.0` when an immutable exact version is required.

### Creating the initial version and major tags

```bash
git switch main
git pull --ff-only
commit=$(git rev-parse HEAD)

git tag v1.0.0 "$commit"
git tag v1 "$commit"
git push --atomic origin refs/tags/v1.0.0 refs/tags/v1
```

### Creating a new v1 version and moving v1

```bash
git switch main
git pull --ff-only
commit=$(git rev-parse HEAD)

git tag v1.0.1 "$commit"
git tag -f v1 "$commit"

# Create the immutable version tag and force-update only the movable v1 tag.
git push --atomic origin refs/tags/v1.0.1 +refs/tags/v1
```

Verify both tags resolve to the intended commit:

```bash
git ls-remote --tags origin refs/tags/v1 refs/tags/v1.0.1
```

Rules:

* Never move or force-push a `v1.x.y` tag.
* Only the major tag (`v1`, `v2`, ...) is movable.
* A published stable `v1.x.y` GitHub Release performs the `v1` movement automatically.
* Prereleases and malformed release tags do not move a major tag.
* Breaking workflow-interface changes start at `v2.0.0` and move `v2`, never `v1`.

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
