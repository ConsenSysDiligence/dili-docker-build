# dili-docker-build

Reusable GitHub Actions workflow that builds a Docker image and pushes it to Amazon ECR via
OIDC.

It is called from a thin caller workflow that supplies the operational config — registry,
role, tags, build args — as inputs. The caller authenticates to AWS via OIDC; this workflow
holds no secrets and runs no caller-controlled code outside `docker buildx`.

## Scope

This workflow is purpose-built for one internal platform and the repositories that produce
images for it. It is not a general-purpose build workflow: the input contract, the tagging
conventions and the release process below are all shaped by that single consumer, and
changes here are driven by its needs.

Adopting it elsewhere means inheriting a coupling you probably do not want — the consumer
authorises builds by pinned commit SHA, so releases are not independent of it. See
[Releases](#releases-and-the-consumer-pin).

## Usage

In the calling repo, add `.github/workflows/build.yml`:

```yaml
name: build

on:
  workflow_dispatch:
    inputs:
      ecr_registry:    { required: true,  type: string }
      ecr_repository:  { required: true,  type: string }
      aws_region:      { required: true,  type: string }
      aws_role_arn:    { required: true,  type: string }
      image_tags:      { required: true,  type: string }
      dispatch_id:     { required: true,  type: string }
      build_args_json: { required: false, type: string, default: '{}' }
      dry_run:         { required: false, type: boolean, default: false }

permissions:
  id-token: write
  contents: read

jobs:
  build:
    uses: ConsenSysDiligence/dili-docker-build/.github/workflows/build.yml@6184ac41a5117be55b8955d21bb5850400acfb18 # v0.5.0
    with:
      ecr_registry:    ${{ inputs.ecr_registry }}
      ecr_repository:  ${{ inputs.ecr_repository }}
      aws_region:      ${{ inputs.aws_region }}
      aws_role_arn:    ${{ inputs.aws_role_arn }}
      image_tags:      ${{ inputs.image_tags }}
      dispatch_id:     ${{ inputs.dispatch_id }}
      build_args_json: ${{ inputs.build_args_json }}
      dry_run:         ${{ inputs.dry_run }}
```

`workflow_dispatch` does not auto-forward its inputs into a `workflow_call`, so each input
is spelled out twice — once in the dispatch contract and once in the `with:` block.

**Pin `uses:` to a commit SHA, not a tag or a branch.** The consumer's IAM trust policy
matches on the exact `job_workflow_ref`, so a floating ref does not merely risk drift, it
fails to assume the role. There is no `v1` moving tag.

If the `Dockerfile` is not at the repo root, set `build_context` and `dockerfile` in the
same `with:` block; the defaults are `.` and `Dockerfile`.

## Inputs

| Name | Required | Default | Description |
| --- | --- | --- | --- |
| `ecr_registry` | yes | — | ECR registry hostname, e.g. `123.dkr.ecr.eu-west-1.amazonaws.com` |
| `ecr_repository` | yes | — | Repository path within the registry, e.g. `myorg/myimage` |
| `aws_region` | yes | — | AWS region for the push |
| `aws_role_arn` | yes | — | IAM role ARN, assumed via OIDC |
| `image_tags` | yes | — | Comma-separated tags. Exactly these are pushed — see the note below |
| `dispatch_id` | yes | — | Caller-supplied identifier, stamped as a `dispatch_id` OCI label for traceability |
| `build_args_json` | no | `{}` | JSON object of Docker build args |
| `dockerfile` | no | `Dockerfile` | Path to Dockerfile, relative to the build context |
| `build_context` | no | `.` | Build context directory |
| `extra_context_artifact` | no | `""` | Name of an Actions artifact to unpack into `build_context` after checkout. Lets the caller stage files — for example a credentialed fetch from a private repo — without exposing the secret to the Dockerfile or to the resulting image |
| `dry_run` | no | `false` | Build only, skip the push |

### `image_tags` pushes exactly what you pass

Up to and including v0.4.1 this workflow also pushed `sha-<caller-commit>` automatically.
**v0.5.0 removed that.** The tags pushed are now precisely the ones in `image_tags`, in
order.

The caller is therefore responsible for including an immutable tag of its own if it wants
one. A caller that passes only mutable pointers such as `latest,dev` will have no
per-commit artifact to roll back to, and — where the ECR repository is configured with
`IMMUTABLE_WITH_EXCLUSION` — re-pushing an unexcluded tag is rejected outright. The
conventional shape is `latest,sha-<commit>`, composed by the caller.

## Image labels

Every pushed image carries:

- `org.opencontainers.image.revision` — caller commit SHA
- `org.opencontainers.image.source` — caller repository URL
- `org.opencontainers.image.url` — URL of the workflow run that produced the image
- `dispatch_id` — the value passed by the caller, opaque to this workflow

## Releases and the consumer pin

Cutting a release here is not self-contained. The consuming platform grants each build role
by matching the OIDC `job_workflow_ref` claim against an allow-list of **specific commit
SHAs of this workflow**, held in its own infrastructure code. A caller running a SHA that is
not on that list gets a `403` when it tries to assume the role, no matter how correct
everything else is.

So the order matters:

1. Merge the change here and tag the release.
2. **Add the new SHA to the consumer's allow-list and apply**, keeping the existing entries
   in place. IAM treats the list as a logical OR, so old and new callers both work during
   the transition.
3. Bump the `uses:` pin in each caller.
4. Once no caller runs the old SHA, remove it from the allow-list and apply again.

Doing step 3 before step 2 breaks every build that has been bumped. Skipping step 4 leaves
an unaudited workflow version able to assume push roles indefinitely.

The rationale for pinning by SHA rather than tag is that a tag can be moved; the allow-list
is what limits ECR push access to build logic that has actually been reviewed.

## Release history

| Tag | Change |
| --- | --- |
| `v0.5.0` | Dropped the automatic `sha-<commit>` tag; callers compose their own tag list |
| `v0.4.1` | Updated `upload/download-artifact` versions |
| `v0.4.0` | Added `extra_context_artifact` for credentialed file staging |
| `v0.3.0` | Added the GitHub Actions cache to `build-push-action` |
| `v0.2.0` | — |

## CI in this repo

`lint.yml` runs `actionlint` and `zizmor` on the workflow definitions. There is nothing else
to test here — the workflow is only exercised by its callers.
