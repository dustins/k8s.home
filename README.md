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
| `networking`    | `cilium`                       | cilium               | `kube-system`    |
| `networking`    | `traefik`                      | traefik              | `traefik`        |
| `security`      | `cert-manager`                 | cert-manager         | `cert-manager`   |
| `security`      | `kubelet-csr-approver`         | kubelet-csr-approver | `kube-system`    |
| `storage`       | `openebs`                      | openebs (ZFS LocalPV) | `openebs`        |
| `observability` | `metrics-server`               | metrics-server       | `kube-system`    |
| `observability` | `headlamp`                     | headlamp             | `kube-system`    |
| `delivery`      | `argocd`                       | argo-cd              | `argocd`         |
| `recovery`      | `velero`                       | velero               | `velero`         |

Every app directory has the same shape:

```
<tier>/<app>/
├── kustomization.yaml      # exactly one helmCharts entry: chart, version, repo, releaseName, namespace
├── values.additional.yaml  # our overrides on top of the chart's defaults
├── manifests/              # optional raw resources (namespaces, IngressRoutes, Cilium LB pools, ...)
├── commands                # notes on how to apply this directory by hand
└── charts/                 # gitignored; populated by `kustomize --enable-helm` when rendering locally
```

The kustomization is the contract. The directory name becomes the Application
name, and Argo CD reads the `helmCharts[0]` entry to pick the destination
(`namespace`), so **adding an app is adding a directory and removing an app is
deleting one**. Directory names match the Helm release name (`cilium`,
`metrics-server`).

## How Argo CD consumes this repo

Argo CD's own directory, `delivery/argocd`, carries six
`AppProject` + `ApplicationSet` pairs, one per tier, in `manifests/`. Each
ApplicationSet uses a git *files* generator over
`<tier>/*/kustomization.yaml` and templates an Application per match:

- **Name** is the app directory name; **namespace** comes from the
  kustomization's `helmCharts[0]`.
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
  storage, delivery and recovery, and `false` for observability. Deleting a
  directory in a preserved tier removes the Application but leaves the workload
  running. Deleting an observability directory tears the workload down too.

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
(needed by `kubectl kustomize --enable-helm`), the `argocd` CLI, and a
kubeconfig with cluster-admin. The SSH deploy key is generated in step 3.
On NixOS, get the `argocd` CLI with:

```sh
nix-channel --update
nix-shell -p argocd
```

Prerequisites on the nodes that live outside this repo:

- The API server is reachable at `10.1.40.32:6443` (cilium runs with
  kube-proxy replacement and points at it directly).
- A `k8s-svc-lb` interface exists on every node for Cilium L2 announcements.
- A ZFS pool exists for OpenEBS ZFS LocalPV.
- `/run/secrets/services/traefik/technitiumApiToken` and
  `/etc/ssl/certs/ca-bundle.crt` exist on nodes that can run Traefik. Traefik
  mounts them from the host to do ACME DNS-01 against Technitium and to trust
  the internal CA at `ca.core.infra.home`.

