# Module 11: EKS Auto Mode

## Overview

Every module up to this point built an EKS cluster the "manual" way: self-managed add-ons, a hand-installed Karpenter, an AWS Load Balancer Controller running as its own Helm release, an `aws-auth` ConfigMap or access entries wiring up node identities, and a dedicated Terraform-managed node group. **EKS Auto Mode**, launched by AWS at re:Invent 2024, is the opposite end of that spectrum: an opinionated "autopilot" flavor of EKS that bundles Karpenter, the AWS Load Balancer Controller, CoreDNS, and the EBS/EFS CSI drivers as add-ons AWS manages for you, running exclusively on a hardened Bottlerocket image that AWS also manages.

This module doesn't throw away anything learned so far — Deployments, Services, HPAs, and Ingress objects all work exactly the same. What changes is who provisions and manages the underlying compute: instead of Terraform-managed node groups or a self-installed Karpenter, Auto Mode ships two built-in NodePools (`general-purpose` and `system`) that you can target but not customize, and it trades away visibility (no SSH, no SSM, no logs or pod access for the components it manages) in exchange for a much lower operational footprint.

## Table of Contents

- [Lesson 1: Introduction to EKS Auto Mode](#lesson-1-introduction-to-eks-auto-mode)
  - [1. What Problem Auto Mode Solves](#1-what-problem-auto-mode-solves)
  - [2. What Changes for Nodes](#2-what-changes-for-nodes)
  - [3. Built-in NodePools and Managed Add-ons](#3-built-in-nodepools-and-managed-add-ons)
  - [4. What Gets Auto-Updated (and What Doesn't)](#4-what-gets-auto-updated-and-what-doesnt)
  - [5. Pros, Cons, and When to Use It](#5-pros-cons-and-when-to-use-it)
  - [Key Takeaways](#key-takeaways)
- [Lesson 2: Enabling Auto Mode on the Cluster](#lesson-2-enabling-auto-mode-on-the-cluster)
  - [1. Tearing Down What Auto Mode Replaces](#1-tearing-down-what-auto-mode-replaces)
  - [2. Expanding the Cluster IAM Role](#2-expanding-the-cluster-iam-role)
  - [3. Enabling Auto Mode on the EKS Cluster Resource](#3-enabling-auto-mode-on-the-eks-cluster-resource)
  - [4. Applying and Verifying Auto Mode Is Enabled](#4-applying-and-verifying-auto-mode-is-enabled)
  - [Key Takeaways](#key-takeaways-1)
- [Lesson 3: First Deployment and the Two Default NodePools](#lesson-3-first-deployment-and-the-two-default-nodepools)
  - [1. Logging Into an "Empty" Cluster](#1-logging-into-an-empty-cluster)
  - [2. Deploying Chip Like Any Other Workload](#2-deploying-chip-like-any-other-workload)
  - [3. Watching Karpenter Provision Nodes](#3-watching-karpenter-provision-nodes)
  - [4. Inspecting the Default NodePools](#4-inspecting-the-default-nodepools)
  - [5. What You Can and Can't Customize](#5-what-you-can-and-cant-customize)
  - [Key Takeaways](#key-takeaways-2)
- [Lesson 4: Scheduling onto the System NodePool](#lesson-4-scheduling-onto-the-system-nodepool)
  - [1. Why Segregate System Components from Applications](#1-why-segregate-system-components-from-applications)
  - [2. The system NodePool's Taint: CriticalAddonsOnly](#2-the-system-nodepools-taint-criticaladdonsonly)
  - [3. Wiring nodeSelector and tolerations into a Deployment](#3-wiring-nodeselector-and-tolerations-into-a-deployment)
  - [4. Applying the Same Pattern to Helm-Installed Add-ons](#4-applying-the-same-pattern-to-helm-installed-add-ons)
  - [Key Takeaways](#key-takeaways-3)
- [Lesson 5: Ingress and Everything Else Still Works](#lesson-5-ingress-and-everything-else-still-works)
  - [1. Auto Mode Doesn't Limit You](#1-auto-mode-doesnt-limit-you)
  - [2. Declaring an IngressClass for Auto Mode's Built-in ALB Controller](#2-declaring-an-ingressclass-for-auto-modes-built-in-alb-controller)
  - [3. Deploying Chip with an Ingress](#3-deploying-chip-with-an-ingress)
  - [4. Verifying the Load Balancer and Testing the Endpoint](#4-verifying-the-load-balancer-and-testing-the-endpoint)
  - [5. Wrapping Up: What's Different, What's the Same](#5-wrapping-up-whats-different-whats-the-same)
  - [Key Takeaways](#key-takeaways-4)

# Lesson 1: Introduction to EKS Auto Mode

## 1. What Problem Auto Mode Solves

Across this course, the cluster has been built piece by piece: node groups, Karpenter, the AWS Load Balancer Controller, CoreDNS, CSI drivers — each installed, versioned, and upgraded by hand. AWS noticed that a large set of add-ons showed up in almost every customer's cluster in roughly the same shape, and packaged them into a single switch: **EKS Auto Mode**, released at re:Invent in late 2024.

Auto Mode is best understood as an aggregation of add-ons and plugins that AWS now operates on your behalf. You don't install them, you don't see their pods, and you don't manage their versions individually — you just declare that Auto Mode is enabled, and the underlying components appear (and stay updated) without further intervention. The explicit goal is reducing operational overhead, at the direct cost of visibility into how that overhead was removed.

## 2. What Changes for Nodes

Auto Mode nodes run exclusively on a **Bottlerocket** image — the same OS this course has already deployed manually in earlier modules, but a variant hardened specifically for Auto Mode, with no option to swap it for another AMI or OS family.

Two constraints are non-negotiable on the default NodePools:

- **21-day node disruption.** Every node is recycled at most 21 days after it's provisioned, regardless of workload. This keeps the fleet continuously rotated onto the latest hardened image and plugin versions.
- **No direct node access.** There is no SSM Session Manager agent on Auto Mode nodes, and no SSH. AWS manages the full node lifecycle, but that also means there's no way to log in and troubleshoot a node directly — only `kubectl describe`/`get events` visibility at the Kubernetes API level.

## 3. Built-in NodePools and Managed Add-ons

Compute is still provisioned by Karpenter under the hood, but you don't install or configure it yourself. Auto Mode ships two default NodePools:

- **`general-purpose`** — where application workloads land unless told otherwise.
- **`system`** — intended for cluster system components that need more stability and shouldn't compete for capacity with rapidly scaling application workloads.

Neither default NodePool is customizable (no changing instance families, capacity types, or disruption settings on them), though both support Spot capacity, and you remain free to create your own additional NodePools/NodeClasses exactly as done manually in the Karpenter modules — those custom ones are fully editable.

Bundled into Auto Mode are the CRDs and controllers for: the AWS Load Balancer Controller (Ingress/Service load balancing), Pod Identity, Karpenter itself, CoreDNS, and block storage (EBS CSI). None of these arrive fully-featured or parametrized — and critically, **none of their pods or logs are visible to you**. The cluster looks empty (no `kube-system` pods for these components show up), yet everything is running and functioning.

## 4. What Gets Auto-Updated (and What Doesn't)

Auto Mode's automatic-update story is narrower than it first sounds. It **does** automatically update:

- Node AMIs, on its own rotation schedule (bounded by the 21-day disruption budget).
- The add-ons it manages internally (Load Balancer Controller, Karpenter, CoreDNS, EBS CSI, Pod Identity).

It does **not** automatically update the Kubernetes control plane version. If the cluster is running 1.31, Auto Mode will not move it to 1.32 on its own — a version upgrade is still a decision and an action you take explicitly, exactly like on a manually managed cluster.

## 5. Pros, Cons, and When to Use It

**In favor of Auto Mode:**

- Drastically simplified operations — the add-ons every cluster tends to need are pre-wired and self-managed.
- Node lifecycle and security patching are handled automatically, respecting the 21-day disruption budget.
- Lower cognitive load from not hand-rolling NodePools for common cases.
- A strong fit for ephemeral, short-lived, or cell-based/sharded architectures.

**Against it, at least in its current form:**

- Zero AMI customization on the default NodePools.
- Zero parametrization of the bundled add-ons.
- Zero log visibility into the managed components, which limits troubleshooting when something goes wrong.
- Linux only — no Windows or macOS/Darwin node support.

Given those trade-offs, Auto Mode is not yet recommended for very critical production environments, but it's an excellent starting point to explore in this lesson before deciding where it fits.

## Key Takeaways

- EKS Auto Mode, released at re:Invent 2024, bundles Karpenter, the AWS Load Balancer Controller, CoreDNS, and EBS CSI as AWS-managed add-ons with no visibility into their pods or logs.
- Nodes run a hardened Bottlerocket image exclusively, are recycled at most every 21 days, and have no SSH or SSM access.
- Two default NodePools ship out of the box — `general-purpose` and `system` — and neither is customizable, though custom NodePools you create yourself remain fully editable.
- Auto Mode auto-updates node AMIs and its bundled add-ons, but never the Kubernetes control plane version itself.
- It trades customization and observability for a much smaller operational footprint — a good fit for ephemeral/cell-based workloads, not yet recommended for the most critical production environments.

---

# Lesson 2: Enabling Auto Mode on the Cluster

## 1. Tearing Down What Auto Mode Replaces

Turning a manually-managed cluster into an Auto Mode cluster starts by removing everything Auto Mode now manages on your behalf. On the `main` branch, that means deleting:

- `addons.tf` — the self-managed `vpc-cni`, `coredns`, `kube-proxy`, and `eks-pod-identity-agent` EKS Addon resources.
- `access_entries.tf` — the `aws_eks_access_entry` used to authenticate the node IAM role.
- `aws_auth.tf` — the (already-commented-out) legacy `aws-auth` ConfigMap.
- `nodes.tf` — the Terraform-managed `aws_eks_node_group`.

The node IAM role itself (`iam_nodes.tf`) and the KMS key used for secrets encryption are both kept as-is — Auto Mode still needs a node role to associate with its NodePools, and cluster-secrets encryption is unrelated to Auto Mode.

## 2. Expanding the Cluster IAM Role

The cluster's IAM role needs additional permissions once Auto Mode takes over compute, load balancing, storage, and networking on your behalf. Four AWS-managed policies are added to the existing `AmazonEKSClusterPolicy` and `AmazonEKSServicePolicy` attachments, and the trust policy gains `sts:TagSession` alongside `sts:AssumeRole`:

```hcl
# iam_cluster.tf
data "aws_iam_policy_document" "cluster" {

  version = "2012-10-17"

  statement {
    actions = [
      "sts:AssumeRole",
      "sts:TagSession"
    ]


    principals {
      type = "Service"
      identifiers = [
        "eks.amazonaws.com"
      ]
    }
  }

}

resource "aws_iam_role" "eks_cluster_role" {
  name               = format("%s-cluster-role", var.project_name)
  assume_role_policy = data.aws_iam_policy_document.cluster.json
}

resource "aws_iam_role_policy_attachment" "eks_cluster_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
  role       = aws_iam_role.eks_cluster_role.name
}

resource "aws_iam_role_policy_attachment" "eks_service_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSServicePolicy"
  role       = aws_iam_role.eks_cluster_role.name
}


resource "aws_iam_role_policy_attachment" "eks_load_balancer_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSLoadBalancingPolicy"
  role       = aws_iam_role.eks_cluster_role.name
}

resource "aws_iam_role_policy_attachment" "eks_ebs_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSBlockStoragePolicy"
  role       = aws_iam_role.eks_cluster_role.name
}

resource "aws_iam_role_policy_attachment" "eks_compute_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSComputePolicy"
  role       = aws_iam_role.eks_cluster_role.name
}

resource "aws_iam_role_policy_attachment" "eks_networking_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSNetworkingPolicy"
  role       = aws_iam_role.eks_cluster_role.name
}
```

Note that these new permissions belong to the **cluster role**, not the **node role** — the node IAM role in `iam_nodes.tf` is untouched.

## 3. Enabling Auto Mode on the EKS Cluster Resource

The EKS documentation lays out exactly which blocks the `aws_eks_cluster` resource needs for Auto Mode: `bootstrap_self_managed_addons` set to `false`, load balancing and block storage enabled inside their respective config blocks, and a `compute_config` block naming the NodePools Auto Mode should provision:

```hcl
# eks.tf
resource "aws_eks_cluster" "main" {
  name    = var.project_name
  version = var.k8s_version

  role_arn = aws_iam_role.eks_cluster_role.arn

  vpc_config {
    subnet_ids = data.aws_ssm_parameter.private_subnets[*].value
  }

  encryption_config {
    provider {
      key_arn = aws_kms_key.main.arn
    }
    resources = ["secrets"]
  }

  access_config {
    authentication_mode                         = "API_AND_CONFIG_MAP"
    bootstrap_cluster_creator_admin_permissions = true
  }

  enabled_cluster_log_types = [
    "api", "audit", "authenticator", "controllerManager", "scheduler"
  ]

  // Auto mode
  bootstrap_self_managed_addons = false

  kubernetes_network_config {
    elastic_load_balancing {
      enabled = true
    }
  }

  storage_config {
    block_storage {
      enabled = true
    }
  }

  compute_config {
    enabled       = true
    node_pools    = ["general-purpose", "system"]
    node_role_arn = aws_iam_role.eks_nodes_role.arn
  }

  zonal_shift_config {
    enabled = true
  }

  tags = {
    "kubernetes.io/cluster/${var.project_name}" = "shared"
  }

}
```

Setting `bootstrap_self_managed_addons = false` tells EKS not to install the classic self-managed add-ons, since Auto Mode brings its own. `kubernetes_network_config.elastic_load_balancing.enabled` and `storage_config.block_storage.enabled` turn on the built-in AWS Load Balancer Controller and EBS CSI driver respectively. `compute_config` is where the two NodePools (`general-purpose` and `system`) are requested, pointing at the same node IAM role kept from before.

## 4. Applying and Verifying Auto Mode Is Enabled

With the deletions and the two file changes in place, `terraform apply` recreates the cluster from scratch as an Auto Mode cluster. Once it's up, the EKS console shows an **Auto Mode: Enabled** panel on the cluster, listing every capability that's active — Application Load Balancing, Block Storage, CSI, GPU support, Karpenter-based autoscaling, cluster DNS, and service networking (kube-proxy) — along with the two NodePools and the IAM role backing them. If anything required for Auto Mode is missing, this same panel calls it out.

## Key Takeaways

- Enabling Auto Mode means deleting the self-managed `addons.tf`, `access_entries.tf`, `aws_auth.tf`, and `nodes.tf` — Auto Mode now owns what they used to manage.
- The node IAM role and the KMS encryption key are kept unchanged; only the **cluster** IAM role needs new permissions (`AmazonEKSLoadBalancingPolicy`, `AmazonEKSBlockStoragePolicy`, `AmazonEKSComputePolicy`, `AmazonEKSNetworkingPolicy`, plus `sts:TagSession` in its trust policy).
- The `aws_eks_cluster` resource turns Auto Mode on via `bootstrap_self_managed_addons = false`, `kubernetes_network_config.elastic_load_balancing.enabled`, `storage_config.block_storage.enabled`, and a `compute_config` block naming the `general-purpose` and `system` NodePools.
- The EKS console's Auto Mode panel is the fastest way to confirm every required capability is actually enabled after applying.

---

# Lesson 3: First Deployment and the Two Default NodePools

## 1. Logging Into an "Empty" Cluster

The first thing that stands out after logging into a fresh Auto Mode cluster is how little is visible. Listing namespaces and pods across the cluster turns up almost nothing — just the handful of Kubernetes API/metrics services in `kube-system`, and the two NodePool objects (`general-purpose` and `system`). Everything specified as "enabled" in Lesson 2 — the AWS Load Balancer Controller, Karpenter, CoreDNS — is running, just not visible through `kubectl`.

## 2. Deploying Chip Like Any Other Workload

Deploying to an Auto Mode cluster looks identical to any other EKS cluster. The lab deployment is the same `chip` application used throughout the Karpenter modules — a Deployment, a `ClusterIP` Service, and an HPA — with a `topologySpreadConstraint` that spreads replicas across Availability Zones:

```yaml
# chip.yml
apiVersion: v1
kind: Namespace
metadata:
  name: chip
---
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
    
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: "topology.kubernetes.io/zone"
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: chip
            
      containers:
      - name: chip
        image: fidelissauro/chip:v1
        ports:
        - containerPort: 8080
          name: http
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
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
        env:
        - name: CHAOS_MONKEY_ENABLED
          value: "false"  
        - name: CHAOS_MONKEY_MODE
          value: "critical" 
        - name: CHAOS_MONKEY_LATENCY
          value: "true"            
        - name: CHAOS_MONKEY_EXCEPTION
          value: "true"   
        - name: CHAOS_MONKEY_APP_KILLER
          value: "true"   
        - name: CHAOS_MONKEY_MEMORY
          value: "true"                                        
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
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: chip
  namespace: chip
spec:
  maxReplicas: 6
  minReplicas: 2
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: chip
```

`kubectl apply` on this manifest is exactly the command used everywhere else in the course — there is nothing Auto-Mode-specific about deploying a workload.

## 3. Watching Karpenter Provision Nodes

Since the cluster starts with zero nodes, the first deployment triggers Karpenter (running invisibly, as part of Auto Mode) to request NodeClaims just as it would in the manually-installed Karpenter modules. With the AZ `topologySpreadConstraint` in place and three replicas, three nodes come up — one per Availability Zone, on the `general-purpose` NodePool, since nothing in the manifest targets a specific NodePool.

## 4. Inspecting the Default NodePools

Describing the `general-purpose` NodePool shows a configuration that looks like any hand-built Karpenter NodePool: a mix of `m`, `c`, and `r` instance families, on-demand-only capacity, Linux-only AMI family, and a 336-hour (14-day) `expireAfter` value baked in. This is the NodePool Karpenter falls back to whenever a workload doesn't specify otherwise.

## 5. What You Can and Can't Customize

The two default NodePools (`general-purpose` and `system`) are not editable — their instance selection, capacity type, and disruption settings are fixed by AWS. What remains fully available is creating your own `NodePool`/`EC2NodeClass` objects, exactly as done manually in the Karpenter modules, and applying them to the cluster; workloads can then be pointed at those custom NodePools like any other.

## Key Takeaways

- An Auto Mode cluster looks empty through `kubectl` even though the Load Balancer Controller, Karpenter, and CoreDNS are all running behind the scenes.
- Deploying workloads is unchanged — the same manifests and `kubectl apply` used throughout the course work as-is.
- Without an explicit NodePool target, workloads land on the built-in `general-purpose` NodePool, provisioned by the same Karpenter mechanics covered earlier in the course.
- The default `general-purpose` and `system` NodePools are fixed and non-customizable, but custom NodePools/EC2NodeClasses you create yourself remain fully editable.

---

# Lesson 4: Scheduling onto the System NodePool

## 1. Why Segregate System Components from Applications

Application workloads on `general-purpose` scale up and down quickly and can churn through nodes often. System components — things like metrics collectors or other cluster-wide utilities — benefit from a more stable NodePool that doesn't vary in capacity right alongside bursty application traffic. That's exactly what the `system` NodePool is for.

## 2. The system NodePool's Taint: CriticalAddonsOnly

Describing the `system` NodePool reveals a `NoSchedule` effect taint keyed `CriticalAddonsOnly`. A `nodeSelector` alone is **not** enough to land a pod there — without a matching `toleration` for that taint, the pod will never be scheduled onto a `system` node, regardless of what the `nodeSelector` says.

## 3. Wiring nodeSelector and tolerations into a Deployment

Getting the `chip` Deployment onto `system` requires both a `nodeSelector` targeting the NodePool by name and a `toleration` for the `CriticalAddonsOnly` taint:

```yaml
# chip-system.yml
apiVersion: v1
kind: Namespace
metadata:
  name: chip
---
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
    
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: "topology.kubernetes.io/zone"
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: chip
            
      nodeSelector:
        karpenter.sh/nodepool: system

      tolerations:
      - key: "CriticalAddonsOnly"
        operator: "Exists"

      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/arch
                operator: In
                values:
                - amd64

      containers:
      - name: chip
        image: fidelissauro/chip:v1
        ports:
        - containerPort: 8080
          name: http
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
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
        env:
        - name: CHAOS_MONKEY_ENABLED
          value: "false"  
        - name: CHAOS_MONKEY_MODE
          value: "critical" 
        - name: CHAOS_MONKEY_LATENCY
          value: "true"            
        - name: CHAOS_MONKEY_EXCEPTION
          value: "true"   
        - name: CHAOS_MONKEY_APP_KILLER
          value: "true"   
        - name: CHAOS_MONKEY_MEMORY
          value: "true"                                        
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
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: chip
  namespace: chip
spec:
  maxReplicas: 6
  minReplicas: 2
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 60
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: chip
```

Applying this shows every replica scheduled onto the same `system` node — the taint/toleration pairing is what actually enforces the placement, with the `nodeSelector` simply picking the NodePool.

## 4. Applying the Same Pattern to Helm-Installed Add-ons

The same `nodeSelector`/`toleration` pairing applies to Helm-installed system components, not just raw Deployments. `kube-state-metrics` and `metrics-server` are both re-pointed at the `system` NodePool through their chart's `set` values:

```hcl
# helm_kube_state_metrics.tf
resource "helm_release" "kube_state_metrics" {
  name             = "kube-state-metrics"
  repository       = "https://prometheus-community.github.io/helm-charts"
  chart            = "kube-state-metrics"
  namespace        = "kube-system"
  create_namespace = true

  set = [
    {
      name  = "apiService.create"
      value = "true"
    },
    {
      name  = "nodeSelector.karpenter\\.sh/nodepool"
      value = "system"
    },
    {
      name  = "tolerations[0].key"
      value = "CriticalAddonsOnly"
    },
    {
      name  = "tolerations[0].operator"
      value = "Exists"
    },
    {
      name  = "metricLabelsAllowlist[0]"
      value = "nodes=[*]"
    },
    {
      name  = "metricAnnotationsAllowList[0]"
      value = "nodes=[*]"
    }
  ]

  depends_on = [
    aws_eks_cluster.main
  ]
}
```

```hcl
# metrics_server.tf
resource "helm_release" "metrics_server" {
  name       = "metrics-server"
  repository = "oci://registry-1.docker.io/bitnamicharts"
  chart      = "metrics-server"
  namespace  = "kube-system"

  wait = false

  version = "7.2.16"

  set = [
    {
      name  = "nodeSelector.karpenter\\.sh/nodepool"
      value = "system"
    },
    {
      name  = "tolerations[0].key"
      value = "CriticalAddonsOnly"
    },
    {
      name  = "tolerations[0].operator"
      value = "Exists"
    },
    {
      name  = "apiService.create"
      value = "true"
    },
  ]

  depends_on = [
    aws_eks_cluster.main,
  ]
}
```

The `\\.` escape in `nodeSelector.karpenter\\.sh/nodepool` is required because Helm's `set` syntax treats `.` as a nesting separator — it has to be escaped to be treated as a literal character inside the label key.

## Key Takeaways

- The `system` NodePool carries a `CriticalAddonsOnly` `NoSchedule` taint — a `nodeSelector` alone will never schedule a pod there without a matching `toleration`.
- Both a `nodeSelector` (`karpenter.sh/nodepool: system`) and a `toleration` (`key: CriticalAddonsOnly, operator: Exists`) are required together to land a workload on `system`.
- The exact same pairing applies to Helm-installed components via `set` values, with `.` in label keys escaped as `\\.` so Helm doesn't treat it as nested YAML.

---

# Lesson 5: Ingress and Everything Else Still Works

## 1. Auto Mode Doesn't Limit You

Everything covered in prior modules — Ingress, Load Balancer annotations, HPA — remains fully usable on an Auto Mode cluster. Auto Mode has its own peculiarities to account for, but it is not a limiter: it layers on top of the same Kubernetes primitives already used throughout the course.

## 2. Declaring an IngressClass for Auto Mode's Built-in ALB Controller

Before deploying an Ingress, Auto Mode needs an `IngressClassParams`/`IngressClass` pair declared explicitly. Skipping this step causes the built-in controller to fall back to provisioning a Classic Load Balancer — not what's wanted here:

```yaml
# ingress-class.yml
apiVersion: eks.amazonaws.com/v1
kind: IngressClassParams
metadata:
  name: alb
spec:
  scheme: internet-facing
---
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: alb
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: eks.amazonaws.com/alb
  parameters:
    apiGroup: eks.amazonaws.com
    kind: IngressClassParams
    name: alb
```

`IngressClassParams` sets the ALB's scheme to `internet-facing` so it's reachable from outside the VPC, and the `IngressClass` ties that back to `eks.amazonaws.com/alb` — Auto Mode's built-in controller — marking it the cluster's default class.

## 3. Deploying Chip with an Ingress

The `chip` manifest for this lesson swaps the `ClusterIP` Service for a `LoadBalancer` Service fronted by an `Ingress`, and adds a second `topologySpreadConstraint` keyed on instance type so replicas also spread across different instance types, not just AZs:

```yaml
# chip-ingress.yml
apiVersion: v1
kind: Namespace
metadata:
  name: chip
---
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

      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: "topology.kubernetes.io/zone"
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: chip

        - maxSkew: 1
          topologyKey: "node.kubernetes.io/instance-type"
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: chip

      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/arch
                operator: In
                values:
                - amd64

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
        env:
        - name: CHAOS_MONKEY_ENABLED
          value: "false"  
        - name: CHAOS_MONKEY_MODE
          value: "critical" 
        - name: CHAOS_MONKEY_LATENCY
          value: "true"            
        - name: CHAOS_MONKEY_EXCEPTION
          value: "true"   
        - name: CHAOS_MONKEY_APP_KILLER
          value: "true"   
        - name: CHAOS_MONKEY_MEMORY
          value: "true"                                        
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
    targetPort: 8080
  selector:
    app: chip
  type: LoadBalancer
--- 
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: chip-ingress
  namespace: chip
  annotations:
    alb.ingress.kubernetes.io/scheme: "internet-facing" 
    alb.ingress.kubernetes.io/subnets:  "subnet-01741f88a082729d7,subnet-064f041f40b8c9b0c,subnet-0bc962e3de60e9e86"
    alb.ingress.kubernetes.io/target-type: "ip" 
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]' 
    alb.ingress.kubernetes.io/healthcheck-path: "/liveness" 
    alb.ingress.kubernetes.io/healthcheck-port: "traffic-port" 
    alb.ingress.kubernetes.io/healthcheck-protocol: "HTTP" 
  labels:
    app.kubernetes.io/name: chip
spec:
  ingressClassName: alb
  rules:
    - host: chip.msfidelis.com.br 
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: chip
                port:
                  number: 8080
```

This is the same Ingress pattern already used in the AWS Load Balancer Controller module (Module 7) — the annotations, subnets, and health-check configuration are unchanged. The only thing tying it to Auto Mode is `ingressClassName: alb`, referencing the `IngressClass` created in the previous step.

## 4. Verifying the Load Balancer and Testing the Endpoint

Applying the manifest triggers Karpenter to provision new nodes on `general-purpose` (no NodePool was specified), diversified by instance type per the second `topologySpreadConstraint`. Once the Ingress resolves, its ALB shows up in the EC2 console under Load Balancers, provisioning exactly as it would on a manually-managed cluster. Sending a request with the `Host: chip.msfidelis.com.br` header to the ALB's DNS name reaches `chip` and gets a response from the application — end to end, no different from the Ingress module's own test.

## 5. Wrapping Up: What's Different, What's the Same

Auto Mode changes how nodes and a fixed set of add-ons are provisioned and managed — it does not change how workloads are deployed, exposed, or scaled. Deployments, Services, HPAs, and Ingress objects all behave identically to a manually-managed cluster; the differences are entirely in the node lifecycle (Bottlerocket-only, 21-day recycling, no direct node access) and in what you can see and customize about the bundled add-ons. As a very new offering, expect its rough edges — like the current lack of log visibility into managed pods — to keep improving over time.

## Key Takeaways

- Ingress, Load Balancer annotations, and every other Kubernetes primitive from earlier modules work unchanged on Auto Mode.
- An explicit `IngressClassParams`/`IngressClass` pair is required before deploying an Ingress — otherwise the built-in controller falls back to a Classic Load Balancer.
- The Ingress manifest and annotations are identical to the AWS Load Balancer Controller module; only `ingressClassName: alb` ties it to Auto Mode's built-in controller.
- Auto Mode's real difference from a manually-managed cluster is confined to node lifecycle and managed add-on visibility — application-level Kubernetes usage is unchanged.
