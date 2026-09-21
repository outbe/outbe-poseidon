# Security policy

## Release signing

Starting with the first release tagged after this file lands (expected: **v0.3.5**, the cog-bumped patch following PR `release/sigstore-signing`), every artifact attached to a `outbe-poseidon` GitHub Release is signed with [sigstore cosign](https://docs.sigstore.dev/cosign/overview/) keyless OIDC and carries an [SLSA v1.0](https://slsa.dev/spec/v1.0/) build-provenance attestation minted by `actions/attest-build-provenance`.

### Signed artifacts

For each release ≥ the cutoff, the following files ship alongside the regular release assets:

| File | What it is |
|---|---|
| `outbe-poseidon-X.Y.Z.crate` | The packaged .crate attached to the GitHub Release and then uploaded to crates.io. |
| `outbe-poseidon-X.Y.Z.crate.sig` | cosign blob signature (raw, base64). |
| `outbe-poseidon-X.Y.Z.crate.pem` | Fulcio-issued ephemeral signing cert (PEM). |
| `SHA256SUMS` | SHA-256 of the .crate. |
| `SHA256SUMS.sig` / `SHA256SUMS.pem` | cosign sig + cert for the manifest. |

Build-provenance attestations are not attached to the Release — they live in GitHub's attestation store and are queried via `gh attestation verify`.

### Threat model

The signing pipeline protects against:

- **Tampered binaries on the Release page.** A re-uploaded `.crate` or `SHA256SUMS` won't verify against the original cert + sig.
- **A compromised crates.io API token.** The same maintainer who can `cargo publish` cannot mint a sigstore signature whose Fulcio cert identity matches `https://github.com/outbe/outbe-poseidon/.github/workflows/release.yml@refs/heads/main` (the manual `workflow_dispatch` release). That identity is only obtainable from inside a GitHub Actions run of this repo's `release.yml` workflow. Releases through v0.11.1 also accepted `@refs/tags/vX.Y.Z` from the old tag-push path; the verify regex still lists that alternative.
- **A forged or locally rebuilt release artifact.** Same identity pin as above.
- **A typo or mis-targeted action update** silently weakening verification. The `verify-release` job hard-fails the workflow on any bad signature before crates.io upload; an upstream change that breaks the cosign sign-blob flow is visible immediately.

It does **not** protect against:

- A compromise of `github.com/outbe/outbe-poseidon` itself (an attacker with push access to `main` can edit the workflow to remove or weaken signing).
- A compromise of the sigstore public-good trust root (Fulcio CA, Rekor transparency log). The verify recipe trusts sigstore's TUF root by default.
- Tampering with the crates.io copy of the tarball. crates.io has no first-party signing channel; the GH-Release-attached `.crate` is packaged from the same tagged tree that `cargo publish` uploads, so a paranoid consumer can `cargo fetch`, hash, and compare against `SHA256SUMS`.
- Existing (pre-cutoff) releases. Those are **not** retroactively signed — see [the spec's D10 rationale](https://github.com/outbe/outbe-poseidon/blob/main/SECURITY.md#retroactive-signing).

### Verification recipe

You need [cosign](https://docs.sigstore.dev/cosign/installation/) and [`gh`](https://cli.github.com/) on `$PATH`.

```sh
# Pick a signed release.
REPO=outbe/outbe-poseidon
TAG=v0.3.5  # or any release ≥ the cutoff
ARTIFACT=outbe-poseidon-${TAG#v}.crate

# Download the artifact + its sig + cert.
gh release download "$TAG" --repo "$REPO" \
  --pattern "$ARTIFACT" \
  --pattern "$ARTIFACT.sig" \
  --pattern "$ARTIFACT.pem"

# Verify. The --certificate-identity-regexp pins the signer to a
# run of THIS repo's release.yml. Manual releases sign as
# @refs/heads/main; releases through v0.11.1 also signed as
# @refs/tags/vX.Y.Z. Any mismatch is a fail.
cosign verify-blob \
  --certificate "$ARTIFACT.pem" \
  --signature   "$ARTIFACT.sig" \
  --certificate-identity-regexp \
    '^https://github\.com/outbe/outbe-poseidon/\.github/workflows/release\.yml@refs/(heads/main|tags/v[0-9]+\.[0-9]+\.[0-9]+)$' \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  "$ARTIFACT"

# Optional: also verify the SLSA build-provenance attestation.
gh attestation verify "$ARTIFACT" --repo "$REPO"
```

Repeat for `SHA256SUMS` (and any other artifact) to verify the whole release set. CI's own `verify-release` job runs the same loop on every GitHub Release before crates.io upload; a green `verify-release` is your signal that the regex above is the correct one for that release.

### Retroactive signing

Releases tagged **before** the cutoff are not signed. Backfilling would mint signatures whose Fulcio identity reads "a manual workflow_dispatch on YYYY-MM-DD by a maintainer," not "a tag-triggered run of the original release," which is weaker provenance than the absence of a signature — and potentially misleading to consumers who don't read the fine print. The next manual release supersedes the unsigned release for any new consumer.

If you need to verify an unsigned release (≤ v0.3.1), you're out of band — diff the `.crate` against the crates.io copy or pin to a signed release.

## Cutting a release

Releases start only from **Actions → release → Run workflow**. Merging to `main` does not cut a tag.

1. Merge everything for the release to `main` and wait for CI (`check.yml`) to go green.
2. **Actions → release → Run workflow**.
3. Use workflow from: **main**.
4. Tag: `vX.Y.Z` (must be greater than the current `Cargo.toml` version; the `v` prefix is optional).
5. Run workflow.

The workflow then:

1. Bumps `Cargo.toml` and `Cargo.lock`, writes `CHANGELOG.md`, commits `chore(version): vX.Y.Z`, and pushes tag `vX.Y.Z`.
2. Packages the crate, creates the signed GitHub Release, and verifies signatures.
3. Publishes to crates.io (skips if that version already exists).

Re-running the same tag skips the bump and retries packaging, the GitHub Release, and crates.io from the existing tag.

## CI secrets

Release automation needs two repository secrets. Set them at **Settings → Secrets and variables → Actions → New repository secret** on `outbe/outbe-poseidon`. Organization secrets also work if this repo is in the selected set. Both secrets are read by `release.yml`: `RELEASE_TOKEN` by the `bump` job, `CARGO_REGISTRY_TOKEN` by the `publish` job.

Do not put either value in the workflow file, in `Cargo.toml`, or in a commit.

### `RELEASE_TOKEN`

A GitHub personal access token used by the `bump` job to push the version commit to `main` and the `v*` tag. The default `GITHUB_TOKEN` cannot push to a protected `main`.

Create the token as a user who is allowed to push to `main` (add that user to the ruleset bypass list if `main` is protected).

Classic PAT (simplest):

1. GitHub → **Settings → Developer settings → Personal access tokens → Tokens (classic) → Generate new token**.
2. Scope: `repo` (contents + tags).
3. Copy the token once. Store it as repository secret `RELEASE_TOKEN`.

Fine-grained PAT:

1. Resource owner: the `outbe` org (org owner must approve).
2. Only this repository.
3. Permissions: **Contents: Read and write**.
4. Same bypass-list requirement as above.

Rotate it if it leaks. After rotation, update the repository secret in place; no workflow change is needed.

### `CARGO_REGISTRY_TOKEN`

A crates.io API token used by the `publish` job. Trusted Publishing cannot be used for the first upload of a new crate name; this token is the bootstrap path.

1. Sign in at [crates.io](https://crates.io/) with the GitHub account that should own `outbe-poseidon`.
2. Verify the account email under [Account Settings](https://crates.io/settings/profile).
3. Create a token at [API tokens](https://crates.io/settings/tokens):
   - First publish of this crate: enable **publish-new** and **publish-update**.
   - After 0.11.0 (or whichever version lands first) is on crates.io: **publish-update** is enough.
4. Copy the token once. Store it as repository secret `CARGO_REGISTRY_TOKEN`.
5. The GitHub user who publishes first becomes the crates.io owner. Add teammates with `cargo owner --add <github-username>` (or `github:outbe:<team>`).

Revoke the token on crates.io if it leaks, then put the replacement in the same secret name.

After the crate exists, owners can switch the job to [Trusted Publishing](https://crates.io/docs/trusted-publishing) (OIDC, no long-lived token) and delete `CARGO_REGISTRY_TOKEN`. Until then the secret is required; the `publish` job fails closed if it is missing.

### Check that both secrets are present

1. **Settings → Secrets and variables → Actions** lists `RELEASE_TOKEN` and `CARGO_REGISTRY_TOKEN`.
2. `RELEASE_TOKEN` is read when you run **Actions → release → Run workflow**. A missing value fails the `bump` job with a pointer back here.
3. `CARGO_REGISTRY_TOKEN` is read by `release.yml`'s `publish` job, which runs after the signed GitHub Release verifies. The first crates.io upload is the next manual release whose version is not yet on the registry.

## Reporting vulnerabilities

For security issues in `outbe-poseidon` itself (not the signing pipeline), open a [private security advisory](https://github.com/outbe/outbe-poseidon/security/advisories/new) on GitHub. Do not file a public issue.

## References

- [sigstore docs](https://docs.sigstore.dev/)
- [SLSA v1.0 specification](https://slsa.dev/spec/v1.0/)
- [`actions/attest-build-provenance`](https://github.com/actions/attest-build-provenance)
- [`sigstore/cosign-installer`](https://github.com/sigstore/cosign-installer)
