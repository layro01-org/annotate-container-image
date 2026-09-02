# annotate-container-image

A GitHub composite action that writes OCI image manifest annotations recording the Git commit
SHA, source repository URL, and ref that produced a built container image. This makes commit
provenance visible to tools such as `deployment resolve`, which can then display which exact
commit is running in each deployed environment.

> **Note**: This action initially lives under `layro01-org` as part of a proof of concept and
> will be transferred to the `veracode` GitHub org before general availability.

---

## Table of contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Wrap mode (preferred)](#wrap-mode-preferred)
4. [Post-hoc mode (fallback)](#post-hoc-mode-fallback)
5. [Reusable workflow integration (Veracode GitHub App customers)](#reusable-workflow-integration)
6. [Inputs reference](#inputs-reference)
7. [Outputs reference](#outputs-reference)
8. [Annotation keys written](#annotation-keys-written)
9. [Reading annotations with `deployment resolve`](#reading-annotations-with-deployment-resolve)
10. [Registry-specific notes](#registry-specific-notes)
11. [Phase 2: signed attestations](#phase-2-signed-attestations)

---

## Overview

Container images carry no provenance information by default. An image tagged `:latest` or even
`:abc123def` gives no guarantee that it was actually built from the commit whose SHA appears in
the tag — the tag is mutable and the image config labels are not part of the content-addressed
manifest.

OCI manifest annotations are different: they are embedded in the image manifest and travel with
the image digest. They survive tag re-pointing and are readable without pulling image layers.

This action writes nine annotation keys to the image manifest:

- **4 standard OCI keys** (`org.opencontainers.image.*`) readable by any OCI-aware tool
- **5 custom Veracode provenance keys** (`com.veracode.provenance.*`) consumed by
  `deployment resolve` and future Trust Authority integrations

The `deployment resolve` CLI reads these annotations and displays a `COMMIT` column (first 12
characters of the commit SHA) and an `ASSURANCE` column (`annotation` for unsigned annotations,
`signed-attestation` for future Sigstore-signed attestations).

---

## Prerequisites

- GitHub Actions workflow running on `ubuntu-latest` (or any Linux runner with bash)
- Docker already logged in to your registry (ECR, GHCR, etc.) earlier in the same job
- `permissions.id-token: write` on the job (already required for AWS OIDC; needed for future
  signing step — costs nothing to add now)
- For **wrap mode**: `docker/setup-buildx-action` must run before this action (see below)
- For **post-hoc mode**: your image must already be pushed before calling this action

---

## Wrap mode (preferred)

Wrap mode computes the annotation values and outputs them as a newline-separated `key=value`
string. You pass this string to `docker/build-push-action`'s `annotations:` input, which
embeds the annotations in the manifest at build time using `docker buildx build --annotation`.

**Advantages over post-hoc**: no network round-trip to mutate an already-pushed manifest; the
original image digest is not changed; no dependency on the `crane` tool.

**Requirement**: Docker Buildx must be set up before calling this action. Buildx is required
for `--annotation` support. Add `docker/setup-buildx-action@v3` before the provenance step.

### Full ECR example

```yaml
name: CI

on:
  push:
    branches: [main]
    tags: ["v*.*.*"]

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      id-token: write   # required for AWS OIDC and future signing step
      contents: read

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
          aws-region: us-east-1

      - name: Login to ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      # Required: enables --annotation support in docker buildx build
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      # Zero required inputs — all provenance values are derived automatically
      # from the GitHub Actions context.
      - name: Compute commit provenance annotations
        id: provenance
        uses: layro01-org/annotate-container-image@v1

      - name: Extract image tags
        id: tags
        env:
          ECR_REGISTRY: ${{ secrets.ECR_REGISTRY }}
        run: |
          SHA_TAG=$ECR_REGISTRY/my-app:${{ github.sha }}
          echo "sha_tag=$SHA_TAG" >> $GITHUB_OUTPUT
          if [[ "${{ github.ref_type }}" == "tag" ]]; then
            echo "version_tag=$ECR_REGISTRY/my-app:${{ github.ref_name }}" >> $GITHUB_OUTPUT
            echo "latest_tag=$ECR_REGISTRY/my-app:latest" >> $GITHUB_OUTPUT
            echo "is_release=true" >> $GITHUB_OUTPUT
          else
            echo "is_release=false" >> $GITHUB_OUTPUT
          fi

      # Replaces separate 'docker build' + 'docker push' steps.
      # docker/build-push-action passes annotations to buildx via --annotation.
      - name: Build and push
        id: build
        uses: docker/build-push-action@v6
        with:
          push: true
          tags: ${{ steps.tags.outputs.sha_tag }}
          annotations: ${{ steps.provenance.outputs.annotations }}

      - name: Push semver and latest tags (releases only)
        if: steps.tags.outputs.is_release == 'true'
        run: |
          docker tag ${{ steps.tags.outputs.sha_tag }} ${{ steps.tags.outputs.version_tag }}
          docker tag ${{ steps.tags.outputs.sha_tag }} ${{ steps.tags.outputs.latest_tag }}
          docker push ${{ steps.tags.outputs.version_tag }}
          docker push ${{ steps.tags.outputs.latest_tag }}
```

### Minimal diff from a bare docker build + push workflow

If your current workflow is:

```yaml
- name: Build image
  run: docker build -t $IMAGE_REF .

- name: Push image
  run: docker push $IMAGE_REF
```

Replace with:

```yaml
- name: Set up Docker Buildx
  uses: docker/setup-buildx-action@v3

- name: Compute commit provenance annotations
  id: provenance
  uses: layro01-org/annotate-container-image@v1

- name: Build and push
  uses: docker/build-push-action@v6
  with:
    push: true
    tags: $IMAGE_REF
    annotations: ${{ steps.provenance.outputs.annotations }}
```

---

## Post-hoc mode (fallback)

Use post-hoc mode when you cannot switch to `docker/build-push-action` — for example, when
your build system calls `docker build` internally and you can't intercept the build step.

Post-hoc mode installs `crane` (if not already present) and calls `crane mutate --annotation`
on the already-pushed image. **This creates a new manifest with a new digest.** The tag is
updated to point to the new digest. If you have stored or pinned the pre-annotation digest,
re-fetch it after this step.

```yaml
- name: Build image
  run: docker build -t ${{ env.IMAGE_REF }} .

- name: Push image
  run: docker push ${{ env.IMAGE_REF }}

- name: Annotate image with commit provenance
  uses: layro01-org/annotate-container-image@v1
  with:
    mode: post-hoc
    image-ref: ${{ env.IMAGE_REF }}
```

If you want to use the annotated image's new digest in subsequent steps, read it back after
this step:

```yaml
- name: Get annotated digest
  id: digest
  run: |
    DIGEST=$(crane digest ${{ env.IMAGE_REF }})
    echo "digest=$DIGEST" >> $GITHUB_OUTPUT
```

### Non-default build context

If your `Dockerfile` is not at the repository root, pass `build-context` so the annotation
records the correct path:

```yaml
- uses: layro01-org/annotate-container-image@v1
  with:
    mode: post-hoc
    image-ref: ${{ env.IMAGE_REF }}
    build-context: services/api
```

---

## Reusable workflow integration

For teams using the **Veracode GitHub App** (`github-actions-integration`), a reusable
workflow is available in your organisation's central `veracode` repo. This wraps the action
in post-hoc mode and keeps all the tooling in one place.

Add a single step after `docker push` in your existing CI workflow:

```yaml
- name: Push image
  run: docker push ${{ steps.tags.outputs.sha_tag }}

- name: Annotate image with commit provenance
  uses: YOUR_ORG/veracode/.github/workflows/veracode-annotate-container-image.yml@main
  with:
    image-ref: ${{ steps.tags.outputs.sha_tag }}
```

Replace `YOUR_ORG` with your GitHub organisation name.

No changes to `veracode.yml`, no new secrets, and no backend configuration are required.

---

## Inputs reference

| Input | Required | Default | Description |
| ----- | -------- | ------- | ----------- |
| `mode` | No | `wrap` | `wrap`: output annotation strings (for `docker/build-push-action`). `post-hoc`: apply annotations to an already-pushed image using `crane mutate`. |
| `image-ref` | Only in `post-hoc` | `''` | Full image reference: `registry/repo:tag` or `registry/repo@sha256:…` |
| `build-context` | No | `.` | Docker build context path within the repo. Recorded in `com.veracode.provenance.build-context`. |

---

## Outputs reference

| Output | Description |
| ------ | ----------- |
| `annotations` | Newline-separated `key=value` pairs. Pass directly to `docker/build-push-action`'s `annotations:` input in wrap mode. |
| `head-sha` | Resolved head commit SHA. On `pull_request` events this is the PR head commit SHA (not `github.sha`, which is an ephemeral merge commit). |

---

## Annotation keys written

| Key | Value | Maps to CLI field |
| --- | ----- | ----------------- |
| `org.opencontainers.image.revision` | Full 40-char commit SHA | `CommitProvenance.HeadSHA` (primary) |
| `org.opencontainers.image.source` | `https://github.com/owner/repo` | `CommitProvenance.SourceURL` |
| `org.opencontainers.image.ref.name` | Git ref (`refs/heads/main`, `refs/tags/v1.2.3`) | `CommitProvenance.SourceRef` |
| `org.opencontainers.image.created` | RFC 3339 build timestamp | (display only) |
| `com.veracode.provenance.head-sha` | Same as `revision`; explicit for PR head disambiguation | `CommitProvenance.HeadSHA` (fallback) |
| `com.veracode.provenance.workflow-ref` | `GITHUB_WORKFLOW_REF` | `CommitProvenance.WorkflowRef` |
| `com.veracode.provenance.run-id` | `GITHUB_RUN_ID` | `CommitProvenance.RunID` |
| `com.veracode.provenance.build-context` | Build context path (from `build-context` input) | `CommitProvenance.BuildContext` |
| `com.veracode.provenance.signing` | `pending` (MVP); `signed` when Phase 2 attestation is written | (informational) |

The standard `org.opencontainers.image.*` keys are written because any OCI-aware tool
understands them. The `com.veracode.provenance.*` keys carry additional detail needed by
`deployment resolve`.

**Why `head-sha` duplicates `revision`**: On `pull_request` events, `github.sha` is an
ephemeral merge commit GitHub creates to run CI. It is not reachable from either branch
history and disappears when the PR closes. The action resolves the actual PR head SHA from
`github.event.pull_request.head.sha` and writes it to both keys, making the intended semantics
explicit.

---

## Reading annotations with `deployment resolve`

After your image is annotated and pushed, run the `deployment resolve` command against a
deployment config that references your image. The CLI fetches the manifest for the resolved
digest and reads the provenance annotations.

**Text output** (default):

```shell
NAME         DECLARED        RESOLVED           COMMIT        ASSURANCE   STATUS
my-app       :abc123def...   sha256:a1b2c3d4…   abc123def456  annotation  ACTIVE
```

**JSON output** (`--output json`):

```json
{
  "name": "my-app",
  "commit_provenance": {
    "head_sha": "abc123def456789012345678901234567890abcd",
    "source_url": "https://github.com/layro01-org/my-app",
    "source_ref": "refs/heads/main",
    "workflow_ref": "layro01-org/my-app/.github/workflows/ci.yml@refs/heads/main",
    "run_id": "12345678901",
    "build_context": ".",
    "assurance": "annotation"
  }
}
```

When no annotations are present, `COMMIT` and `ASSURANCE` display `-` and `commit_provenance`
is omitted from JSON output. This is not an error.

To suppress provenance columns entirely:

```shell
deployment resolve --exclude-commit-provenance <target>
```

You can verify annotations directly with `crane`:

```bash
crane manifest <registry>/my-app:<tag> | jq '.annotations'
```

---

## Registry-specific notes

### Amazon ECR

`crane` uses the ambient AWS credentials set up by `aws-actions/configure-aws-credentials`
earlier in the job. No additional auth configuration is required.

```yaml
- name: Configure AWS credentials
  uses: aws-actions/configure-aws-credentials@v4
  with:
    role-to-assume: ${{ secrets.AWS_ROLE_ARN }}
    aws-region: us-east-1

# crane will use the above credentials automatically
- uses: layro01-org/annotate-container-image@v1
  with:
    mode: post-hoc
    image-ref: ${{ env.ECR_REGISTRY }}/my-app:${{ github.sha }}
```

### GitHub Container Registry (GHCR)

Log in to GHCR with the built-in `GITHUB_TOKEN` before calling the action in post-hoc mode:

```yaml
- name: Login to GHCR
  uses: docker/login-action@v3
  with:
    registry: ghcr.io
    username: ${{ github.actor }}
    password: ${{ secrets.GITHUB_TOKEN }}

- uses: layro01-org/annotate-container-image@v1
  with:
    mode: post-hoc
    image-ref: ghcr.io/${{ github.repository }}/my-app:${{ github.sha }}
```

### Other registries

Run `docker login` (or your registry's equivalent) before calling the action in post-hoc mode.
`crane` picks up Docker credentials from `~/.docker/config.json` automatically.

---

## Phase 2: signed attestations

The `com.veracode.provenance.signing: pending` annotation records that Phase 2 signing has
not yet been applied to this image. In the MVP, annotations are unsigned: they prove only that
whoever pushed the image wrote these values at push time.

Phase 2 will replace the unsigned annotation approach with a Sigstore attestation (OIDC →
Fulcio → RFC 3161 TSA, no public Rekor upload) stored as an OCI referrer attachment alongside
the image. When a verified attestation is present, `deployment resolve` will show
`ASSURANCE: signed-attestation` instead of `annotation`, providing cryptographic proof that
the specific GitHub Actions workflow identity produced the image from the stated commit.

No workflow changes are needed to upgrade from annotations to signed attestations: a future
version of this action will handle attestation transparently when `permissions.id-token: write`
is present (which you already have if you're using AWS OIDC).
