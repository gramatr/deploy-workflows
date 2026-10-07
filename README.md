# deploy-workflows

Shared, reusable GitHub Actions workflows for the gramatr platform's
staging → production release-cut pipeline. Used by `gramatr/gramatr-paas`,
`gramatr/gramatr-console`, and `anneal-app/anneal`.

Background: `gramatr/gramatr-paas`'s
[`docs/scope/EPIC-azure-production-bws-staging-realignment-2026-10-07.md`](https://github.com/gramatr/gramatr-paas/blob/main/docs/scope/EPIC-azure-production-bws-staging-realignment-2026-10-07.md)
(E34) — BWS is staging, Azure is production, a release-cut (tag push)
triggers an automatic staging deploy, and a manual-approval gate promotes
the same artifact to production.

## Why a separate repo

Three different app repos need this (`gramatr-paas`, `gramatr-console`,
`anneal-app/anneal`), but only `gramatr-paas` has an `eng-standards`
submodule — adding one to the other two just for this would be new coupling
for a concern (deployment promotion) that's different from what
`eng-standards` is actually scoped to (lint/coverage/CI-runner standards).
GitHub Actions reusable workflows don't need a submodule at all — any repo
can `uses:` this one directly by ref.

## `bump-and-promote.yml`

Bumps one image's tag in one or more manifest files in a target GitOps repo
(`n90-co/prod-argocd-v2` for staging, `gramatr/azure-infra` for production),
opens a PR, and optionally auto-merges it.

**This workflow has no opinion about staging vs. production.** Whether a
call represents an automatic staging deploy or a gated production promotion
is decided entirely by the *calling* workflow's job definition — a job with
`environment: production-approval` blocks until a human approves before this
workflow ever runs; a plain job runs immediately. Same workflow, same
inputs, different caller.

One call bumps **one** image. A service with multiple images (e.g.
`gramatr-console`'s UI + app tiers) calls this workflow twice.

### Inputs

| Input | Required | Default | Description |
|---|---|---|---|
| `target_repo` | yes | — | `owner/repo` to bump, e.g. `n90-co/prod-argocd-v2` |
| `target_ref` | no | `main` | Branch to PR against |
| `manifest_paths` | yes | — | Space-separated file paths (relative to `target_repo` root) containing the image line to bump |
| `image_name` | yes | — | Full image name without tag, e.g. `ghcr.io/gramatr/intelligence` |
| `image_tag` | yes | — | New tag to set, e.g. `v0.1.11` |
| `pr_title` | yes | — | PR title |
| `auto_merge` | no | `false` | Merge the PR immediately once opened. Leave `false` for production — a human merges after the environment approval gate already ran. |

### Secrets

| Secret | Required | Description |
|---|---|---|
| `deploy_token` | yes | PAT with write access to `target_repo` — e.g. the `github-gramatr-repo-rw` secret already in use elsewhere in this org (fine-grained, `Contents:Write` + `Pull requests:Write` on the target repos). |

### Outputs

| Output | Description |
|---|---|
| `pr_url` | URL of the opened PR, empty if no change was needed |
| `skipped` | `true` if `target_repo` was already at `image_tag` — no PR opened (idempotent no-op, not an error) |

### Example — staging deploy (automatic)

```yaml
jobs:
  deploy-staging:
    needs: build-and-push
    uses: gramatr/deploy-workflows/.github/workflows/bump-and-promote.yml@main
    with:
      target_repo: n90-co/prod-argocd-v2
      manifest_paths: argocd/ml-services/gramatr-paas-staging/intelligence-deployment.yaml
      image_name: ghcr.io/gramatr/intelligence
      image_tag: ${{ github.ref_name }}  # e.g. intelligence-v0.1.11, strip the prefix as needed
      pr_title: "deploy(staging): intelligence ${{ github.ref_name }}"
      auto_merge: true
    secrets:
      deploy_token: ${{ secrets.GRAMATR_REPO_RW_TOKEN }}
```

### Example — production promotion (gated)

```yaml
jobs:
  promote-production:
    needs: deploy-staging
    environment: production-approval   # <- the actual gate; this workflow has no gating logic of its own
    uses: gramatr/deploy-workflows/.github/workflows/bump-and-promote.yml@main
    with:
      target_repo: gramatr/azure-infra
      manifest_paths: k8s/intelligence/deployment.yaml
      image_name: ghcr.io/gramatr/intelligence
      image_tag: ${{ github.ref_name }}
      pr_title: "deploy(production): intelligence ${{ github.ref_name }}"
      auto_merge: false   # a human reviews and merges after the gate, same as any other PR
    secrets:
      deploy_token: ${{ secrets.GRAMATR_REPO_RW_TOKEN }}
```

Each calling repo must configure its own `production-approval` GitHub
Actions environment (Settings → Environments) with required reviewers —
that's what actually blocks the job until approved; it's ordinary GitHub
Actions environment protection, not anything custom to this repo.
