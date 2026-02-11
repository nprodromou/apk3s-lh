# apk3s-lh

Kubernetes cluster running on 3 Mac Studio nodes with [Talos Linux](https://github.com/siderolabs/talos), [Flux](https://github.com/fluxcd/flux2) for GitOps, and [Longhorn](https://github.com/longhorn/longhorn) for persistent storage on a single-disk architecture.

Based on [onedr0p/cluster-template](https://github.com/onedr0p/cluster-template), modified for single-disk Longhorn storage.

## Components

- **OS:** [Talos Linux](https://github.com/siderolabs/talos)
- **GitOps:** [Flux](https://github.com/fluxcd/flux2)
- **CNI:** [Cilium](https://github.com/cilium/cilium)
- **Storage:** [Longhorn](https://github.com/longhorn/longhorn) (single-disk, 3x replication)
- **Ingress:** [Envoy Gateway](https://github.com/envoyproxy/gateway)
- **DNS:** [external-dns](https://github.com/kubernetes-sigs/external-dns) + [k8s-gateway](https://github.com/ori-edge/k8s_gateway)
- **Certificates:** [cert-manager](https://github.com/cert-manager/cert-manager) (Let's Encrypt)
- **Tunnel:** [cloudflared](https://github.com/cloudflare/cloudflared) (Cloudflare Tunnel)
- **Secrets:** [SOPS](https://github.com/getsops/sops) with Age encryption
- **Extras:** [spegel](https://github.com/spegel-org/spegel) (registry mirror), [reloader](https://github.com/stakater/Reloader)

Configuration is rendered from `cluster.sample.yaml` and `nodes.sample.yaml` via [makejinja](https://github.com/mirkolenz/makejinja). Dependency updates are automated with [Renovate](https://www.mend.io/renovate).

---

## Single-Disk Storage Architecture

This cluster uses a **single-disk architecture** where:

1. **Talos OS** and **Longhorn storage** share the same physical NVMe/SSD
2. **EPHEMERAL partition** is limited to ~200GB for container runtime, images, and etcd
3. **UserVolumeConfig** creates a dedicated partition for Longhorn at `/var/mnt/longhorn`
4. **Longhorn** provides replicated block storage across your 3+ node cluster

### Why Single-Disk?

- **Simpler hardware** - No need for additional USB drives or secondary disks
- **Better reliability** - Internal NVMe is more reliable than USB-attached storage
- **Automatic recovery** - When a node fails and is rebuilt, Longhorn automatically rebuilds replicas from other nodes

### Disk Layout (Per Node)

```
┌─────────────────────────────────────────────────────────────┐
│  NVMe/SSD (e.g., 2TB or 4TB)                                │
├─────────────────────────────────────────────────────────────┤
│  Partitions 1-4: Talos System (META, STATE, etc.) ~10GB     │
├─────────────────────────────────────────────────────────────┤
│  Partition 5: EPHEMERAL (~200GB)                            │
│  - Container images and layers                              │
│  - Container runtime data                                   │
│  - Logs                                                     │
│  - etcd data (control plane nodes)                          │
├─────────────────────────────────────────────────────────────┤
│  Partition 6: Longhorn Storage (remaining space)            │
│  - Mounted at /var/mnt/longhorn                             │
│  - XFS filesystem                                           │
│  - Used by Longhorn for PVC data                            │
└─────────────────────────────────────────────────────────────┘
```

### Storage Capacity Planning

With 3-node replication (Longhorn default), your **usable storage** is approximately:

| Drive Size | EPHEMERAL | Longhorn/Node | Usable (3x replication) |
|------------|-----------|---------------|-------------------------|
| 2TB NVMe   | 200GB     | ~1.7TB        | ~1.7TB total            |
| 4TB NVMe   | 200GB     | ~3.7TB        | ~3.7TB total            |

> **Note:** With replication factor 3, data exists on all nodes. Usable capacity equals single-node Longhorn capacity, not the sum.

---

## Deployment Guide

Follow these 6 stages in order to deploy (or rebuild) the cluster.

### Stage 1: Machine Preparation

#### Current Hardware

This repo is pre-configured for 3 Mac Studio trashcan nodes:

| Hostname | IP | MAC | Disk Serial |
|----------|------|------|-------------|
| `apk3s51` | `10.50.50.51` | `00:3e:e1:ca:65:ac` | `OW23092715AEFDC5F` |
| `apk3s52` | `10.50.50.52` | `00:3e:e1:c4:89:b4` | `OW23091215A17F66C` |
| `apk3s53` | `10.50.50.53` | `00:3e:e1:ca:74:a4` | `OW23092715AEFDC52` |

**Talos Schematic ID:** `613e1592b2da41ae5e265e8789429f22e121aab91cb4deb6bc3c0b6262961245`

These values are pre-filled in `nodes.sample.yaml` and `cluster.sample.yaml`. If your hardware changes, update those files before running `task init`.

#### Talos ISO

1. Download the Talos ISO from the [Talos Linux Image Factory](https://factory.talos.dev) using the schematic ID above, or generate a new one with the system extensions for your hardware.

2. Flash the Talos ISO to a USB drive and boot from it on your nodes (or they may already be booted from a previous install).

3. Verify with `nmap` that your nodes are available on the network:

    ```sh
    nmap -Pn -n -p 50000 10.50.50.0/24 -vv | grep 'Discovered'
    ```

### Stage 2: Local Workstation

1. Clone this repository and `cd` into it:

    ```sh
    git clone https://github.com/nprodromou/apk3s-lh.git && cd apk3s-lh
    ```

2. **Install** the [Mise CLI](https://mise.jdx.dev/getting-started.html#installing-mise-cli) on your workstation.

3. **Activate** Mise in your shell by following the [activation guide](https://mise.jdx.dev/getting-started.html#activate-mise).

4. Use `mise` to install the **required** CLI tools:

    ```sh
    mise trust
    pip install pipx
    mise install
    ```

5. Logout of GitHub Container Registry (GHCR) as this may cause authorization problems when using the public registry:

    ```sh
    docker logout ghcr.io
    helm registry logout ghcr.io
    ```

### Stage 3: Cloudflare Configuration

> [!WARNING]
> If any of the commands fail with `command not found` or `unknown command` it means `mise` is either not installed or configured incorrectly.

1. Create a Cloudflare API token for use with cloudflared and external-dns by reviewing the official [documentation](https://developers.cloudflare.com/fundamentals/api/get-started/create-token/) and following the instructions below.

   - Click the blue `Use template` button for the `Edit zone DNS` template.
   - Name your token `kubernetes`
   - Under `Permissions`, click `+ Add More` and add permissions `Zone - DNS - Edit` and `Account - Cloudflare Tunnel - Read`
   - Limit the permissions to a specific account and/or zone resources and then click `Continue to Summary` and then `Create Token`.
   - **Save this token somewhere safe**, you will need it later on.

2. Create the Cloudflare Tunnel:

    ```sh
    cloudflared tunnel login
    cloudflared tunnel create --credentials-file cloudflare-tunnel.json kubernetes
    ```

### Stage 4: Cluster Configuration

1. Generate the config files from the sample files:

    ```sh
    task init
    ```

2. Review `cluster.yaml` and `nodes.yaml`. These are pre-filled from `cluster.sample.yaml` and `nodes.sample.yaml` with your environment values. The only field you need to fill in is `cloudflare_token`.

3. **Single-Disk Storage is pre-configured.** The following templates are already included in this repo:

    - **`talos/patches/global/volumes.yaml`** — Limits the EPHEMERAL partition to 200GB and creates a Longhorn XFS partition on the remaining disk space, mounted at `/var/mnt/longhorn`.
    - **`talos/patches/global/machine-kubelet.yaml`** — Includes a bind mount from `/var/mnt/longhorn` to `/var/lib/longhorn` so Longhorn can access the storage partition.
    - **`kubernetes/apps/longhorn-system/`** — Complete Flux manifests for Longhorn (HelmRepository, HelmRelease, namespace, Kustomization).

    > **Important:** The VolumeConfig/UserVolumeConfig patches MUST be applied during initial Talos installation. They cannot be applied after the EPHEMERAL partition is created.

    To adjust the EPHEMERAL partition size, edit `templates/config/talos/patches/global/volumes.yaml.j2`.

4. **(Optional) Configure Storage Network** for Longhorn replication. If you have a dedicated storage VLAN/interface, add a Multus NetworkAttachmentDefinition after the cluster is bootstrapped.

5. Template out the kubernetes and talos configuration files:

    ```sh
    task configure
    ```

6. Push your changes to git:

    ```sh
    git add -A
    git commit -m "chore: initial commit :rocket:"
    git push
    ```

### Stage 5: Bootstrap Talos, Kubernetes, and Flux

> [!WARNING]
> It might take a while for the cluster to be setup (10+ minutes is normal). During which time you will see a variety of error messages like: "couldn't get current server API group list," "error: no matching resources found", etc. 'Ready' will remain "False" as no CNI is deployed yet. **This is normal.** If this step gets interrupted, e.g. by pressing <kbd>Ctrl</kbd> + <kbd>C</kbd>, you likely will need to [reset the cluster](#reset) before trying again

1. Install Talos:

    ```sh
    task bootstrap:talos
    ```

2. Push your changes to git:

    ```sh
    git add -A
    git commit -m "chore: add talhelper encrypted secret :lock:"
    git push
    ```

3. Install cilium, coredns, spegel, flux and sync the cluster to the repository state:

    ```sh
    task bootstrap:apps
    ```

4. Watch the rollout of your cluster happen:

    ```sh
    kubectl get pods --all-namespaces --watch
    ```

### Stage 6: Verify Longhorn Storage

Longhorn is automatically deployed by Flux after the cluster is bootstrapped. The manifests are pre-configured in `kubernetes/apps/longhorn-system/`.

1. **Verify the Longhorn partition exists** on each node:

    ```sh
    talosctl -n 10.50.50.51 mounts | grep longhorn
    talosctl -n 10.50.50.52 mounts | grep longhorn
    talosctl -n 10.50.50.53 mounts | grep longhorn
    # Each should show: /var/mnt/longhorn
    ```

2. **Verify Longhorn pods are running** (Flux deploys Longhorn automatically):

    ```sh
    kubectl -n longhorn-system get pods
    kubectl -n longhorn-system get nodes.longhorn.io
    ```

3. **Label nodes for Longhorn disk auto-detection** (optional but recommended):

    ```sh
    kubectl label nodes --all node.longhorn.io/create-default-disk=config
    kubectl annotate nodes --all node.longhorn.io/default-disks-config='[{"path":"/var/lib/longhorn","allowScheduling":true}]'
    ```

4. **(Optional) Configure backups** by editing the Longhorn HelmRelease values in `templates/config/kubernetes/apps/longhorn-system/longhorn/app/helmrelease.yaml.j2` and uncommenting the `backupTarget` setting:

    ```yaml
    defaultSettings:
      backupTarget: nfs://<nas-ip>:/backup/longhorn
    ```

---

## Disaster Recovery & Node Recovery

One of the benefits of this architecture is simplified disaster recovery. When a node fails:

1. **Data is safe** - Replicas exist on the other 2+ nodes
2. **Rebuild is automatic** - After the node rejoins, Longhorn rebuilds replicas

### Recovery Runbook: Node Failure

When a node fails and needs to be rebuilt:

```sh
# 1. Reinstall Talos on the failed node
talosctl apply-config --insecure -n <node-ip> --file clusterconfig/<node>.yaml

# 2. Wait for node to rejoin cluster
kubectl get nodes -w

# 3. Check Longhorn node status
kubectl -n longhorn-system get nodes.longhorn.io

# 4. The disk may show as unavailable - update the Longhorn node config
kubectl -n longhorn-system edit nodes.longhorn.io <node-name>
```

In the editor, ensure the disk configuration is correct:

```yaml
spec:
  disks:
    longhorn-disk:
      path: /var/lib/longhorn
      allowScheduling: true
      storageReserved: 0
```

**Or via Longhorn UI:** Navigate to Node > Edit Node and Disks > Update disk path > Save

```sh
# 5. Verify replica rebuild starts
kubectl -n longhorn-system get replicas -w

# 6. Check volume health
kubectl -n longhorn-system get volumes.longhorn.io
```

### Recovery Runbook: Complete Cluster Rebuild

If you need to rebuild the entire cluster:

1. **Ensure you have backups** of critical PVC data (via Longhorn snapshots to NFS/S3)
2. Follow Stage 1 through Stage 6 to rebuild
3. Restore PVCs from Longhorn backup targets

### Longhorn Backup Configuration (Recommended)

Configure Longhorn to backup to an external NFS or S3 target by uncommenting the `backupTarget` setting in `templates/config/kubernetes/apps/longhorn-system/longhorn/app/helmrelease.yaml.j2`:

```yaml
defaultSettings:
  backupTarget: nfs://<nas-ip>:/backup/longhorn
  # OR for S3:
  # backupTarget: s3://bucket-name@region/
  # backupTargetCredentialSecret: longhorn-backup-secret
```

Then create a `RecurringJob` resource for automated backups (apply this after Longhorn is running):

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

## Post installation

### Verifications

1. Check the status of Cilium:

    ```sh
    cilium status
    ```

2. Check Longhorn status:

    ```sh
    kubectl -n longhorn-system get pods
    kubectl -n longhorn-system get nodes.longhorn.io
    kubectl -n longhorn-system get volumes.longhorn.io
    ```

3. Check the status of Flux and if the Flux resources are up-to-date and in a ready state:

    ```sh
    flux check
    flux get sources git flux-system
    flux get ks -A
    flux get hr -A
    ```

4. Check TCP connectivity to both the internal and external gateways:

    ```sh
    nmap -Pn -n -p 443 10.50.50.150 10.50.50.200 -vv
    ```

5. Check the status of your wildcard `Certificate`:

    ```sh
    kubectl -n network describe certificates
    ```

### Public DNS

> [!TIP]
> Use the `envoy-external` gateway on `HTTPRoutes` to make applications public to the internet.

The `external-dns` application created in the `network` namespace will handle creating public DNS records.

### Home DNS

> [!TIP]
> Use the `envoy-internal` gateway on `HTTPRoutes` to make applications private to your network.

`k8s_gateway` will provide DNS resolution to external Kubernetes resources from any device that uses your home DNS server.

---

## Reset

> [!CAUTION]
> **Resetting** the cluster **multiple times in a short period of time** could lead to being **rate limited by DockerHub or Let's Encrypt**.

There might be a situation where you want to destroy your Kubernetes cluster. The following command will reset your nodes back to maintenance mode.

```sh
task talos:reset
```

> **Warning:** This will destroy all Longhorn data on the nodes. Ensure you have backups if needed.

---

## Talos and Kubernetes Maintenance

### Updating Talos node configuration

Update the source templates under `templates/config/talos/` (e.g., patches, talconfig), then re-render and apply:

```sh
task configure
task talos:apply-node IP=? MODE=?
```

### Updating Talos and Kubernetes versions

```sh
# Upgrade node to a newer Talos version
task talos:upgrade-node IP=?

# Upgrade cluster to a newer Kubernetes version
task talos:upgrade-k8s
```

### Adding a node to your cluster

1. **Prepare the new node**: Boot into Talos maintenance mode.
2. **Get disk and network information**:

   ```sh
   talosctl get disks -n <new-node-ip> --insecure
   talosctl get links -n <new-node-ip> --insecure
   ```

3. **Add the node** to `nodes.sample.yaml` (and `nodes.yaml` if already generated) with hostname, IP, MAC, disk serial, and schematic ID.
4. **Re-render and apply**:

   ```sh
   task configure
   task talos:apply-node IP=<new-node-ip>
   ```

5. **Configure Longhorn disk** on the new node:

   ```sh
   kubectl label node <new-node> node.longhorn.io/create-default-disk=config
   kubectl annotate node <new-node> node.longhorn.io/default-disks-config='[{"path":"/var/lib/longhorn","allowScheduling":true}]'
   ```

---

## Troubleshooting

### Longhorn Issues

**Disk not detected:**
```sh
# Check if partition exists
talosctl -n <node-ip> mounts | grep longhorn

# Check Longhorn node status
kubectl -n longhorn-system describe nodes.longhorn.io <node-name>
```

**Replicas not rebuilding:**
```sh
# Check replica status
kubectl -n longhorn-system get replicas

# Check volume status
kubectl -n longhorn-system describe volumes.longhorn.io <volume-name>
```

**Node stuck in "Unschedulable":**
```sh
# Edit node to fix disk configuration
kubectl -n longhorn-system edit nodes.longhorn.io <node-name>

# Ensure disk path is correct and allowScheduling is true
```

### General Kubernetes Issues

1. Check Flux resources:

    ```sh
    flux get sources git -A
    flux get ks -A
    flux get hr -A
    ```

2. Check pod logs:

    ```sh
    kubectl -n <namespace> logs <pod-name> -f
    ```

3. Check namespace events:

    ```sh
    kubectl -n <namespace> get events --sort-by='.metadata.creationTimestamp'
    ```

---

## Optional: Storage Network Configuration

If you have a dedicated storage VLAN, you can isolate Longhorn replication traffic:

1. **Install Multus** (if not already installed)

2. **Create NetworkAttachmentDefinition**:

    ```yaml
    apiVersion: k8s.cni.cncf.io/v1
    kind: NetworkAttachmentDefinition
    metadata:
      name: storage-network
      namespace: longhorn-system
    spec:
      config: |
        {
          "cniVersion": "0.3.1",
          "type": "macvlan",
          "master": "eth1",
          "mode": "bridge",
          "ipam": {
            "type": "host-local",
            "subnet": "10.x.x.0/24",
            "rangeStart": "10.x.x.100",
            "rangeEnd": "10.x.x.200"
          }
        }
    ```

3. **Update Longhorn settings** by uncommenting the `storageNetwork` line in `templates/config/kubernetes/apps/longhorn-system/longhorn/app/helmrelease.yaml.j2`:

    ```yaml
    defaultSettings:
      storageNetwork: longhorn-system/storage-network
    ```

---

## Related Projects

- [onedr0p/cluster-template](https://github.com/onedr0p/cluster-template) - Original template this is forked from
- [onedr0p/home-ops](https://github.com/onedr0p/home-ops) - onedr0p's personal home operations repository
- [longhorn/longhorn](https://github.com/longhorn/longhorn) - Cloud-Native distributed storage for Kubernetes

---

## Thanks

Big shout out to [onedr0p](https://github.com/onedr0p) for the original cluster-template, and all the contributors who have helped on that project.