Off-cluster, Velero needs a Backblaze B2 bucket and an application key for it.
Neither is created by this repo; see [Backups](#backups).

### 1. Cilium (by hand, mandatory)

Nothing schedules without a CNI, so this is the one component that must be
installed before Argo CD can exist:

```sh
cd networking/cilium
kubectl kustomize --enable-helm | kubectl apply -f -
# the CiliumLoadBalancerIPPool and CiliumL2AnnouncementPolicy fail on the
# first pass because the operator has not registered their CRDs yet; once
# the operator is up, run it again
kubectl kustomize --enable-helm | kubectl apply -f -
```

Wait for the nodes to go `Ready`.

### 2. Argo CD (by hand, run twice)

```sh
cd delivery/argocd
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
ssh-keygen -t ed25519 -f /tmp/id_ed25519 -C "forgejo" -N ""
# In Forgejo: repo -> Settings -> Deploy Keys -> add /tmp/id_ed25519.pub (read-only)
argocd repo add ssh://forgejo@git.core.infra.home/dustins/k8s.home.git \
  --ssh-private-key-path /tmp/id_ed25519
```

If `repo add` fails with `ssh: unable to authenticate, attempted methods
[none publickey]`, the deploy key has not been added to the repo in Forgejo
yet.

The ApplicationSet git generators cannot produce anything until this
credential exists, so this step gates everything that follows.

### 4. Watch it converge

Within a minute the six ApplicationSets generate one Application per
directory and start syncing all of them in parallel. Cilium is adopted rather
than reinstalled. Expect a few apps to show `Progressing` or a failed attempt
for a few minutes while their dependencies arrive: Traefik's CRDs unblock the
IngressRoutes, and Cilium LB-IPAM hands Traefik its LoadBalancer IP at
`10.1.41.2`. Once it settles, `argocd.k8s.home` works and the port-forward can
go.

```sh
kubectl -n argocd get applications
```

Optionally, Traefik can also be applied by hand (same command as cilium)
between steps 1 and 2. That makes the Argo CD UI reachable at
`argocd.k8s.home` immediately and avoids the IngressRoute error, but it is not
required. Argo CD adopts anything already installed.

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
| cilium CRDs         | `CiliumLoadBalancerIPPool`, `CiliumL2AnnouncementPolicy` in cilium itself | Soft. Same Application, retry converges.                |
| cilium LB-IPAM      | traefik's LoadBalancer IP                              | Soft. The Service sits `<pending>` until LB-IPAM assigns it. No failure.   |
| kubelet-csr-approver| metrics-server scraping kubelets                       | Soft, runtime only. Metrics are empty until CSRs get approved.             |
| velero node secret  | velero taking any backup                               | Soft, runtime only. The app syncs, the BSL goes Unavailable.               |

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

## Backups

`recovery/velero` runs Velero, which writes a backup of the cluster's
Kubernetes API objects to a Backblaze B2 bucket every night at 02:00 and keeps
them for 30 days. B2 is reached over its S3-compatible API with
`velero-plugin-for-aws`; there is no B2-native plugin and none is needed.

**Volume data is deliberately not backed up.** The nightly backup captures
every namespace's API objects, PVC and PV objects included, but not a single
byte of volume content: `snapshotVolumes` and `defaultVolumesToFsBackup` are
both `false`, and the node-agent DaemonSet is not deployed. So a restore today
rebuilds the cluster's shape and brings volumes back **empty**. That is the
intended state until Ceph replaces OpenEBS ZFS LocalPV.

The plan for volume data is Velero's own CSI snapshot data mover rather than a
second tool: it takes a CSI `VolumeSnapshot`, then a short-lived pod uploads the
snapshot through Kopia to this same bucket, so the first copy of a volume is
full and every copy after it is a diff. Opt-in is per-PVC by label, so backed-up
volumes are declared rather than swept up, and restore stays the same
`velero restore create` used for the API objects. The exact four changes needed
to turn it on are written out at the bottom of
`recovery/velero/values.additional.yaml`. OpenEBS ZFS LocalPV cannot be used
for this even though it supports CSI snapshots, because its snapshots are
node-local.

### Backblaze B2 credentials

The B2 application key follows Traefik's Technitium token, not the Argo CD
deploy key: it is a **sops-nix secret on the nodes**, mounted into the Velero
pod over `hostPath`. Nothing about the credential lives in this repo or in the
cluster's etcd, and because it is a property of the node rather than the
cluster, it survives a cluster rebuild with no manual step.

The node side is in the `core.home` repo:

- `roles/compute-infra/services/velero.nix` declares the secret and the
  `velero` group, gated behind `rolesConfig.compute-infra.veleroBackup`.
  sops-nix fails activation on a declared secret whose file is missing, which
  is why the flag exists rather than the declaration being committed inert —
  the same reason `rustic.nix` next to it is gated.
- `secrets/roles/compute-infra/velero/credentials` holds the key, encrypted to
  the `acorn`, `mini` and admin age keys by the existing
  `secrets/roles/compute-infra` creation rule in `.sops.yaml`.

Two numbers have to agree: the secret is `0440 root:velero` with
`users.groups.velero.gid = 65531` on the host, and the pod carries that gid via
`podSecurityContext.supplementalGroups` in `values.additional.yaml`. It is
`supplementalGroups` and not `fsGroup` because `fsGroup` only re-owns volume
types that support ownership management, and `hostPath` is not one of them.

One operational wrinkle: sops-nix flips `/run/secrets` to a new generation
directory on every activation, so a `colmena` deploy that rotates secrets
leaves the running pod's bind mount pointing at a deleted inode. Restart the
deployment after one:

```sh
kubectl -n velero rollout restart deploy/velero
```

A SOPS-encrypted Secret committed to *this* repo and decrypted by a KSOPS
plugin in `argocd-repo-server` was considered and rejected: the age key would
still have to be created by hand, so the manual step moves rather than
disappears, and it puts a third-party plugin in the render path of every app
here. If this cluster grows enough secrets to want a real in-cluster story, the
answer is External Secrets Operator against the Vault instance that `bedrock`
already manages — not KSOPS.

### Restoring

```sh
nix-shell -p velero
velero backup get
velero backup describe <name> --details
velero restore create --from-backup <name> --include-namespaces <ns>
```

Argo CD is the better restore path for anything this repo already describes:
delete the namespace and let the Application re-sync. Velero is for what the
repo does *not* describe — hand-created Secrets, CRs written by controllers,
and anything stateful once volume data is turned on.

## One-time: retiring MetalLB

LoadBalancer addressing moved from MetalLB to Cilium's own LB-IPAM and L2
announcer. Deleting `networking/metallb/` removed the Application, but the
networking tier has `preserveResourcesOnDeletion: true`, so **the MetalLB
workload keeps running in the cluster until it is removed by hand**. Leaving it
up is not harmless: its controller still allocates from its own
`IPAddressPool` and its speaker still ARPs for `10.1.41.2`, so both it and
Cilium would fight over the same Service and the same VIP.

On a cluster that was running MetalLB, do this once after this commit syncs:

```sh
# confirm cilium took over before tearing anything down
kubectl -n traefik get svc traefik -o jsonpath='{.status.loadBalancer.ingress[0].ip}{"\n"}'
kubectl get ciliumloadbalancerippools
kubectl -n kube-system get leases | grep l2announce

# then remove metallb: CRs first, so no webhook rejects the CRD delete
kubectl -n metallb-system delete l2advertisements,ipaddresspools --all
kubectl delete crd -l app.kubernetes.io/name=metallb
kubectl delete namespace metallb-system
kubectl delete clusterrole,clusterrolebinding -l app.kubernetes.io/name=metallb
kubectl delete validatingwebhookconfiguration,mutatingwebhookconfiguration \
  -l app.kubernetes.io/name=metallb
```

A fresh cluster never had MetalLB and can skip this entirely.

## Cluster-specific values worth knowing

- Pod CIDR `10.42.0.0/16`, native routing, no tunnel, kube-proxy replacement.
- LoadBalancer addressing is Cilium's, not MetalLB's: LB-IPAM allocates from
  `CiliumLoadBalancerIPPool` and the L2 announcer ARPs for the result on the
  `k8s-svc-lb` interface. Both live in `networking/cilium/manifests/`.
- LoadBalancer pool blocks: `10.1.41.2-10.1.41.31` and
  `10.1.41.127-10.1.41.250`. The gap leaves room for bare-metal interface IPs.
  A Service pins an address with `lbipam.cilium.io/ips`.
- Traefik answers on `10.1.41.2` and issues a wildcard `*.k8s.home`
  certificate from the internal ACME CA at `ca.core.infra.home` using a Technitium
  DNS-01 challenge.
- Dashboards: `argocd.k8s.home`, `traefik.k8s.home`, `headlamp.k8s.home`.
- kubelet-csr-approver only signs CSRs from `10.1.40.32/25`.
