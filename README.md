# Greentic Designer Extension Publish

GitHub Action that builds a Greentic Designer Extension project (any of the
four kinds — design / bundle / deploy / provider), packs it deterministically,
and publishes the resulting `.gtxpack` to your registry of choice.

Supports all three registry backends that the `gtdx` CLI ships:

- **Greentic Store HTTP** — set `store-url` + `store-token`, action auto-writes config
- **OCI registries** — `oci://<host>/<namespace>[/<artifact>]` (GHCR, Docker Hub, Harbor, Azure ACR)
- **Local filesystem** — `file://<absolute-path>` (mostly for dry-run PR checks)

## Quick start — publish to Greentic Store on every tag

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags: ['v*']

jobs:
  publish:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: greenticai/greentic-designer-extension-action@v1
        with:
          store-url: http://62.171.174.152:3030
          store-token: ${{ secrets.GREENTIC_STORE_TOKEN }}
          version: ${{ github.ref_name }}
```

That's it. Action writes `~/.greentic/{config,credentials}.toml` internally
and pushes via gtdx.

**Prerequisite:** add `GREENTIC_STORE_TOKEN` as a repo secret.

The Store server issues two bearer token types:

| Type | Format | Lifetime | Use for |
|------|--------|----------|---------|
| **API token** | `gts_...` | long-lived (until revoked) | **CI / this action** ← recommended |
| JWT | `eyJ...` (3 dot-separated base64) | 24 hours | interactive `gtdx` runs |

For CI, always use an API token — JWTs silently expire and break your
release workflow 24 hours after issue. Create one (named e.g.
`ci-github-actions`) via the Store dashboard or API, then paste its value
into the repo secret.

If you only have a JWT (for testing / first-time setup):

```bash
curl -X POST http://62.171.174.152:3030/api/v1/auth/register \
  -H 'Content-Type: application/json' \
  -d '{"name":"your name","handle":"yourhandle","email":"you@example.com","password":"<pw>"}'
# response has a "token" field (JWT, 24h) and "publisher.allowed_prefixes".
```

Your extension's `describe.metadata.id` must start with one of the
`allowed_prefixes` returned by the register response.

## Quick start — publish to GitHub Container Registry on every tag

```yaml
# .github/workflows/release.yml
name: Release

on:
  push:
    tags: ['v*']

jobs:
  publish:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write   # required for pushing to ghcr.io
    steps:
      - uses: actions/checkout@v4

      - uses: greenticai/greentic-designer-extension-action@v1
        with:
          registry: oci://ghcr.io/${{ github.repository_owner }}/${{ github.event.repository.name }}
          version: ${{ github.ref_name }}
```

The action auto-injects `GITHUB_TOKEN` so no secret setup is required for GHCR
as long as the job has `permissions: packages: write`.

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `registry` | conditional | — | Target registry URI. `oci://...`, `file://...`, `local`, or a named Store entry. Optional when `store-url` is set. |
| `store-url` | no | — | Shorthand: Greentic Store HTTP URL. Action auto-writes `~/.greentic/config.toml`. |
| `store-token` | no | — | JWT bearer for the Store in `store-url`. Action writes `~/.greentic/credentials.toml` mode 0600. |
| `manifest` | no | `./Cargo.toml` | Path to the extension project's `Cargo.toml`. |
| `version` | no | *(from describe.json)* | Override `describe.json` version — useful when tags drive CI. |
| `force` | no | `false` | Overwrite an existing version in the target registry. |
| `dry-run` | no | `false` | Validate + build + pack but skip the registry write. Useful for PR checks. |
| `oci-token` | no | *(`GITHUB_TOKEN`)* | Bearer/PAT for `oci://` registries. Action falls back to `GITHUB_TOKEN` automatically. |
| `format` | no | `human` | `human` or `json` output format. |
| `gtdx-ref` | no | *(latest tag)* | Git ref (tag or commit) of `greentic-designer-extensions` to build gtdx from. |
| `rust-toolchain` | no | `1.94` | Rust toolchain (>= 1.94 for edition 2024). |
| `cargo-component-version` | no | *(latest)* | `cargo-component` version pin. |

## Outputs

| Output | Description |
|--------|-------------|
| `sha256` | SHA-256 of the published `.gtxpack`. |
| `registry-url` | URL of the published artifact (file://, https://, or OCI manifest URL). |
| `ext-id` | Extension id that was published. |
| `version` | Published version. |

Example of consuming outputs:

```yaml
      - uses: greenticai/greentic-designer-extension-action@v1
        id: publish
        with:
          registry: oci://ghcr.io/${{ github.repository_owner }}/my-ext

      - name: Comment on release
        run: |
          echo "Published ${{ steps.publish.outputs.ext-id }}@${{ steps.publish.outputs.version }}"
          echo "sha256=${{ steps.publish.outputs.sha256 }}"
          echo "url=${{ steps.publish.outputs.registry-url }}"
```

## Example — PR dry-run validation

Fail the PR if the extension doesn't build or the describe.json isn't schema-valid:

```yaml
name: Validate

on:
  pull_request:
    branches: [main]

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: greenticai/greentic-designer-extension-action@v1
        with:
          registry: local
          dry-run: 'true'
```

## Example — publish to a private Store server

```yaml
      - uses: greenticai/greentic-designer-extension-action@v1
        with:
          store-url: https://my-private-store.example.com
          store-token: ${{ secrets.STORE_TOKEN }}
```

## How it works

Under the hood this action:

1. Installs the requested Rust toolchain + `wasm32-wasip2` target (`dtolnay/rust-toolchain@master`).
2. Caches `~/.cargo/bin` keyed on the action inputs, so `cargo install` only runs on the first run (or when inputs change).
3. Installs `cargo-component` and `gtdx` (from the `greentic-designer-extensions` git repo, since the crates aren't yet published on crates.io).
4. Runs `gtdx publish` with your inputs forwarded as flags.
5. Parses the JSON receipt written to `./dist/publish-*.json` and exposes `sha256` / `registry-url` / `ext-id` / `version` as action outputs.

## Versioning

This action follows semantic versioning. Pin to a major tag (`@v1`) for
automatic non-breaking updates or to a specific release (`@v1.0.0`) for
deterministic builds.

## License

Same license as the upstream `greentic-designer-extensions` repo.
