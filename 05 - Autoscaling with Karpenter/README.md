# Module 5 - Lesson 1
## Introduction to Karpenter

In this lesson we introduce **Karpenter**, one of the most powerful autoscaling solutions available for Kubernetes on AWS.

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

---

# Step 1 - What is Karpenter?

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

# Step 2 - Karpenter vs Cluster Autoscaler

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

# Step 3 - Why Karpenter Is More Efficient

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

# Step 4 - Main Karpenter Components

Karpenter relies on a small set of Custom Resource Definitions (CRDs).

The two most important are:

```txt
EC2NodeClass
NodePool
```

These CRDs define how infrastructure should be provisioned.

---

# Step 5 - Understanding EC2NodeClass

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

# Step 6 - Understanding NodePools

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

# Step 7 - EC2NodeClass vs NodePool

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

# Step 8 - Flexible Capacity Management

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

# Step 9 - Advanced Scheduling Strategies

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

# Step 10 - Faster Node Provisioning

Karpenter is generally faster than Cluster Autoscaler.

Reasons:

```txt
Direct integration with AWS APIs
No dependency on node group scaling
Workload-aware provisioning
```

Instead of scaling an existing node group, Karpenter provisions the exact infrastructure required.

---

# Step 11 - Cost Optimization

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

# Step 12 - Spot Workloads with Karpenter

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

# Step 13 - Real Production Use Cases

Common Karpenter use cases include:

```txt
High-scale production clusters
Machine Learning platforms
Batch processing systems
Cost-optimized Kubernetes environments
```

It is widely adopted because it adapts infrastructure dynamically to workload demand.

---

# Step 14 - Productizing Karpenter

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

# Step 15 - Additional Learning Resources

The course materials include an additional article covering:

```txt
Production Spot strategies
Karpenter best practices
Cost optimization techniques
```

Although written some time ago, the concepts remain highly relevant.

It is strongly recommended reading material.

---

# Step 16 - What We Will Build

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
