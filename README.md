# k8s.home

GitOps source of truth for the `k8s.home` Kubernetes cluster. Everything in
this repo is applied declaratively by Argo CD, including Argo CD itself. Apart
from the CNI and the first Argo CD install, nothing is ever `kubectl apply`'d
by hand.

## Layout

The repo is organised into **tiers**. Each tier is a top-level directory, and
each app inside a tier is one directory containing a kustomization with a
single Helm chart:

| Tier            | App directory                  | Chart                | Namespace        |
| --------------- | ------------------------------ | -------------------- | ---------------- |
| `networking`    | `io.cilium`                    | cilium               | `kube-system`    |
| `networking`    | `io.metallb`                   | metallb              | `metallb-system` |
| `networking`    | `io.traefik`                   | traefik              | `traefik`        |
| `security`      | `io.cert-manager`              | cert-manager         | `cert-manager`   |
| `security`      | `kubelet-csr-approver`         | kubelet-csr-approver | `kube-system`    |
| `storage`       | `io.openebs`                   | openebs (ZFS LocalPV) | `openebs`        |
| `observability` | `io.k8s.sigs.metrics-server`   | metrics-server       | `kube-system`    |
| `observability` | `io.k8s.sigs.headlamp`         | headlamp             | `kube-system`    |
| `delivery`      | `io.argoproj.argocd`           | argo-cd              | `argocd`         |

Every app directory has the same shape:

```
<tier>/<app>/
├── kustomization.yaml      # exactly one helmCharts entry: chart, version, repo, releaseName, namespace
├── values.additional.yaml  # our overrides on top of the chart's defaults
├── manifests/              # optional raw resources (namespaces, IngressRoutes, MetalLB pools, ...)
├── commands                # notes on how to apply this directory by hand
└── charts/                 # gitignored; populated by `kustomize --enable-helm` when rendering locally
```

The kustomization is the contract. Argo CD reads the `helmCharts[0]` entry to
name the Application (`releaseName`) and pick its destination
(`namespace`), so **adding an app is adding a directory and removing an app is
deleting one**. Directory names follow the project's reverse-DNS domain
(`io.cilium`, `io.k8s.sigs.headlamp`) where it has one.

## How Argo CD consumes this repo

Argo CD's own directory, `delivery/io.argoproj.argocd`, carries five
`AppProject` + `ApplicationSet` pairs, one per tier, in `manifests/`. Each
ApplicationSet uses a git *files* generator over
`<tier>/*/kustomization.yaml` and templates an Application per match:

- **Name and namespace** come from the kustomization's `helmCharts[0]`.
- **Sync policy** is `automated` with `prune` and `selfHeal`, so the cluster
  always converges on `main` and hand edits in the cluster are reverted.
- **Server-side apply** is on for every app. The cilium and cert-manager CRDs
  are too large for the client-side last-applied annotation.
