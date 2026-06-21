# CloudPulse — Config (GitOps Manifests)

> **Config repo** — the single source of truth for what runs in the cluster.
> FluxCD watches this repo and continuously reconciles the EKS cluster to match it.

This repository holds the Kubernetes manifests for the CloudPulse application.
It is deliberately kept **separate from the application code** (`cloudpulse-app`)
so that the deploy pipeline can commit a new image tag here **without
re-triggering the build pipeline** — this is standard GitOps separation and it
prevents the build→commit→build webhook loop.

---

## The 5 Repositories

```
Phase 1 → cloudpulse-bootstrap  → Creates Jenkins + Ansible EC2   [Terraform, run locally]
Phase 2 → cloudpulse-ansible    → Configures the Jenkins server    [Ansible, from Ansible server]
Phase 3 → cloudpulse-infra      → Creates VPC + EKS + ECR + Flux   [Terraform via Jenkins]
Phase 4 → cloudpulse-app        → Builds & pushes the image        [Jenkins, on every git push]
Phase 5 → cloudpulse-config     → Holds k8s manifests (THIS)       [Flux watches; pipeline commits tag]
```

---

## How Deployment Works

```
cloudpulse-app pipeline                 cloudpulse-config (this repo)
─────────────────────────               ─────────────────────────────
build image → push to ECR
        │
        └── commit new image tag ───────►  k8s/deployment.yaml updated
                                                     │
                                                     ▼
                                    FluxCD (inside EKS) watches this repo
                                    every ~1 min → applies the change → deploy ✅
```

There is **no Jenkins webhook** on this repo, so the pipeline's tag-commit does
**not** start another build. Flux (pull-based) only reads this repo — it never
pushes back. That is what breaks the self-trigger loop.

---

## Repository Structure

```
cloudpulse-config/
├── k8s/
│   ├── namespace.yaml      # cloudpulse namespace
│   ├── deployment.yaml     # 2 replicas, rolling update, probes, standard labels
│   ├── service.yaml        # LoadBalancer
│   └── kustomization.yaml  # Apply order for Flux (namespace → deploy → service)
│   └── flux-system/        # Added automatically by `flux bootstrap` (GitRepository + Kustomization)
├── .gitignore
└── README.md
```

> `k8s/flux-system/` is created by the `cloudpulse-infra` pipeline when it runs
> `flux bootstrap github --repository=cloudpulse-config --path=k8s`. Do not edit
> it by hand.

---

## Releasing a New Version

You don't edit this repo manually for releases. Instead:

1. Change the app version in `cloudpulse-app` → `git push`
2. The app pipeline builds, scans, pushes to ECR, then **commits the new image
   tag into `k8s/deployment.yaml` here**
3. Flux detects the commit and rolls out the new version automatically

---

## Why a Separate Config Repo?

| Benefit | Explanation |
|---------|-------------|
| No build loop | The tag-commit lands here (no webhook), so it can't re-trigger the app build |
| Clear audit trail | Every deploy is a commit in this repo's history — what changed, when, by whom |
| Separation of concerns | App code lifecycle (frequent) is independent of deploy config (release events) |
| Single source of truth | The cluster always matches this repo; manual drift is auto-reverted by Flux |
