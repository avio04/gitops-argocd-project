# GitOps Deployment using ArgoCD

A hands-on GitOps project demonstrating declarative, Git-driven Kubernetes deployments using ArgoCD — including auto-sync, self-healing, and rollback strategies, running on a local Kubernetes cluster (minikube).

## Overview

This project implements the **GitOps** deployment model: Git is the single source of truth for what should be running in the cluster. Instead of running `kubectl apply` manually, changes are pushed to this repository and **ArgoCD** continuously reconciles the live cluster state to match it.

**Core principle demonstrated:** no `kubectl apply` was used to deploy or update the application at any point after the initial ArgoCD Application was created — every change went through Git.

## Architecture

```
 ┌─────────────────┐         watches/polls         ┌──────────────────┐
 │   GitHub Repo    │ ─────────────────────────────▶│      ArgoCD      │
 │  (this repo)     │                                │  (in-cluster)    │
 │  manifests/       │                                │                  │
 │   - deployment    │◀────── compares desired ───────│  App Controller  │
 │   - service       │        vs actual state         │                  │
 │   - configmap     │                                └────────┬─────────┘
 └─────────────────┘                                         │
                                                     applies diff (sync)
                                                              │
                                                              ▼
                                                  ┌─────────────────────┐
                                                  │  Kubernetes Cluster │
                                                  │     (minikube)      │
                                                  │                     │
                                                  │  Deployment (2 pods)│
                                                  │  Service (NodePort) │
                                                  │  ConfigMap (HTML)   │
                                                  └─────────────────────┘
```

## Tech Stack

- **Kubernetes**: minikube (local, Docker driver)
- **GitOps Engine**: ArgoCD
- **Sample App**: nginx serving HTML from a ConfigMap (no container registry needed)
- **Manifests**: plain Kubernetes YAML (Deployment, Service, ConfigMap)
- **Version Control**: Git / GitHub

## Repository Structure

```
gitops-argocd-project/
├── manifests/
│   ├── deployment.yaml   # nginx Deployment, 2 replicas
│   ├── service.yaml      # NodePort Service
│   └── configmap.yaml    # HTML content served by nginx
└── README.md
```

## Setup Steps

1. **Local cluster**: `minikube start --driver=docker`
2. **Install ArgoCD** into the cluster:
   ```
   kubectl create namespace argocd
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   ```
3. **Access the UI**:
   ```
   kubectl port-forward svc/argocd-server -n argocd 8080:443
   ```
   Login at `https://localhost:8080` with the auto-generated admin password:
   ```
   kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath="{.data.password}" | base64 -d
   ```
4. **Create the ArgoCD Application** pointing at this repo's `manifests/` path, targeting the in-cluster destination.
5. **Sync** — ArgoCD reads the manifests and deploys the ConfigMap, Deployment, and Service.
6. **Verify**:
   ```
   minikube service sample-app-service -n default --url
   ```

## GitOps Workflow Demonstrated

### 1. Declarative sync
The ArgoCD Application continuously compares the Git repo's `manifests/` folder against the live cluster and reports `Synced`/`OutOfSync` accordingly.

### 2. Auto-sync + Self-heal
Enabled via Sync Policy: `Automated`, `Prune Resources`, `Self Heal`.

- **Auto-sync**: pushing a change to `configmap.yaml` and refreshing ArgoCD applies it automatically — no manual Sync click needed.
- **Self-heal**: manually running `kubectl scale deployment sample-app --replicas=5` (bypassing Git) was automatically reverted back to `replicas: 2` by ArgoCD within seconds, since Git still declared 2 replicas. This proves manual cluster drift cannot persist under GitOps.

### 3. Rollback
Two rollback strategies were tested:

