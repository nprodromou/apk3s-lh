# apk3s-lh

Kubernetes cluster on 3 Mac Studio nodes. Talos Linux, Flux GitOps, Longhorn storage, Cilium CNI.

Based on [onedr0p/cluster-template](https://github.com/onedr0p/cluster-template), modified for single-disk Longhorn storage.

## Network Map

```
Network:  10.50.50.0/24
Gateway:  10.50.50.1
API VIP:  10.50.50.50    (kubevip.prodromou.com)
DNS LB:   10.50.50.100   (k8s-gateway)
Int GW:   10.50.50.150   (envoy-internal)
Ext GW:   10.50.50.200   (envoy-external)
Domain:   prodromou.com
```

| Hostname | IP | MAC | Disk Serial |
|----------|------|------|-------------|
| `apk3s51` | `10.50.50.51` | `00:3e:e1:ca:65:ac` | `OW23092715AEFDC5F` |
| `apk3s52` | `10.50.50.52` | `00:3e:e1:c4:89:b4` | `OW23091215A17F66C` |
| `apk3s53` | `10.50.50.53` | `00:3e:e1:ca:74:a4` | `OW23092715AEFDC52` |

**Talos Schematic ID:** `613e1592b2da41ae5e265e8789429f22e121aab91cb4deb6bc3c0b6262961245`

---

## Deploy / Rebuild

### Prerequisites

- All 3 nodes powered on and booted into Talos (either from a previous install or a fresh USB flash from [Talos Image Factory](https://factory.talos.dev) using the schematic ID above)
- Cloudflare API token with `Zone - DNS - Edit` and `Account - Cloudflare Tunnel - Read` permissions (create one [here](https://developers.cloudflare.com/fundamentals/api/get-started/create-token/) using the `Edit zone DNS` template, named `kubernetes`)
- Cloudflare Tunnel credentials file (`cloudflare-tunnel.json`)

Verify nodes are reachable before starting:

```sh
nmap -Pn -n -p 50000 10.50.50.0/24 -vv | grep 'Discovered'
```

### Step 1: Workstation Setup

> Skip this step if you already have the repo cloned and tools installed.

```sh
git clone https://github.com/nprodromou/apk3s-lh.git && cd apk3s-lh
```

Install [Mise CLI](https://mise.jdx.dev/getting-started.html#installing-mise-cli), then [activate it](https://mise.jdx.dev/getting-started.html#activate-mise), then install tools:

```sh
mise trust
pip install pipx
mise install
```

Logout of GHCR to avoid auth issues with public registries:

```sh
docker logout ghcr.io
helm registry logout ghcr.io
```

### Step 2: Cloudflare Tunnel

> Skip this step if you already have a `cloudflare-tunnel.json` in the repo root.

```sh
cloudflared tunnel login
cloudflared tunnel create --credentials-file cloudflare-tunnel.json kubernetes
```

### Step 3: Configure

```sh
task init
```

This copies `cluster.sample.yaml` -> `cluster.yaml` and `nodes.sample.yaml` -> `nodes.yaml`. All values are pre-filled except one:

**Open `cluster.yaml` and paste your `cloudflare_token`.**

Then render all config:

```sh
task configure
```

### Step 4: Commit and Push

```sh
git add -A
git commit -m "chore: initial commit :rocket:"
git push
```

### Step 5: Bootstrap

> [!WARNING]
> This takes 10+ minutes. You'll see errors like "couldn't get current server API group list" and "no matching resources found" while things come up. **This is normal.** If interrupted with Ctrl+C, you may need to `task talos:reset` before retrying.

```sh
task bootstrap:talos
```

Commit the generated Talos secrets:

```sh
git add -A
git commit -m "chore: add talhelper encrypted secret :lock:"
git push
```

Install Cilium, CoreDNS, Spegel, Flux:

```sh
task bootstrap:apps
```

### Step 6: Verify

Watch pods come up:

```sh
kubectl get pods --all-namespaces --watch
```

Verify Longhorn partition exists on each node:

```sh
talosctl -n 10.50.50.51 mounts | grep longhorn
talosctl -n 10.50.50.52 mounts | grep longhorn
talosctl -n 10.50.50.53 mounts | grep longhorn
# Each should show: /var/mnt/longhorn
```

Verify Longhorn pods and nodes (Flux deploys this automatically):

```sh
kubectl -n longhorn-system get pods
kubectl -n longhorn-system get nodes.longhorn.io
```

Label nodes for disk auto-detection:

```sh
kubectl label nodes --all node.longhorn.io/create-default-disk=config
kubectl annotate nodes --all node.longhorn.io/default-disks-config='[{"path":"/var/lib/longhorn","allowScheduling":true}]'
```

Verify the rest of the stack:

```sh
cilium status
flux check
flux get sources git flux-system
flux get ks -A
flux get hr -A
nmap -Pn -n -p 443 10.50.50.150 10.50.50.200 -vv
kubectl -n network describe certificates
```

**The cluster is operational once all Flux Kustomizations and HelmReleases show as ready.**

---

## Disaster Recovery

### Single Node Failure

Data is safe — replicas exist on the other 2 nodes. Longhorn rebuilds automatically after the node rejoins.

```sh
# 1. Reinstall Talos on the failed node (e.g., apk3s51)
talosctl apply-config --insecure -n 10.50.50.51 --file clusterconfig/apk3s51.yaml

# 2. Wait for node to rejoin
kubectl get nodes -w

# 3. Check Longhorn sees the node
kubectl -n longhorn-system get nodes.longhorn.io

# 4. If disk shows unavailable, fix it:
kubectl -n longhorn-system edit nodes.longhorn.io apk3s51
```

In the editor, ensure the disk config is correct:

```yaml
spec:
  disks:
    longhorn-disk:
      path: /var/lib/longhorn
      allowScheduling: true
      storageReserved: 0
```

```sh
# 5. Watch replicas rebuild
kubectl -n longhorn-system get replicas -w

# 6. Verify volume health
kubectl -n longhorn-system get volumes.longhorn.io
```

### Full Cluster Rebuild

1. Ensure you have backups of critical PVC data (Longhorn snapshots to NFS/S3)
2. Follow the [Deploy / Rebuild](#deploy--rebuild) procedure above
3. Restore PVCs from Longhorn backup targets

### Reset Cluster

> [!CAUTION]
> Resetting multiple times in a short period can get you rate-limited by DockerHub or Let's Encrypt. This destroys all Longhorn data. Ensure you have backups.

```sh
task talos:reset
```

---

## Day-2 Operations

### Update Talos Node Configuration

Edit templates under `templates/config/talos/` (patches, talconfig), then:

```sh
task configure
task talos:apply-node IP=? MODE=?
```

### Upgrade Talos or Kubernetes

```sh
task talos:upgrade-node IP=?    # Talos version
task talos:upgrade-k8s          # Kubernetes version
```

### Add a Node

1. Boot the new node into Talos maintenance mode.
2. Get disk and network info:

   ```sh
   talosctl get disks -n <new-node-ip> --insecure
   talosctl get links -n <new-node-ip> --insecure
   ```

3. Add the node to `nodes.sample.yaml` (and `nodes.yaml` if it exists) with hostname, IP, MAC, disk serial, and schematic ID.
4. Re-render and apply:

   ```sh
   task configure
   task talos:apply-node IP=<new-node-ip>
   ```

5. Configure Longhorn on the new node:

   ```sh
   kubectl label node <new-node> node.longhorn.io/create-default-disk=config
   kubectl annotate node <new-node> node.longhorn.io/default-disks-config='[{"path":"/var/lib/longhorn","allowScheduling":true}]'
   ```

### Configure Longhorn Backups

Uncomment the `backupTarget` in `templates/config/kubernetes/apps/longhorn-system/longhorn/app/helmrelease.yaml.j2`:

```yaml
defaultSettings:
  backupTarget: nfs://<nas-ip>:/backup/longhorn
```

Then run `task configure`, commit, and push. Flux will apply the change.

For automated nightly backups, apply a RecurringJob after Longhorn is running:

```yaml
apiVersion: longhorn.io/v1beta2
kind: RecurringJob
metadata:
  name: nightly-backup
  namespace: longhorn-system
spec:
  cron: "0 3 * * *"
  task: backup
  groups:
    - default
  retain: 7
  concurrency: 2
```

---

## Troubleshooting

### Longhorn

**Disk not detected:**
```sh
talosctl -n <node-ip> mounts | grep longhorn
kubectl -n longhorn-system describe nodes.longhorn.io <node-name>
```

**Replicas not rebuilding:**
```sh
kubectl -n longhorn-system get replicas
kubectl -n longhorn-system describe volumes.longhorn.io <volume-name>
```

**Node stuck in "Unschedulable":**
```sh
kubectl -n longhorn-system edit nodes.longhorn.io <node-name>
# Ensure disk path is /var/lib/longhorn and allowScheduling is true
```

### General

```sh
flux get sources git -A
flux get ks -A
flux get hr -A
kubectl -n <namespace> logs <pod-name> -f
kubectl -n <namespace> get events --sort-by='.metadata.creationTimestamp'
```

---

## Reference

### DNS

- **Public:** `envoy-external` gateway + `external-dns` creates Cloudflare records automatically
- **Home:** `envoy-internal` gateway + `k8s-gateway` at `10.50.50.100` resolves `*.prodromou.com` — point your home DNS server at it

### Single-Disk Storage Architecture

Each node's NVMe is partitioned automatically by Talos:

```
┌──────────────────────────────────────────────────────┐
│  NVMe (2TB or 4TB)                                   │
├──────────────────────────────────────────────────────┤
│  Partitions 1-4: Talos System (META, STATE) ~10GB    │
├──────────────────────────────────────────────────────┤
│  Partition 5: EPHEMERAL (~200GB)                     │
│  - Container images, runtime data, logs, etcd        │
├──────────────────────────────────────────────────────┤
│  Partition 6: Longhorn Storage (remaining space)     │
│  - Mounted at /var/mnt/longhorn, XFS                 │
│  - Bind-mounted to /var/lib/longhorn for Longhorn    │
└──────────────────────────────────────────────────────┘
```

With 3x replication, usable capacity equals one node's Longhorn partition (~1.7TB with 2TB drives, ~3.7TB with 4TB drives).

This is configured by two files that are already in the repo:
- `templates/config/talos/patches/global/volumes.yaml.j2` — VolumeConfig (limits EPHEMERAL) + UserVolumeConfig (creates Longhorn partition)
- `templates/config/talos/patches/global/machine-kubelet.yaml.j2` — bind mount from `/var/mnt/longhorn` to `/var/lib/longhorn`

### Optional: Storage Network

If you have a dedicated storage VLAN, you can isolate Longhorn replication traffic with Multus:

1. Deploy Multus as a Flux app.
2. Create a NetworkAttachmentDefinition in `longhorn-system` pointing at your storage interface.
3. Uncomment `storageNetwork: longhorn-system/storage-network` in the Longhorn HelmRelease (`templates/config/kubernetes/apps/longhorn-system/longhorn/app/helmrelease.yaml.j2`).

### Components

| Component | Purpose |
|-----------|---------|
| [Talos Linux](https://github.com/siderolabs/talos) | Immutable OS |
| [Flux](https://github.com/fluxcd/flux2) | GitOps |
| [Cilium](https://github.com/cilium/cilium) | CNI, load balancing, L2 announcements |
| [Longhorn](https://github.com/longhorn/longhorn) | Replicated block storage |
| [Envoy Gateway](https://github.com/envoyproxy/gateway) | Ingress (internal + external) |
| [external-dns](https://github.com/kubernetes-sigs/external-dns) | Cloudflare DNS records |
| [k8s-gateway](https://github.com/ori-edge/k8s_gateway) | Split-horizon home DNS |
| [cert-manager](https://github.com/cert-manager/cert-manager) | Let's Encrypt wildcard certs |
| [cloudflared](https://github.com/cloudflare/cloudflared) | Cloudflare Tunnel |
| [SOPS](https://github.com/getsops/sops) | Secret encryption (Age) |
| [spegel](https://github.com/spegel-org/spegel) | P2P registry mirror |
| [reloader](https://github.com/stakater/Reloader) | Auto-restart on config changes |
| [makejinja](https://github.com/mirkolenz/makejinja) | Template rendering |
| [Renovate](https://www.mend.io/renovate) | Dependency updates |

---

Big shout out to [onedr0p](https://github.com/onedr0p) for the original [cluster-template](https://github.com/onedr0p/cluster-template).
