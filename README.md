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

      - uses: greenticai/greentic-designer-extension-action@v2
        with:
          store-url: https://store.greentic.cloud
          store-token: ${{ secrets.GREENTIC_STORE_TOKEN }}
          signing-key-pem: ${{ secrets.EXT_SIGNING_KEY_PEM }}
          version: ${{ github.ref_name }}
```

That's it. Action writes `~/.greentic/{config,credentials}.toml` internally
and pushes via gtdx.

**Prerequisites:** add `GREENTIC_STORE_TOKEN` and `EXT_SIGNING_KEY_PEM` as repo
secrets.

> ⚠️ **Signing is mandatory by default.** `greentic-ext-runtime` refuses to load
> an extension whose `describe.json` has no `signature` object — the Store
> accepts the upload happily, and every consumer then rejects it at boot with a
> single `warn` line. Without `signing-key-pem` the action fails the job rather
> than shipping such an artifact. See [Signing](#signing).

> ⚠️ **Use `https://` for the Store server.** Over plaintext `http://` your
> bearer token traverses the network unencrypted and can be intercepted. The
> action emits a warning when `store-url` is `http://` and a token is set —
> prefer an `https://` Store URL in any environment you don't fully control.

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
curl -X POST https://store.greentic.cloud/api/v1/auth/register \
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

      - uses: greenticai/greentic-designer-extension-action@v2
        with:
          registry: oci://ghcr.io/${{ github.repository_owner }}/${{ github.event.repository.name }}
          signing-key-pem: ${{ secrets.EXT_SIGNING_KEY_PEM }}
          version: ${{ github.ref_name }}
