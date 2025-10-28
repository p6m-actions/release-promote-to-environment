# Promote Release to Environment

Promote any GitHub release to any environment without rebuilding. Downloads the image digest from the release and deploys it to the target environment using your platform's manifest dispatch system.

## Features

- Promotes releases without rebuilding images
- Downloads digest from GitHub Release artifacts
- Updates platform manifests via dispatch
- Tracks promotion history in release notes
- Supports rollbacks by promoting older releases
- Provides detailed promotion summary
- Works with any environment (dev, stg, prd, etc.)

## Usage

### Basic Example

```yaml
- name: Promote release to production
  uses: p6m-actions/release-promote-to-environment@v1
  with:
    tag: v1.2.3
    environment: prd
    repository: ${{ github.repository }}
    image-name: my-app
    update-manifest-token: ${{ secrets.UPDATE_MANIFEST_TOKEN }}
    platform-dispatch-url: ${{ vars.PLATFORM_DISPATCH_URL }}
```

### Full Promotion Workflow

```yaml
name: Promote Tag

on:
  workflow_dispatch:
    inputs:
      environment:
        description: 'Target environment'
        type: choice
        options:
          - stg
          - prd
        required: true
      tag:
        description: 'Tag to promote (e.g., v1.2.3)'
        type: string
        required: true

run-name: Promote ${{ inputs.tag }} to ${{ inputs.environment }}

permissions:
  contents: write  # Required for updating release notes

jobs:
  promote:
    name: Promote to ${{ inputs.environment }}
    runs-on: ubuntu-latest

    steps:
      - name: Promote release
        uses: p6m-actions/release-promote-to-environment@v1
        with:
          tag: ${{ inputs.tag }}
          environment: ${{ inputs.environment }}
          repository: ${{ github.repository }}
          image-name: my-app
          update-manifest-token: ${{ secrets.UPDATE_MANIFEST_TOKEN }}
          platform-dispatch-url: ${{ vars.PLATFORM_DISPATCH_URL }}
```

### Rollback Example

Rollback is just promoting an older release:

```yaml
# Rollback production to v1.2.0
- uses: p6m-actions/release-promote-to-environment@v1
  with:
    tag: v1.2.0  # Older release
    environment: prd
    repository: ${{ github.repository }}
    image-name: my-app
    update-manifest-token: ${{ secrets.UPDATE_MANIFEST_TOKEN }}
    platform-dispatch-url: ${{ vars.PLATFORM_DISPATCH_URL }}
```

## Inputs

| Input | Description | Required | Default |
|-------|-------------|----------|---------|
| `tag` | Release tag to promote (e.g., v1.2.3) | Yes | - |
| `environment` | Target environment (dev, stg, prd) | Yes | - |
| `repository` | Repository in owner/repo format | Yes | - |
| `image-name` | Application/image name for manifest update | Yes | - |
| `update-manifest-token` | Token for platform manifest dispatch | Yes | - |
| `platform-dispatch-url` | Platform dispatch API URL | Yes | - |
| `directory-name` | Directory name in platform manifests | No | (uses image-name) |
| `github-token` | GitHub token for release operations | No | `${{ github.token }}` |
| `update-release-notes` | Append promotion info to release notes | No | `true` |

## Outputs

| Output | Description |
|--------|-------------|
| `digest` | Promoted image digest |
| `promotion-timestamp` | Promotion timestamp (UTC) |
| `manifest-status` | Manifest update status |

## How It Works

```
┌─────────────────────────────────────────────────────────┐
│ 1. Download digest from GitHub Release                  │
│    gh release download v1.2.3 --pattern digest.txt      │
│    Result: sha256:abc123...                              │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│ 2. Update platform manifest                             │
│    POST /platform-dispatch                               │
│    {                                                     │
│      "environment": "prd",                               │
│      "digest": "sha256:abc123...",                       │
│      "image": "my-app"                                   │
│    }                                                     │
└────────────────────┬────────────────────────────────────┘
                     │
                     ▼
┌─────────────────────────────────────────────────────────┐
│ 3. Update release notes                                 │
│    Append: "Promoted to prd by @user on 2024-10-28"    │
└─────────────────────────────────────────────────────────┘
```

## Promotion History Tracking

This action automatically appends promotion records to release notes:

