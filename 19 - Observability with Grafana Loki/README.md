# Module 19: Observability with Grafana Loki

## Overview

This module starts the course's second major thread of the final project: a dedicated **observability cluster**, built to host the full Grafana stack — Grafana dashboards, Grafana Loki for logs, Grafana Tempo for traces (Module 20), and Grafana Mimir for metrics (Module 21) — all correlated through a single Grafana front end. This module's nine lessons cover only the first two pieces: **Grafana dashboards** and **Grafana Loki**, plus the **Fluent Bit** pipeline that ships logs into Loki from the two workload clusters built in Module 18.

Loki is a Prometheus-inspired log aggregation system: instead of indexing full log content, it indexes only the **labels** attached to each log stream (much like Prometheus indexes only metric labels, not sample values), and stores the actual log lines as compressed chunks in an object store. That trade-off makes it cheaper and more scalable than full-text log indexers, at the cost of not being able to do arbitrary full-text search — queries are written in **LogQL**, Loki's PromQL-inspired query language, and always start from a label selector.

This module assumes Module 18's multicluster setup is fully up: the control-plane cluster running Argo CD, and the two workload clusters (`linuxtips-cluster-01`, `linuxtips-cluster-02`) it federates. The `chip` application's `/login` and `/events` endpoints (used throughout Module 18 to generate canary/rollout traffic) double here as a convenient structured-JSON log generator — hitting them repeatedly produces a steady stream of `stdout` JSON logs to validate the pipeline end-to-end once Fluent Bit and Loki are wired up.

All code for this module lives in a **new, dedicated repository**, `linuxtips-eks-observability-cluster`, on branch `main` — a third repository alongside `linuxtips-eks-vanilla` (Modules 1-17) and `linuxtips-eks-multicluster-management` (Module 18). Lesson 2 builds it by copying the `control-plane` stack from Module 18 wholesale and stripping out everything Argo CD-specific, since the two clusters share the same EKS/Karpenter/Pod-Identity bootstrap boilerplate.

> **Note:** because this observability cluster is built once and grown across three modules, the repository's current `main` branch already contains Terraform and Helm files for **Tempo** and **Mimir** (`helm_tempo.tf`, `iam_tempo.tf`, `s3_tempo.tf`, `lb_tempo.tf`, `helm_mimir.tf`, `iam_mimir.tf`, `s3_mimir.tf`, `lb_mimir.tf`) and an OpenTelemetry Collector `ApplicationSet` (`otel.yml`) — none of which are covered by these 9 lessons. This README documents only the Loki- and Grafana-related content Module 19 actually teaches; the Tempo/Mimir/OTel files are flagged inline as forward-looking scaffolding for Modules 20-21, the same pattern already seen with `argo_fluentbit.tf`/`argo_otel.tf`/`argo_prometheus.tf` in Module 18's control-plane stack.

## Table of Contents

