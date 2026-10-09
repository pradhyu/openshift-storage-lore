# OpenShift Storage & Persistent Volumes: Architecture & Operations Guide

A comprehensive architectural and operations guide covering PersistentVolumeClaim (PVC) file inspection, static NFS provisioning, enterprise multi-protocol NAS, Ceph RBD block storage, CephFS distributed filesystems, and cloud-native database storage.

---

## Table of Contents

* [1. Viewing PVC Files Without a Dedicated Mount / Pod](#1-viewing-pvc-files-without-a-dedicated-mount--pod)
  * [The Short Answer](#the-short-answer)
  * [Practical Solutions & Workarounds](#practical-solutions--workarounds)
    * [Option 1: Existing Running Pod (`oc exec` / `oc rsync`)](#option-1-an-existing-pod-is-already-running-no-new-pod-needed)
    * [Option 2: Self-Cleaning Ephemeral Debug Pod (`oc run --rm`)](#option-2-no-pod-running--ephemeral-one-liner-self-cleaning)
    * [Option 3: Underlying Storage Provider Direct Access](#option-3-access-via-underlying-storage-provider-outside-openshift)
    * [Option 4: Node-Level Inspection via `oc debug node`](#option-4-node-level-inspection-via-oc-debug-node-cluster-admin)
  * [Inspection Comparison Summary](#comparison-summary)
* [2. Setting Up NFS Storage: Step-by-Step (PV & PVC)](#setting-up-nfs-storage-step-by-step-pv--pvc)
* [3. Quick Reference Notes: Cloud & Object Storage Options](#quick-reference-notes-cloud--object-storage-options)
* [4. Enterprise & OpenShift-Native Implementations (Deep Dive)](#deep-dive-enterprise--openshift-native-implementations-options-3-to-5)
  * [Deep Dive 1: OpenShift Data Foundation (ODF / CephFS)](#deep-dive-1-openshift-data-foundation-odf--cephfs)
  * [Deep Dive 2: Enterprise Multi-Protocol NAS CSI (NetApp Trident)](#deep-dive-2-enterprise-multi-protocol-nas-csi-netapp-trident--dell-powerscale)
  * [Deep Dive 3: Universal In-Cluster SFTP Gateway & Sidecar Patterns](#deep-dive-3-universal-in-cluster-sftp-gateway--sidecar-patterns)
* [5. Architecture Selection Decision Matrix](#architecture-selection-decision-matrix)
* [6. Ceph RBD (`ocs-storagecluster-ceph-rbd`) vs. NAS (NFS / CephFS)](#deep-dive-ceph-rbd-ocs-storagecluster-ceph-rbd-vs-nas-nfs--cephfs)
  * [What Do "Ceph" and "RBD" Stand For? (Origins & Acronyms)](#what-do-ceph-and-rbd-actually-stand-for-origins--acronyms)
  * [Core Architectural Difference: Block vs. File](#core-architectural-difference-block-vs-file)
  * [Why Ceph RBD is Better than NAS (7 Technical Advantages)](#why-ceph-rbd-is-better-than-nas-7-technical-advantages)
  * [Workload Decision Matrix: When to Choose RBD vs. NAS](#workload-decision-matrix-when-to-choose-rbd-vs-nas)
* [7. Solving the RWO Constraint: Multi-Pod Read/Write (RWX) Needs](#solving-the-rwo-constraint-how-to-handle-multi-pod-readwrite-rwx-needs)
* [8. How Databases Actually Use RWO Storage in OpenShift (StatefulSets & Operators)](#how-databases-actually-use-rwo-storage-in-openshift-statefulsets--operators)
  * [The Architecture: Share Nothing (Replication via Network, Not Disk)](#the-architecture-share-nothing-replication-via-network-not-disk)
  * [How Client Traffic is Routed: Primary vs. Replicas Services](#how-client-traffic-is-routed-primary-vs-replicas-services)
  * [How Kubernetes Automates This: StatefulSet with volumeClaimTemplates](#how-kubernetes-automates-this-statefulset-with-volumeclaimtemplates)
* [9. Clearing Up the Myth: Does RWO Mean "One Writer, Multiple Readers"?](#clearing-up-the-myth-does-rwo-mean-one-writer-multiple-readers)
* [10. Ceph RBD Architecture: Local Node vs. Remote Storage](#ceph-rbd-architecture-are-files-written-on-the-local-node-or-on-remote-servers)
* [11. Why Ceph RBD is Faster Than NFS Even Though Both Are Remote](#why-ceph-rbd-is-faster-than-nfs-even-though-both-are-remote)
  * [Protocol Chattiness: 1 Block RPC vs. 6+ File RPCs](#1-protocol-chattiness-1-block-rpc-vs-6-file-rpcs)
  * [Striping Across Many Servers (CRUSH) vs. Single-Server Bottleneck](#2-striping-across-many-servers-crush-vs-single-server-bottleneck)
  * [No Filesystem Metadata Overhead on the Storage Cluster](#3-no-filesystem-metadata-overhead-on-the-storage-cluster)
  * [Safe Local Linux Page Caching](#4-safe-local-linux-page-caching)
  * [Multi-Queue Block I/O (`blk-mq`) and Deep Concurrency](#5-multi-queue-block-io-blk-mq-and-deep-concurrency)
  * [Performance Comparison: Ceph RBD vs. Traditional NFS](#performance-comparison-ceph-rbd-vs-traditional-nfs)
* [12. CephFS vs. NFS: Are They the Same? What is the Difference?](#cephfs-vs-nfs-are-they-the-same-what-is-the-difference)
  * [Core Architectural Comparison](#core-architectural-comparison)
  * [6 Key Differences Between CephFS and NFS](#6-key-differences-between-cephfs-and-nfs)
  * [Comparison Matrix: CephFS vs. Traditional NFS](#comparison-matrix-cephfs-vs-traditional-nfs)
* [13. Allowing Users to Download Files from a CephFS PVC](#13-allowing-users-to-download-files-from-a-cephfs-pvc)
  * [The Superpower of CephFS: Concurrent Multi-Pod Access](#the-superpower-of-cephfs-concurrent-multi-pod-access)
  * [Method 1: Web Browser Download Portal (FileBrowser)](#method-1-web-browser-download-portal-via-filebrowser-recommended-for-end-users)
  * [Method 2: High-Speed HTTP Directory Index (Nginx)](#method-2-high-speed-http-directory-index-via-nginx-fastest-for-direct-file-links)
  * [Method 3: In-Cluster SFTP Gateway (WinSCP / FileZilla)](#method-3-in-cluster-sftp-gateway-for-winscp-filezilla-and-batch-jobs)
    * [Connecting WinSCP via Ingress (WebDAV Protocol)](#what-if-you-must-use-openshift-ingress--port-443-with-winscp-the-webdav-solution)
  * [Method 4: Developer & Admin CLI Downloads (oc rsync / oc cp)](#method-4-developer--admin-cli-downloads-no-new-pods-needed)
  * [Security Best Practices for CephFS File Downloads](#security-best-practices-for-cephfs-file-downloads)
  * [Decision Matrix: Which Download Method Should You Use?](#decision-matrix-which-download-method-should-you-use)
* [14. Modern Object Storage Gateway: Using RustFS with OpenShift PVCs](#14-modern-object-storage-gateway-using-rustfs-with-openshift-pvcs)
  * [Why RustFS for This Use Case?](#why-rustfs-for-this-use-case)
  * [Is RustFS a StorageClass or an Application?](#is-rustfs-a-storageclass-or-an-application-how-do-you-actually-use-it)
  * [Is This Available Out-of-the-Box in OpenShift?](#is-this-available-out-of-the-box-in-openshift-native-odf-vs-third-party)
  * [RustFS Architecture on OpenShift Storage](#rustfs-architecture-on-openshift-storage)
  * [Mode 1: Standalone RustFS Gateway on an Existing PVC](#mode-1-standalone-rustfs-gateway-on-an-existing-pvc)
  * [How to Access & Download Files via RustFS](#how-to-access--download-files-via-rustfs)
    * [1. WinSCP via Native Amazon S3 Protocol](#1-winscp-via-native-amazon-s3-protocol-100-via-ingress-port-443)
    * [2. Web Browser Downloads via RustFS Console](#2-web-browser-downloads-via-rustfs-console)
    * [3. Programmatic Pre-Signed URLs](#3-programmatic-pre-signed-urls-time-limited-direct-links)
    * [4. Command Line & Automation (aws-cli / rclone)](#4-command-line--automation-aws-cli--rclone)
  * [Mode 2: Distributed High-Performance RustFS Cluster](#mode-2-distributed-high-performance-rustfs-cluster-statefulset--ceph-rbd)
  * [Architectural Comparison: Traditional Protocols vs. RustFS S3](#architectural-comparison-traditional-protocols-vs-rustfs-s3)

---

## 1. Viewing PVC Files Without a Dedicated Mount / Pod

A common question in OpenShift/Kubernetes administration: **Can you view files in a PersistentVolumeClaim (PVC) without a volume mount and without creating a new pod?**

---

## The Short Answer

**No native `oc view-pvc` command exists** in Kubernetes/OpenShift to browse an unattached PVC directly from the command line.

### Why?

Kubernetes treats storage abstractly:

* A **PVC** is a control-plane API pointer to a **PV** (PersistentVolume).
* The actual filesystem is managed by a CSI (Container Storage Interface) driver or storage appliance (Ceph/ODF, NFS, AWS EBS, NetApp, etc.).
* The underlying storage volume is only partitioned, formatted, and mounted onto an OpenShift node when a **Pod** scheduled on that node declares a volume mount for it.

---

## Practical Solutions & Workarounds

Depending on whether a workload is currently running and what permissions you have, choose the best approach below:

---

### Option 1: An Existing Pod is Already Running (No New Pod Needed)

If an application pod (Deployment, StatefulSet, etc.) is already running and has the PVC mounted, **you do not need a new pod**. Use `oc exec`, `oc rsync`, or `oc cp`.

```bash
# 1. List files in the mounted directory
oc exec <pod-name> -c <container-name> -- ls -la /path/to/mount

# 2. View a specific file's content
oc exec <pod-name> -c <container-name> -- cat /path/to/mount/filename.txt

# 3. Search for files
oc exec <pod-name> -c <container-name> -- find /path/to/mount -name "*.log"

# 4. Copy/Sync files to your local workstation
oc rsync <pod-name>:/path/to/mount/ ./local-folder/

# Or using oc cp:
oc cp <pod-name>:/path/to/mount/filename.txt ./filename.txt
```

> [!TIP]
> **OpenShift Web Console:** Navigate to **Workloads → Pods → [Select Pod] → Terminal tab**, then run `cd /path/to/mount && ls -la`.

---

### Option 2: No Pod Running — Ephemeral One-Liner (Self-Cleaning)

If no pod is running and you do not want to write YAML manifests or manage a permanent pod, use a **self-destroying ephemeral debug pod**:

```bash
# Ephemeral pod that automatically deletes itself upon exit (--rm)
oc run pvc-inspector --rm -it \
  --image=registry.access.redhat.com/ubi9/ubi-minimal \
  --restart=Never \
  --overrides='{
    "spec": {
      "volumes": [
        {
          "name": "pvc-storage",
          "persistentVolumeClaim": {
            "claimName": "<your-pvc-name>"
          }
        }
      ],
      "containers": [
        {
          "name": "inspector",
          "image": "registry.access.redhat.com/ubi9/ubi-minimal",
          "command": ["sh"],
          "stdin": true,
          "tty": true,
          "volumeMounts": [
            {
              "name": "pvc-storage",
              "mountPath": "/mnt/pvc"
            }
          ]
        }
      ]
    }
  }'
```

Inside the interactive shell:

```sh
cd /mnt/pvc
ls -la
exit  # Pod is immediately destroyed
```

---

### Option 3: Access Via Underlying Storage Provider (Outside OpenShift)

If you have infrastructure/storage-level access, you can bypass OpenShift and inspect the storage directly:

| Storage Backend | How to Access Without a Pod |
| :--- | :--- |
| **NFS** | Mount export directly on external host: `sudo mount -t nfs <nfs-server>:<export-path> /mnt` |
| **OpenShift Data Foundation (ODF / CephFS / Rook-Ceph)** | Use the **Ceph Toolbox** pod or Rook-Ceph dashboard to inspect CephFS subvolumes and pools. |
| **Cloud Block Storage (AWS EBS / Azure Disk / GCP PD)** | Take a snapshot of the volume in the cloud console, attach it to a temporary cloud VM / EC2 instance, and mount it read-only. |
| **Enterprise SAN / NAS (NetApp, Pure, Dell)** | Use the storage appliance's web GUI or snapshot manager to browse volume files. |

---

### Option 4: Node-Level Inspection via `oc debug node` (Cluster Admin)

If a volume is already attached to a cluster node by the CSI plugin:

```bash
# 1. Identify which node the PV is attached to
oc get pv <pv-name> -o jsonpath='{.spec.claimRef.name}'

# 2. Start a debug session on that node
oc debug node/<node-name>

# 3. Switch to the host filesystem
chroot /host

# 4. Locate the volume mount path in kubelet
find /var/lib/kubelet/pods/ -name "<your-volume-name>"
# Or check CSI mount points:
ls -la /var/lib/kubelet/plugins/kubernetes.io/csi/
```

---

## Comparison Summary

| Method | Requires New Pod? | Requires Volume Mount? | Best Used When |
| :--- | :---: | :---: | :--- |
| **`oc exec` / `oc rsync`** | ❌ No | ✅ (Uses existing mount) | Application pod is already running |
| **`oc run --rm` (Ephemeral)** | ✅ (Temporary, self-deleting) | ✅ Yes | Quick manual inspection without saving YAMLs |
| **Direct Storage Access** | ❌ No | ❌ (No K8s mount) | NFS or storage-level admin access available |
| **`oc debug node`** | ❌ (Node shell) | ❌ (Reads host path) | Cluster admin troubleshooting CSI mounts |

---

## Setting Up NFS Storage: Step-by-Step (PV & PVC)

When setting up static NFS storage in OpenShift to share files with applications and SFTP:

### Step 1: Export Directory on the NFS Server

On your NFS host, prepare the directory and update `/etc/exports`:

```bash
# Create shared directory
sudo mkdir -p /exports/openshift-data
sudo chmod 777 /exports/openshift-data

# Export in /etc/exports
echo "/exports/openshift-data *(rw,sync,no_root_squash,no_subtree_check)" | sudo tee -a /etc/exports

# Refresh export table
sudo exportfs -rav
```

### Step 2: Create the PersistentVolume (PV)

The PV is a **cluster-scoped** resource that points directly to your NFS server IP and export path.

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-shared-pv
spec:
  capacity:
    storage: 50Gi
  accessModes:
    - ReadWriteMany   # RWX: Multiple pods/nodes can read and write simultaneously
  persistentVolumeReclaimPolicy: Retain  # Retain prevents data loss when PVC is deleted
  nfs:
    server: 192.168.1.100       # NFS / SFTP server IP or hostname
    path: /exports/openshift-data
```

Apply to OpenShift:

```bash
oc apply -f nfs-pv.yaml
```

### Step 3: Create the PersistentVolumeClaim (PVC)

The PVC is a **namespace-scoped** request in your application project that binds to the PV.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nfs-shared-pvc
  namespace: my-project
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 50Gi
  volumeName: nfs-shared-pv     # Explicitly binds to the specific PV created above
```

Apply and verify:

```bash
oc apply -f nfs-pvc.yaml -n my-project

# Check status (should be "Bound")
oc get pvc nfs-shared-pvc -n my-project
```

### Step 4: Mount into Workload & Access via SFTP

Mount the PVC in your Deployment or Pod:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: my-project
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: app
          image: registry.access.redhat.com/ubi9/ubi
          command: ["sh", "-c", "tail -f /dev/null"]
          volumeMounts:
            - name: nfs-vol
              mountPath: /data
      volumes:
        - name: nfs-vol
          persistentVolumeClaim:
            claimName: nfs-shared-pvc
```

* **Inside OpenShift**: The app reads and writes to `/data`.
* **Outside OpenShift**: Connect via SFTP (`sftp user@192.168.1.100`) to `/exports/openshift-data`. Both view and modify the exact same live files.

---

## Quick Reference Notes: Cloud & Object Storage Options

Brief reference for managed cloud services and object storage alternatives:

* **AWS EFS (`efs-sc`) + AWS Transfer Family**: EFS provides dynamic `ReadWriteMany` volumes in AWS. AWS Transfer Family attaches directly to the EFS file system to offer managed SFTP endpoints without running any SFTP pods inside OpenShift.
* **Azure Files (`azurefile-csi`) with Native SFTP**: Azure Storage Accounts support built-in SFTP toggled from the Azure portal. OpenShift dynamically mounts the files via `azurefile-csi`, while external clients connect directly to Azure's SFTP endpoint.
* **Object Storage (`ObjectBucketClaim` / S3) + SFTPGo**: Uses S3 API for cloud-native apps and deploys SFTPGo (or an S3 SFTP gateway) to translate SFTP file transfers into S3 bucket objects.

---

## Deep Dive: Enterprise & OpenShift-Native Implementations (Options 3 to 5)

---

### Deep Dive 1: OpenShift Data Foundation (ODF / CephFS)

**OpenShift Data Foundation (ODF)**, powered by Ceph, is Red Hat's native software-defined storage solution for OpenShift. It provides two main CSI drivers:

* **Ceph RBD (`ocs-storagecluster-ceph-rbd`)**: Block storage (`ReadWriteOnce`).
* **CephFS (`ocs-storagecluster-cephfs`)**: Shared distributed POSIX filesystem (`ReadWriteMany`).

For shared multi-client and SFTP workflows, **CephFS** is the engine of choice.

#### CephFS Shared Storage Architecture

```text
               +------------------------------------------------+
               |        Ceph Storage Cluster (ODF OSDs)         |
               +------------------------------------------------+
                                       │
                         CephFS Filesystem Metadata & Pools
                                       │
                  ┌────────────────────┴───────────────────┐
                  ▼                                        ▼
   ┌───────────────────────────────┐        ┌───────────────────────────────┐
   │    Application Pod (App)      │        │      SFTP Gateway Pod         │
   │  Mount: /var/data             │        │  Mount: /home/sftp/data       │
   │  PVC: cephfs-shared-pvc (RWX) │        │  PVC: cephfs-shared-pvc (RWX) │
   └───────────────────────────────┘        └───────────────────────────────┘
                  │                                        ▲
           Internal Cluster                                │ External Port 22
                  ▼                                        │ (LoadBalancer/NodePort)
             Application                              External Users
```

#### Step 1: Request CephFS Storage via PVC

Because ODF includes dynamic provisioning, you do not need to create manual PVs:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: cephfs-shared-pvc
  namespace: data-exchange
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 200Gi
  storageClassName: ocs-storagecluster-cephfs
```

#### Step 2: Configure OpenShift Security Context Constraints (SCC)

SFTP containers typically require a fixed user or root privileges to bind to port 22 and manage file ownership. In OpenShift, containers run under arbitrary UIDs by default (`restricted-v2` SCC).

Create a dedicated ServiceAccount and grant it `anyuid` or run with an unprivileged port (e.g., port 2222):

```bash
# Create service account
oc create serviceaccount sftp-sa -n data-exchange

# Grant anyuid SCC (requires cluster-admin)
oc adm policy add-scc-to-user anyuid -z sftp-sa -n data-exchange
```

#### Step 3: Deploy Production-Ready CephFS SFTP Gateway

This manifest mounts persistent host keys (to prevent SSH host key warning dialogs on pod restarts) and mounts SSH public keys from a Secret:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: sftp-ssh-keys
  namespace: data-exchange
type: Opaque
stringData:
  # Add user public key: <username>:<password_or_empty>:<UID>:<GID>:<upload_dir>
  users.conf: "partner_user::1001:1001:incoming"
  id_rsa.pub: "ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... partner@example.com"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cephfs-sftp-gateway
  namespace: data-exchange
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: cephfs-sftp
  template:
    metadata:
      labels:
        app: cephfs-sftp
    spec:
      serviceAccountName: sftp-sa
      containers:
        - name: sftp
          image: atmoz/sftp:alpine
          args: ["partner_user::1001:1001:incoming"]
          ports:
            - name: sftp
              containerPort: 22
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi
          volumeMounts:
            - name: cephfs-data
              mountPath: /home/partner_user/incoming
            - name: ssh-keys
              mountPath: /home/partner_user/.ssh/keys/id_rsa.pub
              subPath: id_rsa.pub
              readOnly: true
      volumes:
        - name: cephfs-data
          persistentVolumeClaim:
            claimName: cephfs-shared-pvc
        - name: ssh-keys
          secret:
            secretName: sftp-ssh-keys
```

#### Step 4: Exposing SFTP Outside the Cluster

> [!WARNING]
> **OpenShift Routes do NOT work for SFTP.**
> Standard OpenShift Routes operate at Layer 7 (HTTP, HTTPS with SNI, TLS Re-encrypt). They cannot route raw TCP SSH/SFTP (Port 22) connections.

Use one of the following two methods:

##### Option A: Service Type `LoadBalancer` (Cloud or MetalLB on Bare Metal)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sftp-external
  namespace: data-exchange
spec:
  type: LoadBalancer
  ports:
    - name: sftp
      port: 22
      targetPort: 22
  selector:
    app: cephfs-sftp
```

##### Option B: Service Type `NodePort` (On-Premises without LoadBalancer)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sftp-nodeport
  namespace: data-exchange
spec:
  type: NodePort
  ports:
    - name: sftp
      port: 22
      targetPort: 22
      nodePort: 30022
  selector:
    app: cephfs-sftp
```

Clients connect to any OpenShift worker node IP on port `30022`:

```bash
sftp -P 30022 partner_user@<worker-node-ip>
```

---

### Deep Dive 2: Enterprise Multi-Protocol NAS CSI (NetApp Trident / Dell PowerScale)

In enterprise data centers with existing SAN/NAS hardware (such as **NetApp ONTAP**, **Dell PowerScale / Isilon**, or **Pure Storage FlashBlade**), the storage appliance can serve the **same volume across multiple protocols natively**.

This is called **Multi-Protocol NAS**:

* OpenShift accesses the volume using **NFS** through the CSI driver.
* External users access the volume using **native SFTP or SMB** directly on the storage appliance.
* **No SFTP pods run in OpenShift**, reducing CPU/memory overhead and network hops.

#### Enterprise Multi-Protocol Architecture

```text
              +───────────────────────────────────────────────────+
              │        Enterprise Storage Appliance (NetApp ONTAP) │
              │            SVM Volume: /vol_partner_exchange      │
              +───────────────────────────────────────────────────+
                          /                                   \
       NFS Mount (Trident CSI)                         Native SSH/SFTP
                        /                                       \
                       v                                         v
       +──────────────────────────────+           +──────────────────────────────+
       |   OpenShift Cluster          |           |   External Enterprise Client |
       |   App Pod mounts PVC (RWX)   |           |   WinSCP, FileZilla, Scripts |
       +──────────────────────────────+           +──────────────────────────────+
```

#### Step 1: StorageClass Definition (NetApp Trident Example)

The CSI driver dynamically creates an ONTAP FlexVol with multi-protocol support:

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: trident-ontap-nas
provisioner: csi.trident.netapp.io
parameters:
  backendType: ontap-nas
  media: ssd
  provisioningType: thin
  exportPolicy: openshift-cluster-nodes
  securityStyle: unix
allowVolumeExpansion: true
reclaimPolicy: Retain
```

#### Step 2: Create Multi-Protocol PVC

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: enterprise-shared-pvc
  namespace: finance-app
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 500Gi
  storageClassName: trident-ontap-nas
```

#### Step 3: Appliance-Side SFTP Access Configuration

On the NetApp ONTAP SVM (Storage Virtual Machine):

##### A. Enable SSH/SFTP Service

```bash
security login create -vserver svm_finance -user-or-group-name sftp_user -application ssh -authentication-method publickey
```

##### B. Associate User Public Key

```bash
security login publickey create -vserver svm_finance -username sftp_user -index 1 -publickey "ssh-ed25519 AAAAC3..."
```

##### C. Map Home Directory to Volume Export

Set `sftp_user` home directory to the exact junction path where the PVC was provisioned (`/vol_partner_exchange`).

#### Advantages & Tradeoffs of Enterprise NAS Multi-Protocol

| Benefit | Tradeoff |
| :--- | :--- |
| **Zero Cluster Overhead**: Storage array handles all SSH encryption and bandwidth. | Requires high-end enterprise SAN/NAS hardware (NetApp, Dell, Pure). |
| **Enterprise Auditing**: Centralized logging, snapshot scheduling, and anti-ransomware. | Storage admins must manage user credentials on the storage array. |
| **High Throughput**: 25/100 GbE SAN connections bypass OpenShift SDN overlay. | StorageClass configuration requires Trident/CSI operator installation. |

---

### Deep Dive 3: Universal In-Cluster SFTP Gateway & Sidecar Patterns

This solution works on **ANY OpenShift cluster**, regardless of whether the storage backend is AWS EBS, Google Persistent Disk, VMware vSphere VMDK, Ceph RBD, or Local Storage.

#### Pattern A: Standalone Gateway for `ReadWriteMany` (RWX)

Used when the underlying StorageClass supports `ReadWriteMany` (e.g., CephFS, NFS, Azure Files, EFS).

* The application and the SFTP server run in **separate Deployments**.
* They can scale independently, be restarted independently, and have different resource limits.
* Both reference the same `claimName` in their Pod specs.

```yaml
# Pod 1: Application (e.g., Worker processing files)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: data-processor
spec:
  template:
    spec:
      containers:
        - name: processor
          image: my-company/processor:v1
          volumeMounts:
            - name: shared-storage
              mountPath: /data/input
      volumes:
        - name: shared-storage
          persistentVolumeClaim:
            claimName: shared-data-pvc
---
# Pod 2: Dedicated SFTP Server
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sftp-service
spec:
  template:
    spec:
      containers:
        - name: sftp
          image: atmoz/sftp:alpine
          args: ["transfer_user:SecretPassword123:::input"]
          ports:
            - containerPort: 22
          volumeMounts:
            - name: shared-storage
              mountPath: /home/transfer_user/input
      volumes:
        - name: shared-storage
          persistentVolumeClaim:
            claimName: shared-data-pvc
```

#### Pattern B: Co-Located Sidecar for `ReadWriteOnce` (RWO) Block Storage

##### The Problem with RWO Block Storage

Standard cloud block storage (AWS `gp3-csi`, Azure `managed-csi`, VMware `thin`, Ceph RBD) only supports `ReadWriteOnce`. If you try to mount the same PVC into two separate pods, the CSI attacher fails with:

```text
Multi-Attach error for volume "pvc-xxx" Volume is already used by pod(s)...
```

##### The Sidecar Solution

Because a Pod can run multiple containers that share the exact same volume on the same node, you run the **SFTP server as a sidecar container in the same Pod**:

```text
+───────────────────────────────────────────────────────────────+
|                       Application Pod                         |
|                                                               |
|  ┌─────────────────────────┐     ┌─────────────────────────┐  |
|  │   Main App Container    │     │   SFTP Sidecar (sshd)   │  |
|  │   Processes batch data  │     │   Listens on port 2222  │  |
|  └────────────┬────────────┘     └────────────┬────────────┘  |
|               │                               │               |
|               └───────────────┬───────────────┘               |
|                               ▼                               |
|              Shared Volume Mount: /workspace/data             |
|                  (RWO Block Storage PVC)                      |
+───────────────────────────────────────────────────────────────+
```

##### Full Production RWO Sidecar Manifest

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: batch-app-with-sftp
  namespace: processing-jobs
spec:
  replicas: 1   # Must be 1 for RWO block storage
  selector:
    matchLabels:
      app: batch-app
  template:
    metadata:
      labels:
        app: batch-app
    spec:
      containers:
        # Container 1: Primary Application
        - name: processing-app
          image: registry.access.redhat.com/ubi9/ubi-minimal
          command: ["sh", "-c"]
          args:
            - while true; do
                echo "Checking for incoming files...";
                ls -la /workspace/data;
                sleep 30;
              done
          volumeMounts:
            - name: rwo-storage
              mountPath: /workspace/data

        # Container 2: SFTP Sidecar (Runs non-root on port 2222)
        - name: sftp-sidecar
          image: atmoz/sftp:alpine
          args: ["sftpuser:Pass123!:::data"]
          ports:
            - name: sftp-port
              containerPort: 22
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 256Mi
          volumeMounts:
            - name: rwo-storage
              mountPath: /home/sftpuser/data

      volumes:
        - name: rwo-storage
          persistentVolumeClaim:
            claimName: rwo-block-pvc
---
# Service routing traffic directly to the sidecar's port 22
apiVersion: v1
kind: Service
metadata:
  name: batch-app-sftp-svc
  namespace: processing-jobs
spec:
  type: NodePort
  ports:
    - name: sftp
      port: 22
      targetPort: 22
      nodePort: 31022
  selector:
    app: batch-app
```

---

## Architecture Selection Decision Matrix

| Requirement | Recommended Approach | StorageClass / Technology |
| :--- | :--- | :--- |
| **Native Red Hat OpenShift, fully supported** | **Option 3**: ODF CephFS + Gateway Pod | `ocs-storagecluster-cephfs` |
| **Existing On-Premises SAN/NAS (NetApp, Dell)** | **Option 4**: Multi-Protocol NAS | `ontap-nas` (Trident) or PowerScale |
| **Using Standard Block Storage (EBS, vSphere, Ceph RBD)** | **Option 5 (Pattern B)**: Sidecar Container | `gp3-csi`, `thin`, `ocs-ceph-rbd` |
| **Cloud Managed on AWS (ROSA)** | **Option 1**: EFS + AWS Transfer Family | `efs-sc` |
| **Cloud Managed on Azure (ARO)** | **Option 2**: Azure Files native SFTP | `azurefile-csi` |
| **Massive Unstructured File Storage / Data Lake** | **Option 6**: S3 / ObjectBucketClaim + SFTPGo | `openshift-storage.noobaa.io` / S3 |

---

## Deep Dive: Ceph RBD (`ocs-storagecluster-ceph-rbd`) vs NAS (NFS / CephFS)

In OpenShift Data Foundation (ODF), you have two native storage classes:

1. **Ceph RBD (`ocs-storagecluster-ceph-rbd`)**: Block storage (`ReadWriteOnce`).
2. **CephFS (`ocs-storagecluster-cephfs`)**: Shared file storage (`ReadWriteMany`).

A frequent architectural question is: **Why is Ceph RBD considered superior to NAS (NFS or CephFS) for application workloads, and what are the trade-offs?**

---

### What Do "Ceph" and "RBD" Actually Stand For? (Origins & Acronyms)

Before exploring the architecture, understanding the naming and terminology clears up significant confusion:

#### 1. "Ceph" is NOT an Acronym

**Ceph** is a shortened name derived from **Cephalopod** (the biological class of marine animals that includes octopuses and squids).

* **The Origin**: Created by Sage Weil at the University of California, Santa Cruz. He named it after cephalopods because:
  * Octopuses have multiple distributed tentacles acting independently (symbolizing Ceph's decentralized, autonomous storage nodes).
  * Cephalopods are renowned for high intelligence, flexibility, and adaptability (symbolizing Ceph’s self-healing algorithms).
* **The Logo**: The official Ceph logo is literally a red stylized octopus/squid.

#### 2. "RBD" = RADOS Block Device (Often Mistyped as "RDB")

**RBD** stands for:
> **R**ADOS **B**lock **D**evice

To understand RBD, you must break down **RADOS**:

* **RADOS** stands for:
  > **R**eliable **A**utonomic **D**istributed **O**bject **S**tore

RADOS is the low-level, foundational storage engine at the heart of Ceph that automatically stripes, replicates, and heals raw byte objects across physical disks.

Therefore, **RBD (RADOS Block Device)** is the component of Ceph that presents raw virtual hard drives (`/dev/rbd0`) to worker nodes by striping blocks across the underlying RADOS object cluster.

#### Quick Reference: Essential Ceph Acronyms

| Acronym / Name | Full Form | What It Does in OpenShift |
| :--- | :--- | :--- |
| **Ceph** | Derived from **Cephalopod** *(squid/octopus)* | The overall distributed software-defined storage platform |
| **RBD** | **RADOS Block Device** | Block storage driver (`ReadWriteOnce` disks for VMs, DBs) |
| **RADOS** | **Reliable Autonomic Distributed Object Store** | The core self-healing object engine underneath everything |
| **CephFS** | **Ceph File System** | POSIX-compliant shared file storage (`ReadWriteMany`) |
| **RGW** | **RADOS Gateway** | S3 / OpenStack Swift REST API object gateway |
| **OSD** | **Object Storage Daemon** | Process managing a single physical hard drive or NVMe SSD |
| **MON** | **Monitor Daemon** | Maintains cluster consensus and cluster state maps (Paxos) |
| **MDS** | **Metadata Server** | In-memory directory tree and inode manager for CephFS |
| **CRUSH** | **Controlled Replication Under Scalable Hashing** | Mathematical algorithm that calculates data placement without lookup tables |

---

### Core Architectural Difference: Block vs File

```text
┌──────────────────────────────────────────────┐    ┌──────────────────────────────────────────────┐
│           Ceph RBD (Block Storage)           │    │           NAS / CephFS (File Storage)        │
├──────────────────────────────────────────────┤    ├──────────────────────────────────────────────┤
│ * Acts as a raw virtual hard drive (/dev/rbd)│    │ * Acts as a remote shared folder             │
│ * Formatted locally on worker node (XFS/ext4)│    │ * Filesystem & inodes managed on server      │
│ * Direct block striping over Ceph OSDs       │    │ * Queries Metadata Server (MDS) over network │
│ * Access Mode: ReadWriteOnce (RWO)           │    │ * Access Mode: ReadWriteMany (RWX)           │
│ * Best for: Databases, VMs, message queues   │    │ * Best for: Shared media, CMS, cross-pod data│
└──────────────────────────────────────────────┘    └──────────────────────────────────────────────┘
```

---

### Why Ceph RBD is Better than NAS (7 Technical Advantages)

#### 1. Massive I/O Throughput & Ultra-Low Latency (Databases & State)

* **No Metadata Server (MDS) Bottleneck**:
  In NAS (NFS or CephFS), every file operation (`open`, `stat`, `create`, `rename`, `unlink`, `ls`) requires network round-trips to a metadata server to look up directory trees, permissions, and inodes.
  In Ceph RBD, there is **no metadata server**. The client's kernel calculates exactly which storage daemon (OSD) holds the target block using the mathematical **CRUSH algorithm** and writes directly to it in parallel.
* **Direct Page Cache & `O_DIRECT`**:
  Databases (PostgreSQL, MySQL, Oracle, MongoDB) use `O_DIRECT` to bypass operating system caching and flush writes directly to disk. Ceph RBD handles direct block operations natively with consistent sub-millisecond p99 latency, whereas NFS caches can introduce unpredictable latency spikes.

#### 2. Strict POSIX Compliance & Reliable File Locking (No Corrupted WALs)

* **The Problem with NFS Locks**:
  NFS file locking (`lockd`, `fcntl`, `flock`) is notoriously fragile across networks. Transient network blips, packet drops, or node restarts frequently lead to:
  * Stale file handle errors (`ESTALE`).
  * Unreleased lock leases that deadlock applications.
  * Split-brain writes that can permanently corrupt database Write-Ahead Logs (WAL).
* **The RBD Advantage**:
  Because RBD is presented as a local block device, the Linux kernel formats it with standard local filesystems (`xfs` or `ext4`). All POSIX locks, atomic writes, and filesystem journals are managed locally in the host kernel, preventing database corruption.

#### 3. Instant Copy-on-Write (CoW) Snapshots & Clones

* Ceph RBD operates at the 4MB block object layer.
* Creating a `VolumeSnapshot` or cloning a 1TB database PVC takes **milliseconds** because Ceph creates a copy-on-write pointer table in RADOS rather than duplicating data.
* Restoring a snapshot or spawning a staging environment from production data is near-instantaneous.
* In NAS/NFS, creating volume clones often requires copying the entire directory tree file-by-file across the network.

#### 4. Raw Block Volume Mode (`volumeMode: Block`)

Ceph RBD supports exposing the volume to the container as a raw unformatted block device without any filesystem layer:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: raw-database-disk
spec:
  accessModes:
    - ReadWriteOnce
  volumeMode: Block    # Exposes volume as /dev/xvda inside the container
  resources:
    requests:
      storage: 100Gi
  storageClassName: ocs-storagecluster-ceph-rbd
```

* **OpenShift Virtualization (KubeVirt)**: VMs running Windows or Linux inside OpenShift require raw block devices for their virtual disks (`qemu-img` / raw disk images) to achieve near-bare-metal I/O performance.
* **Specialized Engines**: ScyllaDB, Ceph-on-Ceph, and high-performance databases take full control over disk sectors.
* **NAS cannot do this**: NAS can only serve pre-formatted file hierarchies, never raw block devices.

#### 5. Online Filesystem Expansion on the Fly

With Ceph RBD, you can expand a volume while the database and pod are actively running:

1. Edit the PVC: `storage: 200Gi`.
2. The CSI driver expands the virtual block image in Ceph.
3. The kernel automatically runs `xfs_growfs` or `resize2fs` on the live mounted filesystem without unmounting the disk or restarting the pod.

#### 6. True Security Isolation & Block Encryption

* **Network File Systems (NAS)**:
  NFS exports are shared networks paths. If an administrator misconfigures `/etc/exports` or OpenShift export rules, a rogue container or external host can potentially browse other folders on that export.
* **Ceph RBD**:
  Every PVC is an independent block image encrypted at rest via LUKS using OpenShift cluster keys or HashiCorp Vault. It cannot be accessed by any node or container other than the one that has acquired the exclusive lock.

#### 7. Automated Node Fencing & Split-Brain Prevention

When a Kubernetes worker node becomes unresponsive or network-partitioned, Ceph RBD uses built-in **watchers and exclusive locking (`exclusive-lock`)**:

* If Node A freezes and OpenShift reschedules the database pod to Node B, Ceph forces a fence on Node A's block mapping.
* Node A is barred from sending any further writes, guaranteeing that two pods will never write to the same disk simultaneously and corrupt data.

---

### The Trade-Off: Where NAS (CephFS / NFS) is Still Necessary

While Ceph RBD wins on speed, reliability, and database integrity, it has one major constraint:

| Feature | Ceph RBD (`ocs-ceph-rbd`) | NAS / CephFS (`ocs-cephfs` / NFS) |
| :--- | :--- | :--- |
| **Access Mode** | **`ReadWriteOnce` (RWO)** | **`ReadWriteMany` (RWX)** |
| **Simultaneous Nodes** | Single node only | Unlimited nodes simultaneously |
| **Direct External SFTP** | Requires in-pod sidecar | Direct export / external gateway |
| **Shared Web Uploads** | ❌ Not supported | ✅ Ideal (WordPress, Drupal, static assets) |
| **File Sharing Across Pods** | ❌ Not supported | ✅ Native multi-pod read/write |

---

### Workload Decision Matrix: When to Choose RBD vs NAS

```text
                              What type of workload?
                                        │
             ┌──────────────────────────┴──────────────────────────┐
             ▼                                                     ▼
     Stateful / Performance                               Shared Multi-Client
  (Databases, Queues, VMs)                               (CMS, Assets, SFTP)
             │                                                     │
    ┌────────┴────────┐                                   ┌────────┴────────┐
    ▼                 ▼                                   ▼                 ▼
PostgreSQL,      Kafka, Redis,                       Shared Media,    Cross-Pod
MongoDB, MySQL   Elasticsearch                       SFTP Uploads     Data Exchange
    │                 │                                   │                 │
    └────────┬────────┘                                   └────────┬────────┘
             ▼                                                     ▼
   Use Ceph RBD (RWO)                                     Use NAS / CephFS (RWX)
   `ocs-storagecluster-ceph-rbd`                          `ocs-storagecluster-cephfs`
```

| Workload Type | Recommended Storage | Why |
| :--- | :--- | :--- |
| **PostgreSQL, MySQL, MariaDB** | **Ceph RBD** | Low latency, atomic writes, reliable WAL journaling. |
| **Kafka, RabbitMQ** | **Ceph RBD** | Fast sequential block writes, zero network locking stalls. |
| **Elasticsearch, OpenSearch** | **Ceph RBD** | High IOPS index searching and segment merging. |
| **OpenShift Virtualization (VMs)** | **Ceph RBD** (`volumeMode: Block`) | Direct virtual disk image access, live VM migration. |
| **WordPress, Drupal, Web CMS** | **NAS (CephFS/NFS)** | Multiple web pods need to serve the same `/wp-content/uploads`. |
| **Legacy SFTP Ingestion Pipeline** | **NAS (CephFS/NFS)** | External SFTP drops files; multiple backend pods process them. |

---

## Solving the RWO Constraint: How to Handle Multi-Pod Read/Write (RWX) Needs

If your primary requirement is: **"I need a PVC where multiple pods can read and write simultaneously across the cluster"**, then **Ceph RBD (RWO) cannot be used directly**.

Here is why that limitation exists and how to solve it immediately in OpenShift:

---

### 1. The Direct Solution: Switch to CephFS (`ReadWriteMany`)

In OpenShift Data Foundation (ODF), Ceph provides **both** Ceph RBD (Block) and **CephFS (Shared File)** from the same storage cluster.

To allow multiple pods to read and write simultaneously:

1. Change `accessModes` from `ReadWriteOnce` to **`ReadWriteMany`**.
2. Change `storageClassName` to **`ocs-storagecluster-cephfs`** (or your cluster's NFS storage class).

#### Production CephFS RWX Manifest

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: multi-pod-shared-pvc
  namespace: my-app
spec:
  accessModes:
    - ReadWriteMany   # RWX: Multiple pods across any nodes can read & write concurrently
  resources:
    requests:
      storage: 100Gi
  storageClassName: ocs-storagecluster-cephfs
```

#### Mounting Across Multiple Pod Replicas (Deployment)

Now any Deployment can scale to 2, 5, or 50 replicas distributed across different worker nodes, all sharing the same volume without conflict:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: shared-worker
  namespace: my-app
spec:
  replicas: 4   # All 4 pods mount the exact same PVC and write concurrently
  selector:
    matchLabels:
      app: worker
  template:
    metadata:
      labels:
        app: worker
    spec:
      containers:
        - name: app
          image: registry.access.redhat.com/ubi9/ubi
          command: ["sh", "-c", "echo Pod $(hostname) wrote at $(date) >> /shared/log.txt; tail -f /dev/null"]
          volumeMounts:
            - name: shared-storage
              mountPath: /shared
      volumes:
        - name: shared-storage
          persistentVolumeClaim:
            claimName: multi-pod-shared-pvc
```

---

### 2. What Does RWO ("ReadWriteOnce") Actually Mean?

A common point of confusion in Kubernetes:

* **`ReadWriteOnce` (RWO)** means **Read/Write by ONE NODE at a time**, not necessarily one pod.
* If Pod 1 is scheduled on **Worker Node A**, and Pod 2 is scheduled on **Worker Node B**, mounting the same RWO PVC will **fail** with:

```text
Multi-Attach error for volume "pvc-xxx": Volume is already used by pod(s)...
```

---

### 3. Edge Cases: Can Multiple Pods/Containers Ever Share an RWO Volume?

If you have a scenario where you **must** use Ceph RBD block storage, there are two patterns to share access:

#### Option A: Multiple Containers in the Same Pod (Sidecar Pattern)

Containers running in the **same Pod specification** share the exact same volume on the node:

* Container 1 (Main App): Generates reports or data files.
* Container 2 (SFTP or Exporter): Reads and uploads files.
* Because both run in the same pod, they run on the same node and mount the RWO volume with zero multi-attach conflicts.

#### Option B: Co-Locating Pods on the Same Node (`podAffinity`)

If two separate pods are forced to run on the **same physical worker node** using `podAffinity`, Kubernetes allows both pods on that node to mount the RWO volume:

```yaml
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: app
                operator: In
                values:
                  - primary-app
          topologyKey: "kubernetes.io/hostname"
```

> [!CAUTION]
> **Filesystem Warning with Co-Located Pods:** Standard local filesystems (`ext4`/`xfs`) formatted on block devices are not cluster-aware filesystems. If two distinct processes write to the same file simultaneously without application-level file locking, filesystem corruption can occur. For true multi-client file sharing, **CephFS (RWX) is the recommended, native, and safe choice.**

---

## How Databases Actually Use RWO Storage in OpenShift (StatefulSets & Operators)

A fundamental insight in cloud-native database architecture: **Production databases NEVER share a single storage disk between instances.**

If two PostgreSQL or MySQL pods wrote to the same `/var/lib/postgresql/data` directory at the same time, the data files and index pointers would be immediately corrupted.

Instead, database clustering in Kubernetes/OpenShift works by giving **every database pod its own dedicated, private RWO block disk**, while data synchronization occurs entirely over the **network**.

---

### The Architecture: Share Nothing (Replication via Network, Not Disk)

```text
               ┌────────────────────────────────────────────────────────┐
               │           OpenShift High-Availability Cluster          │
               │                                                        │
               │   ┌────────────────────┐      ┌────────────────────┐   │
               │   │    postgres-0      │      │    postgres-1      │   │
               │   │    (Primary)       │      │   (Read Replica)   │   │
               │   └─────────┬──────────┘      └─────────┬──────────┘   │
               │             │                           │              │
               │             │  WAL Streaming (Network)  │              │
               │             └──────────────────────────►│              │
               └─────────────┼───────────────────────────┼──────────────┘
                             │                           │
                    Fast Local Block I/O        Fast Local Block I/O
                             │                           │
                             ▼                           ▼
               ┌───────────────────────────┐ ┌───────────────────────────┐
               │  data-postgres-0 (RWO)    │ │  data-postgres-1 (RWO)    │
               │  Ceph RBD Block Device    │ │  Ceph RBD Block Device    │
               └───────────────────────────┘ └───────────────────────────┘
```

1. **Every replica gets an independent RWO disk**:
   * Pod `postgres-0` gets its own private Ceph RBD disk (`data-postgres-0`).
   * Pod `postgres-1` gets its own private Ceph RBD disk (`data-postgres-1`).
2. **Replication happens via TCP/IP**:
   * The primary node (`postgres-0`) executes SQL writes to its local fast Ceph RBD disk.
   * It streams its Write-Ahead Log (WAL) or binlog over the cluster network to `postgres-1`.
   * The replica applies those log changes to its own private Ceph RBD disk.
3. **Failover**:
   * If `postgres-0` dies, `postgres-1` is promoted to primary immediately without touching or sharing the dead node's disk.

---

### How Client Traffic is Routed: Primary vs. Replicas Services

In a multi-pod database, how do applications know where to send read and write queries?

OpenShift creates two distinct Services:

```text
                               Application Traffic
                                       │
                  ┌────────────────────┴────────────────────┐
                  │                                         │
           Writes (INSERT, UPDATE)                   Reads (SELECT)
                  │                                         │
                  ▼                                         ▼
        [ db-primary Service ]                    [ db-replicas Service ]
                  │                                    ┌────┴────┐
                  ▼                                    ▼         ▼
        ┌──────────────────┐                 ┌──────────────────┐ ┌──────────────────┐
        │   postgres-0     │   WAL Stream    │   postgres-1     │ │   postgres-2     │
        │   (Primary/Write)├────────────────►│   (Read Replica) │ │   (Read Replica) │
        └─────────┬────────┘    (Network)    └─────────┬────────┘ └─────────┬────────┘
                  │                                    │                    │
                  ▼                                    ▼                    ▼
        ┌──────────────────┐                 ┌──────────────────┐ ┌──────────────────┐
        │  pvc-postgres-0  │                 │  pvc-postgres-1  │ │  pvc-postgres-2  │
        │  Ceph RBD (RWO)  │                 │  Ceph RBD (RWO)  │ │  Ceph RBD (RWO)  │
        └──────────────────┘                 └──────────────────┘ └──────────────────┘
```

1. **`db-primary` Service (Target: Pod with label `role=primary`)**:
   * Directs write queries exclusively to Pod 0.
   * Connection string: `jdbc:postgresql://db-primary:5432/mydb`.
2. **`db-replicas` Service (Target: Pods with label `role=replica`)**:
   * Load-balances heavy read-only reports and analytics queries across Pod 1 and Pod 2.
   * Connection string: `jdbc:postgresql://db-replicas:5432/mydb`.

---

### How Kubernetes Automates This: `StatefulSet` with `volumeClaimTemplates`

Instead of creating PVCs manually, Kubernetes uses a **`StatefulSet`**. The `volumeClaimTemplates` field automatically provisions an isolated Ceph RBD RWO disk for every single replica:

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgresql
  namespace: database-tier
spec:
  serviceName: "postgresql-headless"
  replicas: 3   # Creates postgresql-0, postgresql-1, and postgresql-2
  selector:
    matchLabels:
      app: postgresql
  template:
    metadata:
      labels:
        app: postgresql
    spec:
      containers:
        - name: postgresql
          image: registry.access.redhat.com/rhel9/postgresql-15
          ports:
            - containerPort: 5432
              name: postgresql
          volumeMounts:
            - name: data
              mountPath: /var/lib/pgsql/data
  # Automatically creates an independent RWO Ceph RBD disk for each replica!
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: [ "ReadWriteOnce" ]
        storageClassName: ocs-storagecluster-ceph-rbd
        resources:
          requests:
            storage: 100Gi
```

When you apply this StatefulSet, OpenShift dynamically creates:

* `data-postgresql-0` (Bound to `postgresql-0` on Node 1)
* `data-postgresql-1` (Bound to `postgresql-1` on Node 2)
* `data-postgresql-2` (Bound to `postgresql-2` on Node 3)

---

### What About Single-Instance Databases? (Automated Node Failover)

Even if you run a single-instance database (`replicas: 1`), Ceph RBD (RWO) provides resilient storage:

1. PostgreSQL runs on **Worker Node A** mounted to `pvc-db` (RWO).
2. **Worker Node A loses power / crashes**.
3. OpenShift detects the node failure and reschedules the PostgreSQL pod to **Worker Node B**.
4. The Ceph RBD CSI driver detaches the disk from dead Node A, fences Node A from sending stale writes, and attaches the RWO disk to Node B.
5. PostgreSQL boots up on Node B, replays the journal from the disk, and is back online in seconds without data loss.

---

### Production Cloud-Native Database Operators

All enterprise database operators on OpenShift mandate **RWO block storage (Ceph RBD)**:

* **CloudNativePG** (PostgreSQL)
* **Crunchy Data PGO** (Enterprise PostgreSQL)
* **Percona Operator** (MySQL / MongoDB / Galera)
* **Strimzi Operator** (Apache Kafka)
Every single one of these operators explicitly instructs: **"Do not use shared filesystems (NFS/CephFS). Use fast, reliable Block Storage (RWO) like Ceph RBD."**

---

## Clearing Up the Myth: Does RWO Mean "One Writer, Multiple Readers"?

**NO.** That is the single most common misconception in Kubernetes and OpenShift storage terminology.

`ReadWriteOnce` (RWO) does **NOT** mean:
> *"One pod can read/write, and others can read."* ❌ **(False)**

---

### What the Words Actually Mean

In Kubernetes access modes, the word **"Once"** or **"Many"** always refers to the **number of worker nodes**:

| Keyword Component | What It Actually Dictates |
| :--- | :--- |
| **`ReadWrite`** | The volume is mounted with read AND write permissions. |
| **`Once`** | By **ONE WORKER NODE ONLY**. |
| **`Many`** | By **MULTIPLE WORKER NODES SIMULTANEOUSLY**. |

---

### The Complete Kubernetes Access Mode Decoder

| Access Mode | What It Means | Can Other Nodes Read? | Can Other Nodes Write? | Underlying Storage |
| :--- | :--- | :---: | :---: | :--- |
| **`ReadWriteOnce` (RWO)** | Read and write by **ONE node only**. | ❌ **No (Blocked)** | ❌ **No (Blocked)** | Block storage (Ceph RBD, AWS EBS, vSphere) |
| **`ReadOnlyMany` (ROX)** | Read-only by **multiple nodes**. | ✅ **Yes** | ❌ **No (Nobody can write)** | ISO images, shared read-only assets, git repos |
| **`ReadWriteMany` (RWX)** | Read and write by **multiple nodes**. | ✅ **Yes** | ✅ **Yes** | Shared filesystems (CephFS, NFS, Azure Files) |
| **`ReadWriteOncePod` (RWOP)** | Read and write by **ONE POD ONLY**. | ❌ **No** | ❌ **No** | Strict single-pod block locks (K8s 1.22+) |

If a volume is attached as **RWO**, the storage CSI driver locks the entire disk to that single node. **No other node can even read from that volume.** Any other pod on another node attempting to mount it will fail with `Multi-Attach error`.

---

### Why Can't a Block Disk (RBD) Have "1 Writer + Multiple Readers"?

Why doesn't Kubernetes allow Node A to write to a Ceph RBD volume while Node B reads from it?

* **Operating System Page Cache**:
  Local filesystems (`ext4`, `xfs`) keep metadata, folder structures, and dirty file blocks in the writing node's RAM.
* **Cache Inconsistency & Kernel Panics**:
  If Node B reads raw disk sectors over Ceph while Node A is actively writing to them, Node B has no idea what Node A has cached in RAM. Node B will read half-written sectors, corrupt its local inode tables, and crash the Linux filesystem driver.
* **Conclusion**: Standard block storage must be exclusively locked to one node at a time.

---

### How to ACTUALLY Achieve "1 Writer + Multiple Readers"

If your business requirement is genuinely: **"I have 1 writer pod that generates reports/assets, and 5 reader pods that only read them"**, here is how to do it:

#### Pattern 1: Use `ReadWriteMany` (CephFS / NFS) with `readOnly: true`

Use CephFS (`ocs-storagecluster-cephfs`) which handles network cache coordination. Then enforce read-only permissions inside the pod's `volumeMounts`:

```yaml
# Pod 1: The Writer Pod (readOnly: false)
volumeMounts:
  - name: shared-data
    mountPath: /data
    readOnly: false     # Can write to the volume

# Pod 2, 3, 4: The Reader Pods (readOnly: true)
volumeMounts:
  - name: shared-data
    mountPath: /data
    readOnly: true      # Enforces read-only in the container!
```

#### Pattern 2: Database / Message Queue Pattern (The Modern Approach)

* Pod 1 (Primary Database) has its own private Ceph RBD disk and accepts writes.
* Pod 2, 3 (Read Replicas) have their own private Ceph RBD disks.
* Pod 1 streams changes over TCP/IP to Pod 2 and 3.
* Application reads query the replicas without any disk-level locking conflicts.

---

## Ceph RBD Architecture: Are Files Written on the Local Node or on Remote Servers?

A crucial architectural question: **"In Ceph RBD, are files written on the worker node itself, or are they sent to a remote storage server?"**

### The Short Answer: Both

* The **filesystem structure** (`XFS` or `ext4`, directories, inodes) is managed **locally by the worker node's Linux kernel**.
* The **actual data blocks** are immediately sent across the network and persistently stored on the **remote Ceph storage cluster (replicated 3 times)**.

---

### Step-by-Step: What Happens When a Pod Writes a File to Ceph RBD

```text
 1. Container writes "hello" to /data/report.txt
    │
 2. Worker Node Kernel (VFS / ext4 filesystem)
    Translates file into raw sector blocks (e.g. Block #8192)
    │
 3. Kernel Ceph Driver (/dev/rbd0)
    Uses CRUSH algorithm to calculate which Ceph servers hold that block
    │
 4. High-Speed Storage Network (10/25/100 GbE)
    Sends raw block packets across the cluster
    │
    ▼
 ┌─────────────────────────────────────────────────────────────┐
 │            Remote Ceph Storage Cluster (ODF)                │
 │                                                             │
 │   ┌───────────────┐   ┌───────────────┐   ┌───────────────┐ │
 │   │  Ceph OSD 1   │   │  Ceph OSD 2   │   │  Ceph OSD 3   │ │
 │   │ (Storage Node)│   │ (Storage Node)│   │ (Storage Node)│ │
 │   │  [Primary]    ├───┼──►[Replica 1] ├───┼──►[Replica 2] │ │
 │   └───────────────┘   └───────────────┘   └───────────────┘ │
 └─────────────────────────────────────────────────────────────┘
```

1. **Inside the Pod**: The container writes to `/data/report.txt` just like a local file.
2. **On the Worker Node**: The Linux kernel manages the directory tree, file locks, and filesystem journaling locally on a virtual block device (`/dev/rbd0`).
3. **Over the Network**: The kernel Ceph driver breaks that virtual disk into 4MB chunk objects and transmits them over the storage network to the **remote Ceph storage servers**.
4. **On the Remote Ceph Cluster**: Ceph writes the block to disk and replicates it across **3 separate physical storage nodes and racks**.

---

### The Node Failure Test: What Happens if the Worker Node Dies?

This is the key test that proves data does **not** live on the worker node:

* Suppose **Worker Node 1 completely crashes or loses power**.
* **Is your data lost? NO.**
* Because the actual blocks are stored on the remote Ceph storage nodes, OpenShift simply restarts your pod on **Worker Node 2**.
* Worker Node 2 attaches the remote Ceph RBD block device (`/dev/rbd0`) in **under 3 seconds**, replays the journal, and resumes operation with **zero data loss**.

---

### Comparison: Local Disk vs. Ceph RBD vs. NAS

| Storage Type | Where is Filesystem Managed? | Where is Data Physically Stored? | Survives Worker Node Death? |
| :--- | :--- | :--- | :---: |
| **Local HostPath (`local-storage`)** | Local Worker Node | Local Worker Node's physical SSD | ❌ **No (Tied to that node)** |
| **Ceph RBD (`ocs-ceph-rbd`)** | Local Worker Node | **Remote Ceph Storage Cluster (3x Replicated)** | ✅ **Yes (Instant failover)** |
| **NAS / NFS (`ocs-cephfs` / NFS)** | Remote Storage Server | **Remote Ceph / NFS Server** | ✅ **Yes (Instant failover)** |

---

## Why Ceph RBD is Faster Than NFS Even Though Both Are Remote

A natural follow-up question arises: **"If Ceph RBD sends data blocks across the network to remote storage servers just like NFS does, why is Ceph RBD significantly faster (often 5x to 10x higher IOPS and drastically lower latency) than NFS?"**

The performance difference does not come from physical wire distance—both travel over standard Ethernet or InfiniBand networks. The massive speed advantage comes from **architectural layering, network chattiness, and the fundamental differences between Block-level and File-level storage**.

---

### 1. Protocol Chattiness: 1 Block RPC vs. 6+ File RPCs

When an application writes data to a file, the differences in protocol overhead between NFS and Ceph RBD are dramatic:

#### NFS (File-Level Protocol): Extremely Chatty

In NFS, every filesystem concept (directory traversal, permission checks, file allocation, byte-range locks) must be verified with the remote server over the network:

```text
Container write() ──► Linux VFS ──► NFS Client
                                      │
  (1) RPC: LOOKUP (resolve directory path and inode)   ──────► NFS Server
  (2) RPC: ACCESS (check user permissions on remote host) ───► NFS Server
  (3) RPC: OPEN   (open file descriptor)                ─────► NFS Server
  (4) RPC: SETATTR / LOCK (acquire byte-range lock)     ─────► NFS Server
  (5) RPC: WRITE  (send data payload)                   ─────► NFS Server
  (6) RPC: COMMIT / CLOSE (flush and acknowledge)       ─────► NFS Server
```

* For a single small write, the client and server exchange **4 to 8 network round-trips**.
* If network latency between nodes is 0.5 ms, 6 round trips equal **3.0 ms latency minimum** before the application receives write confirmation.

#### Ceph RBD (Block-Level Protocol): Direct and Lean

In Ceph RBD, the filesystem (`XFS` or `ext4`) lives **inside the worker node's Linux kernel**. Path resolution, permissions, and directory inodes are processed **locally in node RAM**:

```text
Container write() ──► Linux VFS (ext4/XFS in local RAM)
                            │ (Local Inode & Permission Lookup: 0 µs)
                            ▼
                      Kernel Ceph RBD Driver (/dev/rbd0)
                            │
  (1) Single RPC: WRITE (Object ID: rbd_data.1234, Offset: 0, Len: 4096) ──► Ceph OSD
```

* **1 single network round-trip** directly to the target storage disk.
* Latency overhead is purely the raw network packet transit time + NVMe/SSD commit time (~0.3 ms total).

---

### 2. Striping Across Many Servers (CRUSH) vs. Single-Server Bottleneck

```text
        NFS Storage Flow                                  Ceph RBD Storage Flow

        Worker Node Pods                                    Worker Node Pods
       [Pod A]  [Pod B]  [Pod C]                           [Pod A]  [Pod B]  [Pod C]
          │        │        │                                 │        │        │
          └────────┼────────┘                                 │        │        │
                   ▼                                          ▼        ▼        ▼
           ┌───────────────┐                            CRUSH Deterministic Calculation
           │  NFS Server   │                                  │        │        │
           │ (Single IP)   │                                  ▼        ▼        ▼
           │ 1 Network Card│                             ┌────────┐┌────────┐┌────────┐
           │ 1 CPU Core Set│                             │ Ceph   ││ Ceph   ││ Ceph   │
           │ 1 Controller  │                             │ OSD 1  ││ OSD 2  ││ OSD 3  │
           └───────────────┘                             └────────┘└────────┘└────────┘
          All traffic chokes on                       Simultaneous parallel streams across
           single server limits                        tens or hundreds of disks & NICs
```

* **NFS Bottleneck**:
  An NFS mount points to a single IP address (`192.168.1.50:/exports/data`). Every read and write from every pod passes through that single machine's CPU, RAM, and network interface card (NIC).
* **Ceph RBD Parallelism**:
  An RBD disk is not stored on a single machine. It is divided into **4MB chunk objects** and distributed across dozens of physical disks and servers using the **CRUSH algorithm**:
  * Pod writes Block 1 ➔ Streamed directly to **Server 1 / Disk 1**.
  * Pod writes Block 2 ➔ Streamed directly to **Server 2 / Disk 4**.
  * Pod writes Block 3 ➔ Streamed directly to **Server 3 / Disk 2**.
  * Ceph RBD aggregates the network bandwidth and combined IOPS of the **entire storage cluster simultaneously**.

---

### 3. No Filesystem Metadata Overhead on the Storage Cluster

* **NFS Server Overhead**:
  The NFS server must constantly update directory trees, directory modification timestamps (`mtime`), link counts, and byte-range locks. When thousands of files exist in a directory, the NFS server's CPU spends most of its time parsing metadata structures instead of transferring raw data.
* **Ceph Cluster Simplicity with RBD**:
  The Ceph storage cluster **does not know what a file, directory, or folder is**. It only sees numbered byte objects (e.g., `rbd_data.4a8b.000000000001`).
  * Inode allocation, directory traversal, and permission verification happen entirely in the **worker node's own CPU and RAM**.
  * Ceph storage nodes (OSDs) only do one thing: store and replicate raw blocks to fast NVMe/SSD storage.

---

### 4. Safe Local Linux Page Caching

Because Ceph RBD is an **exclusive block device** bound to one worker node at a time (`ReadWriteOnce`), the Linux kernel knows that **no other server can modify those blocks**:

* **RBD**:
  The worker node can safely use its full **Linux Page Cache (RAM)** to buffer reads and writes. Frequently read data stays in worker node memory and returns in nanoseconds without ever hitting the network.
* **NFS**:
  Because NFS assumes multiple clients might modify files at any time, NFS clients enforce strict cache consistency rules (such as `close-to-open` cache coherency and `actimeo` polling timers). Every time an application opens a file, NFS is forced to send network RPCs to check if the remote file has changed, destroying cache efficiency.

---

### 5. Multi-Queue Block I/O (`blk-mq`) and Deep Concurrency

* Modern Linux uses `blk-mq` (Multi-Queue Block I/O), allowing thousands of concurrent I/O requests per CPU core with hardware queue depths of 64, 128, or 256. Ceph RBD plugs directly into `blk-mq`. Databases using asynchronous I/O (`io_uring` or `libaio`) can execute thousands of concurrent I/Os without stalling.
* NFS historically processes requests sequentially or over limited RPC connection slots, causing severe serialization stalls under high concurrency.

---

### Performance Comparison: Ceph RBD vs. Traditional NFS

| Metric | Traditional NFS | Ceph RBD (`ocs-ceph-rbd`) | Why RBD Wins |
| :--- | :--- | :--- | :--- |
| **I/O Protocol Level** | File (VFS over RPC) | Block (Raw block driver) | Zero file-level RPC handshakes |
| **RPCs Per Write** | 4 to 8 network RPCs | **1 network RPC** | Eliminates network round-trips |
| **Network Throughput Target** | Single NFS server IP | **Entire Ceph cluster fabric** | No single NIC bottleneck |
| **Metadata Processing** | Remote NFS host CPU | **Local worker node RAM** | Bypasses remote metadata serialization |
| **Random 4K Write IOPS** | Low (~1,500 – 5,000 IOPS) | **High (20,000 – 100,000+ IOPS)** | Native NVMe striping across cluster |
| **fsync / WAL Latency** | High (5 ms – 25 ms) | **Ultra-Low (0.5 ms – 1.8 ms)** | Immediate replica commit |
| **Best Used For** | Shared configs, shared CMS assets | **Databases (PostgreSQL, MySQL, Kafka)** | Sub-millisecond transactional performance |

---

## CephFS vs. NFS: Are They the Same? What is the Difference?

To an application running inside an OpenShift container, **CephFS and NFS feel almost identical**:

* Both provide a shared network folder with standard directory hierarchies (`/var/data/shared`).
* Both support **`ReadWriteMany` (RWX)**, allowing dozens or hundreds of pods to read and write simultaneously.
* Both support standard POSIX file operations (`open`, `read`, `write`, `close`).

**However, under the hood, they are fundamentally different architectures.**

---

### Core Architectural Comparison

```text
       Traditional NFS (Centralized)                    CephFS (Distributed Scale-Out)

             Worker Nodes                                     Worker Nodes
          [Pod]  [Pod]  [Pod]                              [Pod]  [Pod]  [Pod]
            │      │      │                                  │      │      │
            └──────┼──────┘                                  │      │      │
                   ▼                                         ▼      ▼      ▼
           ┌───────────────┐                       Metadata      Data Payload
           │  NFS Server   │                       (In-Memory)   (Striped in Parallel)
           │ (Single Box)  │                           │            │      │      │
           │  - All CPU    │                           ▼            ▼      ▼      ▼
           │  - All NICs   │                       ┌───────┐    ┌──────┐┌──────┐┌──────┐
           │  - All Disks  │                       │CephMDS│    │ OSD 1││ OSD 2││ OSD 3│
           └───────────────┘                       └───────┘    └──────┘└──────┘└──────┘
        Single Point of Failure                    Scales to hundreds of storage nodes
```

---

### 6 Key Differences Between CephFS and NFS

#### 1. Centralized vs. Distributed Scale-Out

* **NFS (Centralized)**:
  An NFS share lives on **one physical machine or VM** (a single server IP). If 100 pods concurrently read or write, all traffic funnels through that single server's network card, memory, and CPU.
* **CephFS (Distributed)**:
  CephFS is **massively distributed**. A single file is chopped into objects and striped across **tens or hundreds of storage servers (OSDs) in parallel**. If 100 pods write simultaneously, traffic is spread evenly across the entire cluster network.

#### 2. Separation of Data and Metadata (Ceph MDS)

* **NFS**:
  The NFS server handles both file payload data and filesystem metadata (directory trees, timestamps, permissions) on the same disk controller. Under high file counts, directory lookups stall I/O.
* **CephFS**:
  CephFS strictly decouples **Metadata** from **Data**:
  * **Metadata Servers (MDS)**: Keep directory trees and inode maps in fast server RAM.
  * **OSDs (Storage Nodes)**: Store the actual file chunk data.
  * When a pod writes, it consults the MDS for permissions, then streams file data **directly to the storage disks**, bypassing the metadata servers entirely.

#### 3. Client Caching: Coherent Capabilities ("Caps") vs. Polling Timeouts (`actimeo`)

* **NFS**:
  NFS clients use primitive cache timeouts (like `actimeo=30`). A pod might read stale cached data for up to 30 seconds unless configured with `noac` (which destroys performance).
* **CephFS**:
  CephFS uses an advanced **Distributed Capability ("Caps")** engine:
  * When Pod A opens a file, the Ceph MDS grants it a "read and cache capability."
  * If Pod B suddenly opens the same file to write, the Ceph MDS instantly revokes Pod A's cache capability over the network and flushes dirty pages synchronously.
  * **Result**: You get aggressive, high-speed RAM caching with 100% true cache coherence across all pods.

#### 4. High Availability & Self-Healing

* **NFS**:
  If the NFS server crashes, all OpenShift pods lock up with `NFS server not responding`. Setting up high availability requires complex active-passive failover clustering (Pacemaker/Corosync) with shared SAN disks or DRBD replication.
* **CephFS**:
  Ceph is inherently self-healing and zero-downtime. File objects are replicated 3x across the cluster. If an OSD storage disk or node fails, Ceph heals and re-replicates in the background with zero disruption to active pods.

#### 5. Scalability Limits

* **NFS**:
  Typically limited to the maximum disk space and IOPS of a single server appliance (e.g., 50TB–200TB, 10Gbps–40Gbps NIC).
* **CephFS**:
  Can scale out to **tens of petabytes** and hundreds of storage servers simply by adding more disks or nodes to the Ceph cluster.

#### 6. OpenShift Native Integration (ODF)

* **NFS**:
  Requires an external NFS appliance or bastion server, manual export management, and static PV creation.
* **CephFS**:
  Comes pre-packaged as part of **OpenShift Data Foundation (ODF)**. Fully integrated with OpenShift's Web Console, Prometheus monitoring, automated dynamic CSI provisioning, and Red Hat enterprise support.

---

### Comparison Matrix: CephFS vs. Traditional NFS

| Feature | Traditional NFS | CephFS (ODF) |
| :--- | :--- | :--- |
| **Architecture** | Centralized single server | Distributed scale-out cluster |
| **Access Mode** | `ReadWriteMany` (RWX) | `ReadWriteMany` (RWX) |
| **Data Striping** | Single disk array on one host | Distributed parallel striping across cluster OSDs |
| **Metadata Engine** | Integrated on NFS host disk | Dedicated in-memory Ceph MDS cluster |
| **Cache Consistency** | Polling timeouts (`actimeo`) | Distributed capability locks (Caps) |
| **High Availability** | Active-Passive failover | Active-Active self-healing RADOS |
| **Single Point of Failure** | Yes (unless complex HA pair) | None (fully distributed) |
| **OpenShift Management** | External / Manual | Native OpenShift Operator (ODF) |

---

## 13. Allowing Users to Download Files from a CephFS PVC

A frequent real-world requirement: **"I have an application writing reports, data exports, logs, or user-uploaded media to a CephFS PVC in OpenShift. How do I allow people (end-users, business teams, external partners, or developers) to browse and download these files?"**

---

### The Superpower of CephFS: Concurrent Multi-Pod Access

Because CephFS is **`ReadWriteMany` (RWX)**, you do **NOT** need to disrupt or modify your existing backend application pods.

* Your backend producer pods can continue writing to the PVC in `readWrite` mode.
* You can deploy a dedicated **file-serving gateway pod** (Web UI, HTTP server, or SFTP) that mounts the **exact same PVC at the same time**.
* To guarantee complete safety, the download gateway pod mounts the CephFS volume with **`readOnly: true`**, making it physically impossible for downloaders to accidentally delete or corrupt backend data.

```text
                                 ┌─────────────────────────────────┐
                                 │    CephFS Storage Volume (RWX)  │
                                 │      (ocs-storagecluster-cephfs) │
                                 └────────────────┬────────────────┘
                                                  │
                         ┌────────────────────────┴────────────────────────┐
                         │                                                 │
                         ▼                                                 ▼
          ┌─────────────────────────────┐                   ┌─────────────────────────────┐
          │     Backend Producer Pod    │                   │   Download Gateway Pod(s)   │
          │   Mount: /var/data (ReadWrite)│                  │ Mount: /srv/data (ReadOnly) │
          └─────────────────────────────┘                   └──────────────┬──────────────┘
                         ▲                                                 │
                         │ Writes live files                               ▼
                 [Application Logic]                     ┌───────────────────────────────────┐
                                                         │     OpenShift Route / Ingress     │
                                                         │   (HTTPS edge-terminated URL)     │
                                                         └─────────────────┬─────────────────┘
                                                                           │
                                                                           ▼
                                                                  [End Users / Clients]
                                                                (Web Browser, Curl, SFTP)
```

Depending on who needs to download the files and how they work, choose one of the four battle-tested methods below:

---

### Method 1: Web Browser Download Portal via FileBrowser (Recommended for End-Users)

If end-users, analysts, or non-technical stakeholders need to download files, providing a **Web UI** is the best experience. Users simply open an HTTPS URL in their browser, log in, browse folders, and click "Download".

**FileBrowser** is a lightweight, zero-dependency web-based file manager that runs inside a tiny container (~30MB RAM).

#### Key Features of FileBrowser

* Intuitive web file explorer (folders, file size, timestamps).
* One-click file downloads or multi-file ZIP downloads.
* Built-in previews for PDFs, text files, markdown, and images.
* Multi-user authentication with customizable read-only accounts.
* Search bar to locate files quickly across large directory trees.

#### Complete OpenShift Manifest: FileBrowser on CephFS

Save the following as `cephfs-filebrowser.yaml` and apply it to your project:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cephfs-filebrowser
  namespace: my-project
  labels:
    app: cephfs-filebrowser
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cephfs-filebrowser
  template:
    metadata:
      labels:
        app: cephfs-filebrowser
    spec:
      containers:
        - name: filebrowser
          image: docker.io/filebrowser/filebrowser:v2-s6
          imagePullPolicy: IfNotPresent
          env:
            # Tell FileBrowser where to store its internal user database
            - name: FB_DATABASE
              value: /tmp/filebrowser.db
            - name: FB_ROOT
              value: /srv/data
            - name: FB_PORT
              value: "8080"
            - name: FB_NOAUTH
              value: "false" # Set to "true" if you want public access without login
          ports:
            - containerPort: 8080
              name: http
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 500m
              memory: 256Mi
          volumeMounts:
            # Mount your existing CephFS PVC
            - name: cephfs-data
              mountPath: /srv/data
              readOnly: true # Prevents web users from deleting/modifying files!
      volumes:
        - name: cephfs-data
          persistentVolumeClaim:
            claimName: <your-cephfs-pvc-name>
---
apiVersion: v1
kind: Service
metadata:
  name: cephfs-filebrowser-svc
  namespace: my-project
spec:
  selector:
    app: cephfs-filebrowser
  ports:
    - name: http
      port: 8080
      targetPort: 8080
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: cephfs-filebrowser
  namespace: my-project
spec:
  to:
    kind: Service
    name: cephfs-filebrowser-svc
  port:
    targetPort: http
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

#### How Users Access and Download

1. Deploy the manifest:

   ```bash
   oc apply -f cephfs-filebrowser.yaml -n my-project
   ```

2. Retrieve the public HTTPS URL from OpenShift:

   ```bash
   oc get route cephfs-filebrowser -n my-project -o jsonpath='{"https://"}{.spec.host}{"\n"}'
   ```

3. Open the URL in any web browser.
4. Log in with the default credentials:
   * **Username**: `admin`
   * **Password**: `admin`
   *(Immediately navigate to **Settings ➔ User Management** to change the password or create read-only accounts).*
5. Users can browse the directory tree, click any file to download, or select multiple files and click **Download as ZIP**.

---

### Method 2: High-Speed HTTP Directory Index via Nginx (Fastest for Direct File Links)

If you need a lightweight, high-performance, and completely maintenance-free solution where users or automated scripts can download files via direct links (e.g. `https://downloads.example.com/reports/2026-data.csv` or `curl -O`), use an **Nginx autoindex file server**.

#### Key Features of Nginx Autoindex

* Extreme performance: streams large gigabyte files directly from CephFS via Linux `sendfile`.
* Zero user management required.
* Native browser directory listing.
* Works seamlessly with `wget`, `curl`, and automated download scripts.

#### Complete OpenShift Manifest: Nginx Autoindex on CephFS

Save as `cephfs-nginx-downloader.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-downloader-config
  namespace: my-project
data:
  default.conf: |
    server {
        listen 8080;
        server_name localhost;

        location / {
            root /usr/share/nginx/html;
            autoindex on;               # Enables directory browsing!
            autoindex_exact_size off;   # Displays human-readable file sizes (MB/GB)
            autoindex_localtime on;    # Displays local timestamps
            charset utf-8;

            # Optimize for high-throughput file downloads
            sendfile on;
            sendfile_max_chunk 1m;
            tcp_nopush on;
            tcp_nodelay on;
            keepalive_timeout 65;
        }

        # Health probe endpoint
        location /healthz {
            return 200 'OK';
        }
    }
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cephfs-nginx-downloader
  namespace: my-project
spec:
  replicas: 2 # Scale horizontally for high-traffic download spikes!
  selector:
    matchLabels:
      app: cephfs-nginx-downloader
  template:
    metadata:
      labels:
        app: cephfs-nginx-downloader
    spec:
      containers:
        - name: nginx
          image: registry.access.redhat.com/ubi9/nginx-122
          ports:
            - containerPort: 8080
              name: http
          livenessProbe:
            httpGet:
              path: /healthz
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          volumeMounts:
            - name: nginx-conf
              mountPath: /etc/nginx/conf.d/default.conf
              subPath: default.conf
            - name: cephfs-storage
              mountPath: /usr/share/nginx/html
              readOnly: true
      volumes:
        - name: nginx-conf
          configMap:
            name: nginx-downloader-config
        - name: cephfs-storage
          persistentVolumeClaim:
            claimName: <your-cephfs-pvc-name>
---
apiVersion: v1
kind: Service
metadata:
  name: cephfs-nginx-downloader-svc
  namespace: my-project
spec:
  selector:
    app: cephfs-nginx-downloader
  ports:
    - name: http
      port: 8080
      targetPort: 8080
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: cephfs-downloads
  namespace: my-project
spec:
  to:
    kind: Service
    name: cephfs-nginx-downloader-svc
  port:
    targetPort: http
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

#### How Users Download

1. Deploy the manifest:

   ```bash
   oc apply -f cephfs-nginx-downloader.yaml -n my-project
   ```

2. Get the download URL:

   ```bash
   oc get route cephfs-downloads -n my-project -o jsonpath='{"https://"}{.spec.host}{"\n"}'
   ```

3. Users can browse the directory tree in their browser, or scripts can download directly:

   ```bash
   # Download a file via curl
   curl -O https://cephfs-downloads-my-project.apps.cluster.com/exports/daily-report.csv

   # Download an entire directory recursively using wget
   wget -r -np -nH --cut-dirs=1 https://cephfs-downloads-my-project.apps.cluster.com/exports/
   ```

---

### Method 3: In-Cluster SFTP Gateway (For WinSCP, FileZilla, and Batch Jobs)

If users need to download files using desktop graphical file transfer clients (like **WinSCP**, **FileZilla**, or **Cyberduck**), or if external enterprise systems pull files via automated SFTP/SCP scripts:

#### Can WinSCP Connect Over an OpenShift Ingress / Route?

**NO.** A standard OpenShift `Route` or Kubernetes `Ingress` **cannot** be used for WinSCP:

* **Why Routes Fail for WinSCP**:
  * WinSCP connects using the **SFTP protocol**, which runs over **raw SSH (Layer 4 TCP)**.
  * OpenShift Ingress (`Route`) is a **Layer 7 HTTP/HTTPS reverse proxy** (HAProxy).
  * An OpenShift Route listens on ports 80 and 443 and requires HTTP `Host` headers or TLS SNI (Server Name Indication) to determine which service to send traffic to.
  * Standard SSH/SFTP does **not** send TLS SNI hostnames or HTTP headers. If WinSCP tries to connect to an OpenShift Route on port 443, the router drops the connection because the SSH handshake is invalid HTTP/TLS.

---

#### The 3 Ways to Connect WinSCP to OpenShift CephFS

##### Option A: Service Type `NodePort` (On-Premises / Bare Metal)

OpenShift opens a static high port (in the range 30000–32767) on **every worker node** in the cluster.

1. **Service Manifest (`sftp-nodeport.yaml`)**:

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: cephfs-sftp-nodeport
     namespace: my-project
   spec:
     type: NodePort
     selector:
       app: cephfs-sftp
     ports:
       - name: sftp
         port: 22
         targetPort: 22
         nodePort: 32222 # Choose a port between 30000-32767
   ```

2. **WinSCP Connection Settings**:
   * **File protocol**: `SFTP`
   * **Host name**: `<Any-OpenShift-Worker-Node-IP>` (e.g., `192.168.1.50`)
   * **Port number**: `32222` *(Must change from default 22 to your NodePort)*
   * **User name**: `downloader`
   * **Password**: `YourPassword` (or load private `.ppk` key in **Advanced ➔ SSH ➔ Authentication**)

---

##### Option B: Service Type `LoadBalancer` (Cloud or On-Prem with MetalLB)

A dedicated, routable external IP is assigned directly to the SFTP service, enabling standard port 22 access.

1. **Service Manifest (`sftp-loadbalancer.yaml`)**:

   ```yaml
   apiVersion: v1
   kind: Service
   metadata:
     name: cephfs-sftp-lb
     namespace: my-project
   spec:
     type: LoadBalancer
     selector:
       app: cephfs-sftp
     ports:
       - name: sftp
         port: 22
         targetPort: 22
   ```

2. **WinSCP Connection Settings**:
   * **File protocol**: `SFTP`
   * **Host name**: `<LoadBalancer-External-IP-or-DNS>` (e.g., `sftp.company.com` or `10.200.5.15`)
   * **Port number**: `22` *(Standard default port)*
   * **User name**: `downloader`
   * **Password**: `YourPassword`

---

##### Option C: Developer / Admin Port-Forwarding (Zero Cluster Network Changes)

If corporate firewalls block external ports (32222 or 22), developers and administrators can tunnel WinSCP through the OpenShift API using `oc port-forward`:

1. **Start the Port-Forward Tunnel**:

   ```bash
   oc port-forward pod/<sftp-pod-name> 2222:22 -n my-project
   ```

2. **WinSCP Connection Settings**:
   * **File protocol**: `SFTP`
   * **Host name**: `127.0.0.1` (or `localhost`)
   * **Port number**: `2222`
   * **User name**: `downloader`
   * **Password**: `YourPassword`

* **Advantage**: Fully encrypted through OpenShift's TLS API; requires no firewall tickets or external IPs.

---

#### Read-Only Safety for WinSCP Downloaders

To prevent WinSCP users from accidentally deleting or overwriting files on the shared CephFS PVC:

* In the SFTP Gateway Secret, configure the user with `:ro`:

  ```yaml
  # format: user:password[:[uid]:[gid]:[dir]:ro]
  SFTP_USERS: "downloader:SecurePass123:::files:ro"
  ```

* Any attempt in WinSCP to delete, rename, or upload a file will result in `Permission denied`.

---

#### What If You MUST Use OpenShift Ingress / Port 443 with WinSCP? (The WebDAV Solution)

If your enterprise strictly forbids opening `NodePort` (30000–32767) and does NOT have a `LoadBalancer` service, meaning **all traffic MUST enter through standard OpenShift Ingress / Routes on Port 443 (HTTPS)**:

You can still use **WinSCP**!

WinSCP is not only an SFTP client; it natively supports the **WebDAV protocol**. Because WebDAV is an extension of HTTP/HTTPS, it routes **100% natively through standard OpenShift Routes on port 443**.

##### 1. Deploy WebDAV on CephFS (`cephfs-webdav.yaml`)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: cephfs-webdav
  namespace: my-project
spec:
  replicas: 1
  selector:
    matchLabels:
      app: cephfs-webdav
  template:
    metadata:
      labels:
        app: cephfs-webdav
    spec:
      containers:
        - name: webdav
          image: docker.io/bytemark/webdav:latest
          env:
            - name: AUTH_TYPE
              value: "Basic"
            - name: USERNAME
              value: "downloader"
            - name: PASSWORD
              value: "SecurePass123"
            - name: READONLY
              value: "true" # Enforces download-only access!
          ports:
            - containerPort: 80
              name: http
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 500m
              memory: 256Mi
          volumeMounts:
            - name: cephfs-data
              mountPath: /var/lib/dav/data
              readOnly: true
      volumes:
        - name: cephfs-data
          persistentVolumeClaim:
            claimName: <your-cephfs-pvc-name>
---
apiVersion: v1
kind: Service
metadata:
  name: cephfs-webdav-svc
  namespace: my-project
spec:
  selector:
    app: cephfs-webdav
  ports:
    - name: http
      port: 80
      targetPort: 80
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: cephfs-webdav
  namespace: my-project
spec:
  to:
    kind: Service
    name: cephfs-webdav-svc
  port:
    targetPort: http
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

##### 2. Connect WinSCP via OpenShift Route (Port 443)

In the WinSCP login window:

1. **File protocol**: Select **`WebDAV`** from the dropdown (instead of SFTP).
2. **Encryption**: Select **`TLS/SSL Implicit encryption`**.
3. **Host name**: Enter the OpenShift Route hostname (e.g., `cephfs-webdav-my-project.apps.mycluster.com`).
4. **Port number**: `443`.
5. **User name**: `downloader`.
6. **Password**: `SecurePass123`.

* **Result**: Users get the exact same WinSCP dual-pane file explorer experience, transferring files through your corporate OpenShift Route on standard HTTPS, with **zero LoadBalancer and zero NodePort required**.

---

### Method 4: Developer & Admin CLI Downloads (No New Pods Needed)

If the person who needs to download files is a developer, DevOps engineer, or administrator with `oc` cluster credentials, you do **not** need to deploy any Web UI or SFTP server. Use native OpenShift CLI tools against an existing pod that already has the CephFS PVC mounted:

#### A. Download an Entire Folder via `oc rsync` (Fastest for Directories)

```bash
# Sync remote directory from pod to local folder
oc rsync <pod-name>:/path/to/cephfs/mount/ ./local-download-folder/ -n my-project

# Exclude unwanted files or logs
oc rsync <pod-name>:/path/to/cephfs/mount/ ./local-download-folder/ --exclude="*.tmp" -n my-project
```

#### B. Download a Single File via `oc cp`

```bash
# Copy single file from pod to local machine
oc cp <pod-name>:/path/to/cephfs/mount/report.pdf ./report.pdf -n my-project
```

#### C. Stream Directly via `oc exec` Pipeline

```bash
# Stream and extract a compressed archive on the fly
oc exec <pod-name> -n my-project -- tar -czf - -C /path/to/cephfs/mount my-folder | tar -xzf -
```

#### D. Ad-Hoc 5-Minute Download via Ephemeral Python Server

If you need to download a large dataset quickly to your workstation without deploying permanent Routes:

```bash
# 1. Start an ephemeral debug pod mounting the CephFS PVC
oc run cephfs-downloader --rm -it \
  --image=registry.access.redhat.com/ubi9/ubi \
  --restart=Never \
  --overrides='{
    "spec": {
      "volumes": [{"name": "data", "persistentVolumeClaim": {"claimName": "<your-cephfs-pvc>"}}],
      "containers": [{
        "name": "downloader",
        "image": "registry.access.redhat.com/ubi9/python-39",
        "command": ["python3", "-m", "http.server", "8080", "--directory", "/mnt/data"],
        "volumeMounts": [{"name": "data", "mountPath": "/mnt/data", "readOnly": true}]
      }]
    }
  }' -n my-project

# 2. In another terminal, port-forward to your laptop:
oc port-forward pod/cephfs-downloader 8080:8080 -n my-project

# 3. Open http://localhost:8080 in your browser and download whatever you need!
```

---

### Security Best Practices for CephFS File Downloads

1. **Always Set `readOnly: true` on Download Pods**:
   In the download pod's `volumeMounts`, always set `readOnly: true`. Because CephFS supports multi-client mounting, this isolates the download interface from write permissions. Even if the download pod or web portal is compromised, your actual persistent data cannot be wiped or altered.
2. **Protect Web Portals with OpenShift OAuth Proxy**:
   If the files contain sensitive company data, do not expose a public unauthenticated route. Wrap the FileBrowser or Nginx service with the **OpenShift OAuth Proxy sidecar** (`registry.redhat.io/openshift4/ose-oauth-proxy`). This forces all web visitors to log in with their corporate OpenShift / Single Sign-On (SSO) credentials before gaining access to the files.
3. **NetworkPolicy Isolation**:
   If using the SFTP Gateway method, apply an OpenShift `NetworkPolicy` to ensure the gateway pod can only communicate with the storage network and authorized client CIDRs, blocking it from accessing internal database or control-plane services.

---

### Decision Matrix: Which Download Method Should You Use?

#### Audience & User Experience Matrix

| Requirement / Audience | Recommended Solution | Setup Complexity | User Experience |
| :--- | :--- | :---: | :--- |
| **Non-Technical End Users / Business Teams** | **Method 1: FileBrowser** | Low (Single YAML) | ⭐⭐⭐⭐⭐ Rich Web UI, previews, zip download |
| **Public Downloads / Automated `curl` / `wget`** | **Method 2: Nginx Autoindex** | Low (Single YAML) | ⭐⭐⭐⭐ Direct URLs, highest download throughput |
| **WinSCP on Corporate Networks (Port 443 Only)** | **WebDAV Gateway** | Low (Single YAML) | ⭐⭐⭐⭐ Native WinSCP over HTTPS, no custom ports |
| **Desktop FTP Clients (FileZilla / WinSCP) / Partners** | **Method 3: In-Cluster SFTP** | Medium (Secret + Service) | ⭐⭐⭐⭐ Traditional SFTP drag-and-drop |
| **Developers / Admins with `oc` CLI Access** | **Method 4: `oc rsync` / `oc cp`** | Zero (Built-in CLI) | ⭐⭐⭐ Fast terminal commands, no manifests |

---

#### Network Exposure & Protocol Comparison

| Method | Client Protocol | Exposure Type | Needs LoadBalancer? | Works Over Ingress (Port 443)? | Ideal Use Case |
| :--- | :--- | :--- | :---: | :---: | :--- |
| **FileBrowser** | HTTPS | OpenShift Route | ❌ No | ✅ **Yes** | Business users, teams, document sharing |
| **Nginx Autoindex** | HTTPS | OpenShift Route | ❌ No | ✅ **Yes** | Public downloads, CI/CD, script automation |
| **WebDAV Gateway** | WebDAV (HTTPS) | OpenShift Route | ❌ No | ✅ **Yes** | WinSCP users restricted to Port 443 |
| **SFTP (NodePort)** | SFTP / SSH | NodePort (`32222`) | ❌ **No** | ❌ No | On-premises bare metal without cloud LB |
| **SFTP (LoadBalancer)** | SFTP / SSH | LoadBalancer (`22`) | ✅ Yes | ❌ No | Cloud clusters (AWS/Azure/GCP) or MetalLB |
| **SFTP (Port-Forward)** | SFTP / SSH | `oc port-forward` (`2222`) | ❌ **No** | ❌ No | Developer laptop, ad-hoc secure debugging |
| **oc rsync / oc cp** | OpenShift API | Native CLI | ❌ **No** | ❌ No | Developers/DevOps with cluster access |

---

## 14. Modern Object Storage Gateway: Using RustFS with OpenShift PVCs

A modern, cloud-native alternative to traditional file-sharing protocols (SFTP, NFS, and WebDAV) is deploying an **S3-compatible Object Storage Gateway** directly on top of your OpenShift storage.

**RustFS** (<https://github.com/rustfs/rustfs>) has emerged as an open-source, high-performance distributed object storage engine written in **Rust**. It serves as a modern, memory-safe, and permissively licensed alternative to legacy object gateways like MinIO and Ceph RGW.

---

### Why RustFS for This Use Case?

When users ask: *"How do I let people download files from an OpenShift PVC without wrestling with SFTP ports or complex load balancers?"*, RustFS provides an elegant answer:

1. **WinSCP Connects Natively via OpenShift Ingress (Port 443)**:
   WinSCP includes built-in support for the **Amazon S3 protocol**. Because S3 is pure HTTP/HTTPS, you can expose RustFS through a standard OpenShift **Route** on port 443. WinSCP users can browse and download files over standard corporate HTTPS—**no LoadBalancer and no NodePort required**.
2. **Built-in Web Console (Port 9001)**:
   RustFS includes an embedded web management console. By exposing it via an OpenShift Route, non-technical users can log in from Chrome or Firefox, browse buckets, preview files, and download data with one click.
3. **Pre-Signed Download URLs**:
   Backend applications can generate temporary, signed HTTPS download links (e.g., valid for 60 minutes). Users can download files directly from any browser or email link without needing user accounts or credentials.
4. **Apache 2.0 License (Enterprise Friendly)**:
   While MinIO shifted to the restrictive GNU **AGPLv3** license (which triggers compliance and legal concerns for many enterprises), RustFS is released under the permissive **Apache 2.0** license.
5. **Zero Garbage Collection & Ultra-Low Memory**:
   Because it is written in Rust rather than Go or Java, RustFS has zero runtime garbage collection pauses, predictable sub-millisecond latency for small objects, and consumes approximately 1/5th the RAM of comparable storage daemons.

---

### Is RustFS a StorageClass or an Application? How Do You Actually Use It?

A critical conceptual distinction in Kubernetes and OpenShift:

> **RustFS is NOT a StorageClass.**  
> It is an **Application Deployment** (a storage server software), exactly like **MinIO**, **PostgreSQL**, or **RabbitMQ**.

---

#### The Core Difference: StorageClass (POSIX) vs. RustFS (S3 API)

* A **StorageClass** (`ocs-storagecluster-ceph-rbd`, `ocs-storagecluster-cephfs`, `nfs-client`) is a cluster-level Kubernetes infrastructure plugin backed by a **CSI Driver**. It allows pods to declare a `PersistentVolumeClaim` (PVC) and mount a directory (e.g. `/var/data`) directly into a container. Applications interact with it using standard Linux **POSIX file calls** (`open`, `read`, `write`, `close`, `mkdir`).
* **RustFS** is a containerized **application server**. It runs *inside* your cluster as a `Deployment` or `StatefulSet`. It **consumes** existing StorageClasses (like Ceph RBD or CephFS) to persist its own data, and exposes an **HTTP/HTTPS S3 API endpoint** to the rest of the world.

---

#### How Would You Actually Use RustFS? (The 3 Usage Models)

##### Model 1: Cloud-Native Applications (No Volume Mounts Required!)

In modern microservice architectures, application pods **do not mount PVCs at all**. Mounting shared filesystems introduces file-locking bugs, slow pod startup times, and tight coupling to specific nodes.

Instead, applications interact with RustFS over the internal network using standard AWS SDKs:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: invoice-generator
  namespace: my-project
spec:
  replicas: 10 # Scale effortlessly from 1 to 100 replicas!
  template:
    spec:
      containers:
        - name: app
          image: my-invoice-app:latest
          env:
            # Point app to in-cluster RustFS internal service
            - name: AWS_ENDPOINT_URL
              value: "http://rustfs-service.my-project.svc:9000"
            - name: AWS_ACCESS_KEY_ID
              valueFrom:
                secretKeyRef:
                  name: rustfs-credentials
                  key: RUSTFS_ACCESS_KEY
            - name: AWS_SECRET_ACCESS_KEY
              valueFrom:
                secretKeyRef:
                  name: rustfs-credentials
                  key: RUSTFS_SECRET_KEY
          # NOTICE: NO volumeMounts or persistentVolumeClaims needed!
```

**Inside your application code (e.g. Python / Node.js / Java / Go)**:

```python
import boto3
import os

# Connect to internal RustFS
s3 = boto3.client(
    's3',
    endpoint_url=os.environ['AWS_ENDPOINT_URL'],
    aws_access_key_id=os.environ['AWS_ACCESS_KEY_ID'],
    aws_secret_access_key=os.environ['AWS_SECRET_ACCESS_KEY']
)

# Upload invoice directly from memory
s3.put_object(Bucket='invoices', Key='inv-1001.pdf', Body=pdf_bytes)

# Read file directly
response = s3.get_object(Bucket='invoices', Key='inv-1001.pdf')
data = response['Body'].read()
```

* **Why this is powerful**: Your application pods are **100% stateless**. They boot up in milliseconds, scale horizontally without POSIX lock contention, and can be scheduled on any worker node without storage affinity constraints.

---

##### Model 2: External Users & File Transfer Tools (WinSCP, Web Browser)

* **WinSCP Users**: Connect using the **Amazon S3** protocol via the OpenShift Route on standard HTTPS (Port 443).
* **Web Browser Users**: Navigate to the RustFS Web Console Route on standard HTTPS (Port 443) and click **Download**.
* **External Clients**: Receive temporary pre-signed HTTP download URLs.

---

##### Model 3: Can You Create a PVC Based on RustFS? (The S3-CSI Approach)

A frequent follow-up question: **"Can I create a `StorageClass` based on RustFS so applications can use a standard `PersistentVolumeClaim` (PVC) and mount it as a folder, rather than having to modify code to use S3 APIs?"**

**YES, you can.**

Kubernetes allows you to bridge any S3-compatible object storage (including RustFS) into a standard Kubernetes `StorageClass` and PVC using an **S3 Container Storage Interface (CSI) driver** (such as **`csi-s3`**).

```text
 ┌────────────────────────────────────────────────────────────────────────┐
 │                           OpenShift Cluster                            │
 │                                                                        │
 │   ┌───────────────────────────┐        ┌───────────────────────────┐   │
 │   │   Application Pod A       │        │   Application Pod B       │   │
 │   │  Mount: /var/data (POSIX) │        │  Mount: /var/data (POSIX) │   │
 │   └─────────────┬─────────────┘        └─────────────┬─────────────┘   │
 │                 │                                    │                 │
 │                 ▼                                    ▼                 │
 │   ┌────────────────────────────────────────────────────────────────┐   │
 │   │             PersistentVolumeClaim (RWX)                        │   │
 │   │               storageClassName: rustfs-s3                      │   │
 │   └───────────────────────────────┬────────────────────────────────┘   │
 │                                   │                                    │
 │                                   ▼                                    │
 │   ┌────────────────────────────────────────────────────────────────┐   │
 │   │                   CSI-S3 Driver (FUSE Engine)                  │   │
 │   │   Translates POSIX read/write calls into S3 GET/PUT requests   │   │
 │   └───────────────────────────────┬────────────────────────────────┘   │
 │                                   │ Internal HTTP API                  │
 │                                   ▼ (Port 9000)                        │
 │   ┌────────────────────────────────────────────────────────────────┐   │
 │   │                      RustFS Service / Pod                      │   │
 │   └───────────────────────────────┬────────────────────────────────┘   │
 │                                   │ Writes raw blocks                  │
 │                                   ▼                                    │
 │   ┌────────────────────────────────────────────────────────────────┐   │
 │   │              Physical Storage Disk (Ceph RBD / SSD)            │   │
 │   └────────────────────────────────────────────────────────────────┘   │
 └────────────────────────────────────────────────────────────────────────┘
```

###### 1. Step 1: Create the CSI Secret with RustFS Endpoint

Create a Secret containing the internal cluster endpoint of RustFS and your credentials:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: csi-rustfs-secret
  namespace: kube-system
type: Opaque
stringData:
  accessKeyID: "rustfsadmin"
  secretAccessKey: "SuperSecureKey2026!"
  endpoint: "http://rustfs-service.my-project.svc:9000"
  region: "us-east-1"
```

###### 2. Step 2: Define the `StorageClass`

The `StorageClass` tells the CSI driver how to mount RustFS buckets using a FUSE engine (`geesefs` or `s3fs`):

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: rustfs-s3
provisioner: ru.yandex.s3.csi
parameters:
  mounter: geesefs # High-performance S3 FUSE mounter
  csi.storage.k8s.io/provisioner-secret-name: csi-rustfs-secret
  csi.storage.k8s.io/provisioner-secret-namespace: kube-system
  csi.storage.k8s.io/controller-publish-secret-name: csi-rustfs-secret
  csi.storage.k8s.io/controller-publish-secret-namespace: kube-system
  csi.storage.k8s.io/node-stage-secret-name: csi-rustfs-secret
  csi.storage.k8s.io/node-stage-secret-namespace: kube-system
  csi.storage.k8s.io/node-publish-secret-name: csi-rustfs-secret
  csi.storage.k8s.io/node-publish-secret-namespace: kube-system
```

###### 3. Step 3: Request Storage via PVC

Now, your workloads can request storage without knowing anything about S3 APIs:

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: rustfs-pvc
  namespace: my-project
spec:
  accessModes:
    - ReadWriteMany # S3 provides RWX across all cluster nodes!
  storageClassName: rustfs-s3
  resources:
    requests:
      storage: 50Gi
```

###### 4. Step 4: Mount into Application Pod

The application pod mounts the volume just like any traditional filesystem:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: legacy-app
  namespace: my-project
spec:
  containers:
    - name: app
      image: registry.access.redhat.com/ubi9/ubi
      command: ["sh", "-c", "echo 'hello from pod' >> /data/test.txt && sleep 3600"]
      volumeMounts:
        - name: shared-rustfs
          mountPath: /data
  volumes:
    - name: shared-rustfs
      persistentVolumeClaim:
        claimName: rustfs-pvc
```

* **What happens**: The container writes to `/data/test.txt` as a standard local file. The `csi-s3` driver intercepts the call, wraps it in an S3 HTTP PUT request, and sends it to RustFS.
* **Simultaneous External Access**: At the exact same second, an external user connecting via **WinSCP (Amazon S3 protocol)** or the **RustFS Web Console** can view and download `test.txt`!

---

###### Critical Trade-offs: Why Native CephFS is Usually Better for POSIX PVCs

While creating a PVC backed by RustFS works, you must weigh the architectural realities of **emulated POSIX over Object Storage**:

1. **Emulated POSIX vs. True Filesystem**:
   * S3 has no concept of directories or byte offsets—only immutable object keys (`prefix/file.txt`).
   * Renaming a folder in an S3 FUSE mount forces the driver to copy every single object under that prefix and delete the old ones.
   * Appending data (`>>`) requires re-uploading the entire object payload.
2. **File Locking (`fcntl` / `flock`)**:
   * FUSE S3 drivers cannot reliably support strict POSIX file locks. Databases (PostgreSQL, SQLite, MySQL) will crash or corrupt data if run over an S3-backed PVC.
3. **OpenShift Security Context Constraints (SCC)**:
   * FUSE drivers require access to the Linux kernel device `/dev/fuse`. In OpenShift, this requires running with elevated permissions (a custom SCC with `allowHostDirVolumePlugin` or `privileged`).
4. **Latency Overhead**:
   * Every file write passes through Linux VFS ➔ FUSE daemon ➔ HTTP serialization ➔ RustFS ➔ Physical disk. Native CephFS kernel mounts are dramatically faster for POSIX workloads.

---

###### Architectural Verdict

* **If your application needs POSIX volume mounts (`/var/data`)**:  
  Use native OpenShift **CephFS (`ocs-storagecluster-cephfs`)**. It is built into OpenShift, fully supported by Red Hat, kernel-accelerated, and 100% POSIX compliant.
* **If your goal is S3 APIs, WinSCP over port 443, and browser downloads**:  
  Deploy **RustFS directly as an application gateway** on top of your CephFS PVC. This gives you the speed of CephFS for your apps, while giving users WinSCP and Web Console access over HTTPS!

---

#### Comparison Table: StorageClass vs. RustFS

| Feature | Kubernetes StorageClass (e.g., CephFS / RBD) | RustFS (Object Storage Gateway) |
| :--- | :--- | :--- |
| **What is it?** | Infrastructure CSI driver for disk/volume provisioning | Application container serving an S3 API |
| **How Pods Connect** | Container volume mount (`volumeMounts: /data`) | Network HTTP/HTTPS API (`http://service:9000`) |
| **API Protocol** | POSIX system calls (`open`, `read`, `write`) | AWS S3 REST API (`GET`, `PUT`, `DELETE`) |
| **Pod Architecture** | Stateful (tied to volume mount & node CSI) | **Stateless** (scale from 1 to 50 pods instantly) |
| **File Locking** | Prone to POSIX locking conflicts across multiple pods | **Lock-free** atomic object versioning |
| **WinSCP Access** | Requires NodePort / LoadBalancer for SFTP | **Native Amazon S3 via OpenShift Route (Port 443)** |
| **Browser Access** | Requires separate gateway (FileBrowser / Nginx) | **Built-in Web Console & Pre-signed URLs** |

---

### Is This Available Out-of-the-Box in OpenShift? (Native ODF vs. Third-Party)

A critical practical question: **"Are RustFS and S3-CSI built into OpenShift by default, or do they require custom setup?"**

Here is the exact support and availability breakdown:

---

#### 1. What is NOT Built-In (Third-Party / Custom Setup Required)

* **`csi-s3` (S3 CSI Driver)**:
  * ❌ **Not built-in**. Red Hat does not ship an S3-based CSI driver with OpenShift.
  * To use `csi-s3`, cluster administrators must manually install third-party Helm charts and grant elevated permissions (`/dev/fuse` device access) via a custom Security Context Constraint (SCC).
* **RustFS**:
  * ❌ **Not built-in**. RustFS is an open-source project from the community.
  * However, **running RustFS is simple**: because it is a standard container image (`docker.io/rustfs/rustfs`), you can deploy it in 30 seconds into any OpenShift project without cluster-admin permissions using the Deployment manifest provided in [Mode 1](#mode-1-standalone-rustfs-gateway-on-an-existing-pvc).

---

#### 2. What IS Built-In to OpenShift Out-of-the-Box (Red Hat Supported)

If your cluster has **OpenShift Data Foundation (ODF)** installed, you already have enterprise-grade, supported solutions for both POSIX file storage and S3 object storage:

| Storage Type | Native OpenShift (ODF) Solution | How Applications Use It |
| :--- | :--- | :--- |
| **Shared POSIX Filesystem (RWX)** | **CephFS (`ocs-storagecluster-cephfs`)** | Standard `PersistentVolumeClaim` (PVC) |
| **High-Performance Block (RWO)** | **Ceph RBD (`ocs-storagecluster-ceph-rbd`)** | Standard `PersistentVolumeClaim` (PVC) |
| **Native S3 Object Storage** | **NooBaa / Multicloud Object Gateway (MCG)** | **`ObjectBucketClaim` (OBC)** |

---

#### The Red Hat Native S3 Approach: `ObjectBucketClaim` (OBC)

If you want official, out-of-the-box S3 object storage without installing RustFS or third-party CSI drivers, OpenShift provides **`ObjectBucketClaim` (OBC)**:

```yaml
apiVersion: objectbucket.io/v1alpha1
kind: ObjectBucketClaim
metadata:
  name: my-app-bucket
  namespace: my-project
spec:
  # Native ODF Object StorageClass
  storageClassName: openshift-storage.noobaa.io
  generateBucketName: company-reports
```

1. **What OpenShift Does Automatically**:
   * Creates an S3 bucket in the cluster's internal NooBaa / Ceph RGW object store.
   * Generates a **ConfigMap** (`my-app-bucket`) containing the internal S3 endpoint URL and bucket name.
   * Generates a **Secret** (`my-app-bucket`) containing the generated `AWS_ACCESS_KEY_ID` and `AWS_SECRET_ACCESS_KEY`.
2. **WinSCP & Browser Access with Native ODF**:
   * ODF creates a public HTTPS Route on port 443 (e.g., `s3-openshift-storage.apps.mycluster.com`).
   * WinSCP can connect directly using the **Amazon S3 protocol** over port 443 with the credentials from the OBC Secret—exactly like RustFS!

---

#### Decision Guide: Native ODF vs. RustFS Gateway

* **Use Native OpenShift ODF (CephFS + NooBaa OBC)** when:
  * You require full enterprise Red Hat commercial support and SLA.
  * You already have ODF installed and want zero third-party software.
* **Use RustFS as an In-Cluster Gateway** when:
  * You already have an existing CephFS or Ceph RBD volume with files, and you simply need a lightweight, memory-efficient S3 translation layer so people can use **WinSCP or the Web Console over Port 443**.
  * Your cluster does not have NooBaa / ODF Object Storage enabled, and you want an instant, Apache 2.0 S3 server running in your namespace without asking cluster admins for licenses.

---

### RustFS Architecture on OpenShift Storage

RustFS can be deployed directly inside OpenShift in two primary patterns:

```text
  ┌────────────────────────────────────────────────────────────────────────┐
  │                           OpenShift Cluster                            │
  │                                                                        │
  │   ┌───────────────────────────┐       ┌───────────────────────────┐    │
  │   │   CephFS Shared PVC       │  OR   │   Ceph RBD Block PVC      │    │
  │   │  (ocs-storagecluster-     │       │  (ocs-storagecluster-     │    │
  │   │   cephfs / RWX)           │       │   ceph-rbd / RWO)         │    │
  │   └─────────────┬─────────────┘       └─────────────┬─────────────┘    │
  │                 │                                   │                  │
  │                 └─────────────────┬─────────────────┘                  │
  │                                   ▼ Mounts to /data                    │
  │                   ┌───────────────────────────────┐                    │
  │                   │         RustFS Pod            │                    │
  │                   │                               │                    │
  │                   │  - Port 9000: S3 API Engine   │                    │
  │                   │  - Port 9001: Web UI Console  │                    │
  │                   └───────┬───────────────┬───────┘                    │
  │                           │               │                            │
  │               Port 9000   ▼               ▼   Port 9001                │
  │   ┌──────────────────────────┐         ┌──────────────────────────┐    │
  │   │   Route: rustfs-s3       │         │  Route: rustfs-console   │    │
  │   │ (HTTPS Edge Port 443)    │         │ (HTTPS Edge Port 443)    │    │
  │   └─────────────┬────────────┘         └────────────┬─────────────┘    │
  └─────────────────┼───────────────────────────────────┼──────────────────┘
                    │                                   │
                    ▼                                   ▼
        [WinSCP via Amazon S3]                 [Web Browser Users]
        [AWS CLI / SDKs / Boto3]               (One-Click File Downloads)
```

---

### Mode 1: Standalone RustFS Gateway on an Existing PVC

In this architecture, RustFS acts as an S3 frontend sitting on top of your existing CephFS or Ceph RBD PVC. All objects written via S3 are stored directly in your OpenShift persistent volume.

#### Complete OpenShift Manifest (`rustfs-gateway.yaml`)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: rustfs-credentials
  namespace: my-project
type: Opaque
stringData:
  # Configure strong S3 credentials
  RUSTFS_ACCESS_KEY: "rustfsadmin"
  RUSTFS_SECRET_KEY: "SuperSecureKey2026!"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rustfs-gateway
  namespace: my-project
  labels:
    app: rustfs
spec:
  replicas: 1
  strategy:
    type: Recreate
  selector:
    matchLabels:
      app: rustfs
  template:
    metadata:
      labels:
        app: rustfs
    spec:
      containers:
        - name: rustfs
          image: docker.io/rustfs/rustfs:latest
          imagePullPolicy: IfNotPresent
          envFrom:
            - secretRef:
                name: rustfs-credentials
          env:
            # S3 API listener
            - name: RUSTFS_ADDRESS
              value: ":9000"
            # Embedded Web Console listener
            - name: RUSTFS_CONSOLE_ADDRESS
              value: ":9001"
          ports:
            - name: s3-api
              containerPort: 9000
            - name: console
              containerPort: 9001
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 1000m
              memory: 1Gi
          volumeMounts:
            # Mount your existing CephFS or Ceph RBD volume
            - name: storage-data
              mountPath: /data
      volumes:
        - name: storage-data
          persistentVolumeClaim:
            claimName: <your-existing-pvc-name>
---
apiVersion: v1
kind: Service
metadata:
  name: rustfs-service
  namespace: my-project
spec:
  selector:
    app: rustfs
  ports:
    - name: s3-api
      port: 9000
      targetPort: 9000
    - name: console
      port: 9001
      targetPort: 9001
---
# Route 1: Expose S3 API for WinSCP, AWS CLI, and SDKs
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: rustfs-s3
  namespace: my-project
spec:
  to:
    kind: Service
    name: rustfs-service
  port:
    targetPort: s3-api
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
---
# Route 2: Expose Web Console for End-User Browser Downloads
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: rustfs-console
  namespace: my-project
spec:
  to:
    kind: Service
    name: rustfs-service
  port:
    targetPort: console
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect
```

---

### How to Access & Download Files via RustFS

#### 1. WinSCP via Native Amazon S3 Protocol (100% via Ingress Port 443)

WinSCP natively connects to any S3-compatible storage endpoint without third-party plugins.

1. Retrieve your S3 API Route URL:

   ```bash
   oc get route rustfs-s3 -n my-project -o jsonpath='{.spec.host}{"\n"}'
   # Example: rustfs-s3-my-project.apps.mycluster.com
   ```

2. Open WinSCP and configure the login dialog:
   * **File protocol**: Select **`Amazon S3`** from the dropdown menu.
   * **Host name**: Enter the Route hostname (e.g., `rustfs-s3-my-project.apps.mycluster.com`).
   * **Port number**: `443` *(Standard HTTPS port)*.
   * **Access key ID**: `rustfsadmin` (from Secret).
   * **Secret key**: `SuperSecureKey2026!` (from Secret).

3. Click **Login**.
   * WinSCP connects over standard port 443 HTTPS.
   * Users can browse buckets and drag-and-drop files to download or upload, identical to SFTP.

---

#### 2. Web Browser Downloads via RustFS Console

For users who do not have WinSCP installed:

1. Retrieve the Console Route URL:

   ```bash
   oc get route rustfs-console -n my-project -o jsonpath='{"https://"}{.spec.host}{"\n"}'
   ```

2. Open the URL in Google Chrome, Firefox, or Edge.
3. Log in with your `RUSTFS_ACCESS_KEY` and `RUSTFS_SECRET_KEY`.
4. Browse buckets, view object metadata, and click **Download** on any file.

---

#### 3. Programmatic Pre-Signed URLs (Time-Limited Direct Links)

Applications running in OpenShift can generate secure, temporary download URLs so external clients can download files using a web browser without needing credentials:

```python
import boto3
from botocore.config import Config

# Initialize S3 client pointing to in-cluster RustFS
s3_client = boto3.client(
    's3',
    endpoint_url='http://rustfs-service.my-project.svc:9000',
    aws_access_key_id='rustfsadmin',
    aws_secret_access_key='SuperSecureKey2026!',
    config=Config(signature_version='s3v4')
)

# Generate a pre-signed GET URL valid for 1 hour (3600 seconds)
download_url = s3_client.generate_presigned_url(
    'get_object',
    Params={'Bucket': 'reports', 'Key': 'monthly-export-2026.csv'},
    ExpiresIn=3600
)

# Replace internal service hostname with public Route hostname
public_url = download_url.replace(
    'http://rustfs-service.my-project.svc:9000',
    'https://rustfs-s3-my-project.apps.mycluster.com'
)

print(f"Share this link with users to download: {public_url}")
```

Users can paste that link into their browser or click it in an email, and the file downloads immediately over HTTPS.

---

#### 4. Command Line & Automation (`aws-cli` / `rclone`)

Developers and automated pipelines can use standard S3 tools:

```bash
# Configure AWS CLI endpoint
export AWS_ACCESS_KEY_ID="rustfsadmin"
export AWS_SECRET_ACCESS_KEY="SuperSecureKey2026!"

# List buckets
aws --endpoint-url https://rustfs-s3-my-project.apps.mycluster.com s3 ls

# Download a file
aws --endpoint-url https://rustfs-s3-my-project.apps.mycluster.com s3 cp s3://reports/data.csv ./data.csv

# Fast multi-threaded directory sync using rclone
rclone sync :s3,endpoint=https://rustfs-s3-my-project.apps.mycluster.com:reports ./local-reports
```

---

### Mode 2: Distributed High-Performance RustFS Cluster (StatefulSet + Ceph RBD)

For high-throughput AI model serving, distributed caching, or data lake ingestion, RustFS can be deployed as a **distributed cluster** across multiple OpenShift nodes.

In this architecture:

* Each RustFS replica pod runs on a different worker node.
* Each replica mounts its own dedicated, high-speed **Ceph RBD block volume** (`volumeMode: Filesystem`, `ReadWriteOnce`).
* RustFS nodes communicate over internal port 9000 using erasure coding (EC) to provide unified S3 storage that survives node failures without relying on POSIX file locks.

```text
 ┌────────────────────────────────────────────────────────────────────────┐
 │                   Distributed RustFS Storage Cluster                   │
 │                                                                        │
 │  ┌──────────────────────┐┌──────────────────────┐┌──────────────────┐  │
 │  │   RustFS Pod 0       ││   RustFS Pod 1       ││  RustFS Pod 2    │  │
 │  │   (Worker Node 1)    ││   (Worker Node 2)    ││  (Worker Node 3) │  │
 │  │                      ││                      ││                  │  │
 │  │  Mount: /data        ││  Mount: /data        ││  Mount: /data    │  │
 │  │  PVC: ceph-rbd-0     ││  PVC: ceph-rbd-1     ││  PVC: ceph-rbd-2 │  │
 │  │  (RWO Block Storage) ││  (RWO Block Storage) ││  (RWO Block)     │  │
 │  └──────────┬───────────┘└──────────┬───────────┘└─────────┬────────┘  │
 │             │                       │                      │           │
 │             └───────────────────────┼──────────────────────┘           │
 │                                     ▼                                  │
 │                      Internal S3 Erasure-Coded Mesh                    │
 └─────────────────────────────────────┬──────────────────────────────────┘
                                       ▼
                       Unified High-Availability S3 API
```

---

### Architectural Comparison: Traditional Protocols vs. RustFS S3

| Feature | In-Cluster SFTP | In-Cluster WebDAV | In-Cluster FileBrowser | RustFS (S3 Gateway) |
| :--- | :--- | :--- | :--- | :--- |
| **Primary Protocol** | SSH / SFTP (Layer 4) | WebDAV / HTTP (Layer 7) | Web GUI / HTTP | **S3 API / HTTP (Layer 7)** |
| **Ingress Friendly (Port 443)?** | ❌ No (Requires NodePort / LB) | ✅ Yes (OpenShift Route) | ✅ Yes (OpenShift Route) | ✅ **Yes (OpenShift Route)** |
| **WinSCP Support?** | ✅ Native (SFTP) | ✅ Native (WebDAV) | ❌ No (Browser Only) | ✅ **Native (Amazon S3)** |
| **Web Browser Support?** | ❌ No (Requires FTP app) | ⚠️ Primitive (basic auth) | ✅ Rich Web Explorer | ✅ **Embedded Web Console** |
| **Pre-Signed URLs?** | ❌ No | ❌ No | ⚠️ Public share links | ✅ **Standard S3 Signatures** |
| **Language & Engine** | OpenSSH (C) | Apache / Nginx (C) | Go | **Rust (Zero-GC, memory safe)** |
| **Open Source License** | OpenSSH BSD | Apache 2.0 / BSD | AGPLv3 | **Apache 2.0** |
| **Backend Storage** | CephFS (RWX) | CephFS (RWX) | CephFS (RWX) | **CephFS (RWX) or Ceph RBD (RWO)** |
| **Throughput / Concurrency** | Medium | Medium | Medium | **High (Async Rust, Multi-core)** |