```markdown
## Application Details
**Version**: `1.2.3`
**Docker Digest**: `sha256:abc123...`

---

## What's New
- Feature: Added authentication
- Fix: Memory leak resolved

---

**Promoted to `dev`** by @developer on 2024-10-28 10:00:00 UTC

---

**Promoted to `stg`** by @developer on 2024-10-28 14:30:00 UTC

---

**Promoted to `prd`** by @ops-lead on 2024-10-29 09:15:00 UTC
```

This creates an audit trail showing:
- What environments the release has been deployed to
- Who performed each promotion
- When each promotion occurred

## Complete Promotion Flow Example

```bash
# Day 1: Developer cuts release
$ gh workflow run cut-tag.yaml -f version-level=minor
# Creates v1.3.0
# Builds Docker image with digest sha256:abc123...
# Stores digest in GitHub Release

# Day 1: Auto-deploy to dev
# (triggered automatically by cut-tag workflow)
# Deploys sha256:abc123... to dev environment

# Day 2: QA approves, promote to staging
$ gh workflow run promote.yaml -f tag=v1.3.0 -f environment=stg
# Downloads sha256:abc123... from v1.3.0 release
# Deploys to staging (exact same image as dev)
# Updates release notes: "Promoted to stg by @qa-lead on ..."

# Day 3: Staging looks good, promote to production
$ gh workflow run promote.yaml -f tag=v1.3.0 -f environment=prd
# Downloads sha256:abc123... from v1.3.0 release
# Deploys to production (exact same image as dev/stg)
# Updates release notes: "Promoted to prd by @ops-lead on ..."

# Day 4: Issue found in production, rollback to v1.2.0
$ gh workflow run promote.yaml -f tag=v1.2.0 -f environment=prd
# Downloads digest from v1.2.0 release
# Deploys old version to production
# Updates v1.2.0 release notes: "Promoted to prd by @ops-lead on ..."
```

## Benefits

### Immutable Deployments
Every environment gets the **exact same Docker image** (verified by digest):
- Dev tested `sha256:abc123...`
- Staging gets `sha256:abc123...` (same!)
- Production gets `sha256:abc123...` (same!)

No "works on my machine" between environments.

### Easy Rollbacks
Rollback = promote an older release. No special process needed.

```yaml
# Rollback is just promoting v1.2.0 instead of v1.3.0
- uses: release-promote-to-environment@v1
  with:
    tag: v1.2.0  # That's it!
    environment: prd
```

### Audit Trail
Every promotion is recorded in the release notes:
- Who deployed what
- When they deployed it
- Where they deployed it

### No Rebuilds
Images are built once and promoted everywhere:
- Faster (no rebuild time)
- Safer (exact same bits)
- Cheaper (one build instead of N builds)

## Integration with Full Promotion Strategy

This action completes the 3-action promotion strategy:

### 1. Build & Tag
```yaml
# cut-tag.yaml
jobs:
  build:
    - uses: p6m-actions/js-pnpm-cut-tag@v1  # or rust-cut-tag, etc.

  docker-build:
    - uses: p6m-actions/docker-build-with-digest@v1
      outputs:
        digest: sha256:abc123...

  create-release:
    - uses: p6m-actions/release-create-with-digest@v1
      with:
        digest: sha256:abc123...
      # Stores digest in GitHub Release
```

### 2. Promote (This Action)
```yaml
# promote.yaml
jobs:
  promote:
    - uses: p6m-actions/release-promote-to-environment@v1  # ← You are here
      with:
        tag: v1.2.3
        environment: prd
      # Downloads digest from release, deploys
```

## Ecosystem Support

Works with any language ecosystem because it only cares about:
- GitHub Releases (platform feature)
- Docker digests (container standard)
- Platform manifest dispatch (your deployment system)

Tested with:
- JavaScript/PNPM
- Rust
- Python/UV
- .NET
- Go
- Java/Maven

## Required Permissions

Your workflow needs these permissions:

```yaml
permissions:
  contents: write  # To update release notes
```

## Required Secrets/Variables

```yaml
secrets:
  UPDATE_MANIFEST_TOKEN: ${{ secrets.UPDATE_MANIFEST_TOKEN }}

vars:
  PLATFORM_DISPATCH_URL: ${{ vars.PLATFORM_DISPATCH_URL }}
```

## License

MIT
