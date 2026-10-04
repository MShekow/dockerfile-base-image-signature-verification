# Base image verification during a Docker build

Docker builds normally pull a base image without checking whether it was signed
by a trusted publisher. Pinning the image by digest fixes which content is used,
but does not verify who published it.

This repo demonstrates how to use multi-stage Dockerfiles to verify signatures
of base images, using Notation and Cosign, while also verifying the Notation/
Cosign CLI tools themselves:
Notation checks Microsoft-signed MCR images, and Cosign checks Sigstore-signed
images against an expected signer identity.

In both examples, you establish a **trust policy** that explicitly defines
**who** (e.g., which certificate authorities and signer identities) you trust,
such that a valid signature from an unknown publisher is not sufficient.

## Notation: Microsoft Container Registry

Use [`Dockerfile-notation`](Dockerfile-notation):

```sh
docker buildx build -f Dockerfile-notation --load --no-cache-filter verification -t notation-example .
```

The `verification-tooling` stage uses `mcr.microsoft.com/azurelinux/base/core:3.0`, pinned
by digest. It installs packages using only the `azurelinux-official-base`
repository to avoid downloading unnecessary repository indices.
It downloads Notation 1.3.2 from GitHub and checks its
archive against the release's SHA-256 checksums downloaded in advance into
[`notation_1.3.2_checksums.txt`](notation_1.3.2_checksums.txt). The Microsoft
Supply Chain root CA is also downloaded in advance and stored next to the
Dockerfile as `msft_supply_chain.crt`. The stage copies and imports this
certificate and a strict trust policy, then verifies `VERIFICATION_IMAGE`.
The `verification` stage inherits this tooling, checks that both image references
are under `mcr.microsoft.com/`, verifies `BASE_IMAGE`, then creates `/verified`.
The final stage copies this dummy file, making
successful verification a required build dependency.

The default base is Microsoft's OpenJDK 25 distroless image, pinned by digest.
Override it with another signed MCR image:

```sh
docker buildx build -f Dockerfile-notation --load --no-cache-filter verification \
  --build-arg BASE_IMAGE='mcr.microsoft.com/<repository>@sha256:<digest>' \
  -t notation-example .
```

Use a digest so Notation and Docker consume the same immutable image. Tags can
change between verification and Docker's base-image resolution. The
`--no-cache-filter verification` option reruns the `BASE_IMAGE` signature check
on each build while allowing Docker to reuse the cached package installation,
Notation setup, and `VERIFICATION_IMAGE` signature check in the
`verification-tooling` stage. To rerun both image signature checks, use:

```sh
docker buildx build -f Dockerfile-notation --load \
  --no-cache-filter verification-tooling,verification \
  -t notation-example .
```

Notation supports exact repository scopes or `"*"`, not `mcr.microsoft.com/*`.
The policy therefore uses `"*"` and the verification stage rejects references
outside `mcr.microsoft.com/`. This covers all MCR repositories, while requiring
a valid signature from Microsoft's documented SCD Products signing identity;
unsigned images or images signed by other identities fail verification.

### Establishing trust with Notation

The certificate and [`trustpolicy.json`](trustpolicy.json) express your trust
decisions together:

1. **Choose a trust anchor.** Downloading the Microsoft Supply Chain root CA
   from Microsoft's documented certificate URL and importing it into
   `ca:supplychain` tells Notation which CA may authenticate image signers.
   The certificate is already included here as `msft_supply_chain.crt`;
   selecting this certificate is a decision to trust that Microsoft CA.
2. **Authorize a publisher.** `trustedIdentities` in `trustpolicy.json` selects
   Microsoft's `Microsoft SCD Products RSA Signing` certificate subject.
   Trusting the CA alone does not authorize every certificate it issues.
3. **Define scope and enforcement.** The policy references `ca:supplychain`
   and requires `strict` verification. Its `registryScopes` is `"*"`, with
   the Dockerfile's MCR-only check restricting the accepted image references.

Importing the policy makes these rules active. Verification succeeds only when
the image has a valid signature whose certificate chains to the selected CA
and whose signer matches the authorized identity, with all strict checks passing.

## Cosign: Sigstore-signed images

