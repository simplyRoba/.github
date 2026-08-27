# simplyRoba shared GitHub workflows

Reusable GitHub Actions workflows for the `simplyRoba` repositories.

## Rust prereleases

`.github/workflows/rust-create-prerelease.yml` creates a `X.Y.Z-next.N` GitHub prerelease from the exact HEAD of the calling repository's pending Release Please pull request.

The workflow has no inputs. The caller must pass its existing Release Please GitHub App secrets and allow repository contents to be read:

```yaml
jobs:
  prerelease:
    permissions:
      contents: read
    uses: simplyRoba/.github/.github/workflows/rust-create-prerelease.yml@<reviewed-commit-sha>
    secrets: inherit
```

The caller repository must define `RELEASE_BOT_CLIENT_ID` and `RELEASE_BOT_PRIVATE_KEY`. The App must be installed on that repository with `contents: write` and `pull-requests: read`; its installation token creates the tag and release so the resulting `release: published` event can start downstream workflows.

Consumers should pin a reviewed immutable commit while the workflow is initially tested. Once validated, publish a versioned workflow ref (for example `v1`); breaking interface changes must use a new major ref.

## License

GPL-3.0-only. See [LICENSE](LICENSE).
