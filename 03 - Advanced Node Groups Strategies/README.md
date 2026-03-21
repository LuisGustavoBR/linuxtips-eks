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
