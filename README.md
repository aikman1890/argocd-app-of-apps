# argocd-app-of-apps

GitOps bootstrap for Argo CD using the app-of-apps pattern: one root Application watches the `apps/` directory, and every child Application in that directory is picked up automatically. Add a YAML file, get a deployment — no UI clicking required.

## Layout

```text
argocd-app-of-apps/
├── apps/
│   ├── root-app.yaml        # The root app-of-apps Application
│   ├── dev/
│   │   └── app.yaml         # Child Application: dev (auto-sync)
│   └── prod/
│       └── app.yaml         # Child Application: prod (manual sync, gated)
└── applicationsets/
    └── clusters.yaml        # ApplicationSet that fans out per registered cluster
```

## Bootstrap steps

1. **Install Argo CD** in your management cluster using the official manifests:

   ```bash
   kubectl create namespace argocd
   kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
   ```

   Wait for the pods to be ready:

   ```bash
   kubectl wait --for=condition=available -n argocd deployment/argocd-server --timeout=300s
   ```

2. **Point the manifests at your repo.** Replace the `CHANGE_ME` repo URL in `apps/root-app.yaml`, `apps/dev/app.yaml`, `apps/prod/app.yaml`, and `applicationsets/clusters.yaml` with the Git URL of this repository.

3. **Apply the root app:**

   ```bash
   kubectl apply -f apps/root-app.yaml
   ```

   The root Application syncs the `apps/` directory, which creates the dev and prod child Applications. Argo CD then syncs those according to their own sync policies — dev goes automatically, prod waits for a human to press sync.

4. **(Optional) Register workload clusters** and let the ApplicationSet in `applicationsets/clusters.yaml` generate one Application per cluster:

   ```bash
   argocd cluster add <cluster-context>
   kubectl apply -f applicationsets/clusters.yaml
   ```

## How it fans out

```text
root-app (syncs apps/)
├── dev/app.yaml   → syncs apps/dev  → dev workloads, automated sync
├── prod/app.yaml  → syncs apps/prod → prod workloads, manual sync (change control)
└── applicationsets/clusters.yaml → one app per registered cluster
```

Child Applications live in the same repo, so a pull request that adds `apps/staging/app.yaml` gets reviewed like any other code change — and applying it deploys staging. That reviewability is the whole point of the pattern.

## Sync policy philosophy

- **dev**: automated sync with prune and self-heal. Dev should always match Git; drift gets corrected without ceremony.
- **prod**: manual sync. Nothing reaches prod without a person looking at the diff and pressing the button. Pair this with branch protection on your Git repo for a real change-control story.