- **`CreateNamespace`** and **`ApplyOutOfSyncOnly`** are on.
- **Retry** with exponential backoff (up to 10 attempts, 15s doubling to a
  5m cap, roughly half an hour in total). This is what lets apps that depend on
  another app's CRDs or webhooks converge without explicit ordering. See
  [Ordering and dependencies](#ordering-and-dependencies).
- **`preserveResourcesOnDeletion`** is `true` for networking, security,
  storage and delivery, and `false` for observability. Deleting a directory in a
  preserved tier removes the Application but leaves the workload running.
  Deleting an observability directory tears the workload down too.

Two details in the Argo CD kustomization make self-management work:

- `argocd-cm` is patched with `kustomize.buildOptions: --enable-helm` so the
  repo-server can render the `helmCharts` entries in every directory.
- The Argo CD chart is rendered with `includeCRDs: true`, so the Application,
  AppProject and ApplicationSet CRDs are part of the same kustomization as the
  ApplicationSets that use them. That is why the first-time install has to run
  twice (see below).

Repository access is over SSH to
`ssh://forgejo@git.core.infra.home/dustins/k8s.home.git`. The host key and the
internal CA chain are in `values.additional.yaml`. The private deploy key is
**not** in git and is added once by hand at bootstrap.

## Bootstrapping a fresh cluster

Prerequisites on the machine driving the bootstrap: `kubectl`, `helm`
(needed by `kubectl kustomize --enable-helm`), the `argocd` CLI, a kubeconfig
with cluster-admin, and the read-only SSH deploy key for this repo.

Prerequisites on the nodes that live outside this repo:

- The API server is reachable at `10.1.40.32:6443` (cilium runs with
  kube-proxy replacement and points at it directly).
- A `k8s-svc-lb` interface exists on every node for MetalLB L2 advertisement.
- A ZFS pool exists for OpenEBS ZFS LocalPV.
- `/run/secrets/services/traefik/technitiumApiToken` and
  `/etc/ssl/certs/ca-bundle.crt` exist on nodes that can run Traefik. Traefik
  mounts them from the host to do ACME DNS-01 against Technitium and to trust
  the internal CA at `ca.core.infra.home`.

### 1. Cilium (by hand, mandatory)

Nothing schedules without a CNI, so this is the one component that must be
installed before Argo CD can exist:

```sh
cd networking/io.cilium
kubectl kustomize --enable-helm | kubectl apply -f -
```

Wait for the nodes to go `Ready`.

### 2. Argo CD (by hand, run twice)

```sh
cd delivery/io.argoproj.argocd
kubectl kustomize --enable-helm | kubectl apply --server-side --force-conflicts -f -
# wait a few seconds for the CRDs to be established, then run it again
kubectl kustomize --enable-helm | kubectl apply --server-side --force-conflicts -f -
```

The first pass installs Argo CD and its CRDs but fails on the
`AppProject`/`ApplicationSet` resources with `no matches for kind`. The
second pass lands them. Both passes will also fail on the `IngressRoute`
because Traefik's CRDs do not exist yet. That is expected. Argo CD will apply
it itself once Traefik is up.

### 3. Give Argo CD access to this repo (by hand, once)

```sh
kubectl -n argocd port-forward svc/argocd-server 8080:443 &
argocd login localhost:8080 --username admin \
  --password "$(kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d)"
argocd repo add ssh://forgejo@git.core.infra.home/dustins/k8s.home.git \
  --ssh-private-key-path ~/.ssh/<read-only-deploy-key>
```

The ApplicationSet git generators cannot produce anything until this
credential exists, so this step gates everything that follows.

### 4. Watch it converge

Within a minute the five ApplicationSets generate one Application per
directory and start syncing all of them in parallel. Cilium is adopted rather
than reinstalled. Expect a few apps to show `Progressing` or a failed attempt
for a few minutes while their dependencies arrive: Traefik's CRDs unblock the
IngressRoutes, MetalLB's webhook unblocks its own `IPAddressPool`, and MetalLB
hands Traefik its LoadBalancer IP at `10.1.41.2`. Once it settles,
`argocd.k8s.home` works and the port-forward can go.

```sh
kubectl -n argocd get applications
```

Optionally, MetalLB and Traefik can also be applied by hand (same command as
cilium, MetalLB may need two passes) between steps 1 and 2. That makes the
Argo CD UI reachable at `argocd.k8s.home` immediately and avoids the
IngressRoute error, but it is not required. Argo CD adopts anything already
installed.

## Ordering and dependencies

ApplicationSets cannot be prioritised against each other, and this repo does
not try to. Argo CD reconciles every Application independently, and the retry
policy on each one is what turns the dependency graph into eventual
convergence. The actual dependencies are:

| Depends on          | Needed by                                              | Kind of dependency                                                          |
| ------------------- | ------------------------------------------------------ | --------------------------------------------------------------------------- |
| cilium              | everything                                             | **Hard.** No pods run without it. Installed by hand before Argo CD.        |
| Argo CD CRDs        | the ApplicationSets in Argo CD's own directory         | **Hard at apply time.** Handled by running the install twice.              |
| repo credential     | every ApplicationSet                                   | **Hard.** Nothing generates until `argocd repo add` has run.               |
| traefik CRDs        | `IngressRoute` in argocd and headlamp                  | Soft. Apply fails until the CRD exists, retry converges.                   |
| metallb CRDs/webhook| `IPAddressPool`, `L2Advertisement` in metallb itself   | Soft. Same Application, retry converges.                                   |
| metallb             | traefik's LoadBalancer IP                              | Soft. The Service sits `<pending>` until MetalLB assigns it. No failure.   |
| kubelet-csr-approver| metrics-server scraping kubelets                       | Soft, runtime only. Metrics are empty until CSRs get approved.             |

Only the first three are hard, and all three are handled in the bootstrap
steps above. Everything else is "apply fails or sits pending for a few
minutes, then succeeds", which the retry window covers with a lot of room.

If you ever do want explicit ordering, the options are:

- **Sync waves** order resources inside one Application. They do not order
  Applications against each other unless a parent Application owns them
  (app-of-apps) and a custom health check makes the parent wait on child
  health. That would mean replacing the ApplicationSets, so it is not worth it.
- **ApplicationSet progressive syncs** (`strategy.type: RollingSync`) order
  the Applications generated by a single ApplicationSet into steps by label.
  The `k8s.home/tier` label every Application already carries would make this
  a natural fit, but it needs the five tier ApplicationSets collapsed into one,
  the alpha feature flag turned on in the controller, and it changes who
  drives syncs. Hold off unless convergence ever actually fails.

One thing to know: Argo CD will not auto-sync a revision again after its sync
has failed, so an app that exhausts its retries stays `SyncFailed` until the
next commit or a manual sync:

```sh
argocd app sync <name>
```

## Day-to-day

**Add an app.** Create `<tier>/<app>/kustomization.yaml` with one
`helmCharts` entry (copy a neighbour), a `values.additional.yaml`, and any raw
resources under `manifests/`. Commit to `main`. The Application appears
within the ApplicationSet's next reconcile.

**Remove an app.** Delete the directory and commit. Whether the workload is
also removed depends on the tier's `preserveResourcesOnDeletion` (see above).

**Render locally** to check what Argo CD will see:

```sh
kubectl kustomize --enable-helm <tier>/<app>
```

**Diff live state against `main`** for one app:

```sh
argocd app diff <name>
```

**Chart upgrades** are automated. A Renovate workflow in `.forgejo/workflows`
runs daily, bumps the `version` in each `kustomization.yaml`, and opens one
grouped PR for Helm chart versions. Merging the PR is the upgrade.

**Hand edits in the cluster get reverted.** `selfHeal` is on everywhere. Make
the change here instead.

## Cluster-specific values worth knowing

- Pod CIDR `10.42.0.0/16`, native routing, no tunnel, kube-proxy replacement.
- MetalLB pools: `10.1.41.2-10.1.41.31` and `10.1.41.127-10.1.41.250`. The
  gap leaves room for bare-metal interface IPs.
- Traefik answers on `10.1.41.2` and issues a wildcard `*.k8s.home`
  certificate from the internal ACME CA at `ca.core.infra.home` using a Technitium
  DNS-01 challenge.
- Dashboards: `argocd.k8s.home`, `traefik.k8s.home`, `headlamp.k8s.home`.
- kubelet-csr-approver only signs CSRs from `10.1.40.32/25`.