- [Lesson 1: Introduction to the Observability Cluster](#lesson-1-introduction-to-the-observability-cluster)
  - [1. From Single-Cluster Add-ons to a Dedicated Observability Cluster](#1-from-single-cluster-add-ons-to-a-dedicated-observability-cluster)
  - [2. Prerequisites: the Multicluster Argo CD Setup Must Be Running](#2-prerequisites-the-multicluster-argo-cd-setup-must-be-running)
  - [3. Why Loki: Label-Based Indexing Instead of Full-Text Search](#3-why-loki-label-based-indexing-instead-of-full-text-search)
  - [4. A New, Dedicated Repository](#4-a-new-dedicated-repository)
- [Lesson 2: Bootstrapping the Observability Cluster](#lesson-2-bootstrapping-the-observability-cluster)
  - [1. Copying the Control-Plane Stack as a Starting Point](#1-copying-the-control-plane-stack-as-a-starting-point)
  - [2. Leftover Scaffolding Kept From the Copy](#2-leftover-scaffolding-kept-from-the-copy)
  - [3. New Variables for This Cluster](#3-new-variables-for-this-cluster)
  - [4. A Private Route 53 Zone for Internal Service Discovery](#4-a-private-route-53-zone-for-internal-service-discovery)
- [Lesson 3: Deploying Grafana with EFS-Backed Persistence](#lesson-3-deploying-grafana-with-efs-backed-persistence)
  - [1. The EFS CSI Driver via Pod Identity](#1-the-efs-csi-driver-via-pod-identity)
  - [2. Grafana's EFS Volume and Storage Class](#2-grafanas-efs-volume-and-storage-class)
  - [3. Grafana's Helm Values via a `locals` Block](#3-grafanas-helm-values-via-a-locals-block)
  - [4. Installing the `grafana` Helm Release](#4-installing-the-grafana-helm-release)
- [Lesson 4: Exposing Grafana Through an Application Load Balancer](#lesson-4-exposing-grafana-through-an-application-load-balancer)
  - [1. ALB, Target Group, and Target Group Binding for Grafana](#1-alb-target-group-and-target-group-binding-for-grafana)
  - [2. Testing Dashboard Persistence](#2-testing-dashboard-persistence)
- [Lesson 5: Installing Grafana Loki in Simple Scalable Mode](#lesson-5-installing-grafana-loki-in-simple-scalable-mode)
  - [1. Loki's Three Deployment Modes](#1-lokis-three-deployment-modes)
  - [2. The EBS CSI Driver and a GP3 Storage Class](#2-the-ebs-csi-driver-and-a-gp3-storage-class)
  - [3. Three S3 Buckets and an IAM Role for Loki](#3-three-s3-buckets-and-an-iam-role-for-loki)
  - [4. Loki's Helm Values via a `locals` Block](#4-lokis-helm-values-via-a-locals-block)
  - [5. Installing the `loki` Helm Release](#5-installing-the-loki-helm-release)
  - [6. Loki's Simple-Scalable Components at a Glance](#6-lokis-simple-scalable-components-at-a-glance)
- [Lesson 6: Exposing Loki Through a Network Load Balancer](#lesson-6-exposing-loki-through-a-network-load-balancer)
  - [1. Internal NLB and Target Group Binding for the Gateway](#1-internal-nlb-and-target-group-binding-for-the-gateway)
  - [2. A Private DNS Record for the Loki Gateway](#2-a-private-dns-record-for-the-loki-gateway)
  - [3. Testing Log Ingestion with the Loki Push API](#3-testing-log-ingestion-with-the-loki-push-api)
  - [4. Disabling Multi-Tenant Auth](#4-disabling-multi-tenant-auth)
- [Lesson 7: Wiring Loki as a Grafana Data Source](#lesson-7-wiring-loki-as-a-grafana-data-source)
  - [1. Adding Loki to Grafana's `datasources` via `locals`](#1-adding-loki-to-grafanas-datasources-via-locals)
  - [2. Exploring Logs with LogQL](#2-exploring-logs-with-logql)
- [Lesson 8: Shipping Cluster Logs with Fluent Bit](#lesson-8-shipping-cluster-logs-with-fluent-bit)
  - [1. Fluent Bit as a Multicluster `ApplicationSet`](#1-fluent-bit-as-a-multicluster-applicationset)
  - [2. From a Static `cluster` Label to a Per-Cluster One](#2-from-a-static-cluster-label-to-a-per-cluster-one)
  - [3. Filtering and Correlating Logs in Grafana](#3-filtering-and-correlating-logs-in-grafana)
- [Lesson 9: Segregating Capacity with Dedicated NodePools](#lesson-9-segregating-capacity-with-dedicated-nodepools)
  - [1. Dedicated `grafana` and `loki` NodePools](#1-dedicated-grafana-and-loki-nodepools)
  - [2. Pinning Every Component with `nodeSelector`](#2-pinning-every-component-with-nodeselector)
  - [3. Verifying the Split with `kubectl get nodeclaims`](#3-verifying-the-split-with-kubectl-get-nodeclaims)
- [Key Takeaways](#key-takeaways)

---

# Lesson 1: Introduction to the Observability Cluster

## 1. From Single-Cluster Add-ons to a Dedicated Observability Cluster

Every module through 18 added tooling to the clusters that run the applications themselves. Starting with this module, the course builds a **fourth, dedicated EKS cluster** whose only job is observability: aggregating logs, traces, and metrics from every other cluster and correlating them in one place. Across this module and the next two, that cluster grows to host:

- **Grafana Loki** (this module) — log aggregation and indexing.
- **Grafana Tempo** (Module 20) — distributed trace storage.
- **Grafana Mimir** (Module 21) — long-term, centralized Prometheus-remote-write metrics storage.
- **Grafana dashboards**, tying all three data sources together in one UI.

## 2. Prerequisites: the Multicluster Argo CD Setup Must Be Running

This module builds directly on Module 18: the control-plane cluster with Argo CD, and the two workload clusters (`linuxtips-cluster-01`, `linuxtips-cluster-02`) it federates, all need to be up. The shared ingress layer is also required, since the `chip` application's `/login` and `/events` routes are used as a convenient log generator — repeatedly hitting `/events` produces a steady stream of structured JSON logs on `chip`'s `stdout`, which is exactly what gets shipped to Loki later in this module.

## 3. Why Loki: Label-Based Indexing Instead of Full-Text Search

Loki pairs with Grafana the same way Prometheus does, but for logs instead of metrics. The key architectural choice: Loki does **not** index the content of log lines, only their **labels** — the same mental model as querying Prometheus metrics by label rather than by value. Compared to a full-text indexer like Splunk, this makes Loki cheaper to run and more horizontally scalable, since it only ever needs to compress and store chunks in object storage (S3, in this module) rather than build a full inverted index over log content. The trade-off is that any query still has to start from a label selector — there is no free-text search across the entire index.

Queries against Loki are written in **LogQL**, a query language deliberately modeled on PromQL.

## 4. A New, Dedicated Repository

This module's Terraform lives in a brand-new repository, `linuxtips-eks-observability-cluster`, kept separate from both `linuxtips-eks-vanilla` (the app clusters from Modules 1-17) and `linuxtips-eks-multicluster-management` (the ingress/clusters/control-plane stacks from Module 18).

---

# Lesson 2: Bootstrapping the Observability Cluster

## 1. Copying the Control-Plane Stack as a Starting Point

Rather than writing the EKS/Karpenter/Pod-Identity bootstrap from scratch a fourth time, this lesson copies the `control-plane` stack from `linuxtips-eks-multicluster-management` (Module 18) into the new repository almost entirely, then strips out everything that's specific to running Argo CD: the Argo CD Helm release itself, ChartMuseum, and the `system` add-ons (Argo Rollouts, Metrics Server, KEDA) deployed from the control plane. What's kept is the shared EKS/Karpenter bootstrap boilerplate — the same pattern already documented in Module 18's "Provisioning the Two Workload Clusters" and "Installing Karpenter" lessons:

```hcl
# eks.tf
resource "aws_eks_cluster" "main" {
  name    = var.project_name
  version = var.k8s_version

  role_arn = aws_iam_role.eks_cluster_role.arn

  vpc_config {
    subnet_ids = data.aws_ssm_parameter.subnets[*].value
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

  zonal_shift_config {
    enabled = true
  }

  tags = {
    "kubernetes.io/cluster/${var.project_name}" = "shared"
  }
}
```

The IAM roles for the cluster and nodes (`iam_cluster.tf`, `iam_nodes.tf`), the temporary bootstrap node group (`nodes_temp.tf`), the VPC CNI/kube-proxy/Pod Identity Agent/EFS CSI/EBS CSI add-ons (`addons.tf`), the EKS access entries (`access_entry.tf`), the KMS key (`kms.tf`), the wide-open cluster security group rule (`sg.tf`), the S3 Terraform backend (`backend.tf`), and the AWS/Kubernetes/Helm/kubectl providers (`providers.tf`) are all a verbatim reuse of the Module 18 control-plane pattern — same resource names, same policies, same Pod Identity associations. Karpenter itself is installed identically too:

```hcl
# helm_karpenter.tf
resource "helm_release" "karpenter" {
  namespace        = "karpenter"
  create_namespace = true

  name       = "karpenter"
  repository = "oci://public.ecr.aws/karpenter"
  chart      = "karpenter"
  version    = "1.3.3"

  set {
    name  = "settings.clusterName"
    value = var.project_name
  }

  set {
    name  = "settings.clusterEndpoint"
    value = aws_eks_cluster.main.endpoint
  }

  set {
    name  = "aws.defaultInstanceProfile"
    value = aws_iam_instance_profile.nodes.name
  }

  set {
    name  = "controller.resources.requests.cpu"
    value = "1000m"
  }

  set {
    name  = "controller.resources.requests.memory"
    value = "1Gi"
  }

  depends_on = [
    aws_eks_cluster.main,
    aws_eks_node_group.main,
  ]
}
```

```hcl
# karpenter.tf
resource "kubectl_manifest" "ec2_node_class" {
  count = length(var.karpenter_capacity)
  yaml_body = templatefile("${path.module}/files/karpenter/ec2_node_class.yml", {
    NAME              = var.karpenter_capacity[count.index].name
    INSTANCE_PROFILE  = aws_iam_instance_profile.nodes.name
    AMI_ID            = data.aws_ssm_parameter.karpenter_ami[count.index].value
    AMI_FAMILY        = var.karpenter_capacity[count.index].ami_family
    SECURITY_GROUP_ID = aws_eks_cluster.main.vpc_config[0].cluster_security_group_id
    SUBNETS           = data.aws_ssm_parameter.subnets[*].value
  })

  depends_on = [
    helm_release.karpenter
  ]
}

resource "kubectl_manifest" "nodepool" {
  count = length(var.karpenter_capacity)
  yaml_body = templatefile("${path.module}/files/karpenter/nodepool.yml", {
    NAME               = var.karpenter_capacity[count.index].name
    WORKLOAD           = var.karpenter_capacity[count.index].workload
    INSTANCE_FAMILY    = var.karpenter_capacity[count.index].instance_family
    INSTANCE_SIZES     = var.karpenter_capacity[count.index].instance_sizes
    CAPACITY_TYPE      = var.karpenter_capacity[count.index].capacity_type
    AVAILABILITY_ZONES = var.karpenter_capacity[count.index].availability_zones
  })

  depends_on = [
    helm_release.karpenter
  ]
}
```

The AWS Load Balancer Controller is also installed the same way as every prior module, via Pod Identity:

```hcl
# helm_lb_controller.tf
resource "helm_release" "alb_ingress_controller" {
  name             = "aws-load-balancer-controller"
  repository       = "https://aws.github.io/eks-charts"
  chart            = "aws-load-balancer-controller"
  namespace        = "kube-system"
  create_namespace = true

  set {
    name  = "clusterName"
    value = var.project_name
  }

  set {
    name  = "serviceAccount.create"
    value = true
  }

  set {
    name  = "serviceAccount.name"
    value = "aws-load-balancer-controller"
  }

  set {
    name  = "region"
    value = var.region
  }

  set {
    name  = "vpcId"
    value = data.aws_ssm_parameter.vpc.value
  }

  depends_on = [
    aws_eks_cluster.main,
  ]
}
```

During the copy, the lesson explicitly catches and removes a duplicate `cluster.tf` left over from the control-plane stack, applies once just to confirm nothing was missed, and lands on a working fourth cluster — visible in the EKS console alongside `linuxtips-cluster-01`, `linuxtips-cluster-02`, and the control-plane cluster.

## 2. Leftover Scaffolding Kept From the Copy

Not everything from the control-plane copy was cleaned up. The repository still ships an EKS Fargate access entry and IAM role with no `aws_eks_fargate_profile` anywhere that uses them:

```hcl
# access_entry.tf
resource "aws_eks_access_entry" "nodes" {
  cluster_name  = aws_eks_cluster.main.id
  principal_arn = aws_iam_role.eks_nodes_role.arn
  type          = "EC2_LINUX"
}

resource "aws_eks_access_entry" "fargate" {
  cluster_name  = aws_eks_cluster.main.id
  principal_arn = aws_iam_role.fargate.arn
  type          = "FARGATE_LINUX"
}
```

```hcl
# iam_fargate.tf
data "aws_iam_policy_document" "fargate" {
  version = "2012-10-17"

  statement {
    actions = ["sts:AssumeRole"]
    principals {
      type        = "Service"
      identifiers = ["eks-fargate-pods.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "fargate" {
  name               = "${var.project_name}-fargate"
  assume_role_policy = data.aws_iam_policy_document.fargate.json
}

resource "aws_iam_role_policy_attachment" "fargate" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSFargatePodExecutionRolePolicy"
  role       = aws_iam_role.fargate.name
}
```

> **Note:** these two resources are dead weight — a Fargate access entry and execution role with no Fargate profile ever created against this cluster. Every workload in this module runs on Karpenter-provisioned EC2 nodes. This is the same kind of copy-paste leftover the lessons explicitly caught and removed for `cluster.tf`, just one they didn't catch.

## 3. New Variables for This Cluster

`variables.tf` adds one variable not present in Module 18's control-plane stack, for the subnets the Grafana ALB will use, while keeping `clusters_configs` — the Argo CD cluster-secret variable from the control-plane copy — even though this cluster never runs Argo CD:

```hcl
# variables.tf
variable "ssm_grafana_subnets" {
  description = "Lista de paths no SSM para as sub‑redes onde o Grafana será implantado, garantindo isolamento adequado da solução de observabilidade."
  type        = list(string)
}

variable "clusters_configs" {
  description = "Customização dos Secrets do ArgoCD para autenticação entre os clusters que vão ser gerenciados. Permite declarar múltiplos clusters, cada um identificado por cluster_name."
  default = [
    {
      cluster_name = "linuxtips-cluster-01"
    },
    {
      cluster_name = "linuxtips-cluster-02"
    }
  ]
}
```

> **Note:** `clusters_configs` is another leftover from the control-plane copy — nothing in this repository ever reads it, since Argo CD was one of the pieces removed in step 1 above.

## 4. A Private Route 53 Zone for Internal Service Discovery

The first resource created that's exclusive to this cluster is a private, VPC-scoped Route 53 hosted zone named `<project_name>.local`, used to give every internal service (Loki, and later Tempo/Mimir) a stable internal DNS name as it's built:

```hcl
# route53.tf
resource "aws_route53_zone" "private" {
  name = format("%s.local", var.project_name)

  vpc {
    vpc_id = data.aws_ssm_parameter.vpc.value
  }
}

resource "aws_route53_record" "loki" {
  zone_id = aws_route53_zone.private.zone_id
  name    = format("loki.%s.local", var.project_name)
  type    = "CNAME"
  ttl     = "30"
  records = [aws_lb.loki.dns_name]
}

resource "aws_route53_record" "tempo" {
  zone_id = aws_route53_zone.private.zone_id
  name    = format("tempo.%s.local", var.project_name)
  type    = "CNAME"
  ttl     = "30"
  records = [aws_lb.tempo.dns_name]
}

resource "aws_route53_record" "mimir" {
  zone_id = aws_route53_zone.private.zone_id
  name    = format("mimir.%s.local", var.project_name)
  type    = "CNAME"
  ttl     = "30"
  records = [aws_lb.mimir.dns_name]
}
```

> **Note:** the `tempo` and `mimir` records already point at load balancers this module never creates (`aws_lb.tempo`, `aws_lb.mimir` are defined in `lb_tempo.tf`/`lb_mimir.tf`, out of scope for these 9 lessons) — more of the same forward-looking scaffolding flagged in the Overview, present here because `route53.tf` was written once for the whole three-module build.

With `project_name = "linuxtips-observability"` (set in `environment/prod/terraform.tfvars`, covered in Lesson 9), the zone is `linuxtips-observability.local` and the Loki record resolves at `loki.linuxtips-observability.local`.

---

# Lesson 3: Deploying Grafana with EFS-Backed Persistence

## 1. The EFS CSI Driver via Pod Identity

Before Grafana itself, the cluster needs the EFS CSI driver, so Grafana's dashboards can be persisted to a real filesystem instead of being lost every time its pod restarts — the same driver and IAM pattern already used for shared storage in earlier modules:

```hcl
# iam_efs.tf
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

`addons.tf` installs the `aws-efs-csi-driver` add-on itself the same way as every other EKS add-on in this repository — resolved to the most recent version for the cluster's Kubernetes version and depending on the node access entry.

## 2. Grafana's EFS Volume and Storage Class

A dedicated EFS file system, one mount target per private subnet, and a `StorageClass` bound to it:

```hcl
# sg_efs.tf
resource "aws_security_group" "efs" {
  name   = format("%s-efs", var.project_name)
  vpc_id = data.aws_ssm_parameter.vpc.value

  ingress {
    from_port   = 2049
    to_port     = 2049
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

```hcl
# efs_grafana.tf
resource "aws_efs_file_system" "grafana" {
  creation_token   = format("%s-efs-grafana", var.project_name)
  performance_mode = "generalPurpose"

  tags = {
    Name = format("%s-efs-grafana", var.project_name)
  }
}

resource "aws_efs_mount_target" "grafana" {
  count = length(data.aws_ssm_parameter.subnets)

  file_system_id = aws_efs_file_system.grafana.id
  subnet_id      = data.aws_ssm_parameter.subnets[count.index].value
  security_groups = [
    aws_security_group.efs.id
  ]
}

resource "kubectl_manifest" "grafana_efs_storage_class" {
  yaml_body = <<YAML
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-grafana
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: ${aws_efs_file_system.grafana.id}
  directoryPerms: "777"
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
YAML

  depends_on = [
    aws_eks_cluster.main,
    helm_release.karpenter
  ]
}
```

## 3. Grafana's Helm Values via a `locals` Block

Rather than a `values` file or a long chain of `set` blocks, Grafana's Helm values are written as one heredoc string inside a `locals` block — the lesson's suggested pattern for values large enough to need Terraform interpolation:

```hcl
# locals.tf (grafana block)
locals {
  grafana = {
    values : <<-VALUES
adminUser: admin
adminPassword: linuxtips        

persistence:
    enabled: true
    size: 10Gi
    storageClassName: efs-grafana
service:
    type: NodePort
initChownData:
    enabled: false

nodeSelector:
    karpenter.sh/nodepool: grafana


datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
      - name: Loki
        type: loki
        access: proxy
        url: http://loki-gateway.loki.svc.cluster.local
        isDefault: false
        jsonData:
          maxLines: 1000
          derivedFields:
          - datasourceName: Tempo
            datasourceUid: Tempo
            matcherRegex: '\\"traceID\\":\\"([^\\"]+)\\"'
            name: traceID
            url: $$${__value.raw}

      - name: Tempo
        type: tempo
        access: proxy
        url: http://tempo-gateway.tempo.svc.cluster.local
        basicAuth: false
        jsonData:
          tracesToMetrics:
            datasourceUid: 'Mimir'
          serviceMap:
            datasourceUid: 'Mimir'
          nodeGraph:
            enabled: true
          tracesToLogs:
            datasourceUid: 'Loki'

      - name: Mimir
        type: prometheus
        access: proxy
        url: http://mimir-nginx.mimir.svc.cluster.local:80/prometheus
        isDefault: true
        jsonData:
          prometheusType: Mimir
    VALUES
  }
}
```

> **Note:** this `locals.tf` block is the repository's final, cumulative state — it already carries the `nodeSelector` from Lesson 9 below, and `datasources` entries for Tempo and Mimir that don't exist until Modules 20-21. This lesson's own scope is only `adminUser`/`adminPassword`/`persistence`/`service`/`initChownData`; the Loki `datasources` entry is added in Lesson 7, and the Tempo/Mimir entries are forward-looking scaffolding, not something Module 19 builds or explains.

> **Note:** `adminPassword` is a hardcoded plaintext value (`linuxtips`) directly in the Helm values, the same simple-lab pattern used for admin credentials throughout the course — fine for this training environment, not something to carry into production.

## 4. Installing the `grafana` Helm Release

```hcl
# helm_grafana.tf
resource "helm_release" "grafana" {
  name       = "grafana"
  chart      = "grafana"
  repository = "https://grafana.github.io/helm-charts"
  namespace  = "grafana"

  create_namespace = true

  values = [
    local.grafana["values"]
  ]

  depends_on = [
    aws_eks_pod_identity_association.efs_csi,
    aws_eks_addon.efs_csi,
    helm_release.karpenter
  ]
}
```

`kubectl get pods,svc,pvc -n grafana` confirms a running Grafana pod, a `NodePort` service, and a PVC bound to the `efs-grafana` storage class.

---

# Lesson 4: Exposing Grafana Through an Application Load Balancer

## 1. ALB, Target Group, and Target Group Binding for Grafana

A standard internet-facing ALB, deployed into the public subnets referenced by `ssm_grafana_subnets`, target-group-bound to the Grafana `NodePort` service:

```hcl
# lb_grafana.tf
resource "aws_lb" "grafana" {
  name = format("%s-grafana", var.project_name)

  internal           = false
  load_balancer_type = "application"

  subnets = data.aws_ssm_parameter.lb_grafana_subnets[*].value

  security_groups = [aws_security_group.grafana.id]

  enable_cross_zone_load_balancing = true
  enable_deletion_protection       = false

  tags = {
    Name = var.project_name
  }
}

resource "aws_lb_target_group" "grafana" {
  name     = format("%s-grafana", var.project_name)
  port     = 80
  protocol = "HTTP"
  vpc_id   = data.aws_ssm_parameter.vpc.value

  health_check {
    matcher = "200-299"
    path    = "/healthz"
  }
}

resource "aws_lb_listener" "grafana" {
  load_balancer_arn = aws_lb.grafana.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.grafana.arn
  }
}

resource "aws_security_group" "grafana" {
  name = format("%s-grafana", var.project_name)

  vpc_id = data.aws_ssm_parameter.vpc.value

  ingress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = var.project_name
  }
}

resource "kubectl_manifest" "grafana" {
  yaml_body = <<YAML
apiVersion: elbv2.k8s.aws/v1beta1
kind: TargetGroupBinding
metadata:
  name: grafana
  namespace: grafana
spec:
  serviceRef:
    name: grafana
    port: 80
  targetGroupARN: ${aws_lb_target_group.grafana.arn}
  targetType: instance
YAML
  depends_on = [
    helm_release.alb_ingress_controller,
    helm_release.grafana
  ]
}
```

> **Note:** `aws_security_group.grafana` opens **all ports, all protocols** (`from_port`/`to_port` `0`, protocol `-1`) to `0.0.0.0/0` in both directions, not just port 80/443. Functionally the ALB listener still only forwards port 80, but the security group itself is far broader than it needs to be — the same "wide open" pattern worth flagging wherever it shows up, distinct from a tightly-scoped ingress rule.

## 2. Testing Dashboard Persistence

With the ALB's DNS name in hand, the Grafana login page (`admin` / `linuxtips`, from Lesson 3) is reachable directly. The lesson creates a throwaway dashboard from a stock template (no data source configured yet, so no data — just to exercise persistence), deletes the Grafana pod to force a reschedule, and confirms after the pod comes back that the dashboard is still there — proof the EFS-backed `PersistentVolumeClaim` from Lesson 3 is working as intended.

---

# Lesson 5: Installing Grafana Loki in Simple Scalable Mode

## 1. Loki's Three Deployment Modes

Loki's own documentation describes three deployment topologies:

- **Monolithic** — every component (distributor, ingester, querier, query-frontend, compactor, ruler) runs inside a single binary, in a single pod. Simplest, meant for small environments.
- **Simple scalable** — the mode used in this module. Components are split into two independently-scalable groups, `write` and `read`, plus a `backend` group and a `gateway` that fronts both the write and read paths behind one endpoint.
- **Microservice** — every component gets its own independent Deployment, scaling fully independently. Meant for the largest environments with the highest log volume.

## 2. The EBS CSI Driver and a GP3 Storage Class

Loki's `write`/`backend` components are stateful — they need local disk for hot/recent chunks before they're flushed to S3 — so the cluster needs the EBS CSI driver, installed the same way as the EFS driver in Lesson 3:

```hcl
# iam_ebs.tf
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

resource "aws_eks_pod_identity_association" "ebs_csi" {
  cluster_name    = aws_eks_cluster.main.name
  namespace       = "kube-system"
  service_account = "ebs-csi-controller-sa"
  role_arn        = aws_iam_role.ebs_role.arn
}
```

```hcl
# storage_class_gp3.tf
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
    aws_eks_pod_identity_association.ebs_csi,
    helm_release.karpenter
  ]
}
```

## 3. Three S3 Buckets and an IAM Role for Loki

Loki needs three S3 buckets — for compacted log chunks, the ruler, and admin data — and a Pod Identity role scoped to just those three:

```hcl
# s3_loki.tf
resource "aws_s3_bucket" "loki-chunks" {
  bucket = format("%s-%s-loki-chunks", var.project_name, data.aws_caller_identity.current.account_id)
}

resource "aws_s3_bucket_ownership_controls" "loki-chunks" {
  bucket = aws_s3_bucket.loki-chunks.id
  rule {
    object_ownership = "BucketOwnerPreferred"
  }
}

resource "aws_s3_bucket_acl" "loki-chunks" {
  bucket = aws_s3_bucket.loki-chunks.id
  acl    = "private"

  depends_on = [
    aws_s3_bucket_ownership_controls.loki-chunks
  ]
}

resource "aws_s3_bucket" "loki-admin" {
  bucket = format("%s-%s-loki-admin", var.project_name, data.aws_caller_identity.current.account_id)
}

# ... aws_s3_bucket_ownership_controls / aws_s3_bucket_acl for loki-admin, same shape as above

resource "aws_s3_bucket" "loki-ruler" {
  bucket = format("%s-%s-loki-ruler", var.project_name, data.aws_caller_identity.current.account_id)
}

# ... aws_s3_bucket_ownership_controls / aws_s3_bucket_acl for loki-ruler, same shape as above
```

```hcl
# iam_loki.tf
data "aws_iam_policy_document" "loki_role" {
  version = "2012-10-17"

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

resource "aws_iam_role" "loki_role" {
  assume_role_policy = data.aws_iam_policy_document.loki_role.json
  name               = format("%s-loki", var.project_name)
}

data "aws_iam_policy_document" "loki_policy" {
  version = "2012-10-17"

  statement {
    effect = "Allow"
    actions = [
      "s3:*",
    ]

    resources = [
      format("%s/*", aws_s3_bucket.loki-chunks.arn),
      format("%s/*", aws_s3_bucket.loki-admin.arn),
      format("%s/*", aws_s3_bucket.loki-ruler.arn),
      aws_s3_bucket.loki-chunks.arn,
      aws_s3_bucket.loki-admin.arn,
      aws_s3_bucket.loki-ruler.arn,
    ]
  }
}

resource "aws_iam_policy" "loki_policy" {
  name        = format("%s-loki", var.project_name)
  path        = "/"
  description = var.project_name

  policy = data.aws_iam_policy_document.loki_policy.json
}

resource "aws_iam_policy_attachment" "loki" {
  name = "loki"
  roles = [
    aws_iam_role.loki_role.name
  ]

  policy_arn = aws_iam_policy.loki_policy.arn
}

resource "aws_eks_pod_identity_association" "loki" {
  cluster_name    = aws_eks_cluster.main.name
  namespace       = "loki"
  service_account = "loki"
  role_arn        = aws_iam_role.loki_role.arn
}
```

## 4. Loki's Helm Values via a `locals` Block

Same `locals` pattern as Grafana, this time configuring Loki's S3 backend, the `SimpleScalable` deployment mode, per-component replica counts and storage classes, and per-component `nodeSelector`s:

```hcl
# locals.tf (loki block)
locals {
  loki = {
    values : <<-VALUES
loki:
    auth_enabled: false
    schemaConfig:
        configs:
        - from: "2024-04-01"
          store: tsdb
          object_store: s3
          schema: v13
          index:
            prefix: loki_index_
            period: 24h
    storage_config:
        aws:
            region: ${var.region}
            bucketnames: ${aws_s3_bucket.loki-chunks.id}
            s3forcepathstyle: false
    storage:
        type: s3
        bucketNames:
            chunks: ${aws_s3_bucket.loki-chunks.id}
            ruler: ${aws_s3_bucket.loki-ruler.id}
            admin: ${aws_s3_bucket.loki-admin.id}
    ingester:
        chunk_encoding: snappy
    querier:
        max_concurrent: 4
    pattern_ingester:
        enabled: true
    limits_config:
        allow_structured_metadata: true
        volume_enabled: true
        retention_period: 672h

deploymentMode: SimpleScalable

backend:
    replicas: 3
    persistence:
        storageClass: gp3
    nodeSelector:
        karpenter.sh/nodepool: loki

read:
    replicas: 3
    nodeSelector:
        karpenter.sh/nodepool: loki

write:
    replicas: 3 # To ensure data durability with replication
    persistence:
        storageClass: gp3
    nodeSelector:
        karpenter.sh/nodepool: loki        

gateway:
    replicas: 3
    service:
        type: NodePort
    nodeSelector:
        karpenter.sh/nodepool: loki

minio:
    enabled: false
    VALUES
  }
}
```

> **Note:** like the Grafana `locals` block, this is the repository's cumulative final state — every `nodeSelector: karpenter.sh/nodepool: loki` line here is Lesson 9's capacity-segregation work, already baked in. When this lesson first applies it, none of the `loki`-named NodePools exist yet, so pods schedule onto the shared `general` pool until Lesson 9 adds the dedicated one. `auth_enabled: false` is also final-state; Lesson 6 below covers why it's needed and when it was actually flipped.

`retention_period: 672h` (28 days) and `deploymentMode: SimpleScalable` are the two settings that most directly reflect the documentation's recommendation for a small/medium environment: 3 replicas each of `read`/`write`/`backend`, a shared `gateway` fronting both paths, and MinIO disabled since S3 is the real object store here.

## 5. Installing the `loki` Helm Release

```hcl
# helm_loki.tf
resource "helm_release" "loki" {
  name       = "loki"
  chart      = "loki"
  repository = "https://grafana.github.io/helm-charts"
  namespace  = "loki"

  create_namespace = true

  values = [
    local.loki["values"]
  ]

  depends_on = [
    helm_release.karpenter,
    aws_eks_pod_identity_association.loki,
    aws_eks_addon.ebs_csi
  ]
}
```

## 6. Loki's Simple-Scalable Components at a Glance

After `terraform apply`, `kubectl get all -n loki` shows the shape of a simple-scalable deployment: 3 `backend` replicas, 3 `read` replicas, 3 `write` replicas, 3 `gateway` replicas, a `canary` DaemonSet (Loki's own synthetic-log-ingestion health check — it constantly writes and reads back its own test logs to verify the pipeline is healthy), plus `chunks-cache` and `results-cache` — in-memory caching layers that keep recently-used chunks and query results warm.

- **Gateway** — an nginx reverse proxy in front of both the write and read paths, so clients only need one endpoint.
- **Write** — receives incoming log pushes, normalizes them, attaches labels, compresses them into chunks, and hands them to the backend for storage.
- **Read** — handles read/query requests the same way, aggregating and forwarding to the backend.
- **Canary** — a synthetic tester that continuously pushes its own logs into Loki and verifies they come back out, as an always-on health signal.

---

# Lesson 6: Exposing Loki Through a Network Load Balancer

## 1. Internal NLB and Target Group Binding for the Gateway

Loki's gateway is exposed to the rest of the observability cluster (and, via the private zone, to the workload clusters) through an internal Network Load Balancer, target-group-bound to the `loki-gateway` service:

```hcl
# lb_loki.tf
resource "aws_lb" "loki" {
  name = format("%s-loki", var.project_name)

  internal           = true
  load_balancer_type = "network"

  subnets = data.aws_ssm_parameter.lb_subnets[*].value

  enable_cross_zone_load_balancing = true
  enable_deletion_protection       = false

  tags = {
    Name = var.project_name
  }
}

resource "aws_lb_target_group" "loki" {
  name     = format("%s-http", var.project_name)
  port     = 80
  protocol = "TCP"
  vpc_id   = data.aws_ssm_parameter.vpc.value
}

resource "aws_lb_listener" "loki" {
  load_balancer_arn = aws_lb.loki.arn
  port              = 80
  protocol          = "TCP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.loki.arn
  }
}

resource "kubectl_manifest" "loki" {
  yaml_body = <<YAML
apiVersion: elbv2.k8s.aws/v1beta1
kind: TargetGroupBinding
metadata:
  name: loki-gateway
  namespace: loki
spec:
  serviceRef:
    name: loki-gateway
    port: 80
  targetGroupARN: ${aws_lb_target_group.loki.arn}
  targetType: instance
YAML
  depends_on = [
    helm_release.alb_ingress_controller,
    helm_release.loki
  ]
}
```

`ssm_lb_subnets` for this cluster (Lesson 9's `terraform.tfvars`) points at **private** subnets, matching `internal = true` — this NLB is reachable only from inside the VPC, never from the public internet.

## 2. A Private DNS Record for the Loki Gateway

With `aws_route53_record.loki` (Lesson 2) already pointing at this NLB's DNS name, the gateway is reachable internally at `loki.linuxtips-observability.local`, on port 80 — no public exposure needed, since only in-VPC clients (Fluent Bit on the workload clusters, Grafana's data source) ever need to reach it.

## 3. Testing Log Ingestion with the Loki Push API

Before wiring up Fluent Bit, the lesson validates ingestion manually from a throwaway debug pod (`kubectl run` + `curl` installed on the fly), pushing directly to Loki's HTTP push API. The following reconstructs the shape of that request — it was typed live in a terminal, not saved anywhere in the repository, so it's illustrative rather than a quoted file:

```bash
# illustrative — Loki's documented push API, not a file in the repository
TS=$(date +%s%N)   # current Unix time in nanoseconds, Loki's expected timestamp unit

curl -s -X POST "http://loki.linuxtips-observability.local/loki/api/v1/push" \
  -H "Content-Type: application/json" \
  -d '{
        "streams": [
          {
            "stream": { "app": "manual-test" },
            "values": [ [ "'"$TS"'", "hello from the observability lesson" ] ]
          }
        ]
      }'
```

The lesson's first attempts against this endpoint return HTTP `401 Unauthorized` — Loki's default expects a tenant/organization header when multi-tenancy is enabled — which leads directly into the fix in the next section.

## 4. Disabling Multi-Tenant Auth

The Loki Helm chart's `loki.auth_enabled` defaults to `true`, requiring every request to carry an `X-Scope-OrgID` tenant header. Since this lab isn't using multi-tenancy, the fix is the `auth_enabled: false` line already shown in the `locals.tf` block in Lesson 5 — flipped from the chart's default and re-applied. After that, the same push request returns `204 No Content` (Loki's success response for a push with no body to return) instead of `401`.

---

# Lesson 7: Wiring Loki as a Grafana Data Source

## 1. Adding Loki to Grafana's `datasources` via `locals`

Grafana's data sources are provisioned declaratively, as a `datasources.yaml` block nested inside the same `grafana` `locals` value shown in Lesson 3 — this lesson is what actually adds the `Loki` entry:

```yaml
# locals.tf (grafana.values → datasources, Loki entry)
datasources:
  datasources.yaml:
    apiVersion: 1
    datasources:
      - name: Loki
        type: loki
        access: proxy
        url: http://loki-gateway.loki.svc.cluster.local
        isDefault: false
        jsonData:
          maxLines: 1000
```

The URL targets the in-cluster `loki-gateway` Kubernetes Service directly (`loki-gateway.loki.svc.cluster.local`) rather than the external NLB DNS name from Lesson 6 — Grafana and Loki live in the same cluster, so there's no need to hairpin out through the load balancer. `maxLines: 1000` caps how many log lines a single query can return, a safety limit against a runaway query overwhelming the Grafana UI.

> **Note:** as flagged in Lesson 3, this same `datasources` list already carries `Tempo` and `Mimir` entries in the repository's current state — those are Module 20/21 content, not built or explained here.

## 2. Exploring Logs with LogQL

After a `terraform apply` and a refresh of Grafana's **Connections → Data sources** page, the new `Loki` data source appears. From Grafana's **Explore** view, the lesson runs a first label-based query (`{app="manual-test"}`-style selector) against the manually-pushed test logs from Lesson 6, confirming they're indexed and queryable — with a short indexing delay on first ingestion being normal, since Loki has to flush and register the chunk before it's searchable.

---

# Lesson 8: Shipping Cluster Logs with Fluent Bit

## 1. Fluent Bit as a Multicluster `ApplicationSet`

To get real application logs — not just manually-pushed test data — into Loki, Fluent Bit is deployed as a DaemonSet on both workload clusters via an Argo CD `ApplicationSet`, mounting the host's container log directory and shipping everything it finds to the Loki gateway's internal DNS name:

```yaml
# fluent.yml
---
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: fluent-bit
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - cluster: linuxtips-cluster-01
            shard: "01"
          - cluster: linuxtips-cluster-02
            shard: "02"
  template:
    metadata:
      name: fluent-bit-{{shard}}
    spec:
      project: "default"
      source:
        repoURL: 'https://fluent.github.io/helm-charts'
        chart: fluent-bit
        targetRevision: "0.48.10"
        helm:
          releaseName: fluent-bit
          valuesObject:
            serviceAccount:
              create: true
              name: fluent-bit
            config:
              service: |
                [SERVICE]
                    HTTP_Server  On
                    HTTP_Listen  0.0.0.0
                    HTTP_PORT    2020
                    Flush        1
                    Log_Level    info
                    Parsers_File parsers.conf
              inputs: |
                [INPUT]
                    Name              tail
                    Path              /var/log/containers/*.log
                    Parser            cri
                    Tag               kube.*
                    Mem_Buf_Limit     50MB
                    Skip_Long_Lines   On
                    Refresh_Interval  10
              filters: |
                [FILTER]
                    Name                kubernetes
                    Match               kube.*
                    Kube_URL            https://kubernetes.default.svc:443
                    Merge_Log           On
                    K8S-Logging.Parser  On
                    K8S-Logging.Exclude Off
              outputs: |
                [OUTPUT]
                    Name              loki
                    Match             kube.*
                    Host              loki.linuxtips-observability.local
                    Port              80
                    tls               off
                    tls.verify        off        
                    Labels            cluster=teste,job=fluentbit,namespace=$kubernetes['namespace_name'],pod=$kubernetes['pod_name'],container=$kubernetes['container_name']
      destination:
        name: '{{ cluster }}'
        namespace: fluentbit
      syncPolicy:
        syncOptions:
          - CreateNamespace=true
        automated: {}
```

`fluent-bit`'s native `loki` output plugin talks to Loki's `Host`/`Port` directly (no separate Loki client library needed), and attaches four labels to every log line: `cluster`, `job`, `namespace`, and `pod`/`container`, pulled from Fluent Bit's own Kubernetes metadata filter.

## 2. From a Static `cluster` Label to a Per-Cluster One

The `Labels` line above hardcodes `cluster=teste` — a literal string, not a template placeholder — so every log line from both `linuxtips-cluster-01` and `linuxtips-cluster-02` lands in Loki tagged with the same `cluster` value, making it impossible to filter logs by which cluster actually produced them. The lesson calls this out directly and fixes it by applying an equivalent manifest through Terraform on the **control-plane** cluster instead, correctly interpolating the `ApplicationSet`'s own `{{cluster}}` template variable into the label:

```hcl
# argo_fluentbit.tf (linuxtips-eks-multicluster-management, control-plane stack)
resource "kubectl_manifest" "fluentbit" {
  yaml_body = <<YAML
# ... same ApplicationSet as fluent.yml above, except:
              outputs: |
                [OUTPUT]
                    Name              loki
                    Match             kube.*
                    Host              loki.linuxtips-observability.local
                    Port              80
                    tls               off
                    tls.verify        off        
                    Labels            cluster={{cluster}},job=fluentbit,namespace=$kubernetes['namespace_name'],pod=$kubernetes['pod_name'],container=$kubernetes['container_name']
YAML

  depends_on = [
    helm_release.argocd
  ]
}
```

> **Note:** `argo_fluentbit.tf` already existed in the `linuxtips-eks-multicluster-management` repository's control-plane stack as of Module 18, flagged there as forward-looking scaffolding pointing at an observability endpoint that didn't exist yet. This lesson is the payoff — `loki.linuxtips-observability.local` now exists, so this Terraform-managed `ApplicationSet` is what's actually running, superseding the standalone `fluent.yml` shown above (which is left in the observability-cluster repository as the "before" example the lesson walks through, not the final applied manifest).

## 3. Filtering and Correlating Logs in Grafana

With Fluent Bit running as `DaemonSet`s on both clusters (visible as `fluent-bit-01`/`fluent-bit-02` Argo CD Applications), logs start flowing into Loki labeled by `namespace`, `pod`, `container`, and now correctly by `cluster`. From Grafana's Explore view, the lesson filters down to the `chip` application's `nutrition`/app namespace, watches its health-check and `/events` traffic arrive live, and builds a simple table panel filtering by `service_name` — demonstrating the same label-based query model described in Lesson 1, now against real, continuously-flowing application logs instead of manually-pushed test data.

---

# Lesson 9: Segregating Capacity with Dedicated NodePools

## 1. Dedicated `grafana` and `loki` NodePools

Up to this point, every pod in this module scheduled onto the same shared `general` Karpenter NodePool. This lesson adds two more entries to `karpenter_capacity`, dedicated to Grafana and Loki respectively — plus `tempo` and `mimir` entries for the modules still to come:

```hcl
# environment/prod/terraform.tfvars
karpenter_capacity = [
  {
    name               = "general"
    workload           = "general"
    ami_family         = "Bottlerocket"
    ami_ssm            = "/aws/service/bottlerocket/aws-k8s-1.31/x86_64/latest/image_id"
    instance_family    = ["t3", "t3a", "c6", "c6a", "c7", "c7a"]
    instance_sizes     = ["large", "xlarge", "2xlarge"]
    capacity_type      = ["spot", "on-demand"]
    availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]
  },
  {
    name               = "grafana"
    workload           = "grafana"
    ami_family         = "Bottlerocket"
    ami_ssm            = "/aws/service/bottlerocket/aws-k8s-1.31/x86_64/latest/image_id"
    instance_family    = ["t3", "t3a", "c6", "c6a", "c7", "c7a"]
    instance_sizes     = ["large", "xlarge", "2xlarge"]
    capacity_type      = ["spot", "on-demand"]
    availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]
  },
  {
    name               = "loki"
    workload           = "loki"
    ami_family         = "Bottlerocket"
    ami_ssm            = "/aws/service/bottlerocket/aws-k8s-1.31/x86_64/latest/image_id"
    instance_family    = ["t3", "t3a", "c6", "c6a", "c7", "c7a"]
    instance_sizes     = ["large", "xlarge", "2xlarge"]
    capacity_type      = ["spot", "on-demand"]
    availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]
  },
  {
    name     = "tempo"
    workload = "tempo"
    # ... same shape as above
  },
  {
    name     = "mimir"
    workload = "mimir"
    # ... same shape as above
  },
]
```

> **Note:** the `tempo` and `mimir` entries are already present in this tfvars file, consistent with the Overview's flag that this repository is written once for the whole three-module observability build — they provision real NodePools with no workload targeting them yet, since Tempo and Mimir aren't installed until Modules 20-21.

Since each list entry maps 1:1 to a `templatefile()`-rendered `NodePool`/`EC2NodeClass` pair (Lesson 2's `karpenter.tf`), adding `grafana` and `loki` entries here is the entire mechanism — Karpenter labels every node it provisions from a NodePool named `loki` with `karpenter.sh/nodepool: loki` automatically, which is exactly the label the `nodeSelector`s below target.

## 2. Pinning Every Component with `nodeSelector`

With the two new NodePools available, every Loki and Grafana component gets a `nodeSelector` added, already shown inline in Lessons 3 and 5's `locals.tf` blocks:

```yaml
# locals.tf — grafana.values
nodeSelector:
    karpenter.sh/nodepool: grafana
```

```yaml
# locals.tf — loki.values, one nodeSelector per component
backend:
    nodeSelector:
        karpenter.sh/nodepool: loki
read:
    nodeSelector:
        karpenter.sh/nodepool: loki
write:
    nodeSelector:
        karpenter.sh/nodepool: loki
gateway:
    nodeSelector:
        karpenter.sh/nodepool: loki
```

This is the pattern the lesson explicitly says will repeat for every future component (Tempo in Module 20, Mimir in Module 21): give each observability workload its own dedicated NodePool, rather than letting logs, traces, and metrics compete for capacity on one shared pool.

## 3. Verifying the Split with `kubectl get nodeclaims`

After a `terraform apply` to create the new NodePools and a redeploy of the Grafana and Loki Helm releases to pick up the new `nodeSelector`s, `kubectl get nodeclaims` on the observability cluster shows Karpenter provisioning distinct nodes per workload — some backing the `grafana` NodePool, others backing `loki` — confirming Grafana and Loki pods no longer share capacity with each other or with the `general` pool.

## Key Takeaways

- **This is one repository, built across three modules.** `linuxtips-eks-observability-cluster`'s `main` branch already contains Tempo and Mimir Terraform/Helm files, an OpenTelemetry Collector `ApplicationSet`, and Tempo/Mimir entries in `locals.tf`, `route53.tf`, and `karpenter_capacity` — none of it built or explained by these 9 lessons, all of it flagged inline as scaffolding for Modules 20-21.
- **Loki trades full-text search for cheap, scalable label-based indexing** — the same Prometheus-style mental model (index labels, not content), queried with LogQL, storing compressed chunks in S3 rather than a full inverted index.
- **The observability cluster reuses Module 18's control-plane bootstrap almost verbatim** — same EKS/IAM/Karpenter/Pod-Identity/ALB-controller pattern, with Argo CD, ChartMuseum, and the `system` add-ons stripped out — but the copy left real dead weight behind: an unused Fargate access entry/role, and an `clusters_configs` variable nothing reads.
- **Fluent Bit's real, activated log-shipping pipeline lives outside this repository.** The standalone `fluent.yml` shown in Lesson 8 hardcodes `cluster=teste` for every log line; the actually-applied `ApplicationSet` is `argo_fluentbit.tf` in `linuxtips-eks-multicluster-management`'s control-plane stack — flagged as forward-looking scaffolding back in Module 18, now the module that activates it, correctly interpolating `cluster={{cluster}}`.
- **Every observability component gets its own Karpenter NodePool.** Lesson 9 establishes the pattern — `grafana`, `loki`, and (already provisioned, not yet used) `tempo`/`mimir` NodePools — that the next two modules will simply extend rather than redesign.
- **`auth_enabled: false` and the wide-open Grafana/EFS security groups are lab-appropriate, not production defaults** — Loki's multi-tenancy is disabled outright rather than configured, and both `aws_security_group.grafana` and `aws_security_group.efs` allow all ports/protocols from `0.0.0.0/0`, wider than the ports each service actually needs.
