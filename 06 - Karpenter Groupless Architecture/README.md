# Module 6: Karpenter Groupless Architecture

## Overview

In this module we take Karpenter one step further: instead of running it *alongside* EKS managed node groups, we remove node groups from the cluster entirely and let Karpenter manage 100% of the compute capacity.

This is an architectural proposal more than a new feature. The idea is to split the cluster into two categories:

```txt
Volatility-intolerant workloads
  - Karpenter's own controller
  - kube-system add-ons (CoreDNS, metrics-server, kube-state-metrics)
  - anything that cannot handle nodes constantly scaling up and down

Volatility-tolerant workloads
  - the actual applications
  - scaled from 1 to millions of pods by Karpenter's NodePools/EC2NodeClasses
```

Everything in the first group runs on **Fargate Profiles**, so it never depends on — and never disturbs — the node capacity that Karpenter is actively scaling up and down. Everything in the second group is left entirely to Karpenter.

The main benefit of this approach is not having to run two separate compute-management systems (node groups and Karpenter) at the same time, each with its own upgrade cycle and its own scaling logic. It's especially well-suited to bursty, scale-to-zero-style workloads — such as those built with Knative — where capacity needs to grow from zero to a very large number of pods in seconds.

## Table of Contents

- [Lesson 1: Building a Groupless EKS Cluster](#lesson-1-building-a-groupless-eks-cluster)
  - [1. The Groupless Architecture](#1-the-groupless-architecture)
  - [2. Reusing the Fargate Lesson as a Base](#2-reusing-the-fargate-lesson-as-a-base)
  - [3. Creating the Fargate IAM Role and Access Entry](#3-creating-the-fargate-iam-role-and-access-entry)
  - [4. Redeploying the CoreDNS Fix Lambda](#4-redeploying-the-coredns-fix-lambda)
  - [5. Creating the kube-system and karpenter Fargate Profiles](#5-creating-the-kube-system-and-karpenter-fargate-profiles)
  - [6. Deleting the Node Groups](#6-deleting-the-node-groups)
  - [7. Fixing depends_on References](#7-fixing-depends_on-references)
  - [8. Deploying an Application Without a Node Selector](#8-deploying-an-application-without-a-node-selector)
  - [9. Why This Architecture Matters](#9-why-this-architecture-matters)
  - [Key Takeaways](#key-takeaways)

# Lesson 1: Building a Groupless EKS Cluster

## 1. The Groupless Architecture

The goal of this lesson is to end up with a cluster where node groups do not exist at all, and all EC2 compute capacity is provisioned by Karpenter.

For that to work, Karpenter itself — and anything else that can't tolerate nodes constantly appearing and disappearing — needs somewhere stable to run. That's what Fargate Profiles are for:

- A **`kube-system`** Fargate Profile for the add-ons that live in the `kube-system` namespace: CoreDNS, the CoreDNS-fix Lambda's dependency chain, metrics-server, and kube-state-metrics.
- A **`karpenter`** Fargate Profile for the Karpenter controller itself, which cannot be scheduled onto a node that it is responsible for creating.

Everything else — the actual application workloads — is left to Karpenter's NodePools and EC2NodeClasses, exactly as configured in [Module 5](../05%20-%20Autoscaling%20with%20Karpenter/README.md).

## 2. Reusing the Fargate Lesson as a Base

This lesson deliberately reuses the setup from [Module 4: AWS Fargate](../04%20-%20AWS%20Fargate/README.md) rather than reinventing it. The cluster we start from already has Karpenter installed (Module 5), and we bring in the Fargate IAM role, the Fargate access entry, and the CoreDNS-fix Lambda from Module 4 — copying the same code, just applied to this cluster.

If you haven't gone through Module 4 yet, do that first: this lesson assumes the Fargate pod execution role, the CoreDNS-fix Lambda pattern, and the Fargate access entry pattern are already familiar.

## 3. Creating the Fargate IAM Role and Access Entry

The Fargate pod execution role is the same one created in Module 4 — a role that trusts `eks-fargate-pods.amazonaws.com` and carries the `AmazonEKSFargatePodExecutionRolePolicy` managed policy:

```hcl
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
  role       = aws_iam_role.fargate.name
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSFargatePodExecutionRolePolicy"
}
```

That role needs its own EKS access entry, added alongside the node group's existing one:

```hcl
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

Note that the node group's IAM role, instance profile, and access entry are **kept**, even though the node group resource itself is going away later in this lesson. Karpenter-provisioned EC2 instances reuse that same instance profile, so it — and its access entry — stay in place.

## 4. Redeploying the CoreDNS Fix Lambda

Full-Fargate clusters need the same CoreDNS fix covered in [Module 4, Lesson 4](../04%20-%20AWS%20Fargate/README.md#lesson-4-fixing-coredns-in-full-fargate-clusters): CoreDNS ships with an `eks.amazonaws.com/compute-type: ec2` annotation that has to be removed before it can run on Fargate, and it needs a rolling restart afterwards to pick up the change.

We redeploy the exact same Lambda here — same IAM role, same security group, same packaging, same invocation pattern — pointed at this cluster:

```hcl
data "aws_iam_policy_document" "coredns_fix" {
  version = "2012-10-17"

  statement {
    actions = ["sts:AssumeRole"]
    principals {
      type        = "Service"
      identifiers = ["lambda.amazonaws.com"]
    }
  }
}

resource "aws_iam_role" "coredns_fix" {
  name_prefix        = format("%s-coredns-fix", var.project_name)
  assume_role_policy = data.aws_iam_policy_document.coredns_fix.json
}

resource "aws_iam_role_policy_attachment" "coredns_fix" {
  role       = aws_iam_role.coredns_fix.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole"
}

resource "aws_security_group" "coredns_fix" {
  name   = format("%s-coredns-fix", var.project_name)
  vpc_id = data.aws_ssm_parameter.vpc.value

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

data "archive_file" "coredns_archive" {
  type        = "zip"
  source_dir  = "lambda/coredns"
  output_path = "lambda/coredns.zip"
}

resource "aws_lambda_function" "coredns_fix" {
  function_name = format("%s-coredns-fix", var.project_name)
  runtime       = "python3.13"

  handler          = "main.handler"
  role             = aws_iam_role.coredns_fix.arn
  filename         = data.archive_file.coredns_archive.output_path
  source_code_hash = data.archive_file.coredns_archive.output_base64sha256
  timeout          = 120

  vpc_config {
    subnet_ids         = data.aws_ssm_parameter.private_subnets[*].value
    security_group_ids = [aws_security_group.coredns_fix.id]
  }
}
```

The Lambda's invocation is what actually triggers the fix, and it must run **after** the `kube-system` Fargate Profile exists — otherwise there's nowhere for the patched CoreDNS pod to be scheduled:

```hcl
data "aws_lambda_invocation" "coredns_fix" {
  function_name = aws_lambda_function.coredns_fix.function_name
  input         = <<JSON
{
  "endpoint": "${aws_eks_cluster.main.endpoint}",
  "token": "${data.aws_eks_cluster_auth.default.token}"
}
JSON

  depends_on = [
    aws_lambda_function.coredns_fix,
    aws_eks_fargate_profile.kube_system
  ]
}
```

The Lambda's own code (`main.py`) doesn't change from Module 4 at all: it removes the `eks.amazonaws.com/compute-type` annotation from the CoreDNS Deployment and restarts it via a `kubectl.kubernetes.io/restartedAt` patch, retrying with backoff if the API call fails.

## 5. Creating the kube-system and karpenter Fargate Profiles

With the IAM role, access entry, and Lambda in place, we create the two Fargate Profiles this architecture is built around:

```hcl
resource "aws_eks_fargate_profile" "kube_system" {
  cluster_name         = aws_eks_cluster.main.name
  fargate_profile_name = "kube-system"

  pod_execution_role_arn = aws_iam_role.fargate.arn

  subnet_ids = data.aws_ssm_parameter.pod_subnets[*].value

  selector {
    namespace = "kube-system"
  }
}

resource "aws_eks_fargate_profile" "karpenter" {
  cluster_name         = aws_eks_cluster.main.name
  fargate_profile_name = "karpenter"

  pod_execution_role_arn = aws_iam_role.fargate.arn

  subnet_ids = data.aws_ssm_parameter.pod_subnets[*].value

  selector {
    namespace = "karpenter"
  }
}
```

Only these two namespaces get a Fargate Profile at this point — `kube-system` for the cluster add-ons, `karpenter` for the controller itself. Any other namespace that needs the same isolation later can get its own profile the same way.

After applying, delete any pods still running on EC2 in these two namespaces (skipping DaemonSet pods, which don't run on Fargate) so they get rescheduled onto Fargate:

```bash
terraform apply

kubectl get pods -n kube-system -o wide
kubectl get pods -n karpenter -o wide

kubectl delete pod <pod-name> -n kube-system
kubectl delete pod <pod-name> -n karpenter

kubectl get pods -n kube-system -o wide --watch
kubectl get pods -n karpenter -o wide --watch
```

Once the environment stabilizes, the Karpenter controller, metrics-server, kube-state-metrics, and CoreDNS should all show as running on Fargate nodes.

## 6. Deleting the Node Groups

With Karpenter's own controller and every non-negotiable add-on now safely on Fargate, the node group's job is done — from here on, Karpenter takes over all remaining EC2 capacity. Delete the node group resource(s) and apply:

```bash
terraform apply
```

## 7. Fixing depends_on References

Removing the node group breaks every resource that referenced it in a `depends_on` block, since Terraform used it as a signal that the cluster had EC2 capacity available before installing something on top of it. Each of those references needs to point at the relevant Fargate Profile instead.

The Karpenter Helm release now depends on the `karpenter` Fargate Profile instead of the node group, since Karpenter's own controller runs there:

```hcl
resource "helm_release" "karpenter" {
  namespace        = "karpenter"
  create_namespace = true

  name       = "karpenter"
  repository = "oci://public.ecr.aws/karpenter"
  chart      = "karpenter"
  version    = "1.0.8"

  set = [
    {
      name  = "serviceAccount.annotations.eks\\.amazonaws\\.com/role-arn"
      value = aws_iam_role.karpenter.arn
    },
    {
      name  = "settings.clusterName"
      value = var.project_name
    },
    {
      name  = "settings.clusterEndpoint"
      value = aws_eks_cluster.main.endpoint
    },
    {
      name  = "settings.interruptionQueue"
      value = aws_sqs_queue.karpenter.name
    }
  ]

  depends_on = [
    aws_eks_cluster.main,
    aws_eks_fargate_profile.karpenter
  ]
}
```

metrics-server and kube-state-metrics now depend on the `kube_system` Fargate Profile instead, since that's where they're scheduled:

```hcl
resource "helm_release" "metrics_server" {
  name       = "metrics-server"
  repository = "https://charts.bitnami.com/bitnami"
  chart      = "metrics-server"
  namespace  = "kube-system"

  wait = false

  version = "7.2.16"

  set = [
    {
      name  = "apiService.create"
      value = "true"
    }
  ]

  depends_on = [
    aws_eks_cluster.main,
    aws_eks_fargate_profile.kube_system
  ]
}

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
      name  = "metricLabelsAllowlist[0]"
      value = "nodes=[*]"
    },
    {
      name  = "metricAnnotationsAllowList[0]"
      value = "nodes=[*]"
    }
  ]

  depends_on = [
    aws_eks_cluster.main,
    aws_eks_fargate_profile.kube_system
  ]
}
```

Apply again and wait for the node group's instances to finish terminating:

```bash
terraform apply

kubectl get nodes --watch
```

Only Fargate nodes should remain at this point — there is no longer an EC2 node group anywhere in the cluster.

## 8. Deploying an Application Without a Node Selector

Now it's time to see the groupless architecture in action. We reuse the same `chip` application from [Module 5](../05%20-%20Autoscaling%20with%20Karpenter/README.md), including its `topologySpreadConstraints` for availability zone, instance type, and capacity type spread — but this time, unlike the multi-NodePool exercise in Module 5, **we deploy it with no `nodeSelector` at all**. The idea is to let Karpenter figure out on its own what capacity the workload needs, rather than pinning it to a specific NodePool:

```yaml
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
  replicas: 100
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

        - maxSkew: 1
          topologyKey: "karpenter.sh/capacity-type"
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

Apply it and watch the NodeClaims come up:

```bash
kubectl apply -f chip.yaml

kubectl get nodeclaims --watch

kubectl logs -n karpenter -l app.kubernetes.io/name=karpenter --follow
```

The Karpenter logs show the pods sitting pending until Karpenter recognizes the demand and starts requesting nodes to satisfy it — using whichever instance families, sizes, and capacity types are allowed by the NodePool, exactly as configured back in Module 5's multi-NodePool setup.

## 9. Why This Architecture Matters

Since `chip` was deployed to a `nodeSelector`-free NodePool, Karpenter is free to scale it up or down purely based on demand — from 1 pod to 1,000, from 1,000 back down to 2, or up again toward much larger numbers — without that scaling ever touching the Fargate-hosted control-plane components.

The satellite components — Karpenter itself, CoreDNS, metrics-server, kube-state-metrics — stay isolated on an ultra-reduced Fargate footprint and never have to deal with the constant up-and-down of application node capacity.

This split is especially valuable for workloads that are genuinely bursty — for example, applications built on top of Knative, which can scale from zero to a very large number of pods in seconds if allowed to. Karpenter can keep just enough of the essential controllers exposed to react to that kind of demand quickly and economically, without needing capacity permanently allocated for it.

There's also an operational benefit: with node groups removed, there's no longer a second compute-management system to keep in sync with Karpenter — no separate node group AMI/version to upgrade, no risk of the two systems fighting over the same capacity decisions.

## Key Takeaways

- The groupless architecture removes EKS managed node groups entirely, leaving Karpenter as the only source of EC2 compute capacity in the cluster.
- Anything that can't tolerate nodes appearing and disappearing unpredictably — Karpenter's own controller, and `kube-system` add-ons like CoreDNS, metrics-server, and kube-state-metrics — is placed on dedicated Fargate Profiles (`karpenter` and `kube-system`), not left to Karpenter's own NodePools.
- The node IAM role, instance profile, and access entry are kept even after the node group resource is deleted, because Karpenter-provisioned EC2 instances reuse that same instance profile.
- Every resource that previously used the node group as a `depends_on` signal (the Karpenter Helm release, metrics-server, kube-state-metrics) needs that reference updated to point at the appropriate Fargate Profile instead.
- Application workloads are deployed without a `nodeSelector`, letting Karpenter decide freely what capacity to provision — the opposite of the targeted `nodeSelector` approach used for workload segregation in Module 5.
- The main payoff of this architecture is avoiding two parallel compute-management systems (node groups and Karpenter) running at once, and it's particularly well-suited to bursty, scale-to-zero-style workloads such as those built on Knative.
