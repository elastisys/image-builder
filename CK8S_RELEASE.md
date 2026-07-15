# CK8s Image Builder Releases

This document describes how Elastisys maintains its fork of
[kubernetes-sigs/image-builder](https://github.com/kubernetes-sigs/image-builder)
and produces CK8s releases.

Upstream release documentation:
[docs/book/src/capi/releasing.md](docs/book/src/capi/releasing.md)

For the overall Welkin release cycle, see
[Welkin release notes](https://elastisys.io/welkin/release-notes/).

## TL;DR

- `main` mirrors upstream exactly. Sync it with a fast-forward — no CK8s commits,
  no conflicts.
- `ck8s/main` carries all CK8s changes on top of an upstream base.
- One `release-X.Y.Z-ck8s` branch per upstream release.
- Tags: `vX.Y.Z-ck8sN` (`N` starts at `1`, increments per patch).

```text
upstream/main ──fast-forward──► main

upstream/v0.1.56 ──rebase──► ck8s/main
                                    │
                                    ▼
                         release-0.1.56-ck8s
                                    │
                         staging-0.1.56-ck8s1
                                    │
                              v0.1.56-ck8s1
```

## Branches

| Branch | Purpose | Sync with upstream |
|--------|---------|-------------------|
| `main` | Upstream mirror. No CK8s commits. | Always fast-forward from `upstream/main`. |
| `ck8s/main` | All CK8s development and the CK8s patch set. | Rebase onto upstream tag or `main`. |
| `release-X.Y.Z-ck8s` | One branch per upstream release. Backports and release fixes. | Created from upstream tag + CK8s patch set. |
| `staging-X.Y.Z-ck8sN` | Short-lived QA branch. PR back to the release branch. | — |

**Rules:**

- Never commit CK8s changes to `main`.
- Never merge `ck8s/main` into `main`.
- All CK8s feature work merges into `ck8s/main` via PR.
- Do not use personal release branches once the formal branches exist.

## Versioning

```text
vX.Y.Z-ck8sN
```

| Part | Meaning |
|------|---------|
| `X.Y.Z` | Upstream image-builder release |
| `ck8s` | Elastisys CK8s fork marker |
| `N` | CK8s patch number, starting at `1` per upstream base |

Examples: `v0.1.55-ck8s1`, `v0.1.55-ck8s2`

Published tags are immutable. Never move or overwrite an existing tag.

## What CK8s adds

Changes not intended for upstream:

- Unattended security upgrades on Debian-based images
- Kubernetes apiserver audit policy
- GitHub Actions workflows (builder image + Azure/OpenStack VM images)
- CK8s-specific hardening and operational fixes

## Initial setup

Create `ck8s/main` from the current CK8s patch set. Use the cleanest existing
state (e.g. `hani/release-v0.1.55`) and rebase onto the upstream tag:

```bash
git fetch upstream origin --tags

# ensure main is current
git switch main
git merge --ff-only upstream/main

# create ck8s/main from existing CK8s work, rebased onto upstream tag
git switch -c ck8s/main hani/release-v0.1.55   # or other source branch
git rebase -i v0.1.55                          # squash duplicates, clean history
git push -u origin ck8s/main
```

Target a small, logical commit series on `ck8s/main` (unattended upgrades,
audit policy, CI workflows, etc.).

## Managing the fork

### Remotes

```bash
git remote -v
# origin    git@github.com:elastisys/image-builder.git
# upstream  git@github.com:kubernetes-sigs/image-builder.git
```

### Sync main with upstream

This is always conflict-free because `main` has no CK8s commits:

```bash
git fetch upstream
git switch main
git merge --ff-only upstream/main
git push origin main
```

Run this regularly, and always before starting a new CK8s release.

### Update ck8s/main for a new upstream release

When upstream publishes `vX.Y.Z`, rebase the CK8s patch set onto the new tag.
Conflicts are resolved once, only in CK8s files:

```bash
git fetch upstream --tags
git switch ck8s/main
git pull origin ck8s/main

git rebase v0.1.56
# resolve conflicts, run tests, then:
git push --force-with-lease origin ck8s/main
```

Files that commonly conflict:

```text
.github/workflows/build-*.yml
images/capi/ansible/roles/sysprep/tasks/debian.yml
images/capi/ansible/roles/kubernetes/tasks/main.yml
images/capi/packer/goss/goss-vars.yaml
images/capi/ansible/roles/kubernetes/files/etc/audit-policy/apiserver-audit-policy.yaml
```

### Develop CK8s features

```bash
git switch ck8s/main
git pull
git switch -c my-feature

# open PR to ck8s/main
```

Generic fixes that belong upstream should be contributed to
kubernetes-sigs/image-builder separately, then dropped from the CK8s patch set
during the next rebase.

### Create a release branch

After `ck8s/main` is rebased onto the upstream tag:

```bash
git switch ck8s/main
git switch -c release-0.1.56-ck8s
git push -u origin release-0.1.56-ck8s
```

For patch releases on an existing upstream base, use the existing release
branch — do not create a new one.

### Backport a fix

Fix on `ck8s/main` first, then cherry-pick to the release branch:

```bash
git switch release-0.1.56-ck8s
git pull
git cherry-pick <commit-sha>
git push
```

## Release process

Follows the same staging pattern as
[compliantkubernetes-kubespray/release/README.md](https://github.com/elastisys/compliantkubernetes-kubespray/blob/main/release/README.md).

### Prerequisites

- `main` is fast-forwarded to `upstream/main`
- `ck8s/main` is rebased onto the target upstream tag
- Required changes are merged to `ck8s/main` (new bases) or cherry-picked to
  the release branch (patches)

For patch releases:

```bash
export CK8S_GIT_CHERRY_PICK="COMMIT-SHA [COMMIT-SHA ...]"
```

### Staging

```bash
export VERSION=0.1.56-ck8s1

git switch release-0.1.56-ck8s
git pull
git switch -c staging-${VERSION}

for sha in ${CK8S_GIT_CHERRY_PICK:-}; do
  git cherry-pick "${sha}"
done

git push -u origin staging-${VERSION}
```

Open a **draft** PR from `staging-${VERSION}` to `release-0.1.56-ck8s`.

### QA

On the staging branch:

1. Builder image builds and pushes to GHCR
2. Test VM images build (Azure and/or OpenStack)
3. Audit policy and unattended-upgrades present on built image

Trigger **Build CAPI VM image with manual input**:

| Input | Example |
|-------|---------|
| `version` | `1.33.1` |
| `tag` | `0.8` |
| `builder_image` | `ghcr.io/elastisys/image-builder-amd64:v0.1.56-ck8s1` |

Push fixes to the staging branch if needed.

### Release

Merge the staging PR to the release branch, then tag:

```bash
git switch release-0.1.56-ck8s
git pull

export GPG_TTY=$(tty)
git tag -s v0.1.56-ck8s1 -m "CK8s image-builder v0.1.56-ck8s1"
git push origin v0.1.56-ck8s1
```

Verify:

```bash
docker pull ghcr.io/elastisys/image-builder-amd64:v0.1.56-ck8s1
```

Create a GitHub release noting the upstream base, builder image, and CK8s
changes.

### Port fixes back to ck8s/main

After the release, port any QA-only fixes to `ck8s/main`:

```bash
git switch ck8s/main
git pull
git switch -c port-0.1.56-ck8s1
git cherry-pick <fix-commit-sha>
git push -u origin port-0.1.56-ck8s1
```

Open a PR to `ck8s/main`.

## Artifacts

| Artifact | Location |
|----------|----------|
| Builder container | `ghcr.io/elastisys/image-builder-amd64:vX.Y.Z-ck8sN` |
| Azure SIG image | `ubuntu-2404-kube-<k8s-minor>-ck8s-capi-<capi-tag>` |
| OpenStack image | `ubuntu-2404-efi-kube-<k8s-version>-ck8s-capi-<capi-tag>` |

Pin the builder image tag (or digest) for production VM builds. Do not use
`:main`.

## Current releases

| Tag | Upstream base | Status |
|-----|---------------|--------|
| `v0.1.55-ck8s` | `v0.1.55` | Legacy; supersede with `v0.1.55-ck8s1` from `ck8s/main` |

Update this table after each release.
