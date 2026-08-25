# Module 3: Advanced Node Groups Strategies

## Overview

In the previous module we created our **first EKS cluster** and deployed a simple application.

The goal of that lesson was to **quickly provision a working Kubernetes cluster** using Terraform.

Now we will go deeper into **node group strategies**, exploring how to design node groups for different workloads and infrastructure requirements.

## Table of Contents

- [Lesson 1: Advanced Node Group Strategies in Amazon EKS](#lesson-1-advanced-node-group-strategies-in-amazon-eks)
  - [1. Understanding Node Groups](#1-understanding-node-groups)
  - [2. Reviewing the Previous Node Group](#2-reviewing-the-previous-node-group)
  - [3. Defining Capacity Type](#3-defining-capacity-type)
  - [4. Adding Node Labels](#4-adding-node-labels)
  - [5. Deploying the Node Group](#5-deploying-the-node-group)
  - [6. Creating a Spot Node Group](#6-creating-a-spot-node-group)
  - [7. Deploying the Spot Node Group](#7-deploying-the-spot-node-group)
  - [8. Verifying the Node Groups](#8-verifying-the-node-groups)
- [Lesson 2: Using Managed Node Groups with Bottlerocket](#lesson-2-using-managed-node-groups-with-bottlerocket)
  - [1. Understanding What Bottlerocket Is](#1-understanding-what-bottlerocket-is)
  - [2. Why Use Bottlerocket in EKS](#2-why-use-bottlerocket-in-eks)
  - [3. Creating a Managed Node Group with Bottlerocket](#3-creating-a-managed-node-group-with-bottlerocket)
  - [4. Creating the Bottlerocket Node Group File](#4-creating-the-bottlerocket-node-group-file)
  - [5. Applying the Configuration](#5-applying-the-configuration)
  - [6. Checking the Nodes in AWS Console](#6-checking-the-nodes-in-aws-console)
  - [7. Connecting to the Nodes](#7-connecting-to-the-nodes)
  - [8. Creating a Bottlerocket Node Group Using Spot Instances](#8-creating-a-bottlerocket-node-group-using-spot-instances)
  - [9. Checking Node Labels](#9-checking-node-labels)
- [Lesson 3: Using Graviton (ARM64) Instances in EKS](#lesson-3-using-graviton-arm64-instances-in-eks)
  - [1. Understanding What Graviton Is](#1-understanding-what-graviton-is)
  - [2. Benefits of Using Graviton](#2-benefits-of-using-graviton)
  - [3. Important Requirement: Application Compatibility](#3-important-requirement-application-compatibility)
  - [4. Creating a Graviton Node Group](#4-creating-a-graviton-node-group)
  - [5. Configuring the AMI Type](#5-configuring-the-ami-type)
  - [6. Creating the Terraform File](#6-creating-the-terraform-file)
  - [7. Creating a Spot Version](#7-creating-a-spot-version)
  - [8. Applying the Configuration](#8-applying-the-configuration)
  - [9. Validating the Nodes](#9-validating-the-nodes)
  - [10. Why Labels Are Important](#10-why-labels-are-important)
- [Lesson 4: Workload Segregation Using Node Selector](#lesson-4-workload-segregation-using-node-selector)
  - [1. Understanding Node Selector](#1-understanding-node-selector)
  - [2. Using the Example Deployment](#2-using-the-example-deployment)
  - [3. Applying the Deployment](#3-applying-the-deployment)
  - [4. Validating Where Pods Are Running](#4-validating-where-pods-are-running)
  - [5. Changing Strategy to Spot](#5-changing-strategy-to-spot)
  - [6. Real Use Cases](#6-real-use-cases)
  - [7. Combining with Node Groups](#7-combining-with-node-groups)
- [Lesson 5: Using Node Affinity for Smarter Scheduling](#lesson-5-using-node-affinity-for-smarter-scheduling)
  - [1. Understanding Node Affinity](#1-understanding-node-affinity)
  - [2. Creating a Node Affinity Deployment](#2-creating-a-node-affinity-deployment)
  - [3. Applying the Configuration](#3-applying-the-configuration)
  - [4. Observing Pod Distribution](#4-observing-pod-distribution)
  - [5. Important Behavior](#5-important-behavior)
  - [6. Real Use Case](#6-real-use-case)
  - [Key Takeaways](#key-takeaways)
- [Lesson 6: Segregating Workloads by Criticality](#lesson-6-segregating-workloads-by-criticality)
  - [1. Defining Critical vs Non-Critical Nodes](#1-defining-critical-vs-non-critical-nodes)
  - [2. Creating a Critical Node Group](#2-creating-a-critical-node-group)
  - [3. Creating a Soft (Non-Critical) Node Group](#3-creating-a-soft-non-critical-node-group)
  - [4. Applying the Configuration](#4-applying-the-configuration)
  - [5. Validating Node Labels](#5-validating-node-labels)
  - [6. Using Criticality in Deployments](#6-using-criticality-in-deployments)
  - [7. Real Use Case](#7-real-use-case)
  - [Key Takeaways](#key-takeaways-1)
- [Lesson 7: Customizing Node Groups with Launch Templates](#lesson-7-customizing-node-groups-with-launch-templates)
  - [1. Why Use Launch Templates](#1-why-use-launch-templates)
  - [2. What We Can Customize](#2-what-we-can-customize)
  - [3. Preparing User Data](#3-preparing-user-data)
  - [4. Creating AMI Variable](#4-creating-ami-variable)
  - [5. Creating the Launch Template](#5-creating-the-launch-template)
  - [6. Passing Cluster Data to User Data](#6-passing-cluster-data-to-user-data)
  - [7. Validating the Launch Template](#7-validating-the-launch-template)
  - [8. Creating a Custom Node Group](#8-creating-a-custom-node-group)
  - [9. Applying and Validating](#9-applying-and-validating)
  - [10. Real Use Cases](#10-real-use-cases)
  - [11. Best Practice](#11-best-practice)
- [Lesson 8: Cluster Autoscaler and Dynamic Capacity Management](#lesson-8-cluster-autoscaler-and-dynamic-capacity-management)
  - [1. Introduction to Cluster Autoscaler](#1-introduction-to-cluster-autoscaler)
  - [2. Creating IAM Role for Autoscaler (IRSA)](#2-creating-iam-role-for-autoscaler-irsa)
  - [3. Creating the IAM Policy](#3-creating-the-iam-policy)
  - [4. Attaching Policy to the Role](#4-attaching-policy-to-the-role)
  - [5. Deploying Cluster Autoscaler via Helm](#5-deploying-cluster-autoscaler-via-helm)
  - [6. Configuring Node Group Limits](#6-configuring-node-group-limits)
  - [7. Validating Autoscaler Deployment](#7-validating-autoscaler-deployment)
  - [8. Testing Scale Up](#8-testing-scale-up)
  - [9. Observing Node Creation](#9-observing-node-creation)
  - [10. Testing Further Scaling](#10-testing-further-scaling)
  - [11. Scaling Down](#11-scaling-down)
  - [12. Understanding the Behavior](#12-understanding-the-behavior)
  - [13. Introducing Node Termination Handler](#13-introducing-node-termination-handler)
  - [14. Why This Is Important](#14-why-this-is-important)
- [Lesson 9: Node Termination Handler and Safe Workload Eviction](#lesson-9-node-termination-handler-and-safe-workload-eviction)
  - [1. Introduction to Node Termination Handler](#1-introduction-to-node-termination-handler)
  - [2. How the Flow Works](#2-how-the-flow-works)
  - [3. Creating IAM Role (IRSA)](#3-creating-iam-role-irsa)
  - [4. Creating IAM Policy](#4-creating-iam-policy)
  - [5. Attaching Policy to Role](#5-attaching-policy-to-role)
  - [6. Creating SQS Queue](#6-creating-sqs-queue)
  - [7. Creating CloudWatch Event Rules](#7-creating-cloudwatch-event-rules)
  - [8. Testing Event Flow (Before Deployment)](#8-testing-event-flow-before-deployment)
  - [9. Deploying Node Termination Handler (Helm)](#9-deploying-node-termination-handler-helm)
  - [10. Validating Deployment](#10-validating-deployment)
  - [11. Verifying Event Consumption](#11-verifying-event-consumption)
  - [12. Testing Node Termination](#12-testing-node-termination)
  - [13. Observing Kubernetes Behavior](#13-observing-kubernetes-behavior)
  - [14. Why This Matters](#14-why-this-matters)

---

# Lesson 1: Advanced Node Group Strategies in Amazon EKS

---

## 1. Understanding Node Groups

A **Node Group** in Amazon EKS represents a group of EC2 instances that run Kubernetes workloads.

Each node group can be configured with specific characteristics such as:

```txt
Instance types
Capacity type (On-Demand or Spot)
Operating system
CPU architecture
Labels for scheduling
```

Using multiple node groups allows us to **separate workloads and optimize costs and performance**.

---

## 2. Reviewing the Previous Node Group

In the previous module we created a **basic managed node group**.

Example simplified configuration:

```hcl
resource "aws_eks_node_group" "main" {
  cluster_name    = aws_eks_cluster.main.id
  node_group_name = "${var.project_name}-workers"

  node_role_arn = aws_iam_role.eks_nodes_role.arn

  instance_types = var.nodes_instance_sizes

  subnet_ids = data.aws_ssm_parameter.pod_subnets[*].value

  scaling_config {
    desired_size = lookup(var.auto_scale_options, "desired")
    max_size     = lookup(var.auto_scale_options, "max")
    min_size     = lookup(var.auto_scale_options, "min")
  }

  labels = {
    "ingress/ready" = "true"
  }

  tags = {
    "kubernetes.io/cluster/${var.project_name}" = "owned"
  }

  depends_on = [
    # kubernetes_config_map_v1.aws_auth
    aws_eks_access_entry.nodes
  ]

  lifecycle {
    ignore_changes = [
      scaling_config[0].desired_size
    ]
  }

  timeouts {
    create = "1h"
    update = "2h"
    delete = "2h"
  }
}
```

This configuration uses mostly **default values**, which works but does not provide much control over the cluster architecture.

Now we will expand this configuration.

---

## 3. Defining Capacity Type

One important configuration is the **capacity type**.

Example:

```hcl
capacity_type = "ON_DEMAND"
```

If this parameter is not specified, EKS will default to **On-Demand instances**.

On-Demand nodes are:

```txt
Stable
Predictable
Recommended for critical workloads
```

---

## 4. Adding Node Labels

Node labels allow Kubernetes to **schedule workloads to specific nodes**.

Example configuration:

```hcl
labels = {
  "capacity/os" = "AMAZON_LINUX"
  "capacity/arch" = "X86_64"
  "capacity/type" = "ON_DEMAND"
}
```

These labels can later be used in deployments with **node selectors or affinity rules**.

Example use cases:

```txt
Run workloads only on ARM nodes
Run workloads only on Spot instances
Run workloads on specific operating systems
```

---

## 5. Deploying the Node Group

After updating the configuration, apply the Terraform deployment.

```bash
terraform apply --auto-approve --var-file=environment/prod/terraform.tfvars
```

This will provision the node group with the new configuration.

You can verify the result in the AWS console:

```txt
EKS → Cluster → Compute → Node Groups
```

There you will see the node group with its configured labels and capacity type.

---

## 6. Creating a Spot Node Group

Now we will create a **second node group using Spot instances**.

Spot instances allow us to use unused AWS capacity at a much lower cost.

They are ideal for workloads that can tolerate interruptions.

Create a new file:

```txt
nodes_spot.tf
```

```hcl
resource "aws_eks_node_group" "spot" {
  cluster_name    = aws_eks_cluster.main.id
  node_group_name = "${var.project_name}-workers-spot"

  node_role_arn = aws_iam_role.eks_nodes_role.arn

  instance_types = var.nodes_instance_sizes

  subnet_ids = data.aws_ssm_parameter.pod_subnets[*].value

  scaling_config {
    desired_size = lookup(var.auto_scale_options, "desired")
    max_size     = lookup(var.auto_scale_options, "max")
    min_size     = lookup(var.auto_scale_options, "min")
  }

  capacity_type = "SPOT"

  labels = {
    "capacity/os" = "AMAZON_LINUX"
    "capacity/arch" = "X86_64"
    "capacity/type" = "SPOT"
  }

  tags = {
    "kubernetes.io/cluster/${var.project_name}" = "owned"
  }

  depends_on = [
    # kubernetes_config_map_v1.aws_auth
    aws_eks_access_entry.nodes
  ]

  lifecycle {
    ignore_changes = [
      scaling_config[0].desired_size
    ]
  }

  timeouts {
    create = "1h"
    update = "2h"
    delete = "2h"
  }
}
```

---

## 7. Deploying the Spot Node Group

Apply the configuration again.

```bash
terraform apply --auto-approve --var-file=environment/prod/terraform.tfvars
```

Terraform will create the new node group with Spot instances.

Now the cluster will contain:

```txt
On-Demand node group
Spot node group
```

---

## 8. Verifying the Node Groups

Check the node groups in the AWS console.

```txt
EKS → Cluster → Compute
```

You should now see two node groups:

```txt
<project>-workers (On-Demand)
<project>-workers-spot (Spot)
```

Each node will also include the labels we defined earlier.

---

# Lesson 2: Using Managed Node Groups with Bottlerocket

In this lesson we will learn how to create **Managed Node Groups using Bottlerocket** and understand what makes it different from Amazon Linux nodes.

---

## 1. Understanding What Bottlerocket Is

Before creating the node group, it is important to understand what Bottlerocket actually is.

Bottlerocket is an operating system created by AWS specifically to run containers.

It follows the same idea as a minimal Linux distribution.

That means:

```txt
Very small attack surface
Almost no extra libraries
Almost no packages installed
Focused only on running containers
```

It is a lightweight and secure alternative for Kubernetes worker nodes.

---

## 2. Why Use Bottlerocket in EKS

Bottlerocket is not designed to improve performance.  
The main advantage is **security and operational simplicity**.

Main benefits:

```txt
Smaller attack surface
Faster node provisioning
Simpler node lifecycle management
Optimized for Kubernetes workloads
```

It is especially useful when nodes are more **ephemeral** and focused only on running containers.

---

## 3. Creating a Managed Node Group with Bottlerocket

Creating a Bottlerocket node group is very simple.  
It is basically the same process used for Amazon Linux nodes.

We only need to change one parameter:

```hcl
ami_type
```

Example:

```hcl
ami_type = "BOTTLEROCKET_x86_64"
```

This tells EKS to create nodes using the Bottlerocket operating system instead of Amazon Linux.

---

## 4. Creating the Bottlerocket Node Group File

We will duplicate the existing node group configuration and create a new file.

Example:

```txt
nodes_bottlerocket.tf
```

```hcl
resource "aws_eks_node_group" "bottlerocket" {
  cluster_name    = aws_eks_cluster.main.id
  node_group_name = "${var.project_name}-workers-bottlerocket"

  node_role_arn = aws_iam_role.eks_nodes_role.arn

  instance_types = var.nodes_instance_sizes

  subnet_ids = data.aws_ssm_parameter.pod_subnets[*].value

  scaling_config {
    desired_size = lookup(var.auto_scale_options, "desired")
    max_size     = lookup(var.auto_scale_options, "max")
    min_size     = lookup(var.auto_scale_options, "min")
  }

  capacity_type = "ON_DEMAND"

  ami_type = "BOTTLEROCKET_x86_64"

  labels = {
    "capacity/os" = "BOTTLEROCKET"
    "capacity/arch" = "X86_64"
    "capacity/type" = "ON_DEMAND"
  }

  tags = {
    "kubernetes.io/cluster/${var.project_name}" = "owned"
  }

  depends_on = [
    # kubernetes_config_map_v1.aws_auth
    aws_eks_access_entry.nodes
  ]

  lifecycle {
    ignore_changes = [
      scaling_config[0].desired_size
    ]
  }

  timeouts {
    create = "1h"
    update = "2h"
    delete = "2h"
  }
}
```

---

## 5. Applying the Configuration

Now we apply the configuration to create the new node group.

Command:

```bash
terraform apply --auto-approve --var-file=environment/prod/terraform.tfvars
```

After a few minutes, the new nodes will be available inside the cluster.

---

## 6. Checking the Nodes in AWS Console

After the apply finishes, we can verify the nodes in the AWS console.

You will see something like:

```txt
Amazon Linux nodes
Bottlerocket nodes
On-Demand nodes
Spot nodes
```

This confirms the new node group was created successfully.

---

## 7. Connecting to the Nodes

If you connect to an Amazon Linux node using Session Manager, you will see a normal Linux environment.

But if you connect to a Bottlerocket node, you will notice something very different:

```txt
Almost no packages installed
Very minimal environment
Designed only to run containers
```

This is exactly the goal of Bottlerocket.

---

## 8. Creating a Bottlerocket Node Group Using Spot Instances

We can also create a Spot version of the Bottlerocket node group.

The process is the same.

Create a new file:

```txt
nodes_bottlerocket_spot.tf
```

```hcl
resource "aws_eks_node_group" "bottlerocket_spot" {
  cluster_name    = aws_eks_cluster.main.id
  node_group_name = "${var.project_name}-workers-bottlerocket-spot"

  node_role_arn = aws_iam_role.eks_nodes_role.arn

  instance_types = var.nodes_instance_sizes

  subnet_ids = data.aws_ssm_parameter.pod_subnets[*].value

  scaling_config {
    desired_size = lookup(var.auto_scale_options, "desired")
    max_size     = lookup(var.auto_scale_options, "max")
    min_size     = lookup(var.auto_scale_options, "min")
  }

  capacity_type = "SPOT"

  ami_type = "BOTTLEROCKET_x86_64"

  labels = {
    "capacity/os" = "BOTTLEROCKET"
    "capacity/arch" = "X86_64"
    "capacity/type" = "SPOT"
  }

  tags = {
    "kubernetes.io/cluster/${var.project_name}" = "owned"
  }

  depends_on = [
    # kubernetes_config_map_v1.aws_auth
    aws_eks_access_entry.nodes
  ]

  lifecycle {
    ignore_changes = [
      scaling_config[0].desired_size
    ]
  }

  timeouts {
    create = "1h"
    update = "2h"
    delete = "2h"
  }
}
```

---

## 9. Checking Node Labels

Now we can list the nodes using labels to see how they are distributed.

Example command:

```bash
kubectl get nodes --show-labels
```

This allows us to identify:

```txt
Amazon Linux nodes
Bottlerocket nodes
Spot nodes
On-Demand nodes
```

---

# Lesson 3: Using Graviton (ARM64) Instances in EKS

In this lesson we will learn how to use **Graviton instances (ARM64)** in our EKS cluster and how to create node groups using this architecture.

---

## 1. Understanding What Graviton Is

Graviton is a type of AWS instance that uses **ARM64 processors** instead of traditional x86 (Intel/AMD).

Key difference:

```txt
x86 → Intel / AMD  
ARM64 → AWS Graviton processors
```

Instances that support Graviton usually have a **"G" suffix** in their name.

Examples:

```txt
t4g.large  
c7g.large  
m6g.large
```

---

## 2. Benefits of Using Graviton

The main advantage of Graviton is **cost efficiency**.

In many cases, you can get ~20% to 30% cost reduction compared to equivalent x86 instances.

Important note:

```txt
Performance gains are not always significant  
The biggest benefit is cost savings
```

---

## 3. Important Requirement: Application Compatibility

Before using ARM64, you must ensure your applications support it.

That means:

```txt
Applications may need to be compiled for ARM64  
Docker images must support ARM architecture  
Multi-arch images are recommended
```

If your application is not compatible, it will not run.

---

## 4. Creating a Graviton Node Group

To create a Graviton node group, we need to change two things:

```txt
Instance types (must be ARM-based)  
AMI type (must support ARM64)
```

Example instance types:

```txt
t4g.large  
c7g.large
```

---

## 5. Configuring the AMI Type

We must use an ARM-compatible AMI.

Example:

```txt
ami_type = "AL2023_ARM_64_STANDARD"
```

This ensures the node runs with the correct architecture.

---

## 6. Creating the Terraform File

We can reuse an existing node group and adapt it.

Example file:

```txt
nodes_graviton.tf
```

```hcl
resource "aws_eks_node_group" "graviton" {
  cluster_name    = aws_eks_cluster.main.id
  node_group_name = "${var.project_name}-workers-graviton"

  node_role_arn = aws_iam_role.eks_nodes_role.arn

  instance_types = [
    "t4g.large",
    "c7g.large",
  ]

  subnet_ids = data.aws_ssm_parameter.pod_subnets[*].value

  scaling_config {
    desired_size = lookup(var.auto_scale_options, "desired")
    max_size     = lookup(var.auto_scale_options, "max")
    min_size     = lookup(var.auto_scale_options, "min")
  }

  capacity_type = "ON_DEMAND"

  ami_type = "AL2023_ARM_64_STANDARD"

  labels = {
    "capacity/os"   = "AMAZON_LINUX"
    "capacity/arch" = "ARM64"
    "capacity/type" = "ON_DEMAND"
  }

  tags = {
    "kubernetes.io/cluster/${var.project_name}" = "owned"
  }

  depends_on = [
    # kubernetes_config_map_v1.aws_auth
    aws_eks_access_entry.nodes
  ]

  lifecycle {
    ignore_changes = [
      scaling_config[0].desired_size
    ]
  }

  timeouts {
    create = "1h"
    update = "2h"
    delete = "2h"
  }
}
```

---

## 7. Creating a Spot Version

We can also create a Spot version of the Graviton node group.

```txt
nodes_graviton_spot.tf
```

Same configuration, just some changes:

```hcl
resource "aws_eks_node_group" "graviton_spot" {
  cluster_name    = aws_eks_cluster.main.id
  node_group_name = "${var.project_name}-workers-graviton"

  node_role_arn = aws_iam_role.eks_nodes_role.arn

  instance_types = [
    "t4g.large",
    "c7g.large",
  ]

  subnet_ids = data.aws_ssm_parameter.pod_subnets[*].value

  scaling_config {
    desired_size = lookup(var.auto_scale_options, "desired")
    max_size     = lookup(var.auto_scale_options, "max")
    min_size     = lookup(var.auto_scale_options, "min")
  }

  capacity_type = "SPOT"

  ami_type = "AL2023_ARM_64_STANDARD"

  labels = {
    "capacity/os"   = "AMAZON_LINUX"
    "capacity/arch" = "ARM64"
    "capacity/type" = "SPOT"
  }

  tags = {
    "kubernetes.io/cluster/${var.project_name}" = "owned"
  }

  depends_on = [
    # kubernetes_config_map_v1.aws_auth
    aws_eks_access_entry.nodes
  ]

  lifecycle {
    ignore_changes = [
      scaling_config[0].desired_size
    ]
  }

  timeouts {
    create = "1h"
    update = "2h"
    delete = "2h"
  }
}
```

Now we have:

```txt
Graviton On-Demand  
Graviton Spot
```

> **Note:** the real `nodes_graviton_spot.tf` reuses the exact same `node_group_name` (`"${var.project_name}-workers-graviton"`) as the on-demand version above — it does not append a `-spot` suffix, unlike the Bottlerocket Spot file. This is a naming-collision risk in the source Terraform, reproduced here faithfully.

---

## 8. Applying the Configuration

Run:

```bash
terraform apply --auto-approve --var-file=environment/prod/terraform.tfvars
```

After a few minutes, the new ARM64 nodes will be available in the cluster.

---

## 9. Validating the Nodes

Now we can check all nodes in the cluster.

Command:

```bash
kubectl get nodes --show-labels
```

You should now see a mix of:

```txt
x86 nodes  
ARM64 nodes  
Amazon Linux nodes  
Bottlerocket nodes  
Spot nodes  
On-Demand nodes
```

---

## 10. Why Labels Are Important

Labels are critical to control where workloads run.

They allow us to:

```txt
Separate workloads by architecture  
Separate workloads by OS  
Control cost strategies (Spot vs On-Demand)  
Isolate critical applications
```

Example labels we used:

```hcl
"capacity/arch" = "ARM64"
"capacity/os"   = "AMAZON_LINUX"
"capacity/type" = "SPOT" / "ON_DEMAND"
```

---

# Lesson 4: Workload Segregation Using Node Selector

In this lesson we will learn how to control **where pods run inside the cluster** using node labels and `nodeSelector`.

---

## 1. Understanding Node Selector

Node Selector allows us to **force a pod to run only on specific nodes** based on labels.

Example idea:

```txt
Run only on x86 nodes  
Avoid ARM nodes  
Run only on On-Demand  
Avoid Spot nodes
```

This is useful when workloads have **specific requirements**.

---

## 2. Using the Example Deployment

We reuse the previous application and add a `nodeSelector`.

Create a new file `chip-node-selector.yaml`:

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
  replicas: 4
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
      nodeSelector:
        capacity/arch: x86_64
        capacity/type: ON_DEMAND
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
```

This tells Kubernetes:

```txt
Only schedule pods on nodes that match these labels
```

---

## 3. Applying the Deployment

Apply the manifest:

```bash
kubectl apply -f chip-node-selector.yaml
```

Then monitor:

```bash
kubectl get pods -w
```

Pods will only start on nodes that match the selector.

---

## 4. Validating Where Pods Are Running

To confirm:

```bash
kubectl get pods -n chip -o wide
```

Check the node column and verify:

```txt
Node is x86  
Node is On-Demand  
Node is NOT ARM  
Node is NOT Spot
```

This proves the selector is working.

> **Note:** the real node groups label nodes `"capacity/arch" = "X86_64"` (uppercase), while this manifest's `nodeSelector` uses `capacity/arch: x86_64` (lowercase). Kubernetes label matching is case-sensitive, so against the real cluster this selector would never match and the pod would stay `Pending`, contradicting the claim above.

---

## 5. Changing Strategy to Spot

Now we can modify the selector:

```yaml
capacity/type: SPOT
```

Re-apply:

```bash
kubectl apply -f chip-node-selector.yaml
```

Now the same workload will run only on:

```txt
Spot nodes  
Still respecting x86 (if defined)
```

---

## 6. Real Use Cases

This approach is extremely powerful.

Examples:

- Machine Learning workloads:
  - Run only on GPU/Neuron nodes

- Critical applications:
  - Run only on On-Demand nodes

- Cost optimization:
  - Run non-critical workloads on Spot

- Architecture constraints:
  - Force workloads to run only on x86 or ARM

---

## 7. Combining with Node Groups

Since we created multiple node groups:

```txt
x86 + ARM  
On-Demand + Spot  
Amazon Linux + Bottlerocket
```

We can now fully control scheduling using labels.

---

# Lesson 5: Using Node Affinity for Smarter Scheduling

In this lesson we go beyond `nodeSelector` and introduce **Node Affinity**, which allows more flexible and intelligent scheduling.

---

## 1. Understanding Node Affinity

Node Affinity is a more advanced version of node selection.

Instead of forcing strict rules, it allows us to:

```txt
Define preferences  
Suggest distribution  
Create flexible scheduling rules
```

Key difference:

```txt
nodeSelector → strict (must match)  
nodeAffinity → flexible (can prefer or require)
```

---

## 2. Creating a Node Affinity Deployment

We create a new deployment using `affinity`.

Inside the spec:

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
  replicas: 4
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
      nodeSelector:
        capacity/arch: x86_64

      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 50
            preference:
              matchExpressions:
              - key: capacity/type
                operator: In
                values:
                - SPOT
          - weight: 50 
            preference:
              matchExpressions:
              - key: capacity/type
                operator: In
                values:
                - ON_DEMAND

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
          value: "false"                                        
      terminationGracePeriodSeconds: 60
```

This tells Kubernetes:

```txt
Try to balance pods between Spot and On-Demand
```

---

## 3. Applying the Configuration

Apply the deployment:

```bash
kubectl apply -f chip-node-affinity.yaml
```

Then monitor:

```bash
kubectl get pods -n chip -o wide
```

---

## 4. Observing Pod Distribution

Check where pods are running:

```bash
kubectl get pods -o wide
```

Expected behavior:

```txt
~50% on Spot nodes  
~50% on On-Demand nodes
```

Example:

```txt
Pod 1 → Spot  
Pod 2 → On-Demand  
Pod 3 → Spot  
Pod 4 → On-Demand
```

> **Note:** this manifest combines the same lowercase `capacity/arch: x86_64` as a *hard* `nodeSelector` with the soft `nodeAffinity` above. Since the real node groups label nodes `"capacity/arch" = "X86_64"` (uppercase), the `nodeSelector` would never match in the real cluster — the pod would never schedule at all, not just fail to balance ~50/50 as described.

---

## 5. Important Behavior

Node Affinity with `preferredDuringSchedulingIgnoredDuringExecution` is a **soft rule**.

That means:

```txt
If Spot is unavailable → pods go to On-Demand  
If On-Demand is unavailable → pods go to Spot  
```

It does NOT block scheduling.

---

## 6. Real Use Case

This is extremely useful for cost optimization strategies:

```txt
Run part of workload on Spot (cheaper)  
Keep fallback on On-Demand (stable)  
Avoid downtime if Spot is unavailable
```

You can tune weights:

```txt
weight 80 → prefer Spot more  
weight 20 → less preference for On-Demand
```

---

## Key Takeaways

- Node Affinity allows **intelligent workload distribution**
- Works great with multiple node groups
- Ideal for:
  - Cost optimization
  - High availability strategies
  - Flexible scheduling

Compared to nodeSelector, it gives much more control without being restrictive.

---

# Lesson 6: Segregating Workloads by Criticality

In this lesson we explore a practical strategy: **separating workloads based on criticality** using node labels and node groups.

---

## 1. Defining Critical vs Non-Critical Nodes

We can create different node groups based on how critical the workloads are.

Example strategy:

```txt
Critical nodes → stable (On-Demand)  
Non-critical nodes → flexible (Spot)
```

We use labels to represent this:

```txt
severity = critical  
severity = soft
```

---

## 2. Creating a Critical Node Group

We can reuse an existing node group (e.g., Bottlerocket) and adapt it.

Create a new file:

```txt
nodes_critical.tf
```

```hcl
resource "aws_eks_node_group" "critical" {
  cluster_name    = aws_eks_cluster.main.id
  node_group_name = "${var.project_name}-workers-critical"

  node_role_arn = aws_iam_role.eks_nodes_role.arn

  instance_types = var.nodes_instance_sizes

  subnet_ids = data.aws_ssm_parameter.pod_subnets[*].value

  scaling_config {
    desired_size = lookup(var.auto_scale_options, "desired")
    max_size     = lookup(var.auto_scale_options, "max")
    min_size     = lookup(var.auto_scale_options, "min")
  }

  capacity_type = "ON_DEMAND"

  ami_type = "BOTTLEROCKET_x86_64"

  labels = {
    "capacity/os"   = "BOTTLEROCKET"
    "capacity/arch" = "X86_64"
    "capacity/type" = "ON_DEMAND"
    "severity"      = "critical"
  }

  tags = {
    "kubernetes.io/cluster/${var.project_name}" = "owned"
  }

  depends_on = [
    # kubernetes_config_map_v1.aws_auth
    aws_eks_access_entry.nodes
  ]

  lifecycle {
    ignore_changes = [
      scaling_config[0].desired_size
    ]
  }

  timeouts {
    create = "1h"
    update = "2h"
    delete = "2h"
  }
}
```

This node group will host:

```txt
Critical applications  
High availability workloads  
Production-sensitive services
```

---

## 3. Creating a Soft (Non-Critical) Node Group

Now we create another node group for less critical workloads.

Create a new file:

```txt
nodes_soft.tf
```

```hcl
resource "aws_eks_node_group" "soft" {
  cluster_name    = aws_eks_cluster.main.id
  node_group_name = "${var.project_name}-workers-soft"

  node_role_arn = aws_iam_role.eks_nodes_role.arn

  instance_types = var.nodes_instance_sizes

  subnet_ids = data.aws_ssm_parameter.pod_subnets[*].value

  scaling_config {
    desired_size = lookup(var.auto_scale_options, "desired")
    max_size     = lookup(var.auto_scale_options, "max")
    min_size     = lookup(var.auto_scale_options, "min")
  }

  capacity_type = "SPOT"

  ami_type = "BOTTLEROCKET_x86_64"

  labels = {
    "capacity/os"   = "BOTTLEROCKET"
    "capacity/arch" = "X86_64"
    "capacity/type" = "SPOT"
    "severity"      = "soft"
  }

  tags = {
    "kubernetes.io/cluster/${var.project_name}" = "owned"
  }

  depends_on = [
    # kubernetes_config_map_v1.aws_auth
    aws_eks_access_entry.nodes
  ]

  lifecycle {
    ignore_changes = [
      scaling_config[0].desired_size
    ]
  }

  timeouts {
    create = "1h"
    update = "2h"
    delete = "2h"
  }
}
```

This node group is ideal for:

```txt
Batch jobs  
Test workloads  
Non-critical services  
Cost-optimized workloads
```

---

## 4. Applying the Configuration

Run:

```bash
terraform apply --auto-approve --var-file=environment/prod/terraform.tfvars
```

After provisioning, your cluster will contain nodes labeled like:

```txt
severity = critical  
severity = soft
```

---

## 5. Validating Node Labels

You can verify labels with:

```bash
kubectl get nodes --show-labels
```

Or filter specific labels:

```bash
kubectl get nodes -o jsonpath="{.items[*].metadata.labels.severity}"
```

---

## 6. Using Criticality in Deployments

Now we can control where workloads run.

Example:

```yaml
nodeSelector:
  severity: critical
```

This ensures:

```txt
Only critical nodes will run this workload
```

---

## 7. Real Use Case

This pattern is very powerful in real environments:

- Critical workloads:
  - Run only on On-Demand nodes
  - Higher stability and SLA

- Non-critical workloads:
  - Run on Spot nodes
  - Lower cost, higher volatility

This allows:

```txt
Better cost control  
Better reliability for critical systems  
Efficient resource utilization
```

---

## Key Takeaways

By combining:

```txt
Multiple node groups  
Labels  
nodeSelector / affinity
```

You can design a cluster that intelligently separates workloads based on business needs.

---

# Lesson 7: Customizing Node Groups with Launch Templates

In this lesson we learn how to extend Managed Node Groups using **Launch Templates** to gain more control over configuration.

---

## 1. Why Use Launch Templates

Managed Node Groups already handle most things automatically.

But sometimes we need more control, such as:

```txt
Custom AMI  
Custom user_data  
Disk configuration (EBS)  
Additional tags  
Security hardening  
```

For this, we use **Launch Templates**.

---

## 2. What We Can Customize

With a Launch Template, we can define:

```txt
AMI ID  
Instance storage (EBS size/type)  
User data (bootstrap script)  
Tags for instances  
Monitoring settings  
```

This gives us much more flexibility than default node groups.

---

## 3. Preparing User Data

First, we extract the default user data from an existing instance.

Then we create a file:

```txt
files/user-data/user-data.tpl
```

Inside it, we replace static values with variables:

```yaml
MIME-Version: 1.0
Content-Type: multipart/mixed; boundary="//"

--//
Content-Type: application/node.eks.aws

---
apiVersion: node.eks.aws/v1alpha1
kind: NodeConfig
spec:
  cluster:
    apiServerEndpoint: ${KUBERNETES_ENDPOINT}
    certificateAuthority: ${KUBERNETES_CERTIFICATE_AUTHORITY}
    cidr: 172.20.0.0/16
    name: ${CLUSTER_NAME}
  kubelet:
    config:
      maxPods: 35
      clusterDNS:
      - 172.20.0.10

--//--
```

This allows Terraform to dynamically inject values.

---

## 4. Creating AMI Variable

We define a variable for the custom AMI:

```hcl
variable "custom_ami" {
  type        = string
  description = "Customized AMI ID for the nodes"
  default     = "ami-01d396130bcd204a1"
}
```

---

## 5. Creating the Launch Template

Create a new file:

```txt
node_custom.tf
```

Now we create the resource:

```hcl
resource "aws_launch_template" "custom" {
  name = "${var.project_name}-custom"

  block_device_mappings {
    device_name = "/dev/xvda"

    ebs {
      volume_size           = 20
      volume_type           = "gp3"
      delete_on_termination = true
    }
  }

  ebs_optimized = true

  monitoring {
    enabled = false
  }

  tag_specifications {
    resource_type = "instance"

    tags = {
      Name = format("%s-custom", var.project_name)
    }
  }

  user_data = base64encode(templatefile("${path.module}/files/user-data/user-data.tpl", {
    CLUSTER_NAME                     = aws_eks_cluster.main.id
    KUBERNETES_ENDPOINT              = aws_eks_cluster.main.endpoint
    KUBERNETES_CERTIFICATE_AUTHORITY = aws_eks_cluster.main.certificate_authority.0.data
  }))

}
```

> **Note:** `var.custom_ami` is declared but never referenced anywhere in the real Terraform — this launch template sets no `image_id`, and the node group below (§8) sets no `ami_type`. Despite the variable's existence, no custom AMI is actually wired in by this configuration.

---

## 6. Passing Cluster Data to User Data

We inject required values:

```txt
cluster_name  
endpoint  
certificate_authority  
```

These come from the EKS cluster resource.

---

## 7. Validating the Launch Template

After:

```bash
terraform apply --auto-approve --var-file=environment/prod/terraform.tfvars
```

Check in AWS:

```txt
EC2 → Launch Templates
```

Confirm:

```txt
User data is correct  
AMI is correct  
Settings are applied
```

> **Note:** as flagged in §5, this exact configuration never actually overrides the AMI (no `image_id`/`ami_type` wired to `var.custom_ami`), so in practice the "AMI is correct" check would just be confirming the default AMI.

---

## 8. Creating a Custom Node Group

Now we create a new node group using the Launch Template.

In the file `node_custom.tf` create:

```hcl
resource "aws_eks_node_group" "custom" {
  cluster_name    = aws_eks_cluster.main.id
  node_group_name = "${var.project_name}-custom"

  node_role_arn = aws_iam_role.eks_nodes_role.arn

  instance_types = var.nodes_instance_sizes

  subnet_ids = data.aws_ssm_parameter.pod_subnets[*].value

  launch_template {
    id      = aws_launch_template.custom.id
    version = aws_launch_template.custom.latest_version
  }

  scaling_config {
    desired_size = lookup(var.auto_scale_options, "desired")
    max_size     = lookup(var.auto_scale_options, "max")
    min_size     = lookup(var.auto_scale_options, "min")
  }

  capacity_type = "ON_DEMAND"

  labels = {
    "capacity/os"   = "AMAZON_LINUX"
    "capacity/arch" = "X86_64"
    "capacity/type" = "ON_DEMAND"
  }

  tags = {
    "kubernetes.io/cluster/${var.project_name}" = "owned"
  }

  depends_on = [
    # kubernetes_config_map_v1.aws_auth
    aws_eks_access_entry.nodes
  ]

  lifecycle {
    ignore_changes = [
      scaling_config[0].desired_size
    ]
  }

  timeouts {
    create = "1h"
    update = "2h"
    delete = "2h"
  }
}
```

This overrides default behavior.

---

## 9. Applying and Validating

Run:

```bash
terraform apply --auto-approve --var-file=environment/prod/terraform.tfvars
```

Then check EC2 instances:

```txt
Instances should have custom name/tags  
Custom AMI should be used  
Custom disk config should be applied
```

> **Note:** again, since no `image_id`/`ami_type` is actually wired to `var.custom_ami` in this configuration (see §5), the instances will run the default AMI, not a custom one.

---

## 10. Real Use Cases

Launch Templates are useful when you need:

- Golden Images (company standard AMIs)
- Pre-installed agents (security, monitoring)
- Custom bootstrap logic
- Advanced storage configuration

---

## 11. Best Practice

- Prefer default Managed Node Groups when possible
- Use Launch Templates only when necessary
- Avoid unnecessary complexity

They are powerful, but increase operational overhead.

---

# Lesson 8: Cluster Autoscaler and Dynamic Capacity Management

In this lesson, we will explore how to make our Kubernetes cluster dynamically scalable using the Cluster Autoscaler. Instead of manually managing capacity, we will enable the cluster to automatically add or remove nodes based on workload demand. We will also cover how to securely grant permissions using IAM (IRSA), deploy the Autoscaler using Helm, and understand how it reacts to real scenarios like sudden traffic spikes. Finally, we introduce the Node Termination Handler, which ensures workloads are safely handled during instance interruptions, especially when using Spot instances.

---

## 1. Introduction to Cluster Autoscaler

Now that we already know how to create multiple node groups, we need a way to **automatically scale cluster capacity**.

The Cluster Autoscaler is responsible for:

```txt
Add nodes when pods are pending (no capacity)
Remove nodes when they are underutilized
```

It works directly with node groups and their Auto Scaling Groups.

---

## 2. Creating IAM Role for Autoscaler (IRSA)

To allow the Autoscaler to interact with AWS, we must create an IAM Role using IRSA (OIDC).

Key idea:

```txt
Pods assume an IAM Role via OIDC (Web Identity)
```

Main configuration, create a new file `iam_cluster_autoscaler.tf` (full file):

```hcl
data "aws_iam_policy_document" "autoscaler" {
  statement {
    actions = [
      "sts:AssumeRoleWithWebIdentity"
    ]

    effect = "Allow"

    principals {
      type        = "Federated"
      identifiers = [aws_iam_openid_connect_provider.eks.arn]
    }
  }
}

resource "aws_iam_role" "autoscaler" {
  name               = format("%s-autoscaler", var.project_name)
  assume_role_policy = data.aws_iam_policy_document.autoscaler.json
}

data "aws_iam_policy_document" "autoscaler_policy" {
  version = "2012-10-17"

  statement {

    effect = "Allow"
    actions = [
      "autoscaling-plans:DescribeScalingPlans",
      "autoscaling-plans:GetScalingPlanResourceForecastData",
      "autoscaling-plans:DescribeScalingPlanResources",
      "autoscaling:DescribeAutoScalingNotificationTypes",
      "autoscaling:DescribeLifecycleHookTypes",
      "autoscaling:DescribeAutoScalingInstances",
      "autoscaling:DescribeTerminationPolicyTypes",
      "autoscaling:DescribeScalingProcessTypes",
      "autoscaling:DescribePolicies",
      "autoscaling:DescribeTags",
      "autoscaling:DescribeLaunchConfigurations",
      "autoscaling:DescribeMetricCollectionTypes",
      "autoscaling:DescribeLoadBalancers",
      "autoscaling:DescribeLifecycleHooks",
      "autoscaling:DescribeAdjustmentTypes",
      "autoscaling:DescribeScalingActivities",
      "autoscaling:DescribeAutoScalingGroups",
      "autoscaling:DescribeAccountLimits",
      "autoscaling:DescribeScheduledActions",
      "autoscaling:DescribeLoadBalancerTargetGroups",
      "autoscaling:DescribeNotificationConfigurations",
      "autoscaling:DescribeInstanceRefreshes",
      "autoscaling:SetDesiredCapacity",
      "autoscaling:TerminateInstanceInAutoScalingGroup",
      "ec2:DescribeLaunchTemplateVersions"
    ]

    resources = [
      "*"
    ]
  }
}

resource "aws_iam_policy" "autoscaler" {
  name   = format("%s-autoscaler", var.project_name)
  policy = data.aws_iam_policy_document.autoscaler_policy.json
}

resource "aws_iam_role_policy_attachment" "autoscaler" {
  role       = aws_iam_role.autoscaler.name
  policy_arn = aws_iam_policy.autoscaler.arn
}
```

This creates the trust relationship between the cluster and AWS IAM.

---

## 3. Creating the IAM Policy

The Autoscaler needs permissions to manage infrastructure.

Typical permissions include:

```hcl
data "aws_iam_policy_document" "autoscaler_policy" {
  version = "2012-10-17"

  statement {

    effect = "Allow"
    actions = [
      "autoscaling-plans:DescribeScalingPlans",
      "autoscaling-plans:GetScalingPlanResourceForecastData",
      "autoscaling-plans:DescribeScalingPlanResources",
      "autoscaling:DescribeAutoScalingNotificationTypes",
      "autoscaling:DescribeLifecycleHookTypes",
      "autoscaling:DescribeAutoScalingInstances",
      "autoscaling:DescribeTerminationPolicyTypes",
      "autoscaling:DescribeScalingProcessTypes",
      "autoscaling:DescribePolicies",
      "autoscaling:DescribeTags",
      "autoscaling:DescribeLaunchConfigurations",
      "autoscaling:DescribeMetricCollectionTypes",
      "autoscaling:DescribeLoadBalancers",
      "autoscaling:DescribeLifecycleHooks",
      "autoscaling:DescribeAdjustmentTypes",
      "autoscaling:DescribeScalingActivities",
      "autoscaling:DescribeAutoScalingGroups",
      "autoscaling:DescribeAccountLimits",
      "autoscaling:DescribeScheduledActions",
      "autoscaling:DescribeLoadBalancerTargetGroups",
      "autoscaling:DescribeNotificationConfigurations",
      "autoscaling:DescribeInstanceRefreshes",
      "autoscaling:SetDesiredCapacity",
      "autoscaling:TerminateInstanceInAutoScalingGroup",
      "ec2:DescribeLaunchTemplateVersions"
    ]

    resources = [
      "*"
    ]
  }
}
```

This policy allows the Autoscaler to:

```txt
Inspect node groups
Increase/decrease capacity
Manage instances lifecycle
```

---

## 4. Attaching Policy to the Role

Now we attach the policy to the IAM Role:

```hcl
resource "aws_iam_role_policy_attachment" "autoscaler" {
  role       = aws_iam_role.autoscaler.name
  policy_arn = aws_iam_policy.autoscaler.arn
}
```

At this point:

```txt
Role is created
Policy is attached
Trust with OIDC is configured
```

The Autoscaler can now authenticate securely.

---

## 5. Deploying Cluster Autoscaler via Helm

We deploy the Autoscaler using Helm.

Create a new file `helm_cluster_autoscaler.tf`:

```hcl
resource "helm_release" "cluster_autoscaler" {

  repository = "https://kubernetes.github.io/autoscaler"

  chart = "cluster-autoscaler"
  name  = "aws-cluster-autoscaler"

  namespace        = "kube-system"
  create_namespace = true

  values = [
    yamlencode({
      replicaCount = 1
      awsRegion    = var.region
      rbac = {
        serviceAccount = {
          create = true
          annotations = {
            "eks.amazonaws.com/role-arn" = aws_iam_role.autoscaler.arn
          }
        }
      }
      autoscalingGroups = [
        {
          name    = aws_eks_node_group.main.resources[0].autoscaling_groups[0].name
          maxSize = lookup(var.auto_scale_options, "max")
          minSize = lookup(var.auto_scale_options, "min")
        }
      ]
    })
  ]

  depends_on = [
    aws_eks_cluster.main,
    aws_eks_node_group.main,
  ]
}
```

The annotation is what links the pod to the IAM Role.

---

## 6. Configuring Node Group Limits

The Autoscaler respects min/max values defined in node groups.

Example:

```txt
min_size     = 2
max_size     = 10
desired_size = 2
```

It will:

```txt
Scale up → until max
Scale down → until min
```

---

## 7. Validating Autoscaler Deployment

Check if the pod is running:

```bash
kubectl get pods -n kube-system
```

---

## 8. Testing Scale Up

Now we force the cluster to scale.

Increase replicas of an application:

```bash
kubectl scale deployment chip --replicas=300
```

What happens:

```txt
Pods go to Pending
Autoscaler detects lack of capacity
New nodes are created
```

---

## 9. Observing Node Creation

Check nodes:

```bash
kubectl get nodes
```

You will see:

```txt
New nodes joining the cluster
Nodes in NotReady → Ready state
```

Autoscaler keeps adding nodes until all pods are scheduled.

---

## 10. Testing Further Scaling

Increase even more:

```bash
kubectl scale deployment chip --replicas=500
```

Behavior:

```txt
More Pending pods
Autoscaler provisions additional nodes
Cluster stabilizes again
```

---

## 11. Scaling Down

Now reduce workload:

```bash
kubectl scale deployment chip --replicas=4
```

After some time:

```txt
Unused nodes are terminated
Cluster returns to baseline
```

---

## 12. Understanding the Behavior

Cluster Autoscaler works based on:

```txt
Pending pods → scale up
Idle nodes → scale down
```

Important:

```txt
It operates at node level (not pod level)
Depends on node groups
```

---

## 13. Introducing Node Termination Handler

When using Spot instances, nodes can be terminated by AWS.

The Node Termination Handler helps by:

```txt
Detecting termination events
Draining the node
Evicting pods safely
Rescheduling workloads
```

This avoids:

```txt
Sudden pod loss
Application downtime
```

---

## 14. Why This Is Important

Without this:

```txt
Spot instance is terminated → pods die instantly
```

With Node Termination Handler:

```txt
Pods are gracefully moved before termination
```

This is critical for production environments using Spot.

---

# Lesson 9: Node Termination Handler and Safe Workload Eviction

In this lesson, we will learn how to handle infrastructure events in a Kubernetes cluster to avoid unexpected downtime. We will implement the Node Termination Handler, which captures events such as Spot interruptions or instance shutdowns and takes proactive actions like draining nodes and safely rescheduling pods. This ensures that workloads are not abruptly interrupted and improves the overall reliability and resilience of the cluster.

---

## 1. Introduction to Node Termination Handler

The Node Termination Handler is responsible for **capturing AWS infrastructure events** and reacting before nodes are terminated.

It helps to:

```txt
Detect instance shutdown events
Safely drain nodes
Reschedule pods before termination
```

This is especially important when using **Spot instances**.

---

## 2. How the Flow Works

The solution works using AWS event integration:

```txt
AWS Event → CloudWatch Event → SQS Queue → Node Termination Handler
```

Steps:

```txt
AWS emits an event (termination, interruption, etc.)
Event is sent to SQS
Node Termination Handler consumes the message
Action is executed on the node
```

---

## 3. Creating IAM Role (IRSA)

Just like Cluster Autoscaler, we need an IAM Role.

Create a new file:

```txt
iam_node_termination_handler.tf
```

```hcl
data "aws_iam_policy_document" "node_termination_handler" {
  statement {
    actions = [
      "sts:AssumeRoleWithWebIdentity"
    ]

    effect = "Allow"

    principals {
      type        = "Federated"
      identifiers = [aws_iam_openid_connect_provider.eks.arn]
    }
  }
}

resource "aws_iam_role" "node_termination_handler" {
  name               = format("%s-node-termination-handler", var.project_name)
  assume_role_policy = data.aws_iam_policy_document.node_termination_handler.json
}

resource "aws_iam_role" "node_termination" {
  assume_role_policy = data.aws_iam_policy_document.node_termination.json
  name               = format("%s-node-termination-handler", var.project_name)
}
```

> **Note:** this second role block is an orphaned leftover in the real Terraform — it references `data.aws_iam_policy_document.node_termination`, which doesn't exist anywhere in the codebase (only the `node_termination_handler` variant above does), and it duplicates that same role's `name`. As written, this block would fail `terraform apply`.

```hcl
data "aws_iam_policy_document" "aws_node_termination_handler_policy" {
  version = "2012-10-17"

  statement {

    effect = "Allow"
    actions = [
      "autoscaling:CompleteLifecycleAction",
      "autoscaling:DescribeAutoScalingInstances",
      "autoscaling:DescribeTags",
      "ec2:DescribeInstances",
      "sqs:DeleteMessage",
      "sqs:ReceiveMessage"
    ]

    resources = [
      "*"
    ]

  }
}

resource "aws_iam_policy" "node_termination_handler" {
  name   = format("%s-node-termination-handler", var.project_name)
  policy = data.aws_iam_policy_document.aws_node_termination_handler_policy.json
}

resource "aws_iam_role_policy_attachment" "node_termination_handler" {
  role       = aws_iam_role.node_termination_handler.name
  policy_arn = aws_iam_policy.node_termination_handler.arn
}
```

This allows the handler to interact with AWS services securely.

---

## 4. Creating IAM Policy

The Node Termination Handler needs permissions such as:

```hcl
"autoscaling:CompleteLifecycleAction",
"autoscaling:DescribeAutoScalingInstances",
"autoscaling:DescribeTags",
"ec2:DescribeInstances",
"sqs:DeleteMessage",
"sqs:ReceiveMessage"
```

This enables it to:

```txt
Read instance state
Consume messages from SQS
Act based on events
```

---

## 5. Attaching Policy to Role

Attach the policy to the IAM Role:

```hcl
resource "aws_iam_role_policy_attachment" "node_termination_handler" {
  role       = aws_iam_role.node_termination_handler.name
  policy_arn = aws_iam_policy.node_termination_handler.arn
}
```

Now the handler has:

```txt
Authentication (IRSA)
Permissions (IAM Policy)
```

---

## 6. Creating SQS Queue

We create an SQS queue to receive AWS events.

Purpose:

```txt
Central place to store infrastructure events
Decouple AWS events from Kubernetes actions
```

This queue will be consumed by the handler.

Create a new file `sqs_node_termination_handler.tf`:

```hcl
resource "aws_sqs_queue" "node_termination" {
  name                       = format("%s-node-termination", var.project_name)
  delay_seconds              = 0
  message_retention_seconds  = 86400
  receive_wait_time_seconds  = 10
  visibility_timeout_seconds = 60
}

resource "aws_sqs_queue_policy" "node_termination_handler" {
  queue_url = aws_sqs_queue.node_termination.id
  policy    = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": "*",
      "Action": [
        "sqs:SendMessage"
      ],
      "Resource": [
        "${aws_sqs_queue.node_termination.arn}"
      ]
    }
  ]
}
EOF
}

resource "aws_cloudwatch_event_rule" "node_termination_handler_instance_terminate" {

  name        = format("%s-instance-terminate", var.project_name)
  description = var.project_name

  event_pattern = jsonencode({
    source = ["aws.autoscaling"]
    detail-type = [
      "EC2 Instance-terminate Lifecycle Action"
    ]
  })
}

resource "aws_cloudwatch_event_target" "node_termination_handler_instance_terminate" {

  rule      = aws_cloudwatch_event_rule.node_termination_handler_instance_terminate.name
  target_id = "SendToSQS"
  arn       = aws_sqs_queue.node_termination.arn
}

resource "aws_cloudwatch_event_rule" "node_termination_handler_scheduled_change" {

  name        = format("%s-scheduled-change", var.project_name)
  description = var.project_name

  event_pattern = jsonencode({
    source = ["aws.health"]
    detail-type = [
      "AWS Health Event"
    ]
    detail = {
      service = [
        "EC2"
      ]
      eventTypeCategory = [
        "scheduledChange"
      ]
    }
  })
}

resource "aws_cloudwatch_event_target" "node_termination_handler_scheduled_change" {

  rule      = aws_cloudwatch_event_rule.node_termination_handler_scheduled_change.name
  target_id = "SendToSQS"
  arn       = aws_sqs_queue.node_termination.arn
}

resource "aws_cloudwatch_event_rule" "node_termination_handler_spot_termination" {

  name        = format("%s-spot-termination", var.project_name)
  description = var.project_name

  event_pattern = jsonencode({
    source = ["aws.ec2"]
    detail-type = [
      "EC2 Spot Instance Interruption Warning"
    ]
  })
}

resource "aws_cloudwatch_event_target" "node_termination_handler_spot_termination" {

  rule      = aws_cloudwatch_event_rule.node_termination_handler_spot_termination.name
  target_id = "SendToSQS"
  arn       = aws_sqs_queue.node_termination.arn
}

resource "aws_cloudwatch_event_rule" "node_termination_handler_rebalance" {

  name        = format("%s-rebalance", var.project_name)
  description = var.project_name

  event_pattern = jsonencode({
    source = ["aws.ec2"]
    detail-type = [
      "EC2 Instance Rebalance Recommendation"
    ]
  })
}

resource "aws_cloudwatch_event_target" "node_termination_handler_rebalance" {

  rule      = aws_cloudwatch_event_rule.node_termination_handler_rebalance.name
  target_id = "SendToSQS"
  arn       = aws_sqs_queue.node_termination.arn
}

resource "aws_cloudwatch_event_rule" "node_termination_handler_state_change" {

  name        = format("%s-state-change", var.project_name)
  description = var.project_name

  event_pattern = jsonencode({
    source = ["aws.ec2"]
    detail-type = [
      "EC2 Instance State-change Notification"
    ]
  })
}

resource "aws_cloudwatch_event_target" "node_termination_handler_state_change" {

  rule      = aws_cloudwatch_event_rule.node_termination_handler_state_change.name
  target_id = "SendToSQS"
  arn       = aws_sqs_queue.node_termination.arn
}
```

---

## 7. Creating CloudWatch Event Rules

We configure multiple event rules to capture different scenarios:

```txt
Instance termination (Auto Scaling)
Spot interruption warnings
Scheduled maintenance events
Rebalance recommendations
Instance state changes
```

All events are sent to:

```txt
SQS Queue
```

---

## 8. Testing Event Flow (Before Deployment)

Before deploying the handler, we can validate the pipeline.

Example:

```txt
Terminate an EC2 instance manually
```

Then check SQS:

```txt
Messages will appear in the queue
```

This confirms:

```txt
Events are being captured correctly
```

---

## 9. Deploying Node Termination Handler (Helm)

Now we deploy using Helm.

Key configurations:

```txt
Namespace: kube-system
ServiceAccount annotation with IAM Role ARN
SQS Queue URL
```

Create a new file:

```txt
helm_node_termination_handler.tf
```

```hcl
resource "helm_release" "node_termination_handler" {
  name      = "aws-node-termination-handler"
  namespace = "kube-system"

  chart      = "aws-node-termination-handler"
  repository = "https://aws.github.io/eks-charts/"

  values = yamlencode({
    serviceAccount = {
      annotations = {
        "eks.amazonaws.com/role-arn" = aws_iam_role.node_termination_handler.arn
      }
    }
    awsRegion                      = var.region
    queueURL                       = aws_sqs_queue.node_termination.url
    enableSqsTerminationDraining   = true
    enableSpotInterruptionDraining = true
    enableRebalanceMonitoring      = true
    enableRebalanceDraining        = true
    enableScheduledEventDraining   = true
    deleteSqsMsgIfNodeNotFound     = true
    checkTagBeforeDraining         = false
  })
}
```

---

## 10. Validating Deployment

Check if the pod is running:

```bash
kubectl get pods -n kube-system
```

---

## 11. Verifying Event Consumption

After deployment:

```txt
SQS queue should start draining (messages disappear)
```

Meaning:

```txt
Handler is consuming and processing events
```

---

## 12. Testing Node Termination

Now test in practice.

Terminate an instance:

```txt
EC2 → Terminate instance
```

Expected behavior:

```txt
Node is cordoned (no new pods)
Node is drained (pods evicted)
Pods are rescheduled on other nodes
```

---

## 13. Observing Kubernetes Behavior

You can verify with:

```txt
kubectl get nodes
kubectl get pods -o wide
```

You will see:

```txt
Node marked as SchedulingDisabled
Pods moving to other nodes
```

---

## 14. Why This Matters

Without Node Termination Handler:

```txt
Pods are killed abruptly
Possible downtime
```

With it:

```txt
Graceful shutdown
Safer rescheduling
Higher availability
```

This is essential for:

```txt
Spot workloads
Dynamic clusters
Production environments
```
