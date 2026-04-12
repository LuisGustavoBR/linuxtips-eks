# Module 3 - Lesson 1  
## Advanced Node Group Strategies in Amazon EKS

In the previous module we created our **first EKS cluster** and deployed a simple application.

The goal of that lesson was to **quickly provision a working Kubernetes cluster** using Terraform.

Now we will go deeper into **node group strategies**, exploring how to design node groups for different workloads and infrastructure requirements.

---

# Understanding Node Groups

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

# Step 1 - Reviewing the Previous Node Group

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

# Step 2 - Defining Capacity Type

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

# Step 3 - Adding Node Labels

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

# Step 4 - Deploying the Node Group

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

# Step 5 - Creating a Spot Node Group

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

# Step 6 - Deploying the Spot Node Group

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

# Step 7 - Verifying the Node Groups

Check the node groups in the AWS console.

```txt
EKS → Cluster → Compute
```

You should now see two node groups:

```txt
default (On-Demand)
spot-nodes (Spot)
```

Each node will also include the labels we defined earlier.

---

# Module 3 - Lesson 2  
## Using Managed Node Groups with Bottlerocket

In this lesson we will learn how to create **Managed Node Groups using Bottlerocket** and understand what makes it different from Amazon Linux nodes.

---

# Step 8 - Understanding What Bottlerocket Is

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

# Step 9 - Why Use Bottlerocket in EKS

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

# Step 10 - Creating a Managed Node Group with Bottlerocket

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

# Step 11 - Creating the Bottlerocket Node Group File

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

  ami_type = "BOTTLEROCKET_X86_64"

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

# Step 12 - Applying the Configuration

Now we apply the configuration to create the new node group.

Command:

```bash
terraform apply --auto-approve --var-file=environment/prod/terraform.tfvars
```

After a few minutes, the new nodes will be available inside the cluster.

---

# Step 13 - Checking the Nodes in AWS Console

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

# Step 14 - Connecting to the Nodes

If you connect to an Amazon Linux node using Session Manager, you will see a normal Linux environment.

But if you connect to a Bottlerocket node, you will notice something very different:

```txt
Almost no packages installed
Very minimal environment
Designed only to run containers
```

This is exactly the goal of Bottlerocket.

---

# Step 15 - Creating a Bottlerocket Node Group Using Spot Instances

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

  ami_type = "BOTTLEROCKET_X86_64"

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

# Step 16 - Checking Node Labels

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

# Module 3 - Lesson 3  
## Using Graviton (ARM64) Instances in EKS

In this lesson we will learn how to use **Graviton instances (ARM64)** in our EKS cluster and how to create node groups using this architecture.

---

# Step 17 - Understanding What Graviton Is

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

# Step 18 - Benefits of Using Graviton

The main advantage of Graviton is **cost efficiency**.

In many cases, you can get ~20% to 30% cost reduction compared to equivalent x86 instances.

Important note:

```txt
Performance gains are not always significant  
The biggest benefit is cost savings
```

---

# Step 19 - Important Requirement: Application Compatibility

Before using ARM64, you must ensure your applications support it.

That means:

```txt
Applications may need to be compiled for ARM64  
Docker images must support ARM architecture  
Multi-arch images are recommended
```

If your application is not compatible, it will not run.

---

# Step 20 - Creating a Graviton Node Group

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

# Step 21 - Configuring the AMI Type

We must use an ARM-compatible AMI.

Example:

```txt
ami_type = "AL2023_ARM_64"
```

This ensures the node runs with the correct architecture.

---

# Step 22 - Creating the Terraform File

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

# Step 23 - Creating a Spot Version

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

---

# Step 24 - Applying the Configuration

Run:

```bash
terraform apply --auto-approve --var-file=environment/prod/terraform.tfvars
```

After a few minutes, the new ARM64 nodes will be available in the cluster.

---

# Step 25 - Validating the Nodes

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

