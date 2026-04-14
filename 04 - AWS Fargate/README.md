# Module 4 - Lesson 1  
## Introduction to AWS Fargate in EKS

In this lesson we introduce **AWS Fargate in EKS**, a serverless compute option that removes the need to manage nodes.

Unlike traditional Kubernetes setups where you manage EC2 instances (nodes), Fargate allows you to run workloads without provisioning or maintaining infrastructure.

Key idea:

```txt
1 Pod = 1 dedicated Fargate microVM (node)
```

---

# Step 1 - What is Fargate?

Fargate is a compute engine powered by **Firecracker microVMs**.

It sits between:

```txt
Containers → lightweight but less isolated  
Virtual Machines → fully isolated but heavier  
Fargate → lightweight + strong isolation
```

Characteristics:

```txt
No node management  
Automatic provisioning  
Kernel-level isolation  
Fully managed by AWS
```

---

# Step 2 - How Fargate Works in EKS

In EKS, Fargate behaves differently than ECS.

Important concept:

```txt
You do NOT create pods directly on Fargate  
Fargate creates a node per pod
```

That means:

```txt
1 Pod → 1 Fargate node  
10 Pods → 10 Fargate nodes
```

AWS manages:

```txt
Node lifecycle  
Scaling infrastructure  
Networking integration
```

You manage:

```txt
Pods only
```

---

# Step 3 - Comparing EKS Models

We now have three main models:

### 1. Self-managed Kubernetes
```txt
You manage everything  
(Control Plane + Nodes)
```

### 2. EKS (Default)
```txt
AWS manages Control Plane  
You manage Nodes
```

### 3. EKS with Fargate
```txt
AWS manages Control Plane + Nodes  
You manage only Pods
```

---

# Step 4 - Advantages of Fargate

Fargate is great when you want simplicity.

Benefits:

```txt
No infrastructure management  
Strong isolation (microVM)  
Automatic scaling  
Good for ephemeral workloads
```

Ideal use cases:

```txt
Small environments  
Burst workloads  
Isolated workloads
```

---

# Step 5 - Limitations of Fargate

Fargate trades flexibility for simplicity.

Limitations:

```txt
Limited Kubernetes features  
No direct node access  
Only supports specific workload types (Deployments, StatefulSets)  
Less control over networking and storage  
Higher cost in some scenarios
```

---

# Step 6 - When to Use Fargate

Use Fargate when:

```txt
You want zero node management  
You prioritize simplicity over control  
Workloads are stateless or simple
```

Avoid when:

```txt
You need deep Kubernetes control  
You require DaemonSets or custom node configs  
You want maximum cost optimization
```

---

# Step 7 - What We Will Do Next

In this lesson, we will:

```txt
Run Fargate alongside EC2 node groups  
Create a full Fargate-based cluster  
Compare both approaches in practice
```

---
