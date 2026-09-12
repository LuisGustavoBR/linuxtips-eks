# Module 18: ArgoCD Multicluster Architecture

## Overview

This module is the first step toward the course's final project: a corporate-grade, multicluster EKS architecture. Instead of the single-cluster pattern used throughout Modules 1-17, this module federates **three clusters** behind a single Argo CD control plane:

- A dedicated **control-plane cluster** that runs Argo CD, ChartMuseum, and every "system" add-on (Argo Rollouts, Metrics Server, KEDA, and — reserved for a future observability module — Fluent Bit, the OpenTelemetry Collector, and Prometheus).
- Two identical **workload clusters** (`linuxtips-cluster-01` and `linuxtips-cluster-02`) that run the actual applications, registered with Argo CD as remote clusters and authenticated via a cross-account/cross-cluster IAM role chain.
- A shared **ingress layer**: one Application Load Balancer with two weighted target groups, one per workload cluster, so traffic can be split active-active between clusters or shifted entirely off one cluster (e.g. for maintenance or a version upgrade) without any application downtime.

Everything Argo CD applies to the workload clusters — shared add-ons and the `chip` application itself — is expressed as `ApplicationSet`s with a `list` generator enumerating the two clusters, so a single manifest fans out to both clusters at once and any future cluster is just one more `elements` entry away.

All code for this module lives in a **new, dedicated repository**, `linuxtips-eks-multicluster-management`, on branch `main` — a clean break from `linuxtips-eks-vanilla`, which was the companion repo for Modules 1-17. The repository is organized as three independent Terraform stacks — `ingress/`, `clusters/`, `control-plane/` — plus root-level `apply.sh`/`destroy.sh` orchestration scripts that `cd` into each stack in dependency order (ingress → cluster 01 → cluster 02 → control-plane) and run `terraform init`/`apply` (or `destroy`, in reverse order) on each.

`apply.sh` and `destroy.sh` reference `environment/prod/**/backend.tfvars` and `environment/prod/**/terraform.tfvars` paths inside each stack — these `environment/` directories and their real `.tfvars` values are committed in the repository (the `.gitignore` lines that would normally exclude `*.tfvars` are commented out here), so the real values populating every stack's variables are quoted alongside each stack below.

## Table of Contents