# Step 26 - Why Labels Are Important

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
architecture = arm64  
os           = amazon-linux  
capacity     = spot / on-demand
```

---

# Module 3 - Lesson 4  
## Workload Segregation Using Node Selector

In this lesson we will learn how to control **where pods run inside the cluster** using node labels and `nodeSelector`.

---

# Step 27 - Understanding Node Selector

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

# Step 28 - Using the Example Deployment

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

# Step 29 - Applying the Deployment

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

# Step 30 - Validating Where Pods Are Running

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

---

# Step 31 - Changing Strategy to Spot

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

# Step 32 - Real Use Cases

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

# Step 33 - Combining with Node Groups

Since we created multiple node groups:

```txt
x86 + ARM  
On-Demand + Spot  
Amazon Linux + Bottlerocket
```

We can now fully control scheduling using labels.

---

# Module 3 - Lesson 5  
## Using Node Affinity for Smarter Scheduling

In this lesson we go beyond `nodeSelector` and introduce **Node Affinity**, which allows more flexible and intelligent scheduling.

---

# Step 34 - Understanding Node Affinity

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

# Step 35 - Creating a Node Affinity Deployment

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

```
Try to balance pods between Spot and On-Demand
```

---

# Step 36 - Applying the Configuration

Apply the deployment:

```bash
kubectl apply -f chip-node-affinity.yaml
```

Then monitor:

```bash
kubectl get pods -n chip -o wide
```

---

# Step 37 - Observing Pod Distribution

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

---

# Step 38 - Important Behavior

Node Affinity with `preferredDuringSchedulingIgnoredDuringExecution` is a **soft rule**.

That means:

```txt
If Spot is unavailable → pods go to On-Demand  
If On-Demand is unavailable → pods go to Spot  
```

It does NOT block scheduling.

---

# Step 39 - Real Use Case

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

# Step 40 - Key Takeaways

- Node Affinity allows **intelligent workload distribution**
- Works great with multiple node groups
- Ideal for:
  - Cost optimization
  - High availability strategies
  - Flexible scheduling

Compared to nodeSelector, it gives much more control without being restrictive.

---

# Module 3 - Lesson 6  
## Segregating Workloads by Criticality

In this lesson we explore a practical strategy: **separating workloads based on criticality** using node labels and node groups.

---

# Step 41 - Defining Critical vs Non-Critical Nodes

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

# Step 42 - Creating a Critical Node Group

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

# Step 43 - Creating a Soft (Non-Critical) Node Group

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

# Step 44 - Applying the Configuration

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

# Step 45 - Validating Node Labels

You can verify labels with:

```bash
kubectl get nodes --show-labels
```

Or filter specific labels:

```bash
kubectl get nodes -o jsonpath="{.items[*].metadata.labels.severity}"
```

---

# Step 46 - Using Criticality in Deployments

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

# Step 47 - Real Use Case

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

# Step 48 - Key Takeaway

By combining:

```txt
Multiple node groups  
Labels  
nodeSelector / affinity
```

You can design a cluster that intelligently separates workloads based on business needs.

---

# Module 3 - Lesson 7  
## Customizing Node Groups with Launch Templates

In this lesson we learn how to extend Managed Node Groups using **Launch Templates** to gain more control over configuration.

---

# Step 48 - Why Use Launch Templates

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

# Step 49 - What We Can Customize

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

# Step 50 - Preparing User Data

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

# Step 51 - Creating AMI Variable

We define a variable for the custom AMI:

```hcl
variable "custom_ami" {
  type = string
  description = "Customized AMI ID for the nodes"
  default = "ami-01d396130bcd204a1"
}
```

---

# Step 52 - Creating the Launch Template

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

---

# Step 53 - Passing Cluster Data to User Data

We inject required values:

```txt
cluster_name  
endpoint  
certificate_authority  
```

These come from the EKS cluster resource.

---

# Step 54 - Validating the Launch Template

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

---

# Step 55 - Creating a Custom Node Group

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

# Step 56 - Applying and Validating

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

---

# Step 57 - Real Use Cases

Launch Templates are useful when you need:

- Golden Images (company standard AMIs)
- Pre-installed agents (security, monitoring)
- Custom bootstrap logic
- Advanced storage configuration

---

# Step 58 - Best Practice

- Prefer default Managed Node Groups when possible
- Use Launch Templates only when necessary
- Avoid unnecessary complexity

They are powerful, but increase operational overhead.

---
