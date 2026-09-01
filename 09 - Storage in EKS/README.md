# Module 9: Storage in EKS

## Overview

Every workload in the previous modules ran with ephemeral, node-local storage: whatever a pod wrote to disk disappeared the moment it was rescheduled. This module extends the cluster with real, persistent storage by installing AWS's **Container Storage Interface (CSI)** drivers for EBS, EFS, and S3 — three very different storage backends, each solving a different problem (single-pod fast disks, multi-pod shared file systems, and object storage mounted as a volume).

Every driver in this module is installed the same way: as an **EKS Addon**, authenticated through **Pod Identity** instead of IRSA. Pod Identity is introduced here as a simpler alternative to the OIDC-federated role/service-account annotation dance used everywhere so far — it attaches an IAM role to a service account directly, through a single `aws_eks_pod_identity_association` resource, with no cluster OIDC provider involved.

One limitation carries through the whole module: some of what's covered here does not work on Fargate profiles, so this module's cluster runs on managed node groups only.

## Table of Contents

- [Lesson 1: Understanding CSI and the Storage Options in EKS](#lesson-1-understanding-csi-and-the-storage-options-in-eks)
  - [1. What Is CSI (Container Storage Interface)](#1-what-is-csi-container-storage-interface)
  - [2. Three AWS Storage Backends: EBS, EFS, and S3](#2-three-aws-storage-backends-ebs-efs-and-s3)
  - [3. Introducing Pod Identity as an Alternative to IRSA](#3-introducing-pod-identity-as-an-alternative-to-irsa)
  - [4. Installation Strategy: EKS Addons for Everything](#4-installation-strategy-eks-addons-for-everything)
  - [Key Takeaways](#key-takeaways)
- [Lesson 2: Deploying the Pod Identity Agent](#lesson-2-deploying-the-pod-identity-agent)
  - [1. IRSA vs Pod Identity](#1-irsa-vs-pod-identity)
  - [2. Checking the Latest Addon Version](#2-checking-the-latest-addon-version)
  - [3. Declaring the Addon Version Variable](#3-declaring-the-addon-version-variable)
  - [4. Creating the Pod Identity Agent Addon](#4-creating-the-pod-identity-agent-addon)
  - [5. A Fargate Limitation Surfaces](#5-a-fargate-limitation-surfaces)
  - [6. Verifying the DaemonSet](#6-verifying-the-daemonset)
  - [Key Takeaways](#key-takeaways-1)
- [Lesson 3: Installing the EBS CSI Driver](#lesson-3-installing-the-ebs-csi-driver)
  - [1. Attaching the AWS-Managed EBS CSI Policy to the Node Role](#1-attaching-the-aws-managed-ebs-csi-policy-to-the-node-role)
  - [2. Creating the Pod Identity IAM Role for EBS CSI](#2-creating-the-pod-identity-iam-role-for-ebs-csi)
  - [3. Checking the Addon Version and Declaring the Variable](#3-checking-the-addon-version-and-declaring-the-variable)
  - [4. Creating the EBS CSI Driver Addon](#4-creating-the-ebs-csi-driver-addon)
  - [5. Creating the Pod Identity Association](#5-creating-the-pod-identity-association)
  - [6. Verifying the Installation](#6-verifying-the-installation)
  - [Key Takeaways](#key-takeaways-2)
- [Lesson 4: Static EBS Provisioning with a PersistentVolumeClaim](#lesson-4-static-ebs-provisioning-with-a-persistentvolumeclaim)
  - [1. The Lab Deployment: chip-ebs.yml](#1-the-lab-deployment-chip-ebsyml)
  - [2. Applying the Manifest and Watching the Volume Provision](#2-applying-the-manifest-and-watching-the-volume-provision)
  - [3. Testing Filesystem Persistence Through Chip's API](#3-testing-filesystem-persistence-through-chips-api)
  - [4. Proving the Volume Survives Pod Rescheduling](#4-proving-the-volume-survives-pod-rescheduling)
  - [Key Takeaways](#key-takeaways-3)
- [Lesson 5: Dynamic Provisioning with StatefulSets and VolumeClaimTemplates](#lesson-5-dynamic-provisioning-with-statefulsets-and-volumeclaimtemplates)
  - [1. Cleaning Up the Static Test](#1-cleaning-up-the-static-test)
  - [2. From PersistentVolumeClaim to volumeClaimTemplates](#2-from-persistentvolumeclaim-to-volumeclaimtemplates)
  - [3. Applying the StatefulSet and Watching Volumes Provision One by One](#3-applying-the-statefulset-and-watching-volumes-provision-one-by-one)
  - [4. EBS Volumes Are Not Shared Between Pods](#4-ebs-volumes-are-not-shared-between-pods)
  - [5. Scaling Up: One New Volume Per New Replica](#5-scaling-up-one-new-volume-per-new-replica)
  - [Key Takeaways](#key-takeaways-4)
- [Lesson 6: Upgrading to GP3 Storage Classes](#lesson-6-upgrading-to-gp3-storage-classes)
  - [1. Creating the GP3 StorageClass Manifest](#1-creating-the-gp3-storageclass-manifest)
  - [2. Provisioning It Through Terraform Instead](#2-provisioning-it-through-terraform-instead)
  - [3. Switching the StatefulSet to GP3](#3-switching-the-statefulset-to-gp3)
  - [4. Confirming the New Volumes Are GP3](#4-confirming-the-new-volumes-are-gp3)
  - [Key Takeaways](#key-takeaways-5)
- [Lesson 7: Installing the EFS CSI Driver](#lesson-7-installing-the-efs-csi-driver)
  - [1. When to Reach for EFS Instead of EBS](#1-when-to-reach-for-efs-instead-of-ebs)
  - [2. Checking the Addon Version and Declaring the Variable](#2-checking-the-addon-version-and-declaring-the-variable)
  - [3. Creating the Pod Identity IAM Role for EFS CSI](#3-creating-the-pod-identity-iam-role-for-efs-csi)
  - [4. Creating the EFS CSI Driver Addon](#4-creating-the-efs-csi-driver-addon)
  - [5. Verifying the Installation](#5-verifying-the-installation)
  - [Key Takeaways](#key-takeaways-6)
- [Lesson 8: Provisioning and Testing a Shared EFS Volume](#lesson-8-provisioning-and-testing-a-shared-efs-volume)
  - [1. Creating the EFS File System and Security Group](#1-creating-the-efs-file-system-and-security-group)
  - [2. Mounting the File System in the Pod Subnets](#2-mounting-the-file-system-in-the-pod-subnets)
  - [3. Creating the EFS StorageClass and PersistentVolumeClaim](#3-creating-the-efs-storageclass-and-persistentvolumeclaim)
  - [4. Deploying Three Replicas Sharing One Volume](#4-deploying-three-replicas-sharing-one-volume)
  - [5. Proving the Shared Write/Read Across Pods](#5-proving-the-shared-writeread-across-pods)
  - [6. Volumes Survive Pod Deletion Too](#6-volumes-survive-pod-deletion-too)
  - [Key Takeaways](#key-takeaways-7)
- [Lesson 9: Installing the S3 CSI Driver](#lesson-9-installing-the-s3-csi-driver)
  - [1. Mounting S3 Buckets as Volumes: Eventual Consistency](#1-mounting-s3-buckets-as-volumes-eventual-consistency)
  - [2. Checking the Addon Version and Declaring the Variable](#2-checking-the-addon-version-and-declaring-the-variable-1)
  - [3. Creating the Pod Identity IAM Role for S3 CSI](#3-creating-the-pod-identity-iam-role-for-s3-csi)
  - [4. Creating the S3 CSI Driver Addon](#4-creating-the-s3-csi-driver-addon)
  - [5. Verifying the Installation](#5-verifying-the-installation-1)
  - [Key Takeaways](#key-takeaways-8)
- [Lesson 10: Mounting an S3 Bucket as a Volume](#lesson-10-mounting-an-s3-bucket-as-a-volume)
  - [1. Provisioning the Bucket](#1-provisioning-the-bucket)
  - [2. A Static PersistentVolume/PersistentVolumeClaim Pair for the Bucket](#2-a-static-persistentvolumepersistentvolumeclaim-pair-for-the-bucket)
  - [3. Deploying Three Replicas Sharing the Bucket](#3-deploying-three-replicas-sharing-the-bucket)
  - [4. Testing Bidirectional Sync Between Pods and the Bucket](#4-testing-bidirectional-sync-between-pods-and-the-bucket)
  - [5. Wrapping Up: Comparing EBS, EFS, and S3](#5-wrapping-up-comparing-ebs-efs-and-s3)
  - [Key Takeaways](#key-takeaways-9)

# Lesson 1: Understanding CSI and the Storage Options in EKS

## 1. What Is CSI (Container Storage Interface)

CSI is Kubernetes' own plugin standard for storage: a set of APIs that let a storage vendor write a driver Kubernetes can use to provision and attach volumes, without Kubernetes needing to know anything vendor-specific. AWS ships CSI drivers for EBS, EFS, and S3, each turning an AWS storage service into something a pod can mount as a regular volume through the usual `PersistentVolume`/`PersistentVolumeClaim` objects.

## 2. Three AWS Storage Backends: EBS, EFS, and S3

This module covers three backends, each with a different trade-off:

- **EBS (Elastic Block Store)** — the same block disks EC2 instances use. Fast, with strong IOPS options, but a volume can only be mounted by one pod at a time and is locked to a single Availability Zone. This is the most common choice, and the default one to reach for.
- **EFS (Elastic File System)** — a shared, elastic file system. Multiple pods, even across nodes and AZs, can mount and read/write the same volume at once, at some cost to raw throughput compared to EBS.
- **S3 (Simple Storage Service)** — the CSI driver mounts an S3 bucket as a volume. Writes to the mounted path become objects in the bucket and vice versa, but the mount is only eventually consistent.

## 3. Introducing Pod Identity as an Alternative to IRSA

Every IAM-authenticated workload so far has used IRSA: create a role, trust it to the cluster's OIDC provider, then annotate a service account to bind the two together. **Pod Identity** does the same job with one resource instead of that whole chain — an `aws_eks_pod_identity_association` attaches an IAM role straight to a `(namespace, service_account)` pair, with no OIDC federation and no annotation required.

The role's trust policy also looks different. Where IRSA trusts the cluster's OIDC provider ARN and uses `sts:AssumeRoleWithWebIdentity`, a Pod Identity role trusts the `pods.eks.amazonaws.com` service principal directly and grants `sts:AssumeRole` plus `sts:TagSession`:

```hcl
# illustrative shape — the real versions live in iam_ebs_csi.tf, iam_efs_csi.tf, and iam_s3_csi.tf
data "aws_iam_policy_document" "example_role" {
  statement {
    effect = "Allow"

    principals {
      type        = "Service"
      identifiers = ["pods.eks.amazonaws.com"]
    }

    actions = [
      "sts:AssumeRole",
      "sts:TagSession"
    ]
  }
}
```

Pod Identity is powered by its own EKS Addon, the **Pod Identity Agent**, which runs as a DaemonSet in `kube-system` and injects the right credentials into every pod whose service account has an association — the same role this module reaches for every time a CSI driver needs AWS permissions.

Pod Identity doesn't strictly belong to the CSI story — it's a general-purpose IAM mechanism, useful for any workload. It gets folded into this module instead of getting a lesson of its own for a practical reason: it's a genuinely small resource to set up, and every CSI driver installed from here on needs it anyway, so it makes more sense to introduce it right before its first real use than to dedicate a whole separate lesson to it.

## 4. Installation Strategy: EKS Addons for Everything

Every component in this module — the Pod Identity Agent and the EBS, EFS, and S3 CSI drivers — is installed as an **EKS Addon** rather than a standalone Helm release, following the same addon pattern already used for CoreDNS, kube-proxy, and the VPC CNI: check the available versions with `aws eks describe-addon-versions`, store the chosen version in a Terraform variable, and declare an `aws_eks_addon` resource.

The module follows a fixed order for a reason: Pod Identity first, since every driver after it depends on the mechanism it provides; then EBS, the most common and simplest case; then EFS, once the idea of "a volume" is already familiar and the lesson can focus on what's different about sharing it; and S3 last, since it reuses the same installation shape one more time while introducing its own quirks (eventual consistency, no dynamic provisioning). Within each backend, the pattern is also consistent: install and verify the driver first by hand, then provision a lab volume and prove it actually works — the same "do it manually, then formalize it" approach used for the GP3 `StorageClass` in Lesson 6.

## Key Takeaways

- CSI is Kubernetes' storage plugin standard; AWS provides CSI drivers for EBS, EFS, and S3, each solving a different storage problem.
- EBS: fast, single-pod, single-AZ block storage. EFS: shared, multi-pod file storage. S3: object storage mounted as a volume, eventually consistent.
- Pod Identity replaces IRSA's OIDC-federation-plus-annotation chain with a single `aws_eks_pod_identity_association` resource, trusting the `pods.eks.amazonaws.com` principal instead of the cluster's OIDC provider.
- Pod Identity is introduced here, ahead of any single backend, because every driver in this module depends on it — not because it's inherently tied to CSI.
- Every driver in this module installs as an EKS Addon, the same pattern already used for CoreDNS, kube-proxy, and the VPC CNI, and every backend follows the same manual-first, formalize-later teaching order.

# Lesson 2: Deploying the Pod Identity Agent

## 1. IRSA vs Pod Identity

To recap the contrast from Lesson 1: every IRSA setup so far has followed the same four steps — create a role with an `sts:AssumeRoleWithWebIdentity` trust policy, federate that trust to the cluster's OIDC provider (optionally scoped to a specific namespace/service account for extra strictness), attach the needed IAM policies to the role, and finally annotate the target service account with the role's ARN so pods launched under it pick up the identity automatically.

Pod Identity skips the federation and the annotation entirely: the role is attached straight to the service account through an association resource, and that's the whole binding. From this module on, IAM authentication for new components is done with Pod Identity instead of IRSA wherever possible — not as a hard cutover, but a gradual shift, dropping IRSA a little more with each new component this course adds.

## 2. Checking the Latest Addon Version

Same workflow used for every other addon in this course:

```bash
aws eks describe-addon-versions \
  --addon-name eks-pod-identity-agent \
  --kubernetes-version <K8S_VERSION>
```

## 3. Declaring the Addon Version Variable

```hcl
# variables.tf
variable "addon_pod_identity_version" {
  type        = string
  default     = "v1.3.4-eksbuild.1"
  description = "Versão do Addon do Pod Identity"
}
```

## 4. Creating the Pod Identity Agent Addon

The `aws_eks_addon` resource itself barely varies from one addon to the next — name, version variable, and `depends_on` are the only moving parts — so the fastest way to write a new one is to copy an existing block (kube-proxy's, in this case) and swap the two names:

```hcl
# addons.tf
resource "aws_eks_addon" "pod_identity" {
  cluster_name = aws_eks_cluster.main.name
  addon_name   = "eks-pod-identity-agent"

  addon_version               = var.addon_pod_identity_version
  resolve_conflicts_on_create = "OVERWRITE"
  resolve_conflicts_on_update = "OVERWRITE"

  depends_on = [
    aws_eks_access_entry.nodes
  ]
}
```

## 5. A Fargate Limitation Surfaces

Right before applying this addon is where the module's one caveat gets called out explicitly: some of what's coming in this module's CSI lessons does not work on Fargate profiles — CSI volume attachment depends on node-level components that Fargate's per-pod microVMs don't support the same way. This cluster runs on managed node groups only, so it isn't affected.

## 6. Verifying the DaemonSet

```bash
aws eks update-kubeconfig --name <CLUSTER_NAME>
kubectl get pods -n kube-system
```

The addon shows up as a new `eks-pod-identity-agent` DaemonSet in `kube-system`, running one pod per node. From here on, every CSI driver's IAM authentication depends on this DaemonSet being in place.

## Key Takeaways

- The Pod Identity Agent is itself an EKS Addon (`eks-pod-identity-agent`), deployed as a DaemonSet in `kube-system`.
- It must exist before any `aws_eks_pod_identity_association` resource can take effect — every CSI driver installed in this module depends on it.
- `aws_eks_addon` blocks barely differ between addons — copying an existing one (kube-proxy's, here) and swapping the name/version is the fastest way to add a new one.
- Some of this module's CSI functionality does not work on Fargate profiles — this module's cluster uses managed node groups only.
- From this module forward, new IAM-authenticated components use Pod Identity instead of IRSA, as a gradual shift rather than a hard cutover.

# Lesson 3: Installing the EBS CSI Driver

## 1. Attaching the AWS-Managed EBS CSI Policy to the Node Role

Before installing the driver, the node role itself gets the AWS-managed EBS CSI policy attached — belt-and-suspenders alongside the dedicated Pod Identity role created next:

```hcl
# iam_nodes.tf
resource "aws_iam_role_policy_attachment" "ebs_csi" {
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy"
  role       = aws_iam_role.eks_nodes_role.name
}
```

## 2. Creating the Pod Identity IAM Role for EBS CSI

A new `iam_ebs_csi.tf` follows the Pod Identity trust-policy shape from Lesson 1, with the same managed policy attached to a dedicated role:

```hcl
# iam_ebs_csi.tf
data "aws_iam_policy_document" "ebs_role" {
  statement {
    effect = "Allow"

    principals {
      type        = "Service"
      identifiers = ["pods.eks.amazonaws.com"]
    }

    actions = [
      "sts:AssumeRole",
      "sts:TagSession"
    ]
  }
}

resource "aws_iam_role" "ebs_role" {
  assume_role_policy = data.aws_iam_policy_document.ebs_role.json
  name               = format("%s-ebs-csi-role", var.project_name)
}

resource "aws_iam_role_policy_attachment" "ebs_csi_role" {
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy"
  role       = aws_iam_role.ebs_role.name
}
```

Worth double-checking as this gets wired up: the exact service account name the EBS CSI driver expects (`ebs-csi-controller-sa`) and the exact managed policy ARN aren't always obvious on the first try — it's normal to pause and confirm both against the IAM console or the driver's own documentation rather than assume they're right, especially the first time through.

## 3. Checking the Addon Version and Declaring the Variable

```bash
aws eks describe-addon-versions \
  --addon-name aws-ebs-csi-driver \
  --kubernetes-version <K8S_VERSION>
```

```hcl
# variables.tf
variable "addon_ebs_csi_version" {
  type        = string
  default     = "v1.39.0-eksbuild.1"
  description = "Versão do Addon do EBS CSI"
}
```

## 4. Creating the EBS CSI Driver Addon

```hcl
# addons.tf
resource "aws_eks_addon" "ebs_csi" {
  cluster_name = aws_eks_cluster.main.name
  addon_name   = "aws-ebs-csi-driver"

  addon_version               = var.addon_ebs_csi_version
  resolve_conflicts_on_create = "OVERWRITE"
  resolve_conflicts_on_update = "OVERWRITE"

  depends_on = [
    aws_eks_access_entry.nodes
  ]
}
```

## 5. Creating the Pod Identity Association

```hcl
# iam_ebs_csi.tf
resource "aws_eks_pod_identity_association" "ebs_csi" {
  cluster_name    = aws_eks_cluster.main.name
  namespace       = "kube-system"
  service_account = "ebs-csi-controller-sa"
  role_arn        = aws_iam_role.ebs_role.arn
}
```

## 6. Verifying the Installation

```bash
kubectl get pods -n kube-system
kubectl describe sa ebs-csi-controller-sa -n kube-system
kubectl logs -n kube-system -l app=ebs-csi-controller
```

The addon brings up both a DaemonSet (the node-level plugin, one per node) and a controller Deployment — the one whose service account, `ebs-csi-controller-sa`, is bound through the Pod Identity association above. Only GP2 is available as a default `StorageClass` at this point; GP3 is installed explicitly in Lesson 6.

One habit worth carrying over from IRSA doesn't apply here: with IRSA, the way to confirm a service account is wired up correctly is `kubectl describe sa` and checking for the role-ARN annotation. Under Pod Identity, that same `describe sa` comes back with no annotation at all — that's expected, not a sign something's broken, since Pod Identity never touches the service account object itself. The actual place to confirm the binding is the Pod Identity association itself, either via `aws eks list-pod-identity-associations` or the "Access -> Pod Identity Associations" tab in the EKS console.

## Key Takeaways

- The EBS CSI driver's Pod Identity role is only bound to the controller's service account (`ebs-csi-controller-sa`); the node role itself also gets the same managed policy attached directly, as a second layer.
- Installing the addon brings up both a node DaemonSet and a controller Deployment.
- Unlike IRSA, a Pod Identity binding leaves no annotation on the service account — verify it through the association resource (CLI or console), not `kubectl describe sa`.
- Out of the box, EBS only provides a `gp2` `StorageClass` — `gp3` has to be created separately.

# Lesson 4: Static EBS Provisioning with a PersistentVolumeClaim

## 1. The Lab Deployment: chip-ebs.yml

To prove the driver works, `chip` gets a single-replica variant with a `PersistentVolumeClaim` mounted at `/data`:

```yaml
# chip-ebs.yml
apiVersion: v1
kind: Namespace
metadata:
  name: chip
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: chip-pvc
  namespace: chip
spec:
  storageClassName: gp2
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 4Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: chip
  name: chip
  namespace: chip
spec:
  replicas: 1
  selector:
    matchLabels:
      app: chip
  template:
    metadata:
      labels:
        app: chip
        name: chip
        version: v1
    spec:
      containers:
      - name: chip
        image: fidelissauro/chip:v1
        ports:
        - containerPort: 8080
          name: http
        resources:
          requests:
            cpu: 250m
            memory: 512Mi
        startupProbe:
          failureThreshold: 10
          httpGet:
            path: /readiness
            port: 8080
          periodSeconds: 10
        livenessProbe:
          failureThreshold: 10
          httpGet:
            httpHeaders:
            - name: Custom-Header
              value: Awesome
            path: /liveness
            port: 8080
          periodSeconds: 10
        volumeMounts:
          - name: chip-volume
            mountPath: /data
      volumes:
        - name: chip-volume
          persistentVolumeClaim:
            claimName: chip-pvc
      terminationGracePeriodSeconds: 60
---
apiVersion: v1
kind: Service
metadata:
  name: chip
  namespace: chip
  labels:
    app.kubernetes.io/name: chip
    app.kubernetes.io/instance: chip
spec:
  ports:
  - name: web
    port: 8080
    protocol: TCP
  selector:
    app: chip
  type: ClusterIP
```

Only one replica is used deliberately: this test provisions exactly one EBS volume, bound to exactly one pod.

## 2. Applying the Manifest and Watching the Volume Provision

```bash
kubectl apply -f chip-ebs.yml
kubectl get pvc -n chip
kubectl describe pod -n chip -l app=chip
```

`kubectl get pvc` shows the claim moving to `Bound`, and the AWS console shows a brand-new `gp2` EBS volume — separate from the ephemeral root volumes every node already carries. The pod's `describe` output shows the same PVC mounted at `/data` as `chip-volume`.

What the `volumeMounts`/`volumes` pairing actually means in practice: anything the container writes under `/data` from now on lands on that EBS volume instead of the container's own writable layer — which is exactly what makes it survive the container being replaced.

## 3. Testing Filesystem Persistence Through Chip's API

`chip` doubles as a lab Swiss-army knife with a small set of filesystem endpoints purely for testing purposes — never meant for production, since they expose direct filesystem read/write over HTTP. From a temporary bastion pod inside the cluster:

```bash
kubectl run bastionpod --rm -i --tty --image debian -n default -- bash
apt-get update && apt-get install -y curl
```

From there, hitting chip's service DNS name (`chip.chip.svc.cluster.local:8080`) exercises three endpoints: one that lists a directory, one that writes a file given a path and its content base64-encoded, and one that reads a file back — roughly:

```bash
curl http://chip.chip.svc.cluster.local:8080/version

curl -X POST http://chip.chip.svc.cluster.local:8080/filesystem/ls -d '{"path": "/data"}'

curl -X POST http://chip.chip.svc.cluster.local:8080/filesystem/write -d '{"path": "/data/linuxtips", "content": "bGludXh0aXBzCg=="}'

curl -X POST http://chip.chip.svc.cluster.local:8080/filesystem/cat -d '{"path": "/data/linuxtips"}'
```

The very first `ls` against `/data`, right after the volume mounts, doesn't come back perfectly empty — there's already one entry sitting there, left over from the filesystem's own formatting (any freshly-formatted ext4 volume ships with a `lost+found` directory by default; it isn't something chip or the CSI driver created). Listing again after the write shows the new file alongside it, and reading it back returns the content that was written.

## 4. Proving the Volume Survives Pod Rescheduling

```bash
kubectl delete pods -n chip -l app=chip
kubectl get pods -n chip -w
```

Once the replacement pod is running, repeating the `ls`/`cat` calls from step 3 shows the exact same files still there — the EBS volume, and its data, survived the pod being deleted and rescheduled. That's the proof this is genuinely persistent storage, not the node's ephemeral disk.

## Key Takeaways

- A `PersistentVolumeClaim` bound to a single-replica Deployment provisions one EBS volume, mounted at the path declared in `volumeMounts`.
- Deleting and rescheduling the pod does not delete the volume or its data — that's the entire point of persistent storage versus a container's ephemeral filesystem.
- chip's filesystem endpoints used here are lab-only tooling, not something to expose in a real application.
- This static, one-PVC-per-Deployment approach doesn't scale well; Lesson 5 moves to per-replica dynamic provisioning with a StatefulSet.

# Lesson 5: Dynamic Provisioning with StatefulSets and VolumeClaimTemplates

## 1. Cleaning Up the Static Test

```bash
kubectl delete -f chip-ebs.yml
```

The static PVC test from Lesson 4 is removed first, to start this section from a clean namespace.

## 2. From PersistentVolumeClaim to volumeClaimTemplates

A `StatefulSet` with a `volumeClaimTemplates` block provisions one distinct `PersistentVolumeClaim` — and therefore one distinct EBS volume — per replica, each one bound permanently to that replica's identity:

```yaml
# chip-statatefulset-ebs.yml
apiVersion: v1
kind: Namespace
metadata:
  name: chip
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: chip
  namespace: chip
spec:
  serviceName: "chip"
  replicas: 3
  selector:
    matchLabels:
      app: chip
  template:
    metadata:
      labels:
        app: chip
        name: chip
        version: v1
    spec:
      containers:
      - name: chip
        image: fidelissauro/chip:v1
        ports:
        - containerPort: 8080
          name: http
        resources:
          requests:
            cpu: 250m
            memory: 512Mi
        startupProbe:
          failureThreshold: 10
          httpGet:
            path: /readiness
            port: 8080
          periodSeconds: 10
        livenessProbe:
          failureThreshold: 10
          httpGet:
            httpHeaders:
            - name: Custom-Header
              value: Awesome
            path: /liveness
            port: 8080
          periodSeconds: 10
        volumeMounts:
        - name: chip-storage
          mountPath: /data
      terminationGracePeriodSeconds: 60
  volumeClaimTemplates:
  - metadata:
      name: chip-storage
    spec:
      accessModes:
        - ReadWriteOnce
      resources:
        requests:
          storage: 4Gi
      storageClassName: gp2
---
apiVersion: v1
kind: Service
metadata:
  name: chip
  namespace: chip
  labels:
    app.kubernetes.io/name: chip
    app.kubernetes.io/instance: chip
spec:
  ports:
  - name: web
    port: 8080
    protocol: TCP
  selector:
    app: chip
  type: ClusterIP
```

`storageClassName: gp2` is what's available at this point in the walkthrough — Lesson 6 creates a `gp3` `StorageClass` and switches this same manifest over to it. The `4Gi` request is a deliberately small, simplified number for the lab — there's nothing special about it, it's just enough to prove the mechanism without waiting on a larger volume to provision.

## 3. Applying the StatefulSet and Watching Volumes Provision One by One

```bash
kubectl apply -f chip-statatefulset-ebs.yml
kubectl get pods -n chip -w
kubectl get pvc -n chip
```

Unlike a Deployment, a `StatefulSet` brings replicas up one at a time, in order: `chip-0` becomes ready before `chip-1` starts provisioning, and so on. Each replica gets its own PVC (`chip-storage-chip-0`, `chip-storage-chip-1`, `chip-storage-chip-2`) and its own EBS volume, visible individually in the AWS console.

## 4. EBS Volumes Are Not Shared Between Pods

Repeating the filesystem write/read test from Lesson 4 against this StatefulSet makes the key EBS limitation obvious: a file written through one pod (say, whichever `chip-N` a request happens to land on) is only visible when the next request lands on that same pod. Requests routed to a different replica see a different, empty volume. EBS volumes are never shared — this strategy is for workloads that need per-replica persistence (each pod always getting its own volume back after a restart), not for workloads that need every replica to see the same data.

## 5. Scaling Up: One New Volume Per New Replica

```bash
kubectl scale statefulset chip -n chip --replicas=4
kubectl get pvc -n chip
```

Scaling out provisions exactly one new PVC and one new EBS volume for the new replica (`chip-3`), following the same incremental, name-based pattern as the first three.

## Key Takeaways

- `volumeClaimTemplates` provisions one PVC — and one EBS volume — per StatefulSet replica, each permanently associated with that replica's ordinal identity.
- EBS volumes are never shared between pods, even under a StatefulSet: use this pattern when each replica needs its own persistent, non-shared storage, not when replicas need to see the same files.
- Scaling a StatefulSet up provisions new volumes incrementally, one per new replica.

# Lesson 6: Upgrading to GP3 Storage Classes

## 1. Creating the GP3 StorageClass Manifest

`gp2` is the only `StorageClass` the EBS CSI addon creates by default. `gp3` — cheaper and with better baseline IOPS — has to be created explicitly:

```yaml
# storageclassgp3.yml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com
parameters:
  fsType: ext4
  type: gp3
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

Even editor autocomplete doesn't have much to offer here — Copilot kept suggesting the wrong provisioner name for the `provisioner` field, since a `gp3` `StorageClass` isn't common enough boilerplate for it to have memorized correctly. Worth typing out by hand and double-checking against `ebs.csi.aws.com` rather than trusting the suggestion.

Applied directly first, as a quick sanity check:

```bash
kubectl apply -f storageclassgp3.yml
kubectl get storageclass
```

## 2. Provisioning It Through Terraform Instead

This is the same manual-first, formalize-later pattern from Lesson 1: get it working with a plain `kubectl apply`, confirm it behaves the way it should, and only then invest in wiring it into Terraform. The manually-applied `StorageClass` is deleted and recreated as Terraform-managed state instead, using `kubectl_manifest` — the same approach already used elsewhere in this course for CRDs the AWS or Kubernetes Terraform providers don't model natively:

```bash
kubectl delete -f storageclassgp3.yml
```

```hcl
# ebs_storage_class.tf
resource "kubectl_manifest" "gp3" {
  yaml_body = <<YAML
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: gp3
provisioner: ebs.csi.aws.com
parameters:
  fsType: ext4
  type: gp3
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
YAML

  depends_on = [
    aws_eks_addon.ebs_csi,
    aws_eks_pod_identity_association.ebs_csi
  ]
}
```

The explicit `depends_on` matters: the `gp3` class references the `ebs.csi.aws.com` provisioner, which only exists once the addon (and its Pod Identity association) are already in place.

## 3. Switching the StatefulSet to GP3

```bash
kubectl delete -f chip-statatefulset-ebs.yml
```

With the `gp3` class now provisioned, `chip-statatefulset-ebs.yml`'s `volumeClaimTemplates` from Lesson 5 is updated with a single change:

```diff
-      storageClassName: gp2
+      storageClassName: gp3
```

## 4. Confirming the New Volumes Are GP3

```bash
kubectl apply -f chip-statatefulset-ebs.yml
kubectl get pvc -n chip
```

The AWS console confirms the newly-provisioned volumes are `gp3` this time — the same StatefulSet mechanics from Lesson 5, just backed by the more cost-efficient, higher-baseline-IOPS disk type.

## Key Takeaways

- The EBS CSI addon only creates `gp2` by default; `gp3` requires its own `StorageClass`, referencing the `ebs.csi.aws.com` provisioner.
- Provisioning the `StorageClass` through Terraform's `kubectl_manifest`, with an explicit `depends_on` on the addon and its Pod Identity association, keeps it fully IaC-managed instead of a one-off `kubectl apply`.
- Switching a StatefulSet's `volumeClaimTemplates.storageClassName` only affects newly-provisioned volumes going forward.

# Lesson 7: Installing the EFS CSI Driver

## 1. When to Reach for EFS Instead of EBS

EFS solves a different problem than EBS: several pods — even across different nodes — reading and writing the very same volume at once. Typical cases are a configuration file that some automation drops in and every replica of an app needs to see, or one workload publishing files that another workload consumes. Where EBS is one volume per pod, EFS is one volume shared elastically by N pods.

## 2. Checking the Addon Version and Declaring the Variable

The installation follows the exact same shape as the EBS driver in Lesson 3, only the addon name and role change:

```bash
aws eks describe-addon-versions \
  --addon-name aws-efs-csi-driver \
  --kubernetes-version <K8S_VERSION>
```

```hcl
# variables.tf
variable "addon_efs_csi_version" {
  type        = string
  default     = "v2.1.4-eksbuild.1"
  description = "Versão do Addon do EFS CSI"
}
```

## 3. Creating the Pod Identity IAM Role for EFS CSI

```hcl
# iam_efs_csi.tf
data "aws_iam_policy_document" "efs_role" {
  statement {
    effect = "Allow"

    principals {
      type        = "Service"
      identifiers = ["pods.eks.amazonaws.com"]
    }

    actions = [
      "sts:AssumeRole",
      "sts:TagSession"
    ]
  }
}

resource "aws_iam_role" "efs_role" {
  assume_role_policy = data.aws_iam_policy_document.efs_role.json
  name               = format("%s-efs-csi-role", var.project_name)
}

resource "aws_iam_role_policy_attachment" "efs_csi_role" {
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonEFSCSIDriverPolicy"
  role       = aws_iam_role.efs_role.name
}

resource "aws_eks_pod_identity_association" "efs_csi" {
  cluster_name    = aws_eks_cluster.main.name
  namespace       = "kube-system"
  service_account = "efs-csi-controller-sa"
  role_arn        = aws_iam_role.efs_role.arn
}
```

Unlike EBS and S3, the node role itself doesn't need the EFS policy attached — only the controller's dedicated Pod Identity role does. As with EBS in Lesson 3, it's worth checking `efs-csi-controller-sa` against what the addon actually creates before applying the association — the same class of easy-to-typo mismatch that would otherwise leave the role built but never picked up.

## 4. Creating the EFS CSI Driver Addon

```hcl
# addons.tf
resource "aws_eks_addon" "efs_csi" {
  cluster_name = aws_eks_cluster.main.name
  addon_name   = "aws-efs-csi-driver"

  addon_version               = var.addon_efs_csi_version
  resolve_conflicts_on_create = "OVERWRITE"
  resolve_conflicts_on_update = "OVERWRITE"

  depends_on = [
    aws_eks_access_entry.nodes
  ]
}
```

## 5. Verifying the Installation

```bash
kubectl get pods -n kube-system
kubectl describe sa efs-csi-controller-sa -n kube-system
kubectl logs -n kube-system -l app=efs-csi-controller
```

Same shape as the EBS verification: an `efs-csi-controller` Deployment plus a node DaemonSet come up in `kube-system`, with no errors in the controller's logs.

## Key Takeaways

- EFS solves the "many pods, one shared volume" case that EBS structurally can't — a single volume mounted `ReadWriteMany` across pods and nodes.
- The installation pattern is identical to EBS's: check the addon version, create a Pod Identity role trusting `pods.eks.amazonaws.com`, create the addon, associate the role to `efs-csi-controller-sa`.
- The EFS policy is only attached to the dedicated Pod Identity role, not to the node role — unlike EBS and S3, which also attach their policy directly to the nodes.

# Lesson 8: Provisioning and Testing a Shared EFS Volume

## 1. Creating the EFS File System and Security Group

Provisioning an EFS file system reuses the same pattern from earlier container-architecture material: the file system itself, plus a security group opening the NFS port it depends on:

```hcl
# efs.tf
resource "aws_efs_file_system" "main" {
  creation_token   = "chip-efs-sharedd"
  performance_mode = "generalPurpose"

  tags = {
    Name = "chip"
  }
}

resource "aws_security_group" "efs" {
  name   = "efs"
  vpc_id = data.aws_ssm_parameter.vpc.value

  ingress {
    from_port = 2049
    to_port   = 2049
    protocol  = "tcp"
    cidr_blocks = [
      "0.0.0.0/0"
    ]
  }
  egress {
    from_port = 0
    to_port   = 0
    protocol  = "-1"
    cidr_blocks = [
      "0.0.0.0/0"
    ]
  }
}
```

The security group exists because a mount target is a real elastic network interface sitting inside the VPC — EFS traffic reaches the file system over NFS, on that ENI, the same way any other VPC-to-VPC traffic would, so it needs the same kind of port authorization any other network path does. Port 2049 is NFS, which is how EFS mount targets are actually reached. Opening it to `0.0.0.0/0` is a lab-only shortcut — worth tightening to the cluster's own security groups before using this outside a lab.

Applying just this file already makes the file system show up in the AWS console — but it isn't mountable from anywhere yet. A file system with no mount targets exists, but has nothing to actually connect to.

## 2. Mounting the File System in the Pod Subnets

An EFS file system isn't reachable until it has mount targets in the relevant subnets — one per pod subnet, in this cluster's case:

```hcl
# efs.tf
resource "aws_efs_mount_target" "efs_mount_pods" {
  count = length(data.aws_ssm_parameter.pod_subnets)

  file_system_id = aws_efs_file_system.main.id
  subnet_id      = data.aws_ssm_parameter.pod_subnets[count.index].value
  security_groups = [
    aws_security_group.efs.id
  ]
}
```

Once applied, the AWS console shows one elastic network interface per pod subnet, each with its own IP, ready to be mounted from that subnet.

## 3. Creating the EFS StorageClass and PersistentVolumeClaim

With the file system's ID in hand, a `StorageClass` ties it to the EFS CSI provisioner, and a `PersistentVolumeClaim` requests a volume from it — this time with `ReadWriteMany`, since the whole point is sharing:

```yaml
# chip-efs.yml
apiVersion: v1
kind: Namespace
metadata:
  name: chip
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: chip-efs-shared
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: <EFS_FILE_SYSTEM_ID>
  directoryPerms: "777"
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: chip-efs-shared
  namespace: chip
spec:
  storageClassName: chip-efs-shared
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 10Gi
```

## 4. Deploying Three Replicas Sharing One Volume

A plain `Deployment` is enough here — no need for a `StatefulSet` the way Lesson 5 needed one for EBS. The whole reason that lesson reached for a `StatefulSet` was to give each replica its own stable, private volume; with EFS that per-pod identity doesn't matter at all, since every replica shares the exact same volume regardless of which pod it is. The same `chip` Deployment shape from earlier lessons, this time with three replicas all mounting the same `chip-efs-shared` claim:

```yaml
# chip-efs.yml (continued)
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: chip
  name: chip
  namespace: chip
spec:
  replicas: 3
  selector:
    matchLabels:
      app: chip
  template:
    metadata:
      labels:
        app: chip
        name: chip
        version: v1
    spec:
      containers:
      - name: chip
        image: fidelissauro/chip:v1
        ports:
        - containerPort: 8080
          name: http
        resources:
          requests:
            cpu: 250m
            memory: 512Mi
        startupProbe:
          failureThreshold: 10
          httpGet:
            path: /readiness
            port: 8080
          periodSeconds: 10
        livenessProbe:
          failureThreshold: 10
          httpGet:
            httpHeaders:
            - name: Custom-Header
              value: Awesome
            path: /liveness
            port: 8080
          periodSeconds: 10
        volumeMounts:
          - name: chip-volume
            mountPath: /data
      volumes:
        - name: chip-volume
          persistentVolumeClaim:
            claimName: chip-efs-shared
      terminationGracePeriodSeconds: 60
---
apiVersion: v1
kind: Service
metadata:
  name: chip
  namespace: chip
  labels:
    app.kubernetes.io/name: chip
    app.kubernetes.io/instance: chip
spec:
  ports:
  - name: web
    port: 8080
    protocol: TCP
  selector:
    app: chip
  type: ClusterIP
```

```bash
kubectl apply -f chip-efs.yml
kubectl describe pod -n chip -l app=chip
```

Every replica's `describe` output shows the same `chip-volume`, backed by the same `chip-efs-shared` PVC, mounted at `/data`.

## 5. Proving the Shared Write/Read Across Pods

Repeating the filesystem test from Lesson 4 — this time against a Service load-balancing across three replicas — shows the difference EFS makes: listing `/data` right after start shows nothing, writing a file through one request, then listing again (which may land on any of the three pods) shows that same file, visible to all of them. Unlike EBS, it makes no difference which replica actually served the write — it doesn't matter which pod wrote the file, every pod can read it back.

## 6. Volumes Survive Pod Deletion Too

```bash
kubectl delete pods -n chip -l app=chip
kubectl get pods -n chip -w
```

Once the replacements are running, the same files are still there, from any replica — the shared EFS volume survives pod deletion exactly like the EBS volumes did, just with every replica seeing the same content instead of a private one.

## Key Takeaways

- EFS volumes are genuinely shared: a `ReadWriteMany` PVC lets every pod behind a Deployment read and write the exact same files, unlike EBS's one-volume-per-pod model.
- Mount targets must exist in every subnet where pods will actually mount the file system — without them, the volume simply isn't reachable from there.
- Opening the NFS security group to `0.0.0.0/0` is a lab simplification, not something to carry into a real environment.

# Lesson 9: Installing the S3 CSI Driver

## 1. Mounting S3 Buckets as Volumes: Eventual Consistency

The last CSI option mounts an S3 bucket directly as a volume: writing to the mounted path uploads an object, and objects already in the bucket show up as files. The one caveat that matters in practice is that this mount is **eventually consistent** — a write doesn't necessarily show up instantly on the other side, unlike EBS or EFS.

## 2. Checking the Addon Version and Declaring the Variable

The addon has a different name from the other two — `aws-mountpoint-s3-csi-driver` — but the same installation shape:

```bash
aws eks describe-addon-versions \
  --addon-name aws-mountpoint-s3-csi-driver \
  --kubernetes-version <K8S_VERSION>
```

```hcl
# variables.tf
variable "addon_s3_csi_version" {
  type        = string
  default     = "v1.11.0-eksbuild.1"
  description = "Versão do Addon do S3 CSI"
}
```

## 3. Creating the Pod Identity IAM Role for S3 CSI

```hcl
# iam_s3_csi.tf
data "aws_iam_policy_document" "s3_role" {
  statement {
    effect = "Allow"

    principals {
      type        = "Service"
      identifiers = ["pods.eks.amazonaws.com"]
    }

    actions = [
      "sts:AssumeRole",
      "sts:TagSession"
    ]
  }
}

resource "aws_iam_role" "s3_role" {
  assume_role_policy = data.aws_iam_policy_document.s3_role.json
  name               = format("%s-s3-csi-role", var.project_name)
}

resource "aws_iam_role_policy_attachment" "s3_csi_role" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3FullAccess"
  role       = aws_iam_role.s3_role.name
}

resource "aws_eks_pod_identity_association" "s3_csi" {
  cluster_name    = aws_eks_cluster.main.name
  namespace       = "kube-system"
  service_account = "s3-csi-driver-sa"
  role_arn        = aws_iam_role.s3_role.arn
}
```

`AmazonS3FullAccess` is deliberately broad, and it's fair to call it out plainly: it's attached here purely out of a certain laziness, to avoid stopping mid-lab to hand-pick individual S3 actions. This is explicitly not what a production role should look like; a real setup should scope this down to only the `Get`/`Put`/`List` actions the workload actually needs, on the specific bucket(s) it uses.

There's also only one association here, bound to `s3-csi-driver-sa` — worth double-checking against the addon's actual service account name before assuming it, since the S3 CSI driver ships as a single DaemonSet with no separate controller Deployment (unlike EBS and EFS), so there's no second, controller-side service account to second-guess it against.

The node role gets the same policy attached directly as well, exactly like EBS did in Lesson 3:

```hcl
# iam_nodes.tf
resource "aws_iam_role_policy_attachment" "s3_csi" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonS3FullAccess"
  role       = aws_iam_role.eks_nodes_role.name
}
```

## 4. Creating the S3 CSI Driver Addon

```hcl
# addons.tf
resource "aws_eks_addon" "s3_csi" {
  cluster_name = aws_eks_cluster.main.name
  addon_name   = "aws-mountpoint-s3-csi-driver"

  addon_version               = var.addon_s3_csi_version
  resolve_conflicts_on_create = "OVERWRITE"
  resolve_conflicts_on_update = "OVERWRITE"

  depends_on = [
    aws_eks_access_entry.nodes
  ]
}
```

## 5. Verifying the Installation

```bash
kubectl get pods -n kube-system
kubectl describe sa s3-csi-driver-sa -n kube-system
kubectl logs -n kube-system -l app=s3-csi-node
```

The addon shows up as a DaemonSet — this is the component that actually performs the bucket mount inside each node — with no errors in its logs once the Pod Identity association is in place.

## Key Takeaways

- The S3 CSI driver only runs as a DaemonSet (`s3-csi-driver-sa`), with no separate controller Deployment, unlike EBS and EFS.
- `AmazonS3FullAccess` is used here purely for lab simplicity — restrict this to the specific bucket and actions a workload actually needs before using it anywhere real.
- Like EBS, the S3 CSI policy is attached both to the dedicated Pod Identity role and directly to the node role.
- S3-backed volumes are eventually consistent — don't assume a write is immediately visible on the other side of the mount.

# Lesson 10: Mounting an S3 Bucket as a Volume

## 1. Provisioning the Bucket

A plain, private S3 bucket, following the same provisioning pattern used elsewhere in this course:

```hcl
# s3.tf
resource "aws_s3_bucket" "chip" {
  bucket = format("%s-%s-chip-shared", var.project_name, data.aws_caller_identity.current.account_id)
}

resource "aws_s3_bucket_ownership_controls" "chip" {
  bucket = aws_s3_bucket.chip.id
  rule {
    object_ownership = "BucketOwnerPreferred"
  }
}

resource "aws_s3_bucket_public_access_block" "chip" {
  bucket                  = aws_s3_bucket.chip.id
  ignore_public_acls      = true
  block_public_acls       = true
  block_public_policy     = true
  restrict_public_buckets = true

  depends_on = [aws_s3_bucket_ownership_controls.chip]
}

resource "aws_s3_bucket_acl" "chip" {
  bucket = aws_s3_bucket.chip.bucket
  acl    = "private"

  depends_on = [aws_s3_bucket_public_access_block.chip]
}
```

## 2. A Static PersistentVolume/PersistentVolumeClaim Pair for the Bucket

Unlike EBS and EFS, the S3 CSI driver doesn't dynamically provision anything from a `StorageClass` — the bucket already exists, so it's wired up as a **static** `PersistentVolume`, referenced by name from a matching `PersistentVolumeClaim`:

```yaml
# chip-s3.yml
apiVersion: v1
kind: Namespace
metadata:
  name: chip
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: chip-s3-shared
  namespace: chip
spec:
  capacity:
    storage: 100Gi
  accessModes:
    - ReadWriteMany
  csi:
    driver: s3.csi.aws.com
    volumeHandle: "chip-s3-shared"
    volumeAttributes:
      bucketName: "linuxtips-cluster-<ACCOUNT_ID>-chip-shared"
      region: "us-east-1"
  persistentVolumeReclaimPolicy: Retain
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: chip-s3-shared
  namespace: chip
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: ""
  resources:
    requests:
      storage: 10Gi
  volumeName: chip-s3-shared
```

The empty `storageClassName: ""` combined with an explicit `volumeName` is what tells Kubernetes to bind this claim to that specific, pre-existing `PersistentVolume` instead of asking a provisioner to dynamically create one. The `capacity`/`storage` values here are nominal — S3 doesn't actually enforce a size limit the way a block or file volume would.

## 3. Deploying Three Replicas Sharing the Bucket

The same `chip` Deployment shape as EFS's, mounting `chip-s3-shared` instead:

```bash
kubectl apply -f chip-s3.yml
kubectl get pods -n chip
```

## 4. Testing Bidirectional Sync Between Pods and the Bucket

The interesting part of this test runs in both directions, and — per the eventual-consistency caveat from Lesson 9 — sometimes with a short delay before the other side catches up:

- **From the console/CLI to the pods:** upload a file directly to the bucket (`aws s3 cp` or the console) — any file lying around works fine for the test, from an actual dataset down to random old files nobody needs anymore — then `ls /data` from inside a pod — the uploaded file shows up as a regular file.
- **From the pods to the bucket:** write a file at `/data` from inside a pod using chip's filesystem write endpoint (as in Lesson 4), then check the bucket in the AWS console — the file appears there as an object.

Either direction reaches every replica and the bucket alike, since they're all backed by the same underlying S3 objects. In this particular run both directions showed up close to instantly, with no perceptible delay — but that's exactly why the eventual-consistency caveat from Lesson 9 is a caveat and not a bug report: a fast propagation on one test run is not a guarantee, and production logic still shouldn't assume the other side is immediately caught up.

## 5. Wrapping Up: Comparing EBS, EFS, and S3

Across the three backends covered in this module:

| Backend | Sharing model | Best for |
|---|---|---|
| EBS | One volume per pod, single AZ | Fast, per-replica persistent disks (e.g. StatefulSet workloads) |
| EFS | Shared, `ReadWriteMany`, multi-AZ | Many pods reading/writing the same files |
| S3 | Shared, eventually consistent, mounted via a static PV | Object-storage-backed workloads that can tolerate replication delay |

## Key Takeaways

- The S3 CSI driver is wired up through a static `PersistentVolume` referencing an existing bucket, not dynamic provisioning through a `StorageClass` — the empty `storageClassName: ""` plus `volumeName` is what forces that static binding.
- Writes sync in both directions between the mounted path and the bucket, but only eventually — don't build logic that assumes immediate consistency.
- EBS, EFS, and S3 aren't interchangeable: the right one depends on whether storage needs to be per-pod, shared, or object-backed.
