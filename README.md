# Cluster Template (Single-Disk Longhorn Edition)

> **Fork Note:** This is a modified version of [onedr0p/cluster-template](https://github.com/onedr0p/cluster-template) configured for **single-disk architecture with Longhorn storage**. See the [Single-Disk Storage Architecture](#-single-disk-storage-architecture) section for details.

Welcome to my template designed for deploying a single Kubernetes cluster. Whether you're setting up a cluster at home on bare-metal or virtual machines (VMs), this project aims to simplify the process and make Kubernetes more accessible. This template is inspired by my personal [home-ops](https://github.com/onedr0p/home-ops) repository, providing a practical starting point for anyone interested in managing their own Kubernetes environment.

At its core, this project leverages [makejinja](https://github.com/mirkolenz/makejinja), a powerful tool for rendering templates. By reading configuration files—such as [cluster.yaml](./cluster.sample.yaml) and [nodes.yaml](./nodes.sample.yaml)—Makejinja generates the necessary configurations to deploy a Kubernetes cluster with the following features:

- Easy configuration through YAML files.
- Compatibility with home setups, whether on physical hardware or VMs.
- A modular and extensible approach to cluster deployment and management.
- **Single-disk storage** using Longhorn with automatic replica rebuilding.

With this approach, you'll gain a solid foundation to build and manage your Kubernetes cluster efficiently.

## Features

A Kubernetes cluster deployed with [Talos Linux](https://github.com/siderolabs/talos) and an opinionated implementation of [Flux](https://github.com/fluxcd/flux2) using [GitHub](https://github.com/) as the Git provider, [sops](https://github.com/getsops/sops) to manage secrets and [cloudflared](https://github.com/cloudflare/cloudflared) to access applications external to your local network.

- **Required:** Some knowledge of [Containers](https://opencontainers.org/), [YAML](https://noyaml.com/), [Git](https://git-scm.com/), and a **Cloudflare account** with a **domain**.
- **Included components:** [flux](https://github.com/fluxcd/flux2), [cilium](https://github.com/cilium/cilium), [cert-manager](https://github.com/cert-manager/cert-manager), [spegel](https://github.com/spegel-org/spegel), [reloader](https://github.com/stakater/Reloader), [envoy-gateway](https://github.com/envoyproxy/gateway), [external-dns](https://github.com/kubernetes-sigs/external-dns), [cloudflared](https://github.com/cloudflare/cloudflared), and **[longhorn](https://github.com/longhorn/longhorn)**.

**Other features include:**

- Dev env managed w/ [mise](https://mise.jdx.dev/)
- Workflow automation w/ [GitHub Actions](https://github.com/features/actions)
- Dependency automation w/ [Renovate](https://www.mend.io/renovate)
- Flux `HelmRelease` and `Kustomization` diffs w/ [flux-local](https://github.com/allenporter/flux-local)
- **Single-disk storage architecture** with Longhorn for persistent volumes
- **Storage network isolation** via dedicated VLAN (optional)

Does this sound cool to you? If so, continue to read on!

---

## Single-Disk Storage Architecture

This template is configured for a **single-disk architecture** where:

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

## Let's Go!

There are **7 stages** outlined below for completing this project, make sure you follow the stages in order.

### Stage 1: Hardware Configuration

For a **stable** and **high-availability** production Kubernetes cluster, hardware selection is critical.

#### Single-Disk Requirements

| Role           | Cores | Memory | System Disk                    |
|----------------|-------|--------|--------------------------------|
| Control/Worker | 4+    | 16GB+  | 1TB+ NVMe/SSD (single disk)    |

> **Recommended:** Use enterprise or prosumer NVMe drives. Consumer NVMe drives can work but may have issues with power loss protection and write endurance.

#### Why NVMe/SSD Only?

- Longhorn requires low-latency storage for reliable operation
- HDDs are **not recommended** due to latency and IOPS limitations
- USB-attached storage is **not recommended** due to reliability concerns

#### Optional: Storage Network

If you have multiple network interfaces, you can dedicate one for Longhorn replication traffic:

- **Primary interface:** Kubernetes API, pod traffic, external access
- **Storage interface:** Longhorn replica sync (can be a VLAN)

This is configured in Stage 5.

### Stage 2: Machine Preparation

> [!IMPORTANT]
> If you have **3 or more nodes** it is recommended to make 3 of them controller nodes for a highly available control plane. This project configures **all nodes** to be able to run workloads. **Worker nodes** are therefore **optional**.

1. Head over to the [Talos Linux Image Factory](https://factory.talos.dev) and follow the instructions. Be sure to only choose the **bare-minimum system extensions** as some might require additional configuration and prevent Talos from booting without it. Depending on your CPU start with the Intel/AMD system extensions (`i915`, `intel-ucode` & `mei` **or** `amdgpu` & `amd-ucode`), you can always add system extensions after Talos is installed and working.

2. This will eventually lead you to download a Talos Linux ISO (or for SBCs a RAW) image. Make sure to note the **schematic ID** you will need this later on.

3. Flash the Talos ISO or RAW image to a USB drive and boot from it on your nodes.

4. Verify with `nmap` that your nodes are available on the network. (Replace `192.168.1.0/24` with the network your nodes are on.)

    ```sh
    nmap -Pn -n -p 50000 192.168.1.0/24 -vv | grep 'Discovered'
    ```

### Stage 3: Local Workstation

> [!TIP]
> It is recommended to set the visibility of your repository to `Public` so you can easily request help if you get stuck.

1. Create a new repository by clicking the green `Use this template` button at the top of this page, then clone the new repo you just created and `cd` into it. Alternatively you can us the [GitHub CLI](https://cli.github.com/) ...

    ```sh
    export REPONAME="home-ops"
    gh repo create $REPONAME --template onedr0p/cluster-template --disable-wiki --public --clone && cd $REPONAME
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

### Stage 4: Cloudflare configuration

> [!WARNING]
> If any of the commands fail with `command not found` or `unknown command` it means `mise` is either not install or configured incorrectly.

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

### Stage 5: Cluster configuration

1. Generate the config files from the sample files:

    ```sh
    task init
    ```

2. Fill out `cluster.yaml` and `nodes.yaml` configuration files using the comments in those file as a guide.

3. **Configure Single-Disk Storage** by adding the Talos volume patches. Create or update `talos/patches/global/volumes.yaml`:

    ```yaml
    # Limit EPHEMERAL partition to leave room for Longhorn
    apiVersion: v1alpha1
    kind: VolumeConfig
    name: EPHEMERAL
    provisioning:
      maxSize: 200GiB
      grow: false
    ---
    # Create Longhorn storage partition from remaining space
    apiVersion: v1alpha1
    kind: UserVolumeConfig
    name: longhorn
    provisioning:
      diskSelector:
        match: system_disk
      minSize: 500GiB
      grow: true
    mount:
      mountpoint: /var/mnt/longhorn
    filesystem:
      type: xfs
    ```

    > **Important:** This configuration MUST be applied during initial Talos installation. It cannot be applied after the EPHEMERAL partition is created.

4. **Add kubelet extra mounts** for Longhorn. In your `talconfig.yaml` or appropriate patch:

    ```yaml
    machine:
      kubelet:
        extraMounts:
          - destination: /var/lib/longhorn
            type: bind
            source: /var/mnt/longhorn
            options:
              - bind
              - rshared
              - rw
    ```

5. **(Optional) Configure Storage Network** for Longhorn replication. If you have a dedicated storage VLAN/interface, add a Multus NetworkAttachmentDefinition after the cluster is bootstrapped.

6. Template out the kubernetes and talos configuration files:

    ```sh
    task configure
    ```

7. Push your changes to git:

    ```sh
    git add -A
    git commit -m "chore: initial commit :rocket:"
    git push
    ```

### Stage 6: Bootstrap Talos, Kubernetes, and Flux

> [!WARNING]
> It might take a while for the cluster to be setup (10+ minutes is normal). During which time you will see a variety of error messages like: "couldn't get current server API group list," "error: no matching resources found", etc. 'Ready' will remain "False" as no CNI is deployed yet. **This is normal.** If this step gets interrupted, e.g. by pressing <kbd>Ctrl</kbd> + <kbd>C</kbd>, you likely will need to [reset the cluster](#-reset) before trying again

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

### Stage 7: Configure Longhorn Storage

After the cluster is bootstrapped, configure Longhorn:

1. **Verify the Longhorn partition exists** on each node:

    ```sh
    # Check from your workstation
    talosctl -n <node-ip> mounts | grep longhorn
    # Should show: /var/mnt/longhorn
    ```

2. **Install Longhorn** via Flux. Create `kubernetes/apps/longhorn-system/longhorn/`:

    ```yaml
    # kubernetes/apps/longhorn-system/longhorn/ks.yaml
    apiVersion: kustomize.toolkit.fluxcd.io/v1
    kind: Kustomization
    metadata:
      name: longhorn
      namespace: flux-system
    spec:
      interval: 30m
      path: ./kubernetes/apps/longhorn-system/longhorn/app
      prune: true
      sourceRef:
        kind: GitRepository
        name: flux-system
      wait: true
    ```

    ```yaml
    # kubernetes/apps/longhorn-system/longhorn/app/helmrelease.yaml
    apiVersion: helm.toolkit.fluxcd.io/v2
    kind: HelmRelease
    metadata:
      name: longhorn
      namespace: longhorn-system
    spec:
      interval: 30m
      chart:
        spec:
          chart: longhorn
          version: 1.7.2
          sourceRef:
            kind: HelmRepository
            name: longhorn
            namespace: flux-system
      values:
        defaultSettings:
          defaultDataPath: /var/lib/longhorn
          defaultDataLocality: best-effort
          replicaAutoBalance: best-effort
          # Storage network (optional - uncomment if using dedicated storage VLAN)
          # storageNetwork: longhorn-system/storage-network
        persistence:
          defaultClassReplicaCount: 3
        ingress:
          enabled: false
    ```

3. **Label nodes for Longhorn disk auto-detection** (optional but recommended):

    ```sh
    kubectl label nodes --all node.longhorn.io/create-default-disk=config
    kubectl annotate nodes --all node.longhorn.io/default-disks-config='[{"path":"/var/lib/longhorn","allowScheduling":true}]'
    ```

4. **Verify Longhorn is running**:

    ```sh
    kubectl -n longhorn-system get pods
    kubectl -n longhorn-system get nodes.longhorn.io
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
2. Follow Stage 2 through Stage 7 to rebuild
3. Restore PVCs from Longhorn backup targets

### Longhorn Backup Configuration (Recommended)

Configure Longhorn to backup to an external NFS or S3 target:

```yaml
# In your Longhorn HelmRelease values
defaultSettings:
  backupTarget: nfs://<nas-ip>:/backup/longhorn
  # OR for S3:
  # backupTarget: s3://bucket-name@region/
  # backupTargetCredentialSecret: longhorn-backup-secret
```

Create recurring backup jobs:

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
    nmap -Pn -n -p 443 ${cluster_gateway_addr} ${cloudflare_gateway_addr} -vv
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

> [!TIP]
> Ensure you have updated `talconfig.yaml` and any patches with your updated configuration.

```sh
# (Re)generate the Talos config
task talos:generate-config
# Apply the config to the node
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

1. **Prepare the new node**: Boot into maintenance mode
2. **Get disk information**:

   ```sh
   talosctl get disks -n <ip> --insecure
   talosctl get links -n <ip> --insecure
   ```

3. **Update configuration**: Extend `talconfig.yaml` with new node info
4. **Apply configuration**:

   ```sh
   task talos:generate-config
   task talos:apply-node IP=?
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

3. **Update Longhorn settings**:

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