- [Lesson 1: Introduction to the Multicluster Final Project](#lesson-1-introduction-to-the-multicluster-final-project)
- [Lesson 2: Bootstrapping the Three-Stack Terraform Project](#lesson-2-bootstrapping-the-three-stack-terraform-project)
  - [1. A New, Dedicated Repository](#1-a-new-dedicated-repository)
  - [2. Three Independent Stacks: `ingress`, `clusters`, `control-plane`](#2-three-independent-stacks-ingress-clusters-control-plane)
  - [3. Shared Boilerplate: Backend, Variables, Providers, KMS](#3-shared-boilerplate-backend-variables-providers-kms)
  - [4. The `apply.sh`/`destroy.sh` Orchestration Scripts](#4-the-applyshdestroysh-orchestration-scripts)
  - [5. The Real `environment/` tfvars Files](#5-the-real-environment-tfvars-files)
- [Lesson 3: Building the Shared Ingress Layer](#lesson-3-building-the-shared-ingress-layer)
  - [1. Security Group and Application Load Balancer](#1-security-group-and-application-load-balancer)
  - [2. Two Target Groups, One per Cluster](#2-two-target-groups-one-per-cluster)
  - [3. Weighted Forwarding Between Target Groups](#3-weighted-forwarding-between-target-groups)
  - [4. Taking a Cluster Out of Rotation](#4-taking-a-cluster-out-of-rotation)
- [Lesson 4: Adding HTTPS to the Shared Load Balancer](#lesson-4-adding-https-to-the-shared-load-balancer)
  - [1. Route 53 Wildcard DNS Record](#1-route-53-wildcard-dns-record)
  - [2. ACM Certificate and DNS Validation](#2-acm-certificate-and-dns-validation)
  - [3. HTTPS Listener on Port 443](#3-https-listener-on-port-443)
- [Lesson 5: Publishing Target Groups via SSM Parameter Store](#lesson-5-publishing-target-groups-via-ssm-parameter-store)
- [Lesson 6: Provisioning the Two Workload Clusters](#lesson-6-provisioning-the-two-workload-clusters)
  - [1. Variables and Data Sources Shared with Ingress](#1-variables-and-data-sources-shared-with-ingress)
  - [2. IAM Roles for the Cluster, Nodes, and Fargate](#2-iam-roles-for-the-cluster-nodes-and-fargate)
  - [3. The EKS Cluster and a Temporary Node Group](#3-the-eks-cluster-and-a-temporary-node-group)
  - [4. Add-ons: VPC CNI, kube-proxy, Pod Identity Agent](#4-add-ons-vpc-cni-kube-proxy-pod-identity-agent)
- [Lesson 7: Installing the AWS Load Balancer Controller via Pod Identity](#lesson-7-installing-the-aws-load-balancer-controller-via-pod-identity)
- [Lesson 8: Installing Karpenter and Defining NodePool Capacity](#lesson-8-installing-karpenter-and-defining-nodepool-capacity)
- [Lesson 9: Installing Istio and Exposing It Through the Shared ALB](#lesson-9-installing-istio-and-exposing-it-through-the-shared-alb)
  - [1. Target Group Binding for the Ingress Gateway](#1-target-group-binding-for-the-ingress-gateway)
  - [2. A "Mock" Gateway and VirtualService to Open Envoy's Port](#2-a-mock-gateway-and-virtualservice-to-open-envoys-port)
- [Lesson 10: Provisioning the Control Plane Cluster](#lesson-10-provisioning-the-control-plane-cluster)
- [Lesson 11: Installing Argo CD on the Control Plane Cluster](#lesson-11-installing-argo-cd-on-the-control-plane-cluster)
- [Lesson 12: Exposing the Argo CD Dashboard with a Dedicated Load Balancer](#lesson-12-exposing-the-argo-cd-dashboard-with-a-dedicated-load-balancer)
- [Lesson 13: Federating Clusters with IAM Roles and Access Entries](#lesson-13-federating-clusters-with-iam-roles-and-access-entries)
  - [1. The `argocd` Role, Assumed via Pod Identity](#1-the-argocd-role-assumed-via-pod-identity)
  - [2. The `argocd-deployer` Role, Assumed Cross-Cluster](#2-the-argocd-deployer-role-assumed-cross-cluster)
  - [3. Registering Member Clusters as Argo CD Cluster Secrets](#3-registering-member-clusters-as-argo-cd-cluster-secrets)
  - [4. Authorizing the Deployer Role on Each Member Cluster](#4-authorizing-the-deployer-role-on-each-member-cluster)
- [Lesson 14: Managing Shared Add-ons as Multicluster ApplicationSets](#lesson-14-managing-shared-add-ons-as-multicluster-applicationsets)
  - [1. The `system` AppProject](#1-the-system-appproject)
  - [2. Argo Rollouts, Metrics Server, and KEDA as ApplicationSets](#2-argo-rollouts-metrics-server-and-keda-as-applicationsets)
  - [3. From Dashboard-Applied ApplicationSet to Terraform-Managed `kubectl_manifest`](#3-from-dashboard-applied-applicationset-to-terraform-managed-kubectl_manifest)
- [Lesson 15: Deploying an Application Active-Active Across Clusters](#lesson-15-deploying-an-application-active-active-across-clusters)
  - [1. ChartMuseum on the Control Plane Cluster](#1-chartmuseum-on-the-control-plane-cluster)
  - [2. The `chip` ApplicationSet Across Both Clusters](#2-the-chip-applicationset-across-both-clusters)
  - [3. Canary Rollouts and Cluster Failover in Practice](#3-canary-rollouts-and-cluster-failover-in-practice)
- [Key Takeaways](#key-takeaways)

---

# Lesson 1: Introduction to the Multicluster Final Project

## 1. From Single-Cluster Modules to a Federated Architecture

Every previous module built and evolved one EKS cluster. This module takes the first step toward the course's final project: a centralized **control-plane cluster** running Argo CD, used to federate deployments across multiple production/staging workload clusters — instead of applying manifests to each cluster by hand.

## 2. The Target Architecture

- A control-plane cluster running Argo CD (using the ChartMuseum pattern from Module 17) applies `Application`/`ApplicationSet` resources that get deployed onto every workload cluster registered with it.
- Two workload clusters (`linuxtips-cluster-01`, `linuxtips-cluster-02`) host the actual applications and shared add-ons pushed from the control plane.
- An Application Load Balancer sits in front of both workload clusters with **two target groups**, one per cluster, so traffic can be weighted between them — enabling zero-downtime maintenance: shift 100% of traffic to one cluster, patch/upgrade the other, then shift back and repeat for the first.

This lesson only delivers the three clusters and the routing layer; a later module (once the course reaches the observability build) will extend the load-balancer/traffic-shifting strategy further.

> **Note:** this module's lessons move fast and lean heavily on copy-pasting Terraform already built in earlier modules — much of `clusters/` and `control-plane/` is a near-verbatim rebuild of the single-cluster setup from Modules 1-17, adapted to run three times over. This README documents each file exactly as it exists in the repository's `main` branch, not as an incremental diff, since (as in Modules 15-17) the repository carries no intermediate per-lesson history — `main` already reflects every lesson's cumulative changes.

---

# Lesson 2: Bootstrapping the Three-Stack Terraform Project

## 1. A New, Dedicated Repository

This module's code moves out of `linuxtips-eks-vanilla` entirely and into a new repository, `linuxtips-eks-multicluster-management`, built from scratch.

## 2. Three Independent Stacks: `ingress`, `clusters`, `control-plane`

The repository root holds three Terraform stacks, each with its own backend and state file:

```
ingress/            # shared ALB, target groups, weighted routing, optional ACM/HTTPS
clusters/           # reusable EKS + Karpenter + Istio + LB Controller stack, applied once per workload cluster
control-plane/      # the Argo CD / GitOps management cluster (ChartMuseum, Argo CD, Argo-managed add-ons)
```

`clusters/` is applied twice — once per workload cluster — using the same codebase with a different backend state and a different `terraform.tfvars` file per cluster (`cluster-01`, `cluster-02`), rather than duplicating the Terraform code itself.

## 3. Shared Boilerplate: Backend, Variables, Providers, KMS

Each of the three stacks starts with the same skeleton: `backend.tf`, `variables.tf`, `providers.tf`, and `kms.tf`. The backend is an empty S3 backend block, configured at `init` time via a `-backend-config` file:

```hcl
# ingress/backend.tf (identical in clusters/ and control-plane/)
terraform {
  backend "s3" {

  }
}
```

```hcl
# ingress/providers.tf
provider "aws" {
  region = var.region
}
```

Every stack also gets its own KMS key, used purely to validate that `apply.sh` is correctly looping over all three stacks:

```hcl
# ingress/kms.tf (identical in clusters/ and control-plane/)
resource "aws_kms_key" "main" {
  description = var.project_name
}

resource "aws_kms_alias" "main" {
  name          = format("alias/%s", var.project_name)
  target_key_id = aws_kms_key.main.key_id
}
```

## 4. The `apply.sh`/`destroy.sh` Orchestration Scripts

Two scripts at the repository root drive all three stacks in the correct order — `apply.sh` bottom-up (ingress → cluster 01 → cluster 02 → control-plane), `destroy.sh` top-down (control-plane → cluster 02 → cluster 01 → ingress):

```bash
# apply.sh
#!/bin/bash

cd ingress/
echo "Setup do Ingress"
rm -rf  .terraform
terraform init -backend-config=environment/prod/backend.tfvars
terraform apply -var-file=environment/prod/terraform.tfvars --auto-approve

cd ../clusters
echo "Setup do Cluster 01"
rm -rf  .terraform
terraform init -backend-config=environment/prod/cluster-01/backend.tfvars
terraform apply -var-file=environment/prod/cluster-01/terraform.tfvars --auto-approve

echo "Setup do Cluster 02"
rm -rf  .terraform
terraform init -backend-config=environment/prod/cluster-02/backend.tfvars
terraform apply -var-file=environment/prod/cluster-02/terraform.tfvars --auto-approve

cd ../control-plane
rm -rf  .terraform
terraform init -backend-config=environment/prod/backend.tfvars
terraform apply -var-file=environment/prod/terraform.tfvars --auto-approve
```

`destroy.sh` mirrors this exactly, but tears the control plane down first (so nothing keeps re-registering member clusters while they're being destroyed) and the ingress layer last.

## 5. The Real `environment/` tfvars Files

Each stack's `environment/prod/` directory holds the real `backend.tfvars` (S3 bucket/key/region for that stack's state) and `terraform.tfvars` (the actual variable values) that `apply.sh`/`destroy.sh` reference. All four are quoted here in full, since later lessons build on their exact values:

```hcl
# ingress/environment/prod/backend.tfvars
bucket = "linuxtips-s3-eks-state-files"
key    = "eks/multicluster/prod/ingress/state"
region = "us-east-1"
```

```hcl
# ingress/environment/prod/terraform.tfvars
project_name = "linuxtips-ingress"


ssm_vpc = "/linuxtips-vpc/vpc/id"

ssm_subnets = [
  "/linuxtips-vpc/subnets/public/us-east-1a/linuxtips-public-1a",
  "/linuxtips-vpc/subnets/public/us-east-1b/linuxtips-public-1b",
  "/linuxtips-vpc/subnets/public/us-east-1c/linuxtips-public-1c",
]

routing_weight = {
  cluster_01 = 50
  cluster_02 = 50
}

route53_hosted_zone = "Z102505525LUE9SZ7HWTY"
dns_name            = "*.luisgustavo.com.br"
```

```hcl
# clusters/environment/prod/cluster-01/backend.tfvars
bucket = "linuxtips-s3-eks-state-files"
key    = "eks/multicluster/prod/cluster-01/state"
region = "us-east-1"
```

```hcl
# clusters/environment/prod/cluster-01/terraform.tfvars
project_name = "linuxtips-cluster-01"

k8s_version = "1.32"

ssm_vpc = "/linuxtips-vpc/vpc/id"

ssm_subnets = [
  "/linuxtips-vpc/subnets/private/us-east-1a/linuxtips-pods-1a",
  "/linuxtips-vpc/subnets/private/us-east-1b/linuxtips-pods-1b",
  "/linuxtips-vpc/subnets/private/us-east-1c/linuxtips-pods-1c",
]

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
]

istio_ssm_target_group = "/linuxtips-ingress/cluster-01/listener"
```

`cluster-02`'s files are identical except for `project_name` (`linuxtips-cluster-02`), the state `key` (`eks/multicluster/prod/cluster-02/state`), and `istio_ssm_target_group` (`/linuxtips-ingress/cluster-02/listener`) — proof that the two workload clusters really are the same codebase applied twice with only the per-cluster identity/target-group values swapped:

```hcl
# clusters/environment/prod/cluster-02/backend.tfvars
bucket = "linuxtips-s3-eks-state-files"
key    = "eks/multicluster/prod/cluster-02/state"
region = "us-east-1"
```

```hcl
# clusters/environment/prod/cluster-02/terraform.tfvars (excerpt — differences from cluster-01 only)
project_name = "linuxtips-cluster-02"

istio_ssm_target_group = "/linuxtips-ingress/cluster-02/listener"
```

```hcl
# control-plane/environment/prod/backend.tfvars
bucket = "linuxtips-s3-eks-state-files"
key    = "eks/multicluster/prod/control-plane/state"
region = "us-east-1"
```

```hcl
# control-plane/environment/prod/terraform.tfvars
project_name = "linuxtips-control-plane"

k8s_version = "1.32"

ssm_vpc = "/linuxtips-vpc/vpc/id"

ssm_subnets = [
  "/linuxtips-vpc/subnets/private/us-east-1a/linuxtips-pods-1a",
  "/linuxtips-vpc/subnets/private/us-east-1b/linuxtips-pods-1b",
  "/linuxtips-vpc/subnets/private/us-east-1c/linuxtips-pods-1c",
]

ssm_lb_subnets = [
  "/linuxtips-vpc/subnets/public/us-east-1a/linuxtips-public-1a",
  "/linuxtips-vpc/subnets/public/us-east-1b/linuxtips-public-1b",
  "/linuxtips-vpc/subnets/public/us-east-1c/linuxtips-public-1c",
]

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
]
```

> **Note:** `control-plane/environment/prod/terraform.tfvars` sets no `clusters_configs` value, so that stack relies entirely on the variable's own default (`linuxtips-cluster-01`/`linuxtips-cluster-02`) shown in Lesson 10 — the two member clusters Argo CD federates in Lesson 13 are never actually named in this tfvars file.

---

# Lesson 3: Building the Shared Ingress Layer

## 1. Security Group and Application Load Balancer

The `ingress/` stack pulls the VPC and subnet IDs it needs from SSM Parameter Store (the same pattern used to hand off networking values between stacks in earlier modules):

```hcl
# ingress/data.tf
data "aws_ssm_parameter" "vpc" {
  name = var.ssm_vpc
}

data "aws_ssm_parameter" "subnets" {
  count = length(var.ssm_subnets)
  name  = var.ssm_subnets[count.index]
}
```

```hcl
# ingress/sg.tf
resource "aws_security_group" "main" {
  name   = var.project_name
  vpc_id = data.aws_ssm_parameter.vpc.value

  ingress {
    from_port = 0
    to_port   = 0
    protocol  = "-1"
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

  tags = {
    Name = var.project_name
  }
}
```

```hcl
# ingress/main.tf
resource "aws_lb" "main" {
  name = var.project_name

  internal = false

  load_balancer_type = "application"

  subnets = data.aws_ssm_parameter.subnets[*].value

  enable_cross_zone_load_balancing = true
  enable_deletion_protection       = false

  security_groups = [
    aws_security_group.main.id
  ]

  tags = {
    Name = var.project_name
  }
}
```

> **Note:** the ingress security group opens `0.0.0.0/0` on all ports/protocols in both directions — deliberately permissive for the lab, called out live in the lesson as "por facilidade" (for convenience). Tighten this before using the pattern outside a lab.

## 2. Two Target Groups, One per Cluster

Two identical target groups are created — one per workload cluster — health-checked on `/` with a 200-404 matcher, since an unconfigured Istio ingress gateway legitimately answers with a 404 until a route exists:

```hcl
# ingress/tg.tf
resource "aws_lb_target_group" "cluster_01" {
  name     = format("%s-%s", var.project_name, "01")
  port     = 30080
  protocol = "HTTP"

  vpc_id = data.aws_ssm_parameter.vpc.value

  health_check {
    path    = "/"
    matcher = "200-404"
  }
}

resource "aws_lb_target_group" "cluster_02" {
  name     = format("%s-%s", var.project_name, "02")
  port     = 30080
  protocol = "HTTP"

  vpc_id = data.aws_ssm_parameter.vpc.value

  health_check {
    path    = "/"
    matcher = "200-404"
  }
}
```

## 3. Weighted Forwarding Between Target Groups

A `routing_weight` object variable drives a single weighted `forward` action on the HTTP listener, splitting traffic between the two target groups:

```hcl
# ingress/variables.tf (excerpt)
variable "routing_weight" {
  type = object({
    cluster_01 = number
    cluster_02 = number
  })
}
```

```hcl
# ingress/listener.tf
resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.main.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type = "forward"

    forward {
      target_group {
        arn    = aws_lb_target_group.cluster_01.arn
        weight = lookup(var.routing_weight, "cluster_01")
      }
      target_group {
        arn    = aws_lb_target_group.cluster_02.arn
        weight = lookup(var.routing_weight, "cluster_02")
      }
    }
  }
}
```

With `routing_weight = { cluster_01 = 50, cluster_02 = 50 }`, the listener splits traffic 50/50 between both clusters — active-active from day one.

## 4. Taking a Cluster Out of Rotation

Because the weight lives in a Terraform variable, shifting all traffic off a cluster is a one-line change and an `apply`: setting `cluster_01 = 100, cluster_02 = 0` immediately drains cluster 2 (and vice-versa), letting a whole cluster be upgraded, patched, or replaced without any customer-visible downtime.

---

# Lesson 4: Adding HTTPS to the Shared Load Balancer

## 1. Route 53 Wildcard DNS Record

An optional step (already covered several times earlier in the course): a wildcard `A`/alias record pointed at the shared load balancer.

```hcl
# ingress/route53.tf
resource "aws_route53_record" "dns" {
  zone_id = var.route53_hosted_zone
  name    = var.dns_name

  type = "A"

  alias {
    evaluate_target_health = true
    name                   = aws_lb.main.dns_name
    zone_id                = aws_lb.main.zone_id
  }
}
```

## 2. ACM Certificate and DNS Validation

```hcl
# ingress/acm.tf
resource "aws_acm_certificate" "main" {
  domain_name       = var.dns_name
  validation_method = "DNS"

  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_route53_record" "main" {
  for_each = {
    for dvo in aws_acm_certificate.main.domain_validation_options : dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }

  allow_overwrite = true
  name            = each.value.name
  records         = [each.value.record]
  ttl             = 60
  type            = each.value.type
  zone_id         = var.route53_hosted_zone
}

resource "aws_acm_certificate_validation" "main" {
  certificate_arn         = aws_acm_certificate.main.arn
  validation_record_fqdns = [for record in aws_route53_record.main : record.fqdn]
}
```

## 3. HTTPS Listener on Port 443

The HTTPS listener reuses the exact same weighted `forward` block as the HTTP listener on port 80, just with the certificate and a modern SSL policy attached:

```hcl
# ingress/listener.tf (excerpt)
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.main.arn
  port              = 443
  protocol          = "HTTPS"

  ssl_policy = "ELBSecurityPolicy-2016-08"

  certificate_arn = aws_acm_certificate.main.arn

  default_action {
    type = "forward"

    forward {
      target_group {
        arn    = aws_lb_target_group.cluster_01.arn
        weight = lookup(var.routing_weight, "cluster_01")
      }
      target_group {
        arn    = aws_lb_target_group.cluster_02.arn
        weight = lookup(var.routing_weight, "cluster_02")
      }
    }
  }
}
```

Both listeners route to the same two target groups with the same weights, so no separate HTTP/HTTPS target-group binding work is ever needed downstream.

---

# Lesson 5: Publishing Target Groups via SSM Parameter Store

The last step of the ingress stack publishes each target group's ARN to SSM Parameter Store, so the `clusters/` stack can look it up later and perform the target group binding without any hardcoded cross-stack references:

```hcl
# ingress/parameters.tf
resource "aws_ssm_parameter" "cluster_01" {
  type  = "String"
  name  = "/${var.project_name}/cluster-01/listener"
  value = aws_lb_target_group.cluster_01.arn
}

resource "aws_ssm_parameter" "cluster_02" {
  type  = "String"
  name  = "/${var.project_name}/cluster-02/listener"
  value = aws_lb_target_group.cluster_02.arn
}
```

```hcl
# ingress/output.tf
output "ssm_target_group_01" {
  value = aws_ssm_parameter.cluster_01.id
}

output "ssm_target_group_02" {
  value = aws_ssm_parameter.cluster_02.id
}
```

> **Note:** the file is named `output.tf` (singular) in this stack, while `clusters/` and `control-plane/` each ship an (empty) `outputs.tf` (plural) — a naming inconsistency across the three stacks, not a typo introduced by this README.

With the ingress layer complete, the module moves on to the two workload clusters that these target groups will bind to.

---

# Lesson 6: Provisioning the Two Workload Clusters

## 1. Variables and Data Sources Shared with Ingress

The `clusters/` stack reuses the same `ssm_vpc`/`ssm_subnets` pattern as `ingress/`, but points at the **private** subnets instead of the public ones the load balancer uses, plus a `karpenter_capacity` variable (reused verbatim from the Karpenter module) that drives every NodePool/EC2NodeClass pair created later:

```hcl
# clusters/variables.tf
variable "k8s_version" {
  default = "1.32"
}

variable "ssm_vpc" {}

variable "ssm_subnets" {
  type = list(string)
}

variable "node_group_temp_desired" {
  type    = number
  default = 2
}

variable "karpenter_capacity" {
  type = list(object({
    name               = string
    workload           = string
    ami_family         = string
    ami_ssm            = string
    instance_family    = list(string)
    instance_sizes     = list(string)
    capacity_type      = list(string)
    availability_zones = list(string)
  }))
}
```

## 2. IAM Roles for the Cluster, Nodes, and Fargate

Three standard IAM roles are created — one for the EKS control plane, one for the managed node group, one for Fargate — identical in shape to earlier modules:

```hcl
# clusters/iam_cluster.tf
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
```

## 3. The EKS Cluster and a Temporary Node Group

```hcl
# clusters/eks.tf
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

A small, temporary Bottlerocket Spot node group bootstraps the cluster just far enough to run the initial add-ons before Karpenter takes over:

```hcl
# clusters/nodes_temp.tf
resource "aws_eks_node_group" "main" {
  cluster_name    = aws_eks_cluster.main.id
  node_group_name = aws_eks_cluster.main.id

  node_role_arn = aws_iam_role.eks_nodes_role.arn

  instance_types = ["t3a.large"]

  subnet_ids = data.aws_ssm_parameter.subnets[*].value

  scaling_config {
    desired_size = var.node_group_temp_desired
    max_size     = var.node_group_temp_desired
    min_size     = var.node_group_temp_desired
  }

  capacity_type = "SPOT"

  labels = {
    "capacity/os"   = "BOTTLEROCKET"
    "capacity/arch" = "x86_64"
    "capacity/type" = "SPOT"
    "compute-type"  = "ec2"
  }

  tags = {
    "kubernetes.io/cluster/${var.project_name}" = "owned"
  }

  depends_on = [
    aws_eks_access_entry.nodes
  ]

  timeouts {
    create = "1h"
    update = "2h"
    delete = "2h"
  }
}
```

## 4. Add-ons: VPC CNI, kube-proxy, Pod Identity Agent

Each add-on resolves the most recent version compatible with the cluster's Kubernetes version via `most_recent = true`, rather than pinning a version by hand:

```hcl
# clusters/addons.tf (excerpt)
data "aws_eks_addon_version" "cni" {
  addon_name         = "vpc-cni"
  kubernetes_version = aws_eks_cluster.main.version
  most_recent        = true
}

resource "aws_eks_addon" "cni" {
  cluster_name = aws_eks_cluster.main.name
  addon_name   = "vpc-cni"

  addon_version               = data.aws_eks_addon_version.cni.version
  resolve_conflicts_on_create = "OVERWRITE"
  resolve_conflicts_on_update = "OVERWRITE"

  depends_on = [
    aws_eks_access_entry.nodes
  ]
}
```

> **Note:** the CoreDNS add-on is present in the file but entirely commented out, exactly as the lesson describes ("comento meu CoreDNS, que ele demora muito para subir" — commented out purely to save time while recording, not a permanent omission). The Pod Identity Agent add-on (`eks-pod-identity-agent`) is installed uncommented, since every controller from here on (LB Controller, Karpenter, ChartMuseum, Argo CD) authenticates via EKS Pod Identity rather than IRSA annotations.

Both clusters (`linuxtips-cluster-01`, `linuxtips-cluster-02`) are created from this same codebase, applied twice with different backend state and `terraform.tfvars` files.

---

# Lesson 7: Installing the AWS Load Balancer Controller via Pod Identity

Unlike earlier modules, which annotated the controller's service account with an IRSA role ARN, this module associates the IAM role with the service account via **EKS Pod Identity** instead:

```hcl
# clusters/iam_lb_controller.tf (excerpt)
resource "aws_iam_role" "aws_lb_controller" {
  assume_role_policy = data.aws_iam_policy_document.aws_lb_role.json
  name               = format("%s-aws-load-balancer", var.project_name)
}

resource "aws_eks_pod_identity_association" "aws_lb_controller" {
  cluster_name    = aws_eks_cluster.main.name
  namespace       = "kube-system"
  service_account = "aws-load-balancer-controller"
  role_arn        = aws_iam_role.aws_lb_controller.arn
}
```

The trust policy backing that role only allows the `pods.eks.amazonaws.com` service principal to assume it — the same Pod Identity pattern introduced for ChartMuseum in Module 17:

```hcl
# clusters/iam_lb_controller.tf (excerpt)
data "aws_iam_policy_document" "aws_lb_role" {
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
```

The Helm release itself no longer annotates a service account (Pod Identity replaces the annotation entirely):

```hcl
# clusters/helm_lb_controller.tf
resource "helm_release" "alb_ingress_controller" {
  name             = "aws-load-balancer-controller"
  repository       = "https://aws.github.io/eks-charts"
  chart            = "aws-load-balancer-controller"
  namespace        = "kube-system"
  create_namespace = true

  set = [
    { name = "clusterName", value = var.project_name },
    { name = "serviceAccount.create", value = "true" },
    { name = "serviceAccount.name", value = "aws-load-balancer-controller" },
    { name = "region", value = var.region },
    { name = "vpcId", value = data.aws_ssm_parameter.vpc.value }
  ]

  depends_on = [
    aws_eks_cluster.main,
  ]
}
```

---

# Lesson 8: Installing Karpenter and Defining NodePool Capacity

Karpenter is installed the same way as the AWS Load Balancer Controller — an IAM role trusted by `pods.eks.amazonaws.com`, associated to the `karpenter` service account via Pod Identity, then a Helm release with explicit CPU/memory requests for the controller:

```hcl
# clusters/helm_karpenter.tf
resource "helm_release" "karpenter" {
  namespace        = "karpenter"
  create_namespace = true

  name       = "karpenter"
  repository = "oci://public.ecr.aws/karpenter"
  chart      = "karpenter"
  version    = "1.3.3"

  set = [
    { name = "settings.clusterName", value = var.project_name },
    { name = "settings.clusterEndpoint", value = aws_eks_cluster.main.endpoint },
    { name = "aws.defaultInstanceProfile", value = aws_iam_instance_profile.nodes.name },
    { name = "controller.resources.requests.cpu", value = "1000m" },
    { name = "controller.resources.requests.memory", value = "1Gi" },
  ]

  depends_on = [
    aws_eks_cluster.main,
    aws_eks_node_group.main,
  ]
}
```

`EC2NodeClass`/`NodePool` pairs are generated from the same `karpenter_capacity` list/templated-YAML pattern built in the dedicated Karpenter module, just moved into this repository:

```hcl
# clusters/karpenter.tf
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

A minimal `general` capacity entry is enough to unblock the rest of the module — just enough NodePool for setup work, refined in later lessons as real workloads are scheduled.

---

# Lesson 9: Installing Istio and Exposing It Through the Shared ALB

Istio's `base`/`istiod`/`gateway` charts are installed exactly as in the Service Mesh module, with tracing already pointed at a Jaeger collector endpoint that doesn't exist on these clusters yet:

```hcl
# clusters/helm_istio.tf (excerpt)
resource "helm_release" "istiod" {
  name       = "istio"
  chart      = "istiod"
  repository = "https://istio-release.storage.googleapis.com/charts"
  namespace  = "istio-system"

  create_namespace = true
  version          = var.istio_version

  set = [
    { name = "sidecarInjectorWebhook.rewriteAppHTTPProbe", value = "false" },
    { name = "meshConfig.enableTracing", value = "true" },
    { name = "meshConfig.defaultConfig.tracing.zipkin.address", value = "jaeger-collector.tracing.svc.cluster.local:9411" }
  ]

  depends_on = [
    helm_release.istio_base
  ]
}
```

> **Note:** `jaeger-collector.tracing.svc.cluster.local` is not deployed anywhere in this module or repository — it's forward-looking configuration for the observability cluster the course builds in a later module. Until then, trace export from these clusters simply has nowhere to land.

## 1. Target Group Binding for the Ingress Gateway

The ingress gateway Service is bound directly to the target group ARN published by the `ingress/` stack via SSM, using the same `TargetGroupBinding` CRD pattern from the AWS Load Balancer Controller module:

```hcl
# clusters/helm_istio.tf (excerpt)
resource "kubectl_manifest" "target_binding_80" {
  yaml_body = <<YAML
apiVersion: elbv2.k8s.aws/v1beta1
kind: TargetGroupBinding
metadata:
  name: istio-ingress
  namespace: istio-system
spec:
  serviceRef:
    name: istio-ingressgateway
    port: 80
  targetGroupARN: ${data.aws_ssm_parameter.tg.value}
  targetType: instance
YAML
  depends_on = [
    helm_release.istio_ingress
  ]
}
```

## 2. A "Mock" Gateway and VirtualService to Open Envoy's Port

A freshly installed Istio ingress gateway only opens its listening port once at least one `Gateway`/`VirtualService` uses it — until then, the ALB's health checks see the target as unhealthy or time out entirely. A throwaway `Gateway` + `VirtualService` pair, returning a static 200 with no backend routing at all, exists purely to open that port:

```hcl
# clusters/helm_istio.tf (excerpt)
resource "kubectl_manifest" "mock_virtual_service" {
  yaml_body = <<YAML
apiVersion: networking.istio.io/v1
kind: VirtualService
metadata:
  name: mock-istio
  namespace: istio-system
spec:
  hosts:
  -  mock-istio.istio-system.svc.cluster.local
  http:
  - match:
    - uri:
        exact: /
    directResponse:
      status: 200
      body:
        string: "OK"
YAML
  depends_on = [
    helm_release.istio_ingress
  ]
}
```

With this in place, both target groups in the shared ALB report healthy, and `curl`-ing the ALB's DNS name returns Envoy's `404` — proof that requests are reaching Istio on both clusters, even before any real route exists.

Also worth calling out: `clusters/sg.tf` opens the cluster security group to all inbound traffic (`0.0.0.0/0`, all ports) — needed for the ALB's health checks to reach the NodePort behind Envoy, and flagged live in the lesson as an easy step to forget (it caused a timeout mid-recording until it was added).

With Karpenter, the AWS Load Balancer Controller, and Istio running identically on both workload clusters, the module turns to the control-plane cluster that will manage them.

---

# Lesson 10: Provisioning the Control Plane Cluster

The `control-plane/` stack's cluster is built from the exact same `eks.tf` as `clusters/` — copy-pasted, then trimmed of everything the control plane doesn't need: no Istio, no ingress-gateway target group binding, no `istio_ssm_target_group`/`istio_*` variables. What it keeps is the Karpenter capacity list, the AWS Load Balancer Controller, and the shared `backend.tf`/`kms.tf`/`sg.tf` boilerplate:

```hcl
# control-plane/variables.tf (excerpt — what differs from clusters/variables.tf)
variable "ssm_lb_subnets" {
  type = list(string)
}

variable "clusters_configs" {
  default = [
    { cluster_name = "linuxtips-cluster-01" },
    { cluster_name = "linuxtips-cluster-02" }
  ]
}
```

`clusters_configs` is new here — it's this stack's registry of every member cluster Argo CD needs to federate, used again in Lesson 13.

Once applied, `linuxtips-cluster-control-plane` shows up in the EKS console running Karpenter and the LB Controller, same as the two workload clusters — the difference is entirely in what gets deployed onto it next.

---

# Lesson 11: Installing Argo CD on the Control Plane Cluster

Argo CD is installed with the exact same `helm_release` as Module 17 — including the Rollout Extension enabled from day one:

```hcl
# control-plane/helm_argocd.tf (excerpt)
resource "helm_release" "argocd" {
  name             = "argocd"
  namespace        = "argocd"
  create_namespace = true

  repository = "https://argoproj.github.io/argo-helm"
  chart      = "argo-cd"

  set = [
    { name = "server.extraArgs[0]", value = "--insecure" },
    { name = "server.service.type", value = "NodePort" },
    { name = "server.extensions.enabled", value = "true" },
    { name = "server.enable.proxy.extension", value = "true" },
    { name = "server.extensions.image.repository", value = "quay.io/argoprojlabs/argocd-extension-installer" },
    { name = "server.extensions.extensionList[0].name", value = "rollout-extension" },
    { name = "server.extensions.extensionList[0].env[0].name", value = "EXTENSION_URL" },
    { name = "server.extensions.extensionList[0].env[0].value", value = "https://github.com/argoproj-labs/rollout-extension/releases/download/v0.3.6/extension.tar" }
  ]

  depends_on = [
    aws_eks_cluster.main,
    helm_release.karpenter
  ]
}
```

`server.service.type = NodePort` (rather than `ClusterIP`, used in Module 17) is the one real difference — this control-plane cluster exposes the dashboard through its own dedicated ALB via target group binding, not an Istio Gateway, since Argo CD's own cluster doesn't run an in-mesh ingress gateway.

---

# Lesson 12: Exposing the Argo CD Dashboard with a Dedicated Load Balancer

Rather than sharing the same ALB as the workload clusters, the control-plane cluster gets its own dedicated Application Load Balancer — deployed in public subnets published via a new `ssm_lb_subnets` variable:

```hcl
# control-plane/lb_argocd.tf
resource "aws_lb" "main" {
  name = var.project_name

  internal           = false
  load_balancer_type = "application"

  subnets = data.aws_ssm_parameter.lb_subnets[*].value

  enable_cross_zone_load_balancing = true
  enable_deletion_protection       = false

  security_groups = [
    aws_security_group.main.id
  ]

  tags = {
    Name = var.project_name
  }
}

resource "aws_lb_target_group" "argo" {
  name     = var.project_name
  port     = 30080
  protocol = "HTTP"
  vpc_id   = data.aws_ssm_parameter.vpc.value

  health_check {
    path    = "/"
    matcher = "200-404"
  }
}

resource "aws_lb_listener" "argocd" {
  load_balancer_arn = aws_lb.main.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type = "forward"

    forward {
      target_group {
        arn = aws_lb_target_group.argo.arn
      }
    }
  }
}
```

The `argocd-server` Service is bound to that target group with the same `TargetGroupBinding` CRD used throughout the module:

```hcl
# control-plane/helm_argocd.tf (excerpt)
resource "kubectl_manifest" "argocd_target_group" {
  yaml_body = <<YAML
apiVersion: elbv2.k8s.aws/v1beta1
kind: TargetGroupBinding
metadata:
  name: argocd-server
  namespace: argocd
spec:
  serviceRef:
    name: argocd-server
    port: 80
  targetGroupARN: ${aws_lb_target_group.argo.arn}
  targetType: instance
YAML
  depends_on = [
    helm_release.argocd
  ]
}
```

Once the target is healthy, the dashboard is reachable at the load balancer's DNS name over plain HTTP. The initial admin password is retrieved exactly as in Module 17:

```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath='{.data.password}' | base64 -d
```

...followed by an immediate password change from the dashboard's **User Info → Update Password** screen.

---

# Lesson 13: Federating Clusters with IAM Roles and Access Entries

Registering the two workload clusters with this Argo CD instance requires a two-role IAM chain: a role Argo CD's own pods assume via Pod Identity, and a second role that role is allowed to assume *on each member cluster*.

## 1. The `argocd` Role, Assumed via Pod Identity

Associated with three service accounts at once — the Argo CD server, the Application Controller, and the ApplicationSet Controller — all of which need to sign requests to remote clusters:

```hcl
# control-plane/iam_argocd.tf
data "aws_iam_policy_document" "argocd_policy" {
  version = "2012-10-17"

  statement {
    effect = "Allow"
    actions = [
      "sts:AssumeRole",
      "sts:TagSession"
    ]

    resources = [
      "*"
    ]
  }
}

resource "aws_eks_pod_identity_association" "argo_server" {
  cluster_name    = aws_eks_cluster.main.name
  namespace       = "argocd"
  service_account = "argocd-server"
  role_arn        = aws_iam_role.argocd.arn
}

resource "aws_eks_pod_identity_association" "argo_application_controller" {
  cluster_name    = aws_eks_cluster.main.name
  namespace       = "argocd"
  service_account = "argocd-application-controller"
  role_arn        = aws_iam_role.argocd.arn
}

resource "aws_eks_pod_identity_association" "argo_applicationset_controller" {
  cluster_name    = aws_eks_cluster.main.name
  namespace       = "argocd"
  service_account = "argocd-applicationset-controller"
  role_arn        = aws_iam_role.argocd.arn
}
```

## 2. The `argocd-deployer` Role, Assumed Cross-Cluster

A second, deliberately minimal role — no IAM permissions of its own, just a trust policy naming the `argocd` role above as its only principal:

```hcl
# control-plane/iam_argocd_deployer.tf
data "aws_iam_policy_document" "argocd_deployer_assume_role" {
  statement {
    effect = "Allow"

    principals {
      type        = "AWS"
      identifiers = [aws_iam_role.argocd.arn]
    }

    actions = [
      "sts:AssumeRole",
      "sts:TagSession"
    ]
  }
}

resource "aws_iam_role" "argo_deployer" {
  assume_role_policy = data.aws_iam_policy_document.argocd_deployer_assume_role.json
  name               = format("%s-argocd-deployer", var.project_name)
}
```

This is the role that will actually show up as `roleARN` inside each member cluster's `aws-auth`/access entries — Argo CD authenticates to the control-plane cluster via Pod Identity, then hops to this deployer role to reach the member clusters.

## 3. Registering Member Clusters as Argo CD Cluster Secrets

For each entry in `clusters_configs`, a `Secret` labeled `argocd.argoproj.io/secret-type: cluster` registers that cluster with Argo CD, using the deployer role's ARN as the `awsAuthConfig.roleARN`:

```hcl
# control-plane/cluster.tf
data "aws_eks_cluster" "members" {
  count = length(var.clusters_configs)
  name  = lookup(var.clusters_configs[count.index], "cluster_name")
}

resource "kubectl_manifest" "argo_clusters" {
  count = length(var.clusters_configs)

  yaml_body = <<YAML
apiVersion: v1
kind: Secret
metadata:
  name: ${data.aws_eks_cluster.members[count.index].id}
  namespace: argocd
  labels:
    argocd.argoproj.io/secret-type: cluster
type: Opaque
stringData:
  name: ${data.aws_eks_cluster.members[count.index].id}
  config: |
    {
      "awsAuthConfig": {
        "clusterName": "${data.aws_eks_cluster.members[count.index].id}",
        "roleARN": "${aws_iam_role.argo_deployer.arn}"
      },
      "tlsClientConfig": {
        "insecure": false,
        "caData": "${data.aws_eks_cluster.members[count.index].certificate_authority.0.data}"
      }
    }
  server: "${data.aws_eks_cluster.members[count.index].endpoint}"
YAML

  depends_on = [
    helm_release.argocd
  ]
}
```

After this applies, the Argo CD dashboard's **Settings → Clusters** page lists three entries: the built-in `in-cluster` (the control plane itself) plus both workload clusters, each showing its real API server endpoint.

## 4. Authorizing the Deployer Role on Each Member Cluster

Registering the secret is only half the story — each *member* cluster also has to actually authorize the deployer role. A new variable defaulting to the deployer role's real ARN, plus an access entry and policy association granting it `cluster-admin`, is added to the `clusters/` stack (applied to both workload clusters):

```hcl
# clusters/variables.tf (excerpt)
variable "argocd_deployer_role" {
  default = "arn:aws:iam::181560427716:role/linuxtips-control-plane-argocd-deployer"
}
```

```hcl
# clusters/access_entry.tf (excerpt — present only in clusters/, not in control-plane/)
resource "aws_eks_access_entry" "argocd" {
  cluster_name  = aws_eks_cluster.main.id
  principal_arn = var.argocd_deployer_role
  type          = "STANDARD"

  kubernetes_groups = [
    "cluster-admin"
  ]
}

resource "aws_eks_access_policy_association" "argocd" {
  cluster_name  = aws_eks_cluster.main.id
  policy_arn    = "arn:aws:eks::aws:cluster-access-policy/AmazonEKSClusterAdminPolicy"
  principal_arn = var.argocd_deployer_role

  access_scope {
    type = "cluster"
  }
}
```

With this on both workload clusters, Argo CD (via the `argocd` role → `argo_deployer` role hop) has full `cluster-admin` access to deploy anything, anywhere, from a single control plane.

---

# Lesson 14: Managing Shared Add-ons as Multicluster ApplicationSets

## 1. The `system` AppProject

A dedicated `AppProject` groups every add-on/system-level `Application` separately from actual product deployments, exactly as the `nutrition` AppProject did in Module 17:

```hcl
# control-plane/project.tf
resource "kubectl_manifest" "argo_system" {
  yaml_body = <<YAML
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: system
  namespace: argocd
spec:
  clusterResourceWhitelist:
  - group: '*'
    kind: '*'
  destinations:
  - namespace: '*'
    server: '*'
  sourceRepos:
  - '*'
YAML

  depends_on = [
    helm_release.argocd
  ]
}
```

## 2. Argo Rollouts, Metrics Server, and KEDA as ApplicationSets

Every shared add-on the workload clusters need is expressed as one `ApplicationSet` with a `list` generator enumerating both clusters — the same generator shape as Module 17's `chip`/`nutrition` deploys, just pointed at two real remote clusters instead of one:

```yaml
# control-plane/argo_argo_rollouts.tf (embedded YAML, excerpt)
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: argo-rollouts
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
      name: argo-rollouts-{{shard}}
    spec:
      project: "system"
      source:
        repoURL: 'https://argoproj.github.io/argo-helm'
        chart: argo-rollouts
        targetRevision: 2.34.1
        helm:
          releaseName: argo-rollouts
          valuesObject:
            dashboard:
              enabled: true
              controller:
                metrics:
                  enabled: true
                  serviceMonitor:
                    enabled: true
      destination:
        name: '{{ cluster }}'
        namespace: argo-rollouts
      syncPolicy:
        syncOptions:
          - CreateNamespace=true
        automated: {}
```

Applying this once from the control-plane cluster produces two `Application`s — `argo-rollouts-01` and `argo-rollouts-02` — each syncing the same chart independently into its own workload cluster's `argo-rollouts` namespace. Metrics Server and KEDA follow the identical shape, just with different `repoURL`/`chart`/`valuesObject` fields:

```hcl
# control-plane/argo_metrics_server.tf (excerpt)
        repoURL: 'https://charts.bitnami.com/bitnami'
        chart: metrics-server
        targetRevision: 7.2.16
        helm:
          releaseName: metrics-server
          valuesObject:
            apiService:
              create: true
            serviceMonitor:
              enabled: true
      destination:
        name: '{{ cluster }}'
        namespace: kube-system
```

```hcl
# control-plane/argo_keda.tf (excerpt)
        repoURL: 'https://kedacore.github.io/charts'
        chart: keda
        targetRevision: "v2.16.1"
        helm:
          releaseName: keda
      destination:
        name: '{{ cluster }}'
        namespace: keda
```

From here on, upgrading any of these three add-ons on both workload clusters is a one-place, one-`apply` change from the control plane — no more SSHing (or `helm upgrade`-ing) into each cluster individually.

> **Note:** the repository already contains three more Argo-managed `ApplicationSet`s in this same style — `argo_fluentbit.tf`, `argo_otel.tf`, and `argo_prometheus.tf` — none of which are covered by this module's 15 lessons. Each targets an observability backend (`loki.linuxtips-observability.local`, `tempo.linuxtips-observability.local`, `mimir.linuxtips-observability.local`) that doesn't exist yet anywhere in this repository. They're forward-looking scaffolding for the course's next module (a dedicated observability cluster aggregating logs/metrics/traces from every member cluster) and are called out here only for transparency, not documented as this module's content.

## 3. From Dashboard-Applied ApplicationSet to Terraform-Managed `kubectl_manifest`

Argo Rollouts is also demonstrated as a standalone `.yml` file, identical in content to `argo_argo_rollouts.tf`'s embedded YAML, showing how the exact same `ApplicationSet` can be applied directly with `kubectl apply -f` for a quick one-off test before it's promoted into Terraform:

```yaml
# control-plane/argorollouts.yml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: argo-rollouts
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
      name: argo-rollouts-{{shard}}
    spec:
      project: "system"
      source:
        repoURL: 'https://argoproj.github.io/argo-helm'
        chart: argo-rollouts
        targetRevision: 2.34.1
        helm:
          releaseName: argo-rollouts
          valuesObject:
            dashboard:
              enabled: true
              controller:
                metrics:
                  enabled: true
                  serviceMonitor:
                    enabled: true
      destination:
        name: '{{ cluster }}'
        namespace: argo-rollouts
      syncPolicy:
        syncOptions:
          - CreateNamespace=true
        automated: {}
```

---

# Lesson 15: Deploying an Application Active-Active Across Clusters

## 1. ChartMuseum on the Control Plane Cluster

The ChartMuseum setup is a direct copy of Module 17's: an S3 bucket, a Pod Identity-authenticated IAM role scoped to that bucket, and a Helm release configured to use S3 as its backing storage:

```hcl
# control-plane/s3_chartmuseum.tf
resource "aws_s3_bucket" "chartmuseum" {
  bucket = format("%s-%s-chartmuseum", var.project_name, data.aws_caller_identity.current.account_id)
}

resource "aws_s3_bucket_ownership_controls" "chartmuseum" {
  bucket = aws_s3_bucket.chartmuseum.id
  rule {
    object_ownership = "BucketOwnerPreferred"
  }
}

resource "aws_s3_bucket_acl" "chartmuseum" {
  bucket = aws_s3_bucket.chartmuseum.id
  acl    = "private"

  depends_on = [
    aws_s3_bucket_ownership_controls.chartmuseum
  ]
}

resource "aws_s3_object" "linuxtips" {
  bucket = aws_s3_bucket.chartmuseum.id
  key    = "linuxtips-0.1.0.tgz"
  source = "${path.module}/helm/linuxtips-0.1.0.tgz"
  etag   = filemd5("${path.module}/helm/linuxtips-0.1.0.tgz")
}
```

```hcl
# control-plane/helm_chartmuseum.tf
resource "helm_release" "chartmuseum" {
  name       = "chartmuseum"
  repository = "https://chartmuseum.github.io/charts"
  chart      = "chartmuseum"
  namespace  = "chartmuseum"

  create_namespace = true

  set = [
    { name = "serviceAccount.create", value = "true" },
    { name = "env.open.AWS_SDK_LOAD_CONFIG", value = "true" },
    { name = "env.open.DISABLE_API", value = "false" },
    { name = "env.open.STORAGE", value = "amazon" },
    { name = "env.open.DISABLE_STATEFILES", value = "true" },
    { name = "env.open.STORAGE_AMAZON_BUCKET", value = aws_s3_bucket.chartmuseum.id },
    { name = "env.open.STORAGE_AMAZON_REGION", value = var.region }
  ]

  depends_on = [
    aws_eks_cluster.main,
    helm_release.karpenter
  ]
}
```

The `linuxtips` Helm chart itself — `control-plane/helm/linuxtips/` — is the same chart built across Modules 15-16, copied into this repository essentially unchanged (the only two differences from the `linuxtips-eks-vanilla` copy are cosmetic default values: `app.iam` defaults to a placeholder `"dummy-iam-role"` instead of an empty string, and `app.istio.host` defaults to `linuxtips.luisgustavo.com.br` instead of `linuxtips.msfidelis.com.br` — both immediately overridden per-deploy via `valuesObject` anyway).

## 2. The `chip` ApplicationSet Across Both Clusters

`chip` is deployed as an `ApplicationSet` sourced directly from ChartMuseum (`http://chartmuseum.chartmuseum.svc.cluster.local:8080`), applied straight with `kubectl apply -f` rather than wired into Terraform — the application-team deploy path, distinct from the Terraform-managed system add-ons above. The `list` generator's two elements each get their own `VERSION` env var interpolated from `{{cluster}}`, so the two clusters can be told apart from inside the running pods:

```yaml
# control-plane/chip-appset.yml (excerpt)
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: chip
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
      name: chip-{{shard}}
    spec:
      project: "default"
      source:
        repoURL: 'http://chartmuseum.chartmuseum.svc.cluster.local:8080'
        chart: linuxtips
        targetRevision: 0.1.0
        helm:
          releaseName: chip
          valuesObject:
            app:
              name: chip
              namespace: chip
              image:
                repository: fidelissauro/chip
                tag: latest
                pullPolicy: IfNotPresent
              envs:
                - name: ENV
                  value: "dev"
                - name: FOO
                  value: "basr"
                - name: VERSION
                  value: "{{cluster}}-v2"
              rollout:
                revisionHistoryLimit: 3
                version: v1
                strategy:
                  canary:
                    enabled: true
                    steps:
                      - setWeight: 20
                      - pause: {duration: 30s}
                      - analysis:
                          templates:
                            - templateName: chip-success
                      - setWeight: 40
                      - pause: {duration: 30s}
                      - analysis:
                          templates:
                            - templateName: chip-success
                      - setWeight: 60
                      - pause: {duration: 30s}
                      - analysis:
                          templates:
                            - templateName: chip-success
                      - setWeight: 80
                      - pause: {duration: 30s}
                      - analysis:
                          templates:
                            - templateName: chip-success
                      - setWeight: 100
      destination:
        name: '{{ cluster }}'
        namespace: argocd
      syncPolicy:
        syncOptions:
          - CreateNamespace=true
        automated: {}
```

> **Note:** each analysis step's Prometheus `AnalysisTemplate` queries `http://prometheus-kube-prometheus-prometheus.prometheus.svc.cluster.local:9090` — Prometheus isn't deployed on the workload clusters yet in this module, so (as the lesson states directly) the canary analysis fails by design here; it becomes real once Prometheus is deployed in a later lesson. `destination.namespace` is `argocd` in this manifest, left over from an earlier copy/paste in the lesson and not corrected on-screen — the actual `chip`, `chip-canary`, etc. resources still land in the `chip` namespace the chart itself creates via `app.namespace`.

## 3. Canary Rollouts and Cluster Failover in Practice

Applying `chip-appset.yml` produces `chip-01` and `chip-02` Applications, each syncing 10 pods onto its respective cluster. Because the shared ALB splits traffic 50/50 between clusters, a single `curl` loop against the load balancer shows responses landing on both `linuxtips-cluster-01` and `linuxtips-cluster-02`.

Bumping the chart's image/env values and re-applying triggers an Argo Rollouts canary **independently on both clusters at once** — traffic during the rollout is a mix of cluster 1/cluster 2 and v1/v2, all driven from one `apply`. And because the ingress-layer routing weight from Lesson 3 is fully independent of the Argo CD deploy pipeline, a whole cluster can be pulled out of rotation (`cluster_01 = 100, cluster_02 = 0`) at any point during a rollout without touching Argo CD at all — the canary keeps progressing on the drained cluster while 100% of live traffic serves from the other one.

This closes out the module's architecture: two active-active workload clusters, one control-plane cluster driving both system add-ons and application deploys via `ApplicationSet`s, and an ingress layer that can shift load between clusters independently of whatever Argo CD is doing. The next module in the course builds a dedicated observability cluster on top of this foundation, aggregating metrics, logs, and traces from every member cluster in one place — which is exactly what the `argo_fluentbit.tf`/`argo_otel.tf`/`argo_prometheus.tf` files flagged in Lesson 14 are already staged for.

---

## Key Takeaways

- **A dedicated control-plane cluster federates N workload clusters.** Argo CD authenticates to the control plane via EKS Pod Identity, then hops through a second, narrowly-scoped IAM role (`argocd-deployer`) to assume `cluster-admin` access on each registered member cluster — no shared kubeconfig, no long-lived static credentials.
- **`ApplicationSet` + `list` generator is the entire multicluster story.** Every shared add-on (Argo Rollouts, Metrics Server, KEDA) and the `chip` application itself use the exact same two-element `list` generator; adding a third cluster is one more `elements` entry, not a new pipeline.
- **Weighted ALB routing is fully decoupled from Argo CD.** The ingress layer's per-cluster traffic weight is a plain Terraform variable — shifting 100% of load off a cluster for maintenance, mid-rollout or not, never touches the GitOps deploy path at all.
- **Pod Identity replaces IRSA annotations everywhere in this module** — the AWS Load Balancer Controller, Karpenter, ChartMuseum, and Argo CD's own three service accounts are all associated to IAM roles via `aws_eks_pod_identity_association`, not `serviceAccount.annotations`.
- **Terraform-managed system add-ons vs. dashboard/kubectl-applied application deploys is a deliberate split.** Argo Rollouts/Metrics Server/KEDA are `kubectl_manifest` resources inside Terraform (so a `terraform apply` upgrades them everywhere); `chip` is applied by hand with `kubectl apply -f`, modeling the boundary between platform-managed add-ons and application-team-managed deploys.
- **This module's forward references are real, not fabricated:** the Istio `zipkin.address`, and the `argo_fluentbit.tf`/`argo_otel.tf`/`argo_prometheus.tf` `ApplicationSet`s, all point at observability endpoints that don't exist yet — genuine scaffolding left in the repository for a later module, not content this module's lessons teach.
- **The two workload clusters really are one codebase, applied twice.** `clusters/environment/prod/cluster-01/terraform.tfvars` and `cluster-02/terraform.tfvars` differ only in `project_name`, the state `key`, and `istio_ssm_target_group` — everything else (Kubernetes version, subnets, Karpenter capacity) is identical.