Use [`Dockerfile-cosign`](Dockerfile-cosign):

```sh
docker buildx build -f Dockerfile-cosign --load \
  --no-cache-filter verification -t cosign-example .
```

The `verification-tooling` stage uses the digest-pinned official Cosign
`v3.1.3-dev` image, which includes BusyBox. Its shell is `/busybox/sh`, not
`/bin/sh`, so the Dockerfile sets `SHELL` explicitly. No CLI download or package
installation is needed. Cosign verifies the tooling image against Sigstore's
release signer (`keyless@projectsigstore.iam.gserviceaccount.com`) and OIDC
issuer (`https://accounts.google.com`).

The `verification` stage verifies `BASE_IMAGE` and creates
`/home/nonroot/verified`, a writable location for the image's non-root user.
The final stage copies this file to `/verified`. The default final base is
`ghcr.io/mshekow/package-version-check-mcp:latest`, pinned by digest. Its expected signer is the
`MShekow/package-version-check-mcp` GitHub Actions workflow
`.github/workflows/ci-cd.yml`, running for a `vMAJOR.MINOR.PATCH` release tag.
The identity regular expression is anchored to that workflow and release-tag
format, and the OIDC issuer is `https://token.actions.githubusercontent.com`.

### Establishing trust with Cosign

This example expresses its trust policy through verification arguments rather
than a separate policy file:

- **Trust anchors:** Cosign uses the public Sigstore trust roots to validate
  signing certificates and transparency-log evidence.
- **Authorized publisher:** You set `CERTIFICATE_IDENTITY_REGEXP` to select
  the signer identity allowed in the signing certificate. Here it authorizes
  one repository's release workflow, with `^` and `$` anchoring the match to
  the full identity. This is your publisher allowlist, not merely a way to
  locate a signature.
- **Identity provider:** You set `CERTIFICATE_OIDC_ISSUER` to specify who must
  have authenticated that signer. Here it is GitHub Actions; the tooling
  image separately requires Sigstore's release identity and Google issuer.

Choose the expected identity and issuer from the publisher's documented signing
process before verification. These choices authorize that publisher; copying
the identity from any signature you encounter would merely accept its signer.
Cosign enforces both constraints alongside cryptographic and transparency-log
verification. Changing the base image's registry or digest does not automatically
change which publisher is trusted.

To verify another keyless-signed image, supply its digest, expected signer
identity regular expression, and OIDC issuer:

```sh
docker buildx build -f Dockerfile-cosign --load \
  --no-cache-filter verification \
  --build-arg BASE_IMAGE='<registry>/<repository>@sha256:<digest>' \
  --build-arg CERTIFICATE_IDENTITY_REGEXP='<anchored expected signer identity regex>' \
  --build-arg CERTIFICATE_OIDC_ISSUER='<expected OIDC issuer URL>' \
  -t cosign-example .
```

The tooling image's signer is configured separately from the final base's signer.
As with the Notation example, `--no-cache-filter verification` reruns the final
base check and reuses cached tooling verification. Use
`--no-cache-filter verification-tooling,verification` to rerun both checks.
Both default images are pinned by digest so verification and Docker consume
the same immutable images. Keep digest pinning when overriding `BASE_IMAGE`.

Cosign and Notation use different signature formats and trust configuration.
The Microsoft certificate and Notation policy apply to the Notation example;
they cannot be reused to verify those Notation signatures with Cosign. The
Cosign example requires an image published with a Cosign-compatible signature.

## References
- [Notation CLI installation](https://notaryproject.dev/docs/user-guides/installation/cli/)
- [Original checksum file](https://github.com/notaryproject/notation/releases/download/v1.3.2/notation_1.3.2_checksums.txt)
- [Microsoft certificate and signing identity](https://learn.microsoft.com/en-us/java/openjdk/containers#verify-container-image-signatures)
- [Notation trust policy scope constraints](https://github.com/notaryproject/notaryproject/blob/main/specs/trust-store-trust-policy.md#oci-trust-policy-constraints)
- [Cosign installation and container image verification](https://docs.sigstore.dev/cosign/system_config/installation/)
- [Cosign signature verification](https://docs.sigstore.dev/cosign/verifying/verify/)
