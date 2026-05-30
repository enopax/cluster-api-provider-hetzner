# Enopax CAPH image build

This document describes how the **enopax fork** of Cluster API Provider Hetzner
(CAPH) publishes its container image. It is specific to this fork and does not
apply to upstream syself/cluster-api-provider-hetzner.

## Background

This fork carries a small "diagnostics-flags" patch on top of upstream CAPH
(adds `--diagnostics-address` / `--insecure-diagnostics` to the manager — see
[issue #2](https://github.com/enopax/cluster-api-provider-hetzner/issues/2)).
The resulting manager image has, until now, been built and pushed **by hand**;
the most recent published image is `ghcr.io/enopax/caph:v1.0.7-diagnostics`.

This fork is a **retire candidate** (issue #2): we may drop it entirely once
upstream exposes the flags or we change approach. The build tooling here is
therefore deliberately kept minimal.

## Image namespace transition

As part of adopting the org-wide package naming schema
(enopax/enopax PR #15), the image is being moved to a namespaced path:

| Path | Status |
| --- | --- |
| `ghcr.io/enopax/caph` | Legacy. Kept working during the transition. |
| `ghcr.io/enopax/images/caph` | New namespaced path. Preferred going forward. |

During the cutover the image is published to **both** paths so existing
consumers keep working. Known consumers (not changed by this repo):

- `enopax/caph-helm` — chart pins `ghcr.io/enopax/caph:v1.0.7-diagnostics`.
- `enopax/k0cluster` — manifests reference CAPH.

These should be repointed to `ghcr.io/enopax/images/caph` as a follow-up before
the legacy path is retired.

## Publishing via CI (preferred)

Use the `Publish Enopax CAPH Image` workflow
(`.github/workflows/publish-enopax-image.yml`). It is a manual
(`workflow_dispatch`) workflow that builds the manager image once and pushes it
to **both** paths above.

1. Go to **Actions → Publish Enopax CAPH Image → Run workflow**.
2. Set the `tag` input (e.g. `v1.0.7-diagnostics`).
3. Run. The image is pushed to:
   - `ghcr.io/enopax/caph:<tag>`
   - `ghcr.io/enopax/images/caph:<tag>`

The workflow builds the local `builder` image first (Go toolchain,
`images/builder/Dockerfile`) and passes it as the `builder_image` build-arg to
`images/caph/Dockerfile`, mirroring the Makefile `docker-build` target.

## Publishing manually (fallback)

The image can still be built and pushed by hand using the existing Makefile
targets with registry/name/tag overrides. Publish to both paths during the
transition:

```sh
TAG=v1.0.7-diagnostics

# Build the builder image (Go toolchain used to compile the manager).
docker build -t caph-builder:${TAG} -f images/builder/Dockerfile .

# Build the manager image.
docker build \
  --build-arg builder_image=caph-builder:${TAG} \
  --build-arg ARCH=amd64 \
  -t ghcr.io/enopax/caph:${TAG} \
  -t ghcr.io/enopax/images/caph:${TAG} \
  -f images/caph/Dockerfile .

# Push to both paths.
docker push ghcr.io/enopax/caph:${TAG}
docker push ghcr.io/enopax/images/caph:${TAG}
```

> Note: upstream's `build.yml` / `release.yml` push to `ghcr.io/syself` and are
> unrelated to the enopax image; they are intentionally left untouched.
