# flux-image-automation-lab

Reproducible kind lab to validate Flux image automation pushing to a side branch and a GitHub workflow auto-opening a PR that `lobicornis` then auto-merges.

Mirrors the prod setup in `traefik/infra` (PR: open PR for flux image automation updates) on a single-node kind cluster, with two image-policy-driven workloads (`traefik/traefik` and `traefik/whoami`).

## What this validates

- `ImageUpdateAutomation` pushes its commit to `flux-image-automation/<base>/<automation-name>` (not the default branch).
- The `.github/workflows/flux-auto-pr.yaml` workflow turns that push into a PR with merge labels.
- `lobicornis` merges the PR based on the labels.
- A single automation cycle batches multiple `ImagePolicy` setter updates into one commit / one PR.

## Layout

```
kind/cluster.yaml                   kind cluster config
bootstrap/                          kubectl apply -k here once
flux/                               GitOps entry point synced by flux
  image-repository.yaml             traefik + whoami
  image-policy.yaml                 semver ranges per image
  image-update-automation.yaml      push to flux-image-automation/main/lab-main
apps/                               workloads carrying setter markers
lobicornis/                         merge bot (Deployment+Service+CronJob)
.github/workflows/
  flux-auto-pr.yaml                 the workflow under test
  ci.yaml                           yamllint check (gives `gh pr checks --watch` something to wait on)
```

## Prerequisites

- `kind`, `kubectl`, `flux`, `gh`, `docker`
- A GitHub personal account (lab uses `darkweaver87` — replace with yours throughout)
- A GitHub PAT (classic, `repo` scope, or fine-grained with contents+PRs write on this repo). Two consumers use it:
  1. Flux `ImageUpdateAutomation` pushes via HTTPS
  2. Lobicornis hits the GitHub API to merge

## One-time GitHub setup

1. Create a public repo `darkweaver87/flux-image-automation-lab` and push this tree.
2. Create the 3 labels the workflow applies:
   ```bash
   gh label create bot/approve          --color BFD4F2
   gh label create bot/merge-method-ff  --color BFD4F2
   gh label create status/3-needs-merge --color D93F0B
   ```
3. Generate the PAT and stash it locally:
   ```bash
   export PAT='ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'
   ```
4. Set the PAT as a repo Actions secret so the workflow can open PRs:
   ```bash
   gh secret set TRAEFIKER_GITHUB_TOKEN --body "$PAT"
   ```
   (Name kept identical to prod so the workflow file is verbatim.)

## Cluster setup

```bash
kind create cluster --config kind/cluster.yaml

flux install \
  --version=v2.8.3 \
  --components-extra=image-reflector-controller,image-automation-controller

kubectl create secret generic github-credentials \
  -n flux-system \
  --from-literal=username=darkweaver87 \
  --from-literal=password="$PAT"

kubectl create namespace lobicornis
kubectl create secret generic lobicornis \
  -n lobicornis \
  --from-literal=github-token="$PAT"

kubectl apply -k bootstrap/
```

## Reconciliation walkthrough

Wait for everything to settle (~1-2 min):
```bash
flux get sources git -A
flux get kustomizations -A
flux get images all -A
kubectl -n lobicornis get pods
```

Expected:
- `GitRepository/lab-main` Ready
- `Kustomization/lab-flux`, `lab-apps`, `lab-lobicornis` Ready
- `ImageRepository/traefik` and `whoami` have `Latest image` set
- `ImagePolicy/traefik` → `traefik:3.0.0` (exact match)
- `ImagePolicy/whoami`  → `whoami:v1.10.0` (exact match)
- `ImageUpdateAutomation/lab-main` `LastPushCommit` empty / `No update made`
- Traefik + whoami pods Running in `lab-apps`
- Lobicornis Deployment Ready

## Test 1 — locked semvers, no PR

With the initial ranges (`=3.0.0`, `=1.10.0`) flux finds nothing newer, so no commit is pushed to `flux-image-automation/main/lab-main` and the workflow never fires.

```bash
git ls-remote origin 'refs/heads/flux-image-automation/*'   # should be empty
gh pr list                                                   # should be empty
```

## Test 2 — unlock both ranges, multi-setter PR + auto-merge

Loosen both `ImagePolicy` ranges so newer tags become eligible:

```yaml
# flux/image-policy.yaml — both policies
    semver:
      range: '>=3.0.0'   # traefik
      range: '>=1.10.0'  # whoami
```

Commit and push to `main`:
```bash
git add flux/image-policy.yaml
git commit -m "test: loosen image policy ranges"
git push origin main
```

Watch the chain:
```bash
flux reconcile source git lab-main
flux reconcile kustomization lab-flux
flux logs --kind=ImageUpdateAutomation --tail=20 -f
```

Within ~1 minute:
1. `ImageUpdateAutomation/lab-main` produces one commit on branch `flux-image-automation/main/lab-main` that updates **both** image tags (`apps/traefik/deployment.yaml` and `apps/whoami/deployment.yaml`).
2. GitHub Action `🤖 Flux Auto-PR` fires, opens a PR titled `ci(flux): :rocket: lab-main image automation updates`, waits for `ci` to pass, then adds `bot/approve`, `bot/merge-method-ff`, `status/3-needs-merge`.
3. Within the next 2 min (cronjob cadence), the lobicornis tick CronJob curls the lobicornis service, lobicornis sees the labels and fast-forward-merges the PR.
4. Flux re-syncs `main`, `Kustomization/lab-apps` reapplies the new image tags, both Deployments roll to the new versions.

Verify:
```bash
gh pr list --state merged --limit 5
kubectl -n lab-apps get deploy -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.template.spec.containers[0].image}{"\n"}{end}'
```

## Safety / scoping notes

- Lobicornis is configured with `github.user: darkweaver87`, which scopes its PR search to your personal repos. If you have **other** repos with `status/3-needs-merge` PRs, lobicornis will try to merge them too. Either avoid that label elsewhere, or flip `extra.dryRun: true` in `lobicornis/configmap.yaml` to gate first.
- The PAT has `repo` scope; treat it accordingly. The `github-credentials` secret in `flux-system` and the `lobicornis` secret in `lobicornis` both hold it.
- The lab repo is public so flux source-controller can read without auth. The PAT is only needed for write (image-automation push) and lobicornis API calls.

## Tear-down

```bash
kind delete cluster --name flux-auto-lab
gh repo delete darkweaver87/flux-image-automation-lab --yes
```

## Lobicornis merge probe

This section was added by a test PR to verify lobicornis can fast-forward-merge with the bot labels.