- **ArgoCD UI rollback** (via History and Rollback): redeploys a previous revision's manifests to the cluster *without changing Git*. Useful for an immediate "stop the bleeding" action, but **fragile** — since Git's `HEAD` still points at the bad commit, the next sync (manual or auto) re-applies the bad version. ArgoCD explicitly disables auto-sync when you attempt a UI rollback, for exactly this reason.
- **`git revert`** (used to finish this demo): creates a new commit undoing the bad change, so Git and cluster state converge again. This is the safer, audit-friendly approach for real teams, since auto-sync and self-heal can stay enabled the whole time.

## Troubleshooting Log

Real issues hit during this build, and how they were diagnosed and fixed — kept here because working through incidents like these is a core part of the DevOps/GitOps skill set.

**1. `argocd-applicationset-controller` stuck in `CrashLoopBackOff`**
- **Symptom**: pod kept restarting with increasing restart counts.
- **Root cause**: `kubectl logs --previous` showed `no matches for kind "ApplicationSet" in version "argoproj.io/v1alpha1"` — the `ApplicationSet` CRD was missing from the cluster (drift from an earlier, unrelated project's ArgoCD install).
- **Fix**: re-applied the official ArgoCD install manifest, which is idempotent — it restored the missing CRD without disturbing already-healthy components.

**2. Namespace mismatch on install**
- **Symptom**: installing into a custom namespace name still caused some resources to land in the default `argocd` namespace instead.
- **Root cause**: ArgoCD's official install manifest hardcodes `namespace: argocd` inside several resource definitions (ConfigMaps, Secrets), which ignores the `-n <namespace>` flag on `kubectl apply` for those specific objects.
- **Fix**: standardized on the default `argocd` namespace, as all ArgoCD tooling/docs assume it.

**3. Git push rejected — divergent branches**
- **Symptom**: `git push` failed with `[rejected] ... (fetch first)`.
- **Root cause**: a file had been edited directly on GitHub's web UI at some point, so the local repo's history and `origin/main` diverged (confirmed via the commit author showing GitHub's `noreply` email format instead of the configured local Git email).
- **Fix**: `git pull --rebase origin main`, manually resolved the merge conflict in `configmap.yaml`, then `git rebase --continue` and pushed.

**4. Rollback silently undone by auto-sync**
- **Symptom**: after rolling back to a known-good revision in the ArgoCD UI, the app reverted right back to the "bad" version after a manual Sync.
- **Root cause**: manual Sync always deploys Git's current `HEAD` — since the bad commit was still the latest thing on `main`, syncing re-applied it, overwriting the rollback.
- **Fix**: switched to `git revert` instead, which changes what Git considers "current," making the fix durable under auto-sync.

**5. Forgot to push after committing a fix**
- **Symptom**: ArgoCD kept showing the old (bad) commit as `LAST SYNC` even after supposedly fixing things.
- **Root cause**: a local revert commit existed but had never actually been pushed to `origin/main` — ArgoCD only ever reads from GitHub, never a local machine.
- **Fix**: `git push origin main`; confirmed by checking `git log --oneline` for both local `HEAD` and `origin/main` matching.

## Key Learnings

- ArgoCD's Application Controller reconciles based on **Git alone** — local filesystem state is invisible to it until pushed.
- **Self-heal** is what makes GitOps trustworthy: manual `kubectl` changes cannot persist, which prevents "hidden" production drift.
- **Auto-sync and manual/UI rollback actively conflict** — understanding this interaction matters more than either feature alone.
- ConfigMap-only changes do **not** trigger a rolling pod restart by default (no new ReplicaSet) — Kubernetes only considers the Pod template (image, env vars, etc.) when deciding whether to roll pods. In production, teams often add a checksum annotation on the pod template to force a restart when a mounted ConfigMap changes.

## Possible Future Improvements

- Add a **checksum/hash annotation** on the Deployment's pod template to force rolling restarts on ConfigMap changes.
- Split into **staging/prod overlays** using Kustomize.
- Replace Git polling with a **GitHub webhook** for instant sync instead of the ~3 minute default poll interval.
- Move from a ConfigMap-mounted static page to a real containerized app with its own image tag, to demonstrate image-based rolling updates.