```

The action auto-injects `GITHUB_TOKEN` so no secret setup is required for GHCR
as long as the job has `permissions: packages: write`.

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `registry` | conditional | — | Target registry URI. `oci://...`, `file://...`, `local`, or a named Store entry. Optional when `store-url` is set. |
| `store-url` | no | — | Shorthand: Greentic Store HTTP URL. Action auto-writes `~/.greentic/config.toml`. |
| `store-token` | no | — | Bearer for the Store in `store-url` — either a long-lived API token (`gts_...`, recommended for CI) or a 24h JWT. Action writes `~/.greentic/credentials.toml` mode 0600. |
| `manifest` | no | `./Cargo.toml` | Path to the extension project's `Cargo.toml`. |
| `version` | no | *(from describe.json)* | Override `describe.json` version — useful when tags drive CI. |
| `force` | no | `false` | Overwrite an existing version in the target registry. |
| `dry-run` | no | `false` | Validate + build + pack but skip the registry write. Useful for PR checks. |
| `oci-token` | no | *(`GITHUB_TOKEN`)* | Bearer/PAT for `oci://` registries. Action falls back to `GITHUB_TOKEN` automatically. |
| `format` | no | `human` | `human` or `json` output format. |
| `gtdx-version` | no | *(latest release)* | Version constraint for `greentic-extension-sdk-cli` (e.g. `1.2` or `=1.2.3`). Stable pins come from crates.io; a `-research` pin (e.g. `=1.2.4-research`) installs from the SDK git tag. |
| `rust-toolchain` | no | `1.95` | Rust toolchain (>= 1.95 for edition 2024). |
| `cargo-component-version` | no | *(latest)* | `cargo-component` version pin. |
| `signing-key-pem` | **yes** *(unless `allow-unsigned`)* | — | Ed25519 PKCS8 PEM signing key, as emitted by `gtdx keygen`. Pass a secret. See [Signing](#signing). |
| `signing-key-id` | no | *(`env`)* | Label recorded as the signature key id in registry metadata. `[A-Za-z0-9._-]+` only. |
| `expected-public-key` | no | — | Base64 ed25519 public key the produced artifact must be signed with. Fails the job on a mismatch. |
| `allow-unsigned` | no | `false` | Publish without a signature. The artifact will not load under `greentic-ext-runtime`. |

## Signing

`greentic-ext-runtime` verifies every extension it loads in three steps: the
`describe.json` inside the `.gtxpack` must carry a valid `signature`, the
`manifest.json` ledger must match the files on disk, and the signing key must be
the one **pinned for that extension id on first load** (trust on first use).

Two consequences shape this action's defaults:

1. **An unsigned publish is not a build failure — it is a silent outage.** The
   Store accepts the artifact, and every host then refuses it at boot with
   `signature verification failed: missing signature field`, logged at `warn`.
   If an older signed version is still installed, the UI looks completely
   healthy while no tenant ever receives the update. So the action **fails
   closed**: no `signing-key-pem`, no publish.
2. **The key must not change between versions.** Publishing v2 of an extension
   under a different key than v1 was pinned with is refused as a publisher
   mismatch — as unloadable as no signature at all. Set `expected-public-key` to
   the key already pinned for these ids and the action will assert it.

After `gtdx publish` returns, the action unzips the artifact it just produced,
reads `describe.json` back out, and fails if the `signature` object is missing
or was made by an unexpected key. The check runs on `dry-run: 'true'` too, which
makes a PR job a genuine pre-flight for the release.

```yaml
      - uses: greenticai/greentic-designer-extension-action@v2
        with:
          store-url: https://store.greentic.cloud
          store-token: ${{ secrets.GREENTIC_STORE_TOKEN }}
          signing-key-pem: ${{ secrets.EXT_SIGNING_KEY_PEM }}
          expected-public-key: 66L1j2dtYwoRQ2R6Bt8qkshdZmdb2IW8lLRgibxAI60=
```

Mint a key with `gtdx keygen` and store the PKCS8 PEM as a repository or
organization secret. Fork PRs get an empty secret, so keep signed publishes off
the `pull_request` trigger, or run them with `dry-run: 'true'` and
`allow-unsigned: 'true'`.

## Outputs

| Output | Description |
|--------|-------------|
| `sha256` | SHA-256 of the published `.gtxpack`. |
| `registry-url` | URL of the published artifact (file://, https://, or OCI manifest URL). |
| `ext-id` | Extension id that was published. |
| `version` | Published version. |

Example of consuming outputs:

```yaml
      - uses: greenticai/greentic-designer-extension-action@v2
        id: publish
        with:
          registry: oci://ghcr.io/${{ github.repository_owner }}/my-ext
          signing-key-pem: ${{ secrets.EXT_SIGNING_KEY_PEM }}

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
      - uses: greenticai/greentic-designer-extension-action@v2
        with:
          registry: local
          dry-run: 'true'
          # Fork PRs cannot read secrets, so there is no key to sign with. This
          # job only checks that the project builds and validates.
          allow-unsigned: 'true'
```

## Example — publish to a private Store server

```yaml
      - uses: greenticai/greentic-designer-extension-action@v2
        with:
          store-url: https://my-private-store.example.com
          store-token: ${{ secrets.STORE_TOKEN }}
          signing-key-pem: ${{ secrets.EXT_SIGNING_KEY_PEM }}
```

## How it works

Under the hood this action:

1. Installs the requested Rust toolchain + `wasm32-wasip2` target (`dtolnay/rust-toolchain`, pinned to a commit SHA).
2. Resolves the effective `gtdx` / `cargo-component` versions (the latest on crates.io when you don't pin one) and caches `~/.cargo/bin` keyed on them, so `cargo install` only runs on the first run, when you change an input, or when a newer version is published.
3. Installs `cargo-component` and `gtdx` (`greentic-extension-sdk-cli`, which ships the `gtdx` binary) from crates.io — except a `-research` `gtdx-version` pin, which installs `gtdx` from the SDK git tag (research builds aren't published to crates.io).
4. Resolves the signing key, failing the job when none was supplied and `allow-unsigned` is not set.
5. Runs `gtdx publish --sign` with your inputs forwarded as flags.
6. Parses `gtdx publish`'s stdout — the one-line JSON object (`--format json`) or the labelled human output — and exposes `sha256` / `registry-url` / `ext-id` / `version` as action outputs.
7. Unzips the produced `.gtxpack` and asserts its `describe.json` really carries a signature (and, with `expected-public-key`, the right one).

> **Runner requirement:** the signature check uses `unzip` and `jq`, and parsing
> `--format json` output uses `jq`. Both are preinstalled on GitHub-hosted
> `ubuntu-latest`. On a minimal or self-hosted runner, install them first.

## Versioning

This action follows semantic versioning. Pin to the current major tag (`@v2`)
for automatic non-breaking updates or to a specific release (`@v2.1.0`) for
deterministic builds. The older `@v1` line predates the crates.io install and
is no longer maintained — migrate to `@v2`.

## License

Same license as the upstream `greentic-designer-extensions` repo.
