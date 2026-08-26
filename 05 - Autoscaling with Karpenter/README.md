# Module 5: Autoscaling with Karpenter

## Overview

In this module we introduce **Karpenter**, one of the most powerful autoscaling solutions available for Kubernetes on AWS.

Previously we explored:

```txt
Cluster Autoscaler
```

which is a good starting point for node autoscaling.

Now we will move to a more advanced solution that provides:

```txt
Faster provisioning
Better cost optimization
More scheduling intelligence
Greater workload flexibility
```

Karpenter is currently one of the most recommended approaches for scaling EKS clusters in AWS.

## Table of Contents

- [Lesson 1: Introduction to Karpenter](#lesson-1-introduction-to-karpenter)
  - [1. What is Karpenter?](#1-what-is-karpenter)
  - [2. Karpenter vs Cluster Autoscaler](#2-karpenter-vs-cluster-autoscaler)
  - [3. Why Karpenter Is More Efficient](#3-why-karpenter-is-more-efficient)
  - [4. Main Karpenter Components](#4-main-karpenter-components)
  - [5. Understanding EC2NodeClass](#5-understanding-ec2nodeclass)
  - [6. Understanding NodePools](#6-understanding-nodepools)
  - [7. EC2NodeClass vs NodePool](#7-ec2nodeclass-vs-nodepool)
  - [8. Flexible Capacity Management](#8-flexible-capacity-management)
  - [9. Advanced Scheduling Strategies](#9-advanced-scheduling-strategies)
  - [10. Faster Node Provisioning](#10-faster-node-provisioning)
  - [11. Cost Optimization](#11-cost-optimization)
  - [12. Spot Workloads with Karpenter](#12-spot-workloads-with-karpenter)
  - [13. Real Production Use Cases](#13-real-production-use-cases)
  - [14. Productizing Karpenter](#14-productizing-karpenter)
  - [15. Additional Learning Resources](#15-additional-learning-resources)
  - [16. What We Will Build](#16-what-we-will-build)
- [Lesson 2: Installing Karpenter on EKS](#lesson-2-installing-karpenter-on-eks)
  - [1. Starting from a Vanilla Cluster](#1-starting-from-a-vanilla-cluster)
  - [2. Creating the Karpenter IAM Resources](#2-creating-the-karpenter-iam-resources)
  - [3. Required Permissions](#3-required-permissions)
  - [4. Understanding SQS Permissions](#4-understanding-sqs-permissions)
  - [5. Creating the IAM Role](#5-creating-the-iam-role)
  - [6. Applying the IAM Configuration](#6-applying-the-iam-configuration)
  - [7. Installing Karpenter with Helm](#7-installing-karpenter-with-helm)
  - [8. Using the Official Repository](#8-using-the-official-repository)
  - [9. Configuring IRSA](#9-configuring-irsa)
  - [10. Required Helm Parameters](#10-required-helm-parameters)
    - [Cluster Name](#cluster-name)
    - [Cluster Endpoint](#cluster-endpoint)
    - [Instance Profile](#instance-profile)
  - [11. Understanding the Instance Profile](#11-understanding-the-instance-profile)
  - [12. Deploying Karpenter](#12-deploying-karpenter)
  - [13. Verifying the Installation](#13-verifying-the-installation)
  - [14. Monitoring Karpenter Logs](#14-monitoring-karpenter-logs)
  - [15. What Has Been Installed?](#15-what-has-been-installed)
  - [Key Takeaways](#key-takeaways)
- [Lesson 3: Understanding NodePools and EC2NodeClasses in Karpenter](#lesson-3-understanding-nodepools-and-ec2nodeclasses-in-karpenter)
  - [1. Creating an EC2NodeClass](#1-creating-an-ec2nodeclass)
  - [2. Understanding Each EC2NodeClass Setting](#2-understanding-each-ec2nodeclass-setting)
    - [AMI Family](#ami-family)
    - [Instance Profile](#instance-profile-1)
    - [Subnets](#subnets)
    - [Security Groups](#security-groups)
    - [AMI Selection](#ami-selection)
  - [3. Creating a NodePool](#3-creating-a-nodepool)
  - [4. Defining Instance Families](#4-defining-instance-families)
  - [5. Choosing Spot or On-Demand](#5-choosing-spot-or-on-demand)
  - [6. Understanding Consolidation](#6-understanding-consolidation)
  - [7. Applying the Configuration](#7-applying-the-configuration)
  - [8. Deploying a Test Application](#8-deploying-a-test-application)
  - [9. Triggering Node Provisioning](#9-triggering-node-provisioning)
  - [10. Watching Karpenter Create Nodes](#10-watching-karpenter-create-nodes)
  - [11. Scaling Back Down](#11-scaling-back-down)
  - [12. Watching Consolidation in Action](#12-watching-consolidation-in-action)
  - [13. Why Karpenter Is More Powerful Than Cluster Autoscaler](#13-why-karpenter-is-more-powerful-than-cluster-autoscaler)
  - [14. What We Will Improve Next](#14-what-we-will-improve-next)
- [Lesson 4: Automating Karpenter NodePools with Terraform](#lesson-4-automating-karpenter-nodepools-with-terraform)
  - [1. Installing the Kubectl Manifest Provider](#1-installing-the-kubectl-manifest-provider)
  - [2. Defining the Capacity Configuration](#2-defining-the-capacity-configuration)
  - [3. Creating the Capacity Object](#3-creating-the-capacity-object)
  - [4. Retrieving the Latest AMI Automatically](#4-retrieving-the-latest-ami-automatically)
  - [5. Creating Template Files](#5-creating-template-files)
  - [6. Building the EC2NodeClass Template](#6-building-the-ec2nodeclass-template)
  - [7. Handling Dynamic Subnets](#7-handling-dynamic-subnets)
  - [8. Creating the EC2NodeClass Resource](#8-creating-the-ec2nodeclass-resource)
  - [9. Validating the EC2NodeClass](#9-validating-the-ec2nodeclass)
  - [10. Building the NodePool Template](#10-building-the-nodepool-template)
  - [11. Generating Instance Families](#11-generating-instance-families)
  - [12. Generating Instance Sizes](#12-generating-instance-sizes)
  - [13. Generating Capacity Types](#13-generating-capacity-types)
  - [14. Generating Availability Zones](#14-generating-availability-zones)
  - [15. Creating Dynamic Workload Labels](#15-creating-dynamic-workload-labels)
  - [16. Creating the NodePool Resource](#16-creating-the-nodepool-resource)
  - [17. Applying the Configuration](#17-applying-the-configuration)
  - [18. Benefits of This Approach](#18-benefits-of-this-approach)
  - [19. Preparing for Workload Segregation](#19-preparing-for-workload-segregation)
- [Lesson 5: Improving Availability with Topology Spread Constraints](#lesson-5-improving-availability-with-topology-spread-constraints)
  - [1. The Problem: Uneven Distribution](#1-the-problem-uneven-distribution)
  - [2. Spreading Pods Across Availability Zones](#2-spreading-pods-across-availability-zones)
  - [3. Diversifying Instance Types and Sizes](#3-diversifying-instance-types-and-sizes)
  - [4. Mixing Spot and On-Demand Capacity](#4-mixing-spot-and-on-demand-capacity)
  - [5. Verifying the Distribution](#5-verifying-the-distribution)
  - [Key Takeaways](#key-takeaways-1)
- [Lesson 6: Segregating Workloads with Multiple NodePools](#lesson-6-segregating-workloads-with-multiple-nodepools)
  - [1. Why Segregate Workloads](#1-why-segregate-workloads)
  - [2. Adding a Dedicated NodePool via Terraform](#2-adding-a-dedicated-nodepool-via-terraform)
  - [3. Understanding the karpenter.sh/nodepool Label](#3-understanding-the-karpentershnodepool-label)
  - [4. Targeting a NodePool with nodeSelector](#4-targeting-a-nodepool-with-nodeselector)
  - [5. Creating Criticality-Based NodePools](#5-creating-criticality-based-nodepools)
  - [6. Creating OS/AMI-Specific NodePools](#6-creating-osami-specific-nodepools)
  - [7. Switching Workloads Between NodePools](#7-switching-workloads-between-nodepools)
  - [Key Takeaways](#key-takeaways-2)
- [Lesson 7: Native Interruption Handling in Karpenter](#lesson-7-native-interruption-handling-in-karpenter)
  - [1. How Karpenter Handles Interruptions](#1-how-karpenter-handles-interruptions)
  - [2. Creating the SQS Queue](#2-creating-the-sqs-queue)
  - [3. Allowing EventBridge to Publish to the Queue](#3-allowing-eventbridge-to-publish-to-the-queue)
  - [4. Creating EventBridge Rules for Instance Lifecycle Events](#4-creating-eventbridge-rules-for-instance-lifecycle-events)
  - [5. Enabling the Interruption Queue](#5-enabling-the-interruption-queue)
  - [6. Testing Interruption Handling](#6-testing-interruption-handling)
  - [7. Karpenter vs. the Standalone Node Termination Handler](#7-karpenter-vs-the-standalone-node-termination-handler)
  - [Key Takeaways](#key-takeaways-3)

---

# Lesson 1: Introduction to Karpenter

---

## 1. What is Karpenter?

Karpenter is an open-source node autoscaling solution originally created by AWS.

Its goal is simple:

```txt
Provision the right capacity
At the right time
Using the right infrastructure
```

Unlike Cluster Autoscaler, Karpenter does not simply increase the size of existing node groups.

Instead, it analyzes pending workloads and creates infrastructure that best fits those workloads.

---

## 2. Karpenter vs Cluster Autoscaler

Cluster Autoscaler works primarily by:

```txt
Detecting pending pods
Increasing node group size
Waiting for new nodes
```

Karpenter goes much further.

It evaluates:

```txt
Pod requirements
Instance families
Instance sizes
Spot availability
Cost optimization
Scheduling constraints
```

This allows Karpenter to make smarter infrastructure decisions.

---

## 3. Why Karpenter Is More Efficient

When a pod becomes pending, Karpenter evaluates the workload requirements before provisioning capacity.

Example:

```txt
CPU requirements
Memory requirements
Scheduling constraints
Node labels
Architecture requirements
```

Instead of simply adding another node, Karpenter determines which instance type best satisfies the workload.

Benefits:

```txt
Faster scaling
Lower cost
Better resource utilization
```

---

## 4. Main Karpenter Components

Karpenter relies on a small set of Custom Resource Definitions (CRDs).

The two most important are:

```txt
EC2NodeClass
NodePool
```

These CRDs define how infrastructure should be provisioned.

---

## 5. Understanding EC2NodeClass

The EC2NodeClass defines the infrastructure template that Karpenter will use.

Think of it as:

```txt
EC2 configuration template
```

Typical settings include:

```txt
AMI selection
Subnet selection
Security groups
Instance profile
Storage configuration
```

Everything related to the EC2 instance itself is defined here.

---

## 6. Understanding NodePools

The NodePool defines how Karpenter should provision and manage capacity.

Examples:

```txt
Spot instances
On-Demand instances
Instance families
Instance sizes
Availability zones
```

This is where we define the behavior of the autoscaling strategy.

---

## 7. EC2NodeClass vs NodePool

A simple way to think about the relationship is:

```txt
EC2NodeClass
  = How the machine looks

NodePool
  = How the machine is used
```

Example:

```txt
EC2NodeClass
  Amazon Linux 2023
  Specific subnets
  Specific security groups

NodePool
  Spot instances
  C-family instances
  Large and XLarge sizes
```

---

## 8. Flexible Capacity Management

One of Karpenter's biggest advantages is flexibility.

Different workloads can use different NodePools.

Examples:

```txt
Machine Learning workloads
Batch processing
Critical applications
Development workloads
```

Each workload can have its own scaling strategy.

---

## 9. Advanced Scheduling Strategies

Karpenter allows much more granular control over infrastructure.

Examples:

```txt
Dedicated Spot pools
Dedicated On-Demand pools
GPU workloads
ARM workloads
High-memory workloads
```

This enables workload segregation without creating multiple managed node groups.

---

## 10. Faster Node Provisioning

Karpenter is generally faster than Cluster Autoscaler.

Reasons:

```txt
Direct integration with AWS APIs
No dependency on node group scaling
Workload-aware provisioning
```

Instead of scaling an existing node group, Karpenter provisions the exact infrastructure required.

---

## 11. Cost Optimization

One of Karpenter's strongest features is cost optimization.

It can evaluate:

```txt
Instance prices
Spot capacity
Available instance types
Workload requirements
```

And choose the most efficient option.

This often results in significant savings.

---

## 12. Spot Workloads with Karpenter

Karpenter is especially powerful when combined with Spot Instances.

Benefits:

```txt
Automatic diversification
Capacity-aware scheduling
Reduced interruption risk
Lower costs
```

Many production environments run large percentages of Spot capacity using Karpenter.

---

## 13. Real Production Use Cases

Common Karpenter use cases include:

```txt
High-scale production clusters
Machine Learning platforms
Batch processing systems
Cost-optimized Kubernetes environments
```

It is widely adopted because it adapts infrastructure dynamically to workload demand.

---

## 14. Productizing Karpenter

One of the goals of this module is to make Karpenter easy to consume.

We will focus on:

```txt
Reusable Terraform modules
Standardized NodePools
Standardized EC2NodeClasses
Production-ready patterns
```

The objective is to reduce operational complexity while preserving flexibility.

---

## 15. Additional Learning Resources

The course materials include an additional article covering:

```txt
Production Spot strategies
Karpenter best practices
Cost optimization techniques
```

Although written some time ago, the concepts remain highly relevant.

It is strongly recommended reading material.

---

## 16. What We Will Build

Throughout this module we will:

```txt
Install Karpenter
Integrate it with EKS
Create EC2NodeClasses
Create NodePools
Provision nodes dynamically
Explore production-ready patterns
```

By the end of the module, you will have a complete understanding of how to use Karpenter as the primary autoscaling solution for EKS.

---

# Lesson 2: Installing Karpenter on EKS

In this lesson we will install **Karpenter** in our EKS cluster.

The goal is to prepare the environment so Karpenter can start provisioning EC2 instances dynamically based on workload demands.

At this stage, we will focus only on:

```txt
Creating the IAM permissions
Deploying Karpenter with Helm
Connecting Karpenter to the cluster
Validating the installation
```

We will configure NodePools and EC2NodeClasses in the next lessons.

---

## 1. Starting from a Vanilla Cluster

To simplify the setup, we will use a clean EKS cluster.

Current environment:

```txt
EKS Cluster
No Karpenter
No NodePools
No EC2NodeClasses
```

This allows us to understand exactly what Karpenter requires to operate.

---

## 2. Creating the Karpenter IAM Resources

The first requirement is creating the IAM resources used by Karpenter.

Create a new file:

```txt
iam_karpenter.tf
```

```hcl
data "aws_iam_policy_document" "karpenter" {
  statement {
    actions = ["sts:AssumeRoleWithWebIdentity"]
    effect  = "Allow"

    principals {
      identifiers = [aws_iam_openid_connect_provider.eks.arn]
      type        = "Federated"
    }
  }
}

resource "aws_iam_role" "karpenter" {
  assume_role_policy = data.aws_iam_policy_document.karpenter.json
  name               = format("%s-karpenter", var.project_name)
}

data "aws_iam_policy_document" "karpenter_policy" {
  version = "2012-10-17"

  statement {

    effect = "Allow"
    actions = [
      "eks:DescribeCluster",
      "ec2:CreateLaunchTemplate",
      "ec2:CreateFleet",
      "ec2:CreateTags",
      "ec2:DescribeLaunchTemplates",
      "ec2:DescribeInstances",
      "ec2:DescribeSecurityGroups",
      "ec2:DescribeSubnets",
      "ec2:DescribeImages",
      "ec2:DescribeInstanceTypes",
      "ec2:DescribeInstanceTypeOfferings",
      "ec2:DescribeAvailabilityZones",
      "ec2:DescribeSpotPriceHistory",
      "pricing:GetProducts",
      "ec2:RunInstances",
      "ec2:TerminateInstances",
      "ec2:DeleteLaunchTemplate",
      "ssm:GetParameter",
      "iam:PassRole",
      "sqs:*"
    ]

    resources = [
      "*"
    ]

  }
}

resource "aws_iam_policy" "karpenter" {
  name   = format("%s-karpenter", var.project_name)
  path   = "/"
  policy = data.aws_iam_policy_document.karpenter_policy.json
}

resource "aws_iam_role_policy_attachment" "karpenter" {
  role       = aws_iam_role.karpenter.name
  policy_arn = aws_iam_policy.karpenter.arn
}
```

The installation requires:

```txt
IAM Policy
IAM Role
Policy Attachment
```

These resources will allow Karpenter to interact with AWS services.

---

## 3. Required Permissions

Karpenter needs permissions to manage infrastructure dynamically.

Examples include:

```txt
Describe EC2 resources
Create Launch Templates
Launch EC2 instances
Terminate EC2 instances
Read SSM parameters
Consume SQS messages
```

These permissions allow Karpenter to provision and manage capacity automatically.

---

## 4. Understanding SQS Permissions

One interesting permission is SQS access.

Why?

Because Karpenter includes functionality similar to Node Termination Handler.

It can react to events such as:

```txt
Spot interruptions
Instance terminations
Infrastructure events
```

This allows Karpenter to:

```txt
Drain nodes
Reschedule workloads
Replace capacity automatically
```

> **Note:** the `sqs:*` IAM permission is a prerequisite, not the full mechanism. To actually receive interruption events, Karpenter also needs a dedicated SQS queue plus EventBridge rules forwarding Spot interruption, rebalance-recommendation, and instance state-change notifications to it, along with the `settings.interruptionQueue` Helm value pointing at that queue. None of that infrastructure is created in this lesson, so interruption handling is not yet functional at this point in the course — it's covered by the IAM permission here so it's ready when that queue/EventBridge setup is added in [Lesson 7](#lesson-7-native-interruption-handling-in-karpenter).

---

## 5. Creating the IAM Role

Create an IAM Role for Karpenter.

This role will later be associated with the Karpenter service account through IRSA.

Flow:

```txt
Karpenter Pod
↓
Service Account
↓
IAM Role (IRSA)
↓
AWS API Access
```

This is the same authentication model used throughout previous lessons.

---

## 6. Applying the IAM Configuration

Deploy the IAM resources:

```bash
terraform apply --auto-approve --var-file=environment/prod/terraform.tfvars
```

After the deployment completes, Karpenter will have the permissions required to manage AWS infrastructure.

---

## 7. Installing Karpenter with Helm

Now we can install Karpenter itself.

Create:

```txt
helm_karpenter.tf
```

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
    }
  ]

  depends_on = [
    aws_eks_cluster.main,
    aws_eks_node_group.main
  ]

}
```

Main components:

```txt
Helm Release
Karpenter Controller
Service Account
IRSA Annotation
```

---

## 8. Using the Official Repository

Karpenter is installed directly from the official AWS repository.

Example configuration:

```txt
Repository:
public.ecr.aws/karpenter
```

Version used in the course:

```txt
1.0.8
```

---

## 9. Configuring IRSA

Because we are still using IRSA, we must annotate the service account.

The annotation links:

```txt
Service Account
↓
IAM Role
```

This allows Karpenter to authenticate securely against AWS APIs without static credentials.

---

## 10. Required Helm Parameters

Some parameters are mandatory during installation.

### Cluster Name

```txt
clusterName
```

Used by Karpenter to identify the EKS cluster.

### Cluster Endpoint

```txt
clusterEndpoint
```

Allows communication with the Kubernetes API.

### Instance Profile

Unlike `clusterName` and `clusterEndpoint`, the instance profile is **not** a Helm chart value in current Karpenter versions — the `aws.defaultInstanceProfile` setting was removed from the chart in earlier Karpenter releases. Instead, it is configured directly on each `EC2NodeClass` resource via the `instanceProfile` field, as shown in Lesson 3.

---

## 11. Understanding the Instance Profile

This configuration is very important.

Karpenter creates EC2 instances dynamically.

Those instances must be able to join the cluster.

Therefore, Karpenter needs the same node instance profile already used by your worker nodes.

Flow:

```txt
Karpenter
↓
Creates EC2 Instance
↓
EC2 receives Instance Profile
↓
Node joins EKS cluster
```

Without an instance profile, nodes would be created but could not register in Kubernetes.

Unlike earlier Karpenter versions, this is no longer configured at the Helm chart level — it is defined per `EC2NodeClass`, which we will do in Lesson 3.

---

## 12. Deploying Karpenter

Apply the Helm release:

```bash
terraform apply --auto-approve --var-file=environment/prod/terraform.tfvars
```

Terraform will:

```txt
Install the Helm chart
Create the deployment
Create the service account
Configure IRSA
```

---

## 13. Verifying the Installation

Update kubeconfig:

```bash
aws eks update-kubeconfig --name <cluster-name>
```

Check the Karpenter namespace:

```bash
kubectl get pods -n karpenter
```

Expected result:

```txt
Karpenter controller running
Pods in Running state
```

---

## 14. Monitoring Karpenter Logs

It is useful to keep the logs open while configuring NodePools later.

List the pods:

```bash
kubectl get pods -n karpenter
```

Describe the deployment:

```bash
kubectl describe deployment -n karpenter
```

Follow the logs:

```bash
kubectl logs -l app.kubernetes.io/name=karpenter -n karpenter -f
```

This makes troubleshooting much easier during provisioning tests.

---

## 15. What Has Been Installed?

At this point we have:

```txt
IAM Role
IAM Policies
IRSA Configuration
Karpenter Controller
Helm Release
```

What we do NOT have yet:

```txt
NodePools
EC2NodeClasses
Provisioning rules
Scaling policies
```

Karpenter is installed, but it does not yet know how to create nodes.

---

## Key Takeaways

In this lesson we completed the Karpenter installation.

Installation flow:

```txt
Create IAM permissions
Create IAM Role
Configure IRSA
Install Helm chart
Validate controller deployment
```

Current status:

```txt
Karpenter is running
Karpenter can access AWS APIs
Karpenter is connected to EKS
```

In the next lesson, we will configure the resources that actually define how Karpenter provisions infrastructure:

```txt
EC2NodeClasses
NodePools
```

These resources are the foundation of Karpenter's autoscaling behavior.

---

# Lesson 3: Understanding NodePools and EC2NodeClasses in Karpenter

Now that Karpenter is installed in the cluster, we can start configuring how it will provision and manage EC2 instances.

The two most important Custom Resources (CRDs) in Karpenter are:

```txt
EC2NodeClass
NodePool
```

Think about them like this:

```txt
EC2NodeClass = Defines HOW an EC2 instance should be created

NodePool = Defines WHEN and WHY Karpenter should create EC2 instances
```

A simple analogy:

```txt
EC2NodeClass = Launch Template

NodePool = Auto Scaling Rules
```

The combination of both resources tells Karpenter everything it needs to know to provision infrastructure automatically.

---

## 1. Creating an EC2NodeClass

The EC2NodeClass defines the infrastructure configuration that Karpenter will use when creating nodes.

This includes:

```txt
AMI
Subnets
Security Groups
Instance Profile
Operating System
```

Create a new file:

```txt
karpenter.yaml
```

Start with the EC2NodeClass definition.

```yaml
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: linuxtips
spec:
  instanceProfile: "linuxtips-eks-cluster"
  amiFamily: "AL2023"
  amiSelectorTerms:
  - id: ami-094fb6db0f574f0d6
  securityGroupSelectorTerms:
  - id: sg-0bb94cbd741aa3105
  subnetSelectorTerms:
  - id: subnet-0091a3edd1fc94df2
  - id: subnet-0df064a0ae575582b
  - id: subnet-027f0c1ec495abba6
```

---

## 2. Understanding Each EC2NodeClass Setting

Let's understand what each field controls.

### AMI Family

Defines the operating system used by worker nodes.

Example:

```yaml
amiFamily: AL2023
```

Available options include:

```txt
AL2
AL2023
Bottlerocket
Windows2019
Windows2022
Custom
```

For this course we will use:

```txt
Amazon Linux 2023
```

---

### Instance Profile

The instance profile allows EC2 instances to authenticate with AWS services.

Example:

```yaml
instanceProfile: PROJECT_INSTANCE_PROFILE
```

This should be the same instance profile currently used by your EKS node groups.

---

### Subnets

Defines where nodes can be created.

Example:

```yaml
subnetSelectorTerms:
  - id: subnet-a
  - id: subnet-b
  - id: subnet-c
```

Usually these are the same pod subnets already used by EKS.

---

### Security Groups

Defines which security groups will be attached to newly created nodes.

Example:

```yaml
securityGroupSelectorTerms:
  - id: sg-cluster
```

For the first tests, using the same security group as your existing nodes is usually the easiest option.

---

### AMI Selection

Defines the AMI Karpenter should use.

Example:

```yaml
amiSelectorTerms:
  - id: ami-xxxxxxxx
```

A simple approach is to reuse the AMI currently used by your EKS node groups.

---

## 3. Creating a NodePool

With the infrastructure definition ready, we can create a NodePool.

The NodePool defines the provisioning rules that Karpenter will follow.

Add the following resource to the same file.

```yaml
---
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: linuxtips
spec:
  disruption:
    consolidationPolicy: WhenEmptyOrUnderutilized
    consolidateAfter: 2m
  template:
    metadata:
      labels:
        workload: "etc"  
    spec:
      requirements:
        - key: karpenter.k8s.aws/instance-family
          operator: In
          values:   
          - t3
          - t3a

        - key: karpenter.sh/capacity-type
          operator: In
          values:
          - "spot"

        - key: karpenter.k8s.aws/instance-size
          operator: In
          values:
          - large
    
        - key: "topology.kubernetes.io/zone" 
          operator: In
          values:
          - "us-east-1a"
          - "us-east-1b"
          - "us-east-1c"          
          
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: linuxtips
```

---

## 4. Defining Instance Families

One of the biggest advantages of Karpenter is controlling exactly which EC2 families can be used.

Example:

```yaml
values:
  - t3
  - t3a
```

This means Karpenter may choose:

```txt
t3.large
t3a.large
```

If we later allow additional sizes:

```yaml
values:
  - medium
  - large
```

Karpenter could choose:

```txt
t3.medium
t3.large
t3a.medium
t3a.large
```

This flexibility allows Karpenter to optimize both cost and availability.

---

## 5. Choosing Spot or On-Demand

The capacity type controls how instances are purchased.

Spot only:

```yaml
values:
  - spot
```

On-Demand only:

```yaml
values:
  - on-demand
```

Or both:

```yaml
values:
  - spot
  - on-demand
```

For this lesson we will use:

```txt
Spot instances only
```

to reduce infrastructure costs.

---

## 6. Understanding Consolidation

One of Karpenter's most powerful features is node consolidation.

Karpenter continuously evaluates whether nodes are still necessary.

Example configuration:

```yaml
disruption:
  consolidationPolicy: WhenEmptyOrUnderutilized
  consolidateAfter: 1m
```

Meaning:

```txt
Check nodes every 1 minute

Remove nodes that are:
- Empty
- Underutilized
```

Available policies:

```txt
WhenEmpty
WhenEmptyOrUnderutilized
```

---

## 7. Applying the Configuration

Deploy the NodeClass and NodePool.

```bash
kubectl apply -f karpenter.yaml
```

Validate:

```bash
kubectl get ec2nodeclasses
kubectl get nodepools
```

Expected output:

```txt
1 EC2NodeClass

1 NodePool
```

At this point Karpenter is ready to provision nodes.

---

## 8. Deploying a Test Application

To test Karpenter we will use the same `chip` application from previous lessons.

```txt
chip.yaml
```

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
```

For the initial test:

```txt
1 replica
No HPA
```

Deploy the application.

```bash
kubectl apply -f chip.yaml
```

Because the cluster already has available capacity, Karpenter will not create additional nodes yet.

---

## 9. Triggering Node Provisioning

Now let's force a large scale event.

Scale the deployment:

```bash
kubectl scale deployment chip --replicas=100
```

Immediately Karpenter starts evaluating:

```txt
Pending pods
Required CPU
Required memory
Available instance types
Pricing options
```

Instead of simply adding more nodes, Karpenter calculates the most efficient infrastructure needed to satisfy the workload.

---

## 10. Watching Karpenter Create Nodes

Monitor the logs:

```bash
kubectl logs -n karpenter -l app.kubernetes.io/name=karpenter -f
```

Monitor nodes:

```bash
kubectl get nodes -w
```

You will see Karpenter:

```txt
Creating NodeClaims
Requesting EC2 instances
Joining nodes to the cluster
Scheduling pods
```

In a short time the cluster can scale from:

```txt
1 Pod

to

100 Pods
```

while automatically provisioning all required compute capacity.

---

## 11. Scaling Back Down

Now reduce the workload.

```bash
kubectl scale deployment chip --replicas=1
```

The excess nodes will become:

```txt
Empty
or
Underutilized
```

Based on the consolidation policy, Karpenter will start removing them automatically.

---

## 12. Watching Consolidation in Action

Since we configured:

```yaml
consolidateAfter: 1m
```

Karpenter evaluates the cluster every minute.

As nodes become unnecessary, they are:

```txt
Cordoned
Drained
Terminated
```

This process happens automatically and safely.

---

## 13. Why Karpenter Is More Powerful Than Cluster Autoscaler

Cluster Autoscaler focuses primarily on adding nodes when pods are pending.

Karpenter goes much further.

It considers:

```txt
CPU requirements
Memory requirements
Instance families
Instance sizes
Spot pricing
Availability
Node consolidation
```

Because of this, Karpenter can:

```txt
Scale faster
Reduce costs
Optimize resource utilization
Improve scheduling decisions
```

This is why it has become the preferred autoscaling solution for modern EKS environments.

---

## 14. What We Will Improve Next

The current configuration works, but it still contains several hardcoded values:

```txt
AMI IDs
Subnet IDs
Security Group IDs
Instance Profiles
```

In the next lesson we will use Terraform to generate these resources dynamically and make the Karpenter configuration reusable across environments.

This is where we start moving from a proof of concept into a production-ready implementation.

---

# Lesson 4: Automating Karpenter NodePools with Terraform

So far, we have been creating our `EC2NodeClass` and `NodePool` resources manually.

While this works well for learning purposes, it quickly becomes difficult to maintain in real environments where multiple workloads require different node configurations.

The goal of this lesson is to automate the creation of Karpenter resources using Terraform.

Instead of manually writing YAML files, we will generate them dynamically from Terraform variables and deploy them directly into Kubernetes.

This approach makes it easier to:

```txt
Create multiple NodePools
Create multiple EC2NodeClasses
Reuse configurations across environments
Standardize infrastructure
Reduce manual work
```

By the end of this lesson, Karpenter resources will be fully generated from Terraform inputs.

---

## 1. Installing the Kubectl Manifest Provider

To apply Kubernetes manifests directly from Terraform, we will use the `kubectl_manifest` provider.

This provider allows Terraform to send raw YAML manifests to the cluster.

Conceptually:

```txt
Terraform Variables
        ↓
Template Files
        ↓
Generated YAML
        ↓
kubectl_manifest
        ↓
Kubernetes Cluster
```

```txt
provider.tf
```

```hcl
terraform {
  required_providers {
    kubectl = {
      source  = "gavinbunney/kubectl"
      version = ">= 1.7.0"
    }
  }
}
```

> **Note:** `gavinbunney/kubectl` is a real, functioning provider, but it has seen little maintenance in recent years. Many current setups use the `alekc/kubectl` fork instead, which offers the same `kubectl_manifest`/`yaml_body` API and is actively maintained.

After adding the provider configuration, run:

```bash
terraform init -backend-config=environment/prod/backend.tfvars
```

Terraform will download the required provider and make it available for use.

---

## 2. Defining the Capacity Configuration

Before generating manifests dynamically, we need a structure that describes our desired capacity.

Create a variable called:

```hcl
variable "karpenter_capacity"
```

Instead of creating a single NodePool, we will create a list of NodePools.

This gives us the flexibility to create:

```txt
General workloads
Critical workloads
Spot workloads
Machine Learning workloads
GPU workloads
```

all from the same Terraform logic.

Our variable will contain information such as:

```txt
Name
Workload Label
AMI Family
Instance Families
Instance Sizes
Capacity Type
Availability Zones
```

---

## 3. Creating the Capacity Object

A single entry in the list might look like:

```hcl
karpenter_capacity = [
  {
    name               = "linux-apps"
    workload           = "linux"
    ami_family         = "AL2023"
    ami_ssm            = "/aws/service/eks/optimized-ami/1.35/amazon-linux-2023/x86_64/standard/recommended/image_id"

    instance_family    = ["t3a"]
    instance_sizes     = ["large"]

    capacity_type      = ["spot"]

    availability_zones = [
      "us-east-1a",
      "us-east-1b",
      "us-east-1c"
    ]
  }
]
```

This object now contains everything necessary to generate both:

```txt
EC2NodeClass
NodePool
```

---

## 4. Retrieving the Latest AMI Automatically

Hardcoding AMI IDs is not a good practice.

AWS already publishes the latest EKS-compatible AMIs through Systems Manager Parameter Store (SSM).

Instead of storing AMI IDs manually, we can retrieve them dynamically.

Example:

```hcl
data "aws_ssm_parameter" "karpenter_ami" {
  count = length(var.karpenter_capacity)

  name = var.karpenter_capacity[count.index].ami_ssm
}
```

Benefits:

```txt
Always use the latest AMI
No manual updates
Less maintenance
More automation
```

---

## 5. Creating Template Files

Instead of embedding large YAML documents inside Terraform resources, we will use template files.

Create a new directory:

```txt
files/
```

Inside it create:

```txt
files/
├── ec2_node_class.yaml
└── node_pool.yaml
```

These files will contain the YAML templates used to generate the Karpenter resources.

---

## 6. Building the EC2NodeClass Template

The first template is the EC2NodeClass.

Inside the template we replace static values with variables.

Example:

```yaml
apiVersion: karpenter.k8s.aws/v1
kind: EC2NodeClass
metadata:
  name: ${name}

spec:
  amiFamily: ${ami_family}

  instanceProfile: ${instance_profile}

  amiSelectorTerms:
    - id: ${ami_id}

  securityGroupSelectorTerms:
    - id: ${security_group}
```

Terraform will replace these placeholders automatically during execution.

---

## 7. Handling Dynamic Subnets

Subnets are stored as a list.

Because of that we need to generate them dynamically inside the template.

Example:

```yaml
subnetSelectorTerms:
%{ for subnet in subnets ~}
  - id: ${subnet}
%{ endfor ~}
```

Terraform loops through every subnet and generates valid YAML.

This allows the same template to work for any environment.

---

## 8. Creating the EC2NodeClass Resource

Now we can connect the template to Terraform.

Example:

```hcl
resource "kubectl_manifest" "ec2_node_class" {
  count = length(var.karpenter_capacity)

  yaml_body = templatefile(
    "${path.module}/files/ec2_node_class.yaml",
    {
      name             = var.karpenter_capacity[count.index].name
      instance_profile = aws_iam_instance_profile.nodes.name
      ami_family       = var.karpenter_capacity[count.index].ami_family
      ami_id           = data.aws_ssm_parameter.karpenter_ami[count.index].value
      subnets          = local.pod_subnets
      security_group   = aws_security_group.cluster.id
    }
  )
}
```

At this point Terraform can generate EC2NodeClasses dynamically.

---

## 9. Validating the EC2NodeClass

Apply the configuration:

```bash
terraform apply
```

Verify the resource:

```bash
kubectl get ec2nodeclasses
```

Expected result:

```txt
linux-apps
```

The resource is now being managed entirely through Terraform.

---

## 10. Building the NodePool Template

The NodePool template is slightly more complex because it contains multiple lists.

Examples:

```txt
Instance Families
Instance Sizes
Capacity Types
Availability Zones
```

Each of these values must be rendered dynamically.

---

## 11. Generating Instance Families

Inside the template:

```yaml
- key: karpenter.k8s.aws/instance-family
  operator: In
  values:
%{ for family in instance_family ~}
    - ${family}
%{ endfor ~}
```

Terraform will convert:

```hcl
["t3a", "m6a"]
```

into:

```yaml
values:
  - t3a
  - m6a
```

---

## 12. Generating Instance Sizes

The same technique applies to instance sizes.

Template:

```yaml
- key: karpenter.k8s.aws/instance-size
  operator: In
  values:
%{ for size in instance_sizes ~}
    - ${size}
%{ endfor ~}
```

Example output:

```yaml
values:
  - large
  - xlarge
```

---

## 13. Generating Capacity Types

Capacity type can also be generated dynamically.

Template:

```yaml
- key: karpenter.sh/capacity-type
  operator: In
  values:
%{ for type in capacity_type ~}
    - ${type}
%{ endfor ~}
```

Possible values:

```txt
spot
on-demand
```

or both.

---

## 14. Generating Availability Zones

Availability zones follow the same dynamic-list pattern as instance families, sizes, and capacity types.

Template:

```yaml
- key: "topology.kubernetes.io/zone"
  operator: In
  values:
%{ for zone in availability_zones ~}
    - ${zone}
%{ endfor ~}
```

This ensures nodes are only created in the availability zones we explicitly allow, matching the manual example from Lesson 3.

---

## 15. Creating Dynamic Workload Labels

We can also generate labels automatically.

Example:

```yaml
labels:
  workload: ${workload}
```

This becomes extremely useful later when we start segregating workloads across different NodePools.

Examples:

```txt
workload=critical
workload=apps
workload=batch
workload=ml
```

---

## 16. Creating the NodePool Resource

Once the template is ready, create the Terraform resource.

Example:

```hcl
resource "kubectl_manifest" "node_pool" {
  count = length(var.karpenter_capacity)

  yaml_body = templatefile(
    "${path.module}/files/node_pool.yaml",
    {
      name               = var.karpenter_capacity[count.index].name
      workload           = var.karpenter_capacity[count.index].workload
      instance_family    = var.karpenter_capacity[count.index].instance_family
      instance_sizes     = var.karpenter_capacity[count.index].instance_sizes
      capacity_type      = var.karpenter_capacity[count.index].capacity_type
      availability_zones = var.karpenter_capacity[count.index].availability_zones
    }
  )
}
```

---

## 17. Applying the Configuration

Run:

```bash
terraform apply
```

Terraform will now generate:

```txt
EC2NodeClass
NodePool
```

from the capacity configuration.

Verify:

```bash
kubectl get ec2nodeclasses
kubectl get nodepools
```

---

## 18. Benefits of This Approach

Instead of creating YAML files manually, everything is now driven by Terraform variables.

Adding a new NodePool becomes as simple as adding another object to the list.

Example:

```hcl
karpenter_capacity = [
  {...},
  {...},
  {...}
]
```

Terraform automatically creates:

```txt
Multiple EC2NodeClasses
Multiple NodePools
```

without duplicating code.

---

## 19. Preparing for Workload Segregation

At this point we have automated the entire Karpenter provisioning process.

The next step is to take advantage of this flexibility.

We will start creating multiple NodePools for different workload types and learn how to:

```txt
Distribute workloads across AZs
Separate critical and non-critical applications
Use Spot instances safely
Improve availability
Reduce infrastructure costs
```

This is where Karpenter becomes significantly more powerful than traditional node groups and Cluster Autoscaler.

---

# Lesson 5: Improving Availability with Topology Spread Constraints

Now that Karpenter can dynamically create `EC2NodeClasses` and `NodePools`, let's look at the application side: how the `chip` Deployment itself can use Karpenter more intelligently to improve its own availability.

Karpenter always takes the path of least resistance. Left on its own, it provisions capacity whichever way is fastest and cheapest — which often means most replicas end up concentrated in a single Availability Zone, a single instance type, or a single capacity type.

Kubernetes gives us a mechanism to change that: `topologySpreadConstraints`. Karpenter reads these constraints when deciding what capacity to provision, not just where to place pods on nodes that already exist.

In this lesson we will apply three progressively richer spread constraints to the `chip` Deployment and observe how Karpenter reacts to each one.

---

## 1. The Problem: Uneven Distribution

Scale the current `chip` Deployment up significantly:

```bash
kubectl scale deployment chip --replicas=100 -n chip
```

Without any further guidance, Karpenter may satisfy this demand unevenly — for example, concentrating most of the new nodes in `us-east-1a` and very few in `us-east-1b` or `us-east-1c`.

Check the zone every node landed in:

```bash
kubectl get nodes -L topology.kubernetes.io/zone
```

`topology.kubernetes.io/zone` is a standard Kubernetes well-known label that every cloud provider populates automatically, and it's exactly what we'll use to fix this.

---

## 2. Spreading Pods Across Availability Zones

Add a `topologySpreadConstraints` block to the `chip` Deployment's `spec.template.spec`:

```yaml
spec:
  template:
    spec:
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: "topology.kubernetes.io/zone"
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: chip
      containers:
      # ...
```

Each field controls the balancing behavior:

```txt
maxSkew: the maximum allowed difference between the zone with the most matching pods and the zone with the fewest
topologyKey: the node label used to group nodes into "zones" for this comparison
whenUnsatisfiable: ScheduleAnyway lets pods still be scheduled if perfect balance can't be reached, instead of leaving them Pending
labelSelector: which pods count toward the skew calculation
```

`ScheduleAnyway` matters here because Karpenter is actively creating capacity to satisfy this constraint — a strict `DoNotSchedule` isn't necessary to get a well-balanced result.

Apply and force the existing capacity to be reconsidered:

```bash
kubectl apply -f chip.yaml
kubectl scale deployment chip --replicas=1 -n chip
kubectl scale deployment chip --replicas=100 -n chip
```

Check the distribution again:

```bash
kubectl get nodes -L topology.kubernetes.io/zone
```

This time the new nodes land in a much more balanced way across the three Availability Zones.

---

## 3. Diversifying Instance Types and Sizes

Back in Lesson 4, our `karpenter_capacity` entry only allowed a single instance family and size:

```hcl
instance_family = ["t3a"]
instance_sizes  = ["large"]
```

Widen both lists so Karpenter has more instance shapes to choose from:

```hcl
instance_family = ["c6i", "c6a", "c7i", "c7a"]
instance_sizes  = ["large", "xlarge", "2xlarge"]
```

Apply the change:

```bash
terraform apply
```

Now add a second `topologySpreadConstraint`, this time keyed on the instance type itself:

```yaml
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
```

`node.kubernetes.io/instance-type` is another standard well-known label — every node is labeled with its underlying EC2 instance type.

Existing nodes won't retroactively change shape, so force Karpenter to reprovision:

```bash
kubectl get nodeclaims
kubectl delete nodeclaim <nodeclaim-name>
```

---

## 4. Mixing Spot and On-Demand Capacity

The same `karpenter_capacity` entry currently only allows Spot:

```hcl
capacity_type = ["spot"]
```

Allow both capacity types:

```hcl
capacity_type = ["spot", "on-demand"]
```

Apply the change:

```bash
terraform apply
```

Add a third `topologySpreadConstraint`, keyed on Karpenter's own capacity-type label:

```yaml
  - maxSkew: 1
    topologyKey: "karpenter.sh/capacity-type"
    whenUnsatisfiable: ScheduleAnyway
    labelSelector:
      matchLabels:
        app: chip
```

`karpenter.sh/capacity-type` is set by Karpenter itself on every node it provisions, with a value of either `spot` or `on-demand`.

Delete the current NodeClaims once more to let Karpenter reprovision under the new constraint:

```bash
kubectl get nodeclaims
kubectl delete nodeclaim <nodeclaim-name>
```

---

## 5. Verifying the Distribution

Check all three dimensions at once:

```bash
kubectl get nodes -L topology.kubernetes.io/zone,node.kubernetes.io/instance-type,karpenter.sh/capacity-type
```

The `chip` workload should now be spread across multiple AZs, multiple instance types, and a mix of Spot and On-Demand capacity.

The end goal isn't only balance for its own sake — it's Spot safety. AWS interruptions tend to affect a specific instance type in a specific AZ at a time. The more diversified the workload is across zone, instance type, and capacity type, the less likely a single interruption event takes out a large share of the running replicas at once.

---

## Key Takeaways

```txt
topologySpreadConstraints let the application influence what capacity Karpenter provisions, not just where existing pods are placed
maxSkew and whenUnsatisfiable control how strict the balancing is
Multiple constraints can be combined: Availability Zone, instance type, and capacity type
Diversifying across all three dimensions is what makes running Spot in production safer
Existing nodes are not retroactively rebalanced — deleting NodeClaims forces Karpenter to reprovision under the new constraints
```

---

# Lesson 6: Segregating Workloads with Multiple NodePools

With the `karpenter_capacity` list from Lesson 4, we already have the building blocks to run more than one NodePool. In this lesson we use that flexibility to isolate workloads that have different — sometimes conflicting — requirements from the general-purpose pool created so far.

---

## 1. Why Segregate Workloads

Some real-world scenarios can't share a single NodePool:

```txt
Machine learning workloads that need GPU instance types
Latency-critical applications that can't tolerate Spot interruption
Asynchronous or batch workloads that tolerate Spot just fine
Applications that require a specific AMI family, like Windows or Bottlerocket
```

With traditional node groups, each of these required its own manually managed Auto Scaling Group. With Karpenter, it's just another entry in `karpenter_capacity`.

---

## 2. Adding a Dedicated NodePool via Terraform

Append a new object to the `karpenter_capacity` list, dedicated to the `chip` workload:

```hcl
karpenter_capacity = [
  {
    name               = "linux-apps"
    workload           = "linux"
    ami_family         = "AL2023"
    ami_ssm            = "/aws/service/eks/optimized-ami/1.35/amazon-linux-2023/x86_64/standard/recommended/image_id"
    instance_family    = ["c6i", "c6a", "c7i", "c7a"]
    instance_sizes     = ["large", "xlarge", "2xlarge"]
    capacity_type      = ["spot", "on-demand"]
    availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]
  },
  {
    name               = "chip-capacity"
    workload           = "chip"
    ami_family         = "Bottlerocket"
    ami_ssm            = "/aws/service/bottlerocket/aws-k8s-1.35/x86_64/latest/image_id"
    instance_family    = ["c6i", "c6a"]
    instance_sizes     = ["large", "xlarge"]
    capacity_type      = ["spot"]
    availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]
  }
]
```

Apply and confirm a second NodePool exists:

```bash
terraform apply
kubectl get nodepools
kubectl get ec2nodeclasses
```

---

## 3. Understanding the karpenter.sh/nodepool Label

Every node Karpenter provisions is automatically labeled with the name of the NodePool that created it:

```bash
kubectl get nodes --show-labels | grep karpenter.sh/nodepool
```

This is the same well-known label used as a `topologySpreadConstraint` key in Lesson 5 — here we'll use it directly as a `nodeSelector` to pin a workload to one specific NodePool.

---

## 4. Targeting a NodePool with nodeSelector

Add a `nodeSelector` to the `chip` Deployment's `spec.template.spec`:

```yaml
spec:
  template:
    spec:
      nodeSelector:
        karpenter.sh/nodepool: chip-capacity
      containers:
      # ...
```

Apply the change:

```bash
kubectl apply -f chip.yaml
kubectl get nodeclaims
```

From this point on, Karpenter provisions new capacity for `chip` exclusively from the `chip-capacity` NodePool — the general `linux-apps` pool stops receiving its pods.

---

## 5. Creating Criticality-Based NodePools

The same pattern works for isolating workloads by criticality tier. Add entries for each tier, differing only by `capacity_type`:

```hcl
{
  name          = "critical"
  workload      = "critical"
  ami_family    = "AL2023"
  ami_ssm       = "/aws/service/eks/optimized-ami/1.35/amazon-linux-2023/x86_64/standard/recommended/image_id"
  instance_family    = ["c6i", "c6a"]
  instance_sizes     = ["large", "xlarge"]
  capacity_type      = ["on-demand"]
  availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]
},
{
  name          = "soft"
  workload      = "soft"
  ami_family    = "AL2023"
  ami_ssm       = "/aws/service/eks/optimized-ami/1.35/amazon-linux-2023/x86_64/standard/recommended/image_id"
  instance_family    = ["c6i", "c6a"]
  instance_sizes     = ["large", "xlarge"]
  capacity_type      = ["spot"]
  availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]
},
{
  name          = "general"
  workload      = "general"
  ami_family    = "AL2023"
  ami_ssm       = "/aws/service/eks/optimized-ami/1.35/amazon-linux-2023/x86_64/standard/recommended/image_id"
  instance_family    = ["c6i", "c6a"]
  instance_sizes     = ["large", "xlarge"]
  capacity_type      = ["spot", "on-demand"]
  availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]
}
```

```txt
critical: always On-Demand — for workloads that can't tolerate interruption
soft: always Spot — for workloads that tolerate interruption and want the cost savings
general: a mix of both — the default for everything else
```

---

## 6. Creating OS/AMI-Specific NodePools

The same list also isolates workloads that require a specific operating system or AMI family:

```hcl
{
  name       = "windows-2019"
  workload   = "windows-2019"
  ami_family = "Windows2019"
  ami_ssm    = "/aws/service/ami-windows-latest/Windows_Server-2019-English-Core-EKS_Optimized-1.35/image_id"
  instance_family    = ["c6i"]
  instance_sizes     = ["xlarge"]
  capacity_type      = ["on-demand"]
  availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]
},
{
  name       = "windows-2022"
  workload   = "windows-2022"
  ami_family = "Windows2022"
  ami_ssm    = "/aws/service/ami-windows-latest/Windows_Server-2022-English-Core-EKS_Optimized-1.35/image_id"
  instance_family    = ["c6i"]
  instance_sizes     = ["xlarge"]
  capacity_type      = ["on-demand"]
  availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]
}
```

`Windows2019`, `Windows2022`, `AL2`, and `AL2023` are all part of the same `amiFamily` enum we already validated back in Lesson 3 — this is the same list, just applied more deliberately per workload instead of once for the whole cluster.

---

## 7. Switching Workloads Between NodePools

Because targeting a NodePool is just a `nodeSelector`, moving a workload between them only takes a value change:

```yaml
nodeSelector:
  karpenter.sh/nodepool: critical
```

```bash
kubectl apply -f chip.yaml
kubectl get nodeclaims -w
```

Watch the NodeClaims from the old NodePool get decommissioned over time as new ones from the target NodePool take over the workload.

---

## Key Takeaways

```txt
Every Karpenter-provisioned node carries a karpenter.sh/nodepool label identifying its origin
A nodeSelector on that label is enough to pin a workload to a specific NodePool
The same karpenter_capacity Terraform pattern from Lesson 4 scales to dedicated, criticality-based, and OS-specific NodePools
Moving a workload between NodePools is just a nodeSelector change plus a rollout
This is where Karpenter clearly surpasses traditional node groups: no manual Auto Scaling Group per workload type
```

---

# Lesson 7: Native Interruption Handling in Karpenter

Karpenter includes functionality equivalent to a standalone Node Termination Handler: it listens for AWS events about instance lifecycle changes and reacts by draining the affected node and provisioning replacement capacity automatically.

Back in Lesson 2 we granted the Karpenter controller `sqs:*` IAM permissions but noted the interruption-handling infrastructure itself — the actual SQS queue and EventBridge rules — was outside that lesson's scope. This lesson builds that missing piece.

---

## 1. How Karpenter Handles Interruptions

The flow is:

```txt
AWS publishes an instance lifecycle event (Spot interruption, rebalance recommendation, state-change, health event)
EventBridge matches the event and forwards it to an SQS queue
Karpenter polls that queue
Karpenter cordons and drains the affected node, then provisions replacement capacity
```

None of this is active yet. Karpenter only starts polling once the Helm chart's `settings.interruptionQueue` value is set — until we do that, the `sqs:*` permission granted in Lesson 2 has had nothing to act on.

---

## 2. Creating the SQS Queue

```hcl
resource "aws_sqs_queue" "karpenter" {
  name                       = format("%s-karpenter-interruption", var.project_name)
  message_retention_seconds  = 86400
  receive_wait_time_seconds  = 10
  visibility_timeout_seconds = 60
}
```

The retention period has to cover the slowest event type this queue carries, not just the fastest one: Spot interruption warnings give about two minutes of notice, but the scheduled-change health events and ASG lifecycle events added below aren't on that clock. A full day of retention is a safer default than a short window tuned to only one event type. `receive_wait_time_seconds` turns on long polling, so Karpenter isn't hammering SQS between messages, and `visibility_timeout_seconds` gives it a full minute to process a message before it becomes eligible for redelivery.

---

## 3. Allowing EventBridge to Publish to the Queue

```hcl
data "aws_iam_policy_document" "karpenter_sqs" {
  statement {
    effect    = "Allow"
    actions   = ["sqs:SendMessage"]
    resources = [aws_sqs_queue.karpenter.arn]

    principals {
      type        = "Service"
      identifiers = ["events.amazonaws.com"]
    }
  }
}

resource "aws_sqs_queue_policy" "karpenter" {
  queue_url = aws_sqs_queue.karpenter.id
  policy    = data.aws_iam_policy_document.karpenter_sqs.json
}
```

Without this policy, EventBridge would not be allowed to deliver messages into the queue.

---

## 4. Creating EventBridge Rules for Instance Lifecycle Events

Define the event patterns Karpenter needs to react to:

```hcl
locals {
  karpenter_interruption_events = {
    instance_terminate = {
      source      = ["aws.autoscaling"]
      detail-type = ["EC2 Instance-terminate Lifecycle Action"]
    }
    spot_interruption = {
      source      = ["aws.ec2"]
      detail-type = ["EC2 Spot Instance Interruption Warning"]
    }
    rebalance_recommendation = {
      source      = ["aws.ec2"]
      detail-type = ["EC2 Instance Rebalance Recommendation"]
    }
    state_change = {
      source      = ["aws.ec2"]
      detail-type = ["EC2 Instance State-change Notification"]
    }
    health_event = {
      source      = ["aws.health"]
      detail-type = ["AWS Health Event"]
      detail = {
        service           = ["EC2"]
        eventTypeCategory = ["scheduledChange"]
      }
    }
  }
}

resource "aws_cloudwatch_event_rule" "karpenter" {
  for_each = local.karpenter_interruption_events

  name          = format("%s-karpenter-%s", var.project_name, each.key)
  event_pattern = jsonencode(each.value)
}

resource "aws_cloudwatch_event_target" "karpenter" {
  for_each = local.karpenter_interruption_events

  rule      = aws_cloudwatch_event_rule.karpenter[each.key].name
  target_id = "karpenter-sqs"
  arn       = aws_sqs_queue.karpenter.arn
}
```

Every one of these rules targets the same SQS queue — Karpenter doesn't care which rule matched, only that a relevant instance lifecycle event arrived.

Two of these are easy to misread:

- `health_event` is scoped with a `detail` block to `service: ["EC2"]` and `eventTypeCategory: ["scheduledChange"]`. Without that filter, the rule would match every AWS Health event across every service in the account, not just EC2 scheduled maintenance.
- `instance_terminate` listens on `aws.autoscaling`, not `aws.ec2` — it only fires for instances that belong to an Auto Scaling Group with a lifecycle hook, which doesn't include NodePool-provisioned capacity directly. It's part of Karpenter's own reference interruption setup so a cluster that mixes Karpenter with traditional node groups still gets full coverage from one queue.

---

## 5. Enabling the Interruption Queue

Add `settings.interruptionQueue` to the `helm_release` resource created in Lesson 2:

```hcl
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
```

Apply the change:

```bash
terraform apply
```

This value is passed to the Karpenter controller as the `INTERRUPTION_QUEUE` environment variable — it's the only thing that actually turns interruption handling on.

---

## 6. Testing Interruption Handling

Pick a node and terminate its underlying instance manually:

```bash
kubectl get nodes
aws ec2 terminate-instances --instance-ids <instance-id>
```

Check that a message arrived in the queue:

```bash
aws sqs receive-message --queue-url <queue-url>
```

Watch Karpenter react to it:

```bash
kubectl logs -n karpenter -l app.kubernetes.io/name=karpenter -f
```

The logs should show the interruption message being captured, the affected node being drained, and a replacement node being provisioned to cover the lost capacity.

---

## 7. Karpenter vs. the Standalone Node Termination Handler

Karpenter's native interruption handling behaves exactly like a standalone Node Termination Handler — cordon, drain, replace. Because of that, AWS does not recommend running both at the same time: they would both attempt to react to the same events and could interfere with each other's draining.

If Karpenter manages the cluster's capacity, its native interruption handling should be the only one enabled.

---

## Key Takeaways

```txt
Karpenter's interruption handling stays dormant until settings.interruptionQueue is set — the sqs:* permission from Lesson 2 alone is not enough
The pipeline is: EventBridge rules match instance lifecycle events, forward them to an SQS queue, Karpenter polls that queue
The queue's retention has to cover the slowest event type it carries — Spot warnings give two minutes of notice, but scheduled-change and ASG lifecycle events don't, which is why the queue keeps messages for a full day
Never run the standalone Node Termination Handler alongside Karpenter's native handling
```
