# dili-docker-build

Reusable GitHub Actions workflow that builds a Docker image and pushes it to Amazon ECR via OIDC.

Designed to be called from a thin caller workflow that supplies the operational config (registry, role, tags, build args) as inputs. The caller authenticates to AWS via OIDC; this workflow holds no secrets and runs no caller-controlled code outside `docker buildx`.

## Usage

In your repo, add `.github/workflows/build.yml`:

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
    uses: ConsenSysDiligence/dili-docker-build/.github/workflows/build.yml@v1
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

`workflow_dispatch` doesn't auto-forward its inputs into a `workflow_call`, so each input is spelled out twice — once in the dispatch contract and once in the `with:` block. Pin the `uses:` to a release tag or commit SHA, never a floating branch.

If your `Dockerfile` isn't at the repo root, set `build_context` and `dockerfile` in the same `with:` block (defaults are `.` and `Dockerfile`).

## Inputs

| Name | Required | Default | Description |
| --- | --- | --- | --- |
| `ecr_registry` | yes | — | ECR registry hostname, e.g. `123.dkr.ecr.eu-west-1.amazonaws.com` |
| `ecr_repository` | yes | — | Repository path within the registry, e.g. `myorg/myimage` |
| `aws_region` | yes | — | AWS region for the push |
| `aws_role_arn` | yes | — | IAM role ARN, assumed via OIDC |
| `image_tags` | yes | — | Comma-separated tags. `sha-<commit>` is always pushed in addition. |
| `dispatch_id` | yes | — | Caller-supplied identifier; stamped as a `dispatch_id` OCI label for traceability |
| `build_args_json` | no | `{}` | JSON object of Docker build args |
| `dockerfile` | no | `Dockerfile` | Path to Dockerfile, relative to the build context |
| `build_context` | no | `.` | Build context directory |
| `dry_run` | no | `false` | Build only, skip push |

## Image labels

Every pushed image carries:

- `org.opencontainers.image.revision` — caller commit SHA
- `org.opencontainers.image.source` — caller repository URL
- `org.opencontainers.image.url` — URL of the workflow run that produced the image
- `dispatch_id` — value passed by the caller, opaque to this workflow
