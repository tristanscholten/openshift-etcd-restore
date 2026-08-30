# Release Policy

## Versioning

Use [Semantic Versioning 2.0.0](https://semver.org/): `MAJOR.MINOR.PATCH`.

- **MAJOR:** incompatible API or behaviour change.
- **MINOR:** backward-compatible feature.
- **PATCH:** backward-compatible bug or security fix.

Release versions use a `v`-prefixed immutable Git tag, for example `v1.4.2`.

## Required release procedure

1. Update every authoritative version declaration in the source tree.
2. Run the full project verification suite and security checks.
3. Commit the exact release source on a feature branch and merge through a PR; never commit directly to `main`.
4. Build, test, and publish immutable artifacts from that exact commit.
5. Create and push an annotated signed-where-available Git tag: `vMAJOR.MINOR.PATCH`.
6. For production systems, deploy the immutable tagged artifact through GitOps and record the rollback tag.

Never retag, overwrite, or reuse a published release version. If a released artifact is defective, publish the next patch version.
