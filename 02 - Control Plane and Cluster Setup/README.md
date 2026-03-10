# Module 2: Creating the First EKS Cluster (Vanilla Setup)

Welcome to Module 2 of the LinuxTips EKS Learning Path.

The main goal here is to build what we will call a **vanilla structure**.

This means a **minimal but functional EKS cluster** that will serve as the **foundation for future scenarios**. Instead of evolving directly on top of a complex structure, we will start with a clean base and later expand it with new capabilities.

The idea works like this:

- We create a **vanilla EKS cluster**
- Later we add **specific capabilities**
- Then we may return to the **vanilla base**
- And implement **different architectural variations**

So this cluster becomes the **seed of everything else we will build in the course**.

This lesson is also a **continuation of the previous one**, so it is extremely important that the **VPC infrastructure created earlier is already deployed**.

Especially the **SSM Parameter Store entries**, because we will use them here to retrieve networking information like:

- VPC ID
- Subnets
- Other infrastructure values

Launching an **EKS cluster from scratch is never trivial**, but we prepared a **clear step-by-step process** so the setup becomes much easier.

Just like in the previous lesson, we will follow a simple rule:

- Copy and paste repetitive structures when needed
- Avoid wasting time rewriting boilerplate code
- Focus on understanding what actually matters

There are **many moving parts in an EKS cluster**, so saving time on repetitive work helps us focus on the architecture.

Now let's start coding.

---

# Step 1 — Creating the Repository

The first thing we need to do is **create a repository for this lesson**.

We will follow the **same naming structure used in the networking project** so everything stays organized.

Example repository name:

```txt
linuxtips-eks-vanilla
```

This repository will contain the **minimum Terraform configuration necessary to create an EKS cluster**.

When creating the repository:

- Make it **public**
- Initialize it with a **README**
- Do **include a Terraform `.gitignore`**
- Do **include a MIT license**

After creating it:

1. Clone the repository
2. Open it in **VS Code**
3. Start creating the Terraform structure

---

# Step 2 — Initial Terraform Structure

We will follow **exactly the same folder structure used in the networking module**.

Create the following structure:

```bash
mkdir -p environment/prod/
```

```txt
environment/
environment/prod/
```

Inside `environment/prod`, create the following Terraform files:

```bash
touch environment/prod/{backend,terraform}.tfvars
```

```txt
backend.tfvars
terraform.tfvars
```

And in the root folder:

```bash
touch {variables,provider,outputs,backend,data}.tf
```

```txt
variables.tf
provider.tf
outputs.tf
backend.tf
data.tf
```


At this point, the structure should look like this:

```txt
├── backend.tf
├── data.tf
├── outputs.tf
├── provider.tf
├── variables.tf
└── environment
    └── prod
        ├── backend.tfvars
        └── terraform.tfvars
```

For now, this minimal structure is enough.

---

# Step 3 — Creating Base Variables

Inside `variables.tf`, create the base variables used across the project.

We will define:

- project name
- AWS region

Example structure:

```hcl
variable "project_name" {
  type = string
}

variable "region" {
  type = string
}
```

These values will be provided later through `terraform.tfvars`.

---

# Step 4 — Configuring the Terraform Backend

Now we configure the **remote state backend** using **S3**, just like in the previous lesson.

In `backend.tf`:

```hcl
terraform {
  backend "s3" {
    
  }
}
```

We keep the backend **empty in the code** because we will pass the configuration **during Terraform initialization**.

Example `backend.tfvars` values used during init:

```hcl
bucket  = "linuxtips-s3-eks-state-files-bordon"
key     = "eks/cluster/prod/state"
region  = "us-east-1"
```

You should adapt these values to your own environment.

---

# Step 5 — Terraform Variables File

Now create `terraform.tfvars` with the project configuration.

Example:

```hcl
project_name = "linuxtips-cluster"
region       = "us-east-1"
```

These values will be loaded automatically by Terraform.

---

# Step 6 — Configuring the AWS Provider

Now we configure the **AWS provider**.

In `provider.tf`:

```hcl
provider "aws" {
  region = var.region
}
```

This ensures Terraform will interact with AWS in the correct region.

---

# Step 7 — Loading Values From Parameter Store

Now comes an important part.

Instead of **hardcoding infrastructure values**, we will retrieve them from **AWS Systems Manager Parameter Store (SSM)**.

These parameters were created in the **previous networking lesson**.

We will load:

- VPC ID
- Public subnets
- Private subnets
- Pod subnets

This approach allows us to **decouple the networking infrastructure from the cluster infrastructure**.

---

# Step 8 — Declaring SSM Variables

Inside `variables.tf`, add the variables that will store the **SSM parameter names**.

First, the VPC parameter:

```hcl
variable "ssm_vpc" {
  type = string
}
```

Now the **public subnets list**:

```hcl
variable "ssm_public_subnets" {
  type = list(string)
}
```

Now the **private subnets**:

```hcl
variable "ssm_private_subnets" {
  type = list(string)
}
```

And finally the **pods subnets**:

```hcl
variable "ssm_pod_subnets" {
  type = list(string)
}
```

For now, we will **not use database subnets**, so we can ignore them.

---

# Step 9 — Declaring the TFVars Values

Now populate these variables in `terraform.tfvars`.

Example structure:

```hcl
ssm_vpc = "/linuxtips-vpc/vpc/id"

ssm_public_subnets = [
  "/linuxtips-vpc/subnets/public/us-east-1a/linuxtips-public-1a",
  "/linuxtips-vpc/subnets/public/us-east-1b/linuxtips-public-1b",
  "/linuxtips-vpc/subnets/public/us-east-1c/linuxtips-public-1c",
]

ssm_private_subnets = [
  "/linuxtips-vpc/subnets/private/us-east-1a/linuxtips-private-1a",
  "/linuxtips-vpc/subnets/private/us-east-1b/linuxtips-private-1b",
  "/linuxtips-vpc/subnets/private/us-east-1c/linuxtips-private-1c",
]

ssm_pod_subnets = [
  "/linuxtips-vpc/subnets/private/us-east-1a/linuxtips-pods-1a",
  "/linuxtips-vpc/subnets/private/us-east-1b/linuxtips-pods-1b",
  "/linuxtips-vpc/subnets/private/us-east-1c/linuxtips-pods-1c",
]
```

These paths must match the **parameters created in the networking module**.

---

# Step 10 — Initial Terraform Run

Now we test if everything is configured correctly.

Run:

```bash
terraform fmt --recursive
```

And:

```bash
terraform init --backend-config=environment/prod/backend.tfvars
```

Then:

```bash
terraform apply --auto-approve --var-file=environment/prod/terraform.tfvars
```

At this stage nothing will be created yet, but Terraform will validate:

- provider configuration
- backend configuration
- variables

If no errors appear, we are ready to continue.

---

# Step 11 — Loading Parameters Using Data Sources

Now we will **load the Parameter Store values into memory** during Terraform execution.

Inside `data.tf`, we declare the data sources.

First, load the VPC ID:

```hcl
data "aws_ssm_parameter" "vpc" {
  name = var.ssm_vpc
}
```

---

# Step 12 — Loading Public Subnets

For the public subnets we need a **loop**, because we have multiple parameters.

```hcl
data "aws_ssm_parameter" "public_subnets" {
  count = length(var.ssm_public_subnets)
  name = var.ssm_public_subnets[count.index]
}
```

This loads **each subnet parameter dynamically**.

---

# Step 13 — Loading Private Subnets

The same logic applies to private subnets.

```hcl
data "aws_ssm_parameter" "private_subnets" {
  count = length(var.ssm_private_subnets)
  name = var.ssm_private_subnets[count.index]
}
```

---

# Step 14 — Loading Pod Subnets

And finally the pod subnets:

```hcl
data "aws_ssm_parameter" "pod_subnets" {
  count = length(var.ssm_pod_subnets)
  name = var.ssm_pod_subnets[count.index]
}
```

---

# Step 15 — Testing Data Sources

Run Terraform again to confirm everything loads correctly:

```bash
terraform apply --auto-approve --var-file=environment/prod/terraform.tfvars
```

If everything is correct, Terraform will successfully retrieve:

- VPC ID
- Public subnet IDs
- Private subnet IDs
- Pod subnet IDs

Now all networking values are **available in memory for Terraform**.

---

# Module 2 — Lesson 2  
## Creating IAM Roles and KMS for the EKS Cluster

Now let's start creating the **IAM resources required by our EKS cluster**.

The first thing we will do is create a file called:

```txt
iam_cluster.tf
```

Inside this file we will define the **IAM Role used by the EKS Control Plane**.

The EKS control plane needs specific permissions in order to manage:

- Kubernetes API operations  
- Worker node communication  
- Cluster lifecycle actions  

So the first thing we do is define an **IAM policy document**.

---

# Step 16 — Creating the Assume Role Policy

First we create a **data source** that defines the **assume role policy**.

This document tells AWS **which service can assume this role**.

```hcl
data "aws_iam_policy_document" "cluster" {

  version = "2012-10-17"

  statement {
    actions = [
      "sts:AssumeRole"
    ]

    principals {
      type = "Service"
      identifiers = [
        "eks.amazonaws.com"
      ]
    }
  }
}
```

This structure is very common when creating IAM roles for AWS services.

The important part here is:

```hcl
sts:AssumeRole
```

And the principal:

```hcl
eks.amazonaws.com
```

This means the **EKS service itself will assume this role**.

---

# Step 17 — Creating the Cluster IAM Role

Now we create the actual IAM role.

```hcl
resource "aws_iam_role" "eks_cluster_role" {
  name               = format("%s-cluster-role", var.project_name)
  assume_role_policy = data.aws_iam_policy_document.cluster.json
}
```

At this point the role exists, but **it still has no permissions attached**.

So we now attach the required AWS managed policies.

---

# Step 18 — Attaching the EKS Cluster Policy

The first required policy is:

```hcl
AmazonEKSClusterPolicy
```

We attach it using:

```hcl
resource "aws_iam_role_policy_attachment" "eks_cluster_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSClusterPolicy"
  role       = aws_iam_role.eks_cluster_role.name
}
```

This policy allows the EKS control plane to **manage cluster resources**.

---

# Step 19 — Attaching the EKS Service Policy

The second required policy is:

```
AmazonEKSServicePolicy
```

```hcl
resource "aws_iam_role_policy_attachment" "eks_service_policy" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSServicePolicy"
  role       = aws_iam_role.eks_cluster_role.name
}
```

Now our **EKS control plane role is ready**.

---

# Step 20 — Creating the Worker Nodes IAM Role

Now we create another file:

```hcl
iam_nodes.tf
```

This file will contain the **IAM role used by the worker nodes**.

The structure is very similar to the cluster role.

First we create the **assume role policy document**.

```hcl
data "aws_iam_policy_document" "nodes" {

  version = "2012-10-17"

  statement {
    actions = [
      "sts:AssumeRole"
    ]

    principals {
      type = "Service"
      identifiers = [
        "ec2.amazonaws.com"
      ]
    }
  }
}
```

Notice that here the principal changes to:

```hcl
ec2.amazonaws.com
```

This is because **worker nodes are EC2 instances**.

---

# Step 21 — Creating the Nodes Role

Now we create the nodes IAM role.

```hcl
resource "aws_iam_role" "eks_nodes_role" {
  name               = format("%s-nodes-role", var.project_name)
  assume_role_policy = data.aws_iam_policy_document.nodes.json
}
```

Now we need to attach several policies.

Worker nodes require **more permissions than the control plane role**.

---

# Step 22 — Attaching Required Policies to Nodes

Worker nodes require several AWS managed policies.

### CNI Policy

Required for Kubernetes networking.

```hcl
resource "aws_iam_role_policy_attachment" "cni" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKS_CNI_Policy"
  role       = aws_iam_role.eks_nodes_role.name
}
```

---

### Worker Node Policy

Allows nodes to communicate with the control plane.

```hcl
resource "aws_iam_role_policy_attachment" "nodes" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy"
  role       = aws_iam_role.eks_nodes_role.name
}
```

---

### ECR Read Access

Allows nodes to pull container images from ECR.

```hcl
resource "aws_iam_role_policy_attachment" "ecr" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonEC2ContainerRegistryReadOnly"
  role       = aws_iam_role.eks_nodes_role.name
}
```

---

### SSM Access

Allows access to instances via Systems Manager.

```hcl
resource "aws_iam_role_policy_attachment" "ssm" {
  policy_arn = "arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore"
  role       = aws_iam_role.eks_nodes_role.name
}
```

---

### CloudWatch Agent

Permissions required for logging and monitoring.

```hcl
resource "aws_iam_role_policy_attachment" "cloudwatch" {
  policy_arn = "arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy"
  role       = aws_iam_role.eks_nodes_role.name
}
```

Now our **nodes role has all required permissions**.

---

# Step 23 — Creating the Instance Profile

EC2 instances cannot assume roles directly.

They must use an **Instance Profile**.

```hcl
resource "aws_iam_instance_profile" "nodes_instance_profile" {
  name = var.project_name
  role = aws_iam_role.eks_nodes_role.name
}
```

This profile will be used later by **worker nodes**.

---

# Step 24 — Creating the KMS Key

We create another file:

```hcl
kms.tf
```

Now we create the **KMS key used to encrypt EKS resources at rest**.

```hcl
resource "aws_kms_key" "main" {
  description = var.project_name
}
```

This key will be used to **encrypt sensitive cluster data**.

For example:

- Kubernetes secrets  
- Cluster state data  

---

# Step 25 — Creating a KMS Alias

It is good practice to create an alias for easier identification.

```hcl
resource "aws_kms_alias" "main" {
  name          = format("alias/%s", var.project_name)
  target_key_id = aws_kms_key.main.id
}
```

Now the key can easily be identified inside AWS.

---

# Module 2 — Lesson 3  
## Creating the EKS Control Plane

Now we are going to create our **first real EKS resource**, which is the **EKS Cluster itself**.

This resource represents the **Kubernetes control plane**.

So far we have prepared everything required to reach this point:

- VPC and networking
- IAM roles
- KMS key for encryption
- Instance profiles
- Subnet references

Now we will finally create the **EKS control plane**.

---

# Step 26 — Creating the EKS Cluster Resource

Create a new file:

```hcl
eks.tf
```

Inside it we define the main resource:

```hcl
resource "aws_eks_cluster" "main" {

}
```

This resource represents the **Kubernetes control plane managed by AWS**.

---

# Step 27 — Defining the Cluster Name

The cluster needs a name.

```hcl
name = var.project_name
```

Using the project name helps keep everything standardized.

---

# Step 28 — Creating the Kubernetes Version Variable

Before continuing, let's create a new variable called:

```hcl
k8s_version  
```

Inside `variables.tf`:

```hcl
variable "k8s_version" {
  type = string
}
```

And inside `terraform.tfvars`:

```hcl
k8s_version = "1.34"
```

At the time this course was recorded, **Kubernetes 1.31** was the latest supported version.

---

# Step 29 — Setting the Kubernetes Version

Back in the cluster resource:

```hcl
version = var.k8s_version
```

This defines which Kubernetes version the control plane will run.

---

# Step 30 — Defining the Cluster IAM Role

Now we attach the **IAM role created earlier for the control plane**.

```hcl
role_arn = aws_iam_role.eks_cluster_role.arn
```

This role allows EKS to manage cluster resources.

---

# Step 31 — Configuring VPC Networking

Now we define the **VPC configuration block**.

```hcl
vpc_config {

}
```

Inside this block we define the subnets used by the control plane.

The control plane will run inside the **private subnets**.

```hcl
subnet_ids = data.aws_ssm_parameter.private_subnets[*].value
```

This loads the subnet IDs from **Parameter Store**.

---

# Step 32 — Configuring Encryption

Now we configure **encryption for Kubernetes secrets**.

```hcl
  encryption_config {
    provider {
      key_arn = aws_kms_key.main.arn
    }
    resources = ["secrets"]
  }
```

This tells EKS to use our **KMS key to encrypt Kubernetes secrets at rest**.

---

# Step 33 — Access Configuration

Now we configure **cluster access settings**.

```hcl
  access_config {
    authentication_mode                         = "API_AND_CONFIG_MAP"
    bootstrap_cluster_creator_admin_permissions = true
  }
```

This enables two authentication methods:

- API based access (modern method)
- aws-auth ConfigMap (legacy method)

Using both allows us to **demonstrate both approaches during the course**.

However, in production environments it is recommended to use **API access entries only**.

---

# Step 34 — Bootstrap Admin Permissions

Another very important parameter is:

```hcl
bootstrap_cluster_creator_admin_permissions = true
```

This setting ensures that **the IAM identity that created the cluster automatically receives admin permissions inside Kubernetes**.

This is extremely important.

Sometimes Kubernetes authentication configurations can break, and you may lose access to the cluster.

With this option enabled, the **cluster creator always keeps admin privileges**, which helps recover access.

---

# Step 35 — Enabling Control Plane Logs

Now we enable **control plane logging**.

```hcl
  enabled_cluster_log_types = [
    "api", "audit", "authenticator", "controllerManager", "scheduler"
  ]
```

These logs will be sent to **CloudWatch Logs**.

They include:

- API calls
- Audit logs
- Authentication logs
- Controller Manager logs
- Scheduler logs

This is extremely useful for **debugging and auditing cluster activity**.

---

# Step 36 — Cluster Tag

It is also good practice to include the Kubernetes cluster tag.

```hcl
  tags = {
    "kubernetes.io/cluster/${var.project_name}" = "shared"
  }
```

This tag helps AWS services identify resources associated with the cluster.

---

# Step 37 — API Endpoint Security

There are also security configurations related to **cluster API access**.

For example:

```hcl
endpoint_private_access = false
endpoint_public_access  = true
```

By default, the Kubernetes API is **publicly accessible**.

You can also restrict access using **CIDR blocks**.

Example:

```hcl
public_access_cidrs = ["0.0.0.0/0"]
```

This allows access from anywhere.

However, in a production environment it is highly recommended to restrict this to **corporate IP ranges**.

For example:

```hcl
public_access_cidrs = ["203.0.113.10/32"]
```

In this course we will leave it open for simplicity.

---

# Step 38 — Creating the OIDC Provider

Before finishing the cluster setup, we also need to configure **OIDC integration**.

This allows Kubernetes **Service Accounts to assume IAM roles** using Web Identity.

First we retrieve the **TLS certificate from the OIDC issuer**.

```hcl
data "tls_certificate" "eks" {
  url = aws_eks_cluster.main.identity[0].oidc[0].issuer
}
```

This certificate will be used to configure the OIDC provider.

---

# Step 39 — Creating the OpenID Connect Provider

Now we create the IAM OIDC provider.

```hcl
resource "aws_iam_openid_connect_provider" "eks" {
  client_id_list = ["sts.amazonaws.com"]
  thumbprint_list = [
    data.tls_certificate.eks.certificates[0].sha1_fingerprint,
    "9e99a48a9960b14926bb7f3b02e22da2b0ab7280"
  ]
  url = flatten(concat(aws_eks_cluster.main[*].identity[*].oidc.0.issuer, [""]))[0]
}
```

This resource allows **Kubernetes service accounts to authenticate with AWS IAM**.

It is required for features like:

- IAM Roles for Service Accounts (IRSA)
- External controllers
- AWS integrations inside Kubernetes

---

# Step 40 — Creating the Cluster

Now we can finally run Terraform.

First initialize the providers:

```bash
terraform init --backend-config=environment/prod/backend.tfvars
```

Then apply the configuration:

```bash
terraform apply --auto-approve --var-file=environment/prod/terraform.tfvars
```

Terraform will now create the **EKS control plane**.

This process usually takes **several minutes**.

---

# Step 41 — Verifying the Cluster

After Terraform finishes, you can open the **EKS console**.

You should see your new cluster running.

Important information available in the console includes:

- Kubernetes version
- API endpoint
- IAM role used by the control plane
- VPC configuration

---

# Step 42 — Connecting to the Cluster

To connect to the cluster we need:

- AWS CLI
- kubectl installed

We can generate the kubeconfig automatically using the AWS CLI.

```bash
aws eks update-kubeconfig --region us-east-1 --name linuxtips-cluster
```

This command updates your **local kubeconfig file**.

It creates a new Kubernetes context for the cluster.

---

# Step 43 — Testing the Connection

Now we can test the connection.

```bash
kubectl get nodes
```

You will see that the cluster currently has **no worker nodes**.

This is expected.

At this stage we only created the **control plane**.

Running:

```bash
kubectl get pods -A
```

Will show only the **system components**.

---

# Module 2 — Lesson 4  
## Managing the Cluster Security Group

One important detail about EKS clusters is that **every cluster automatically creates its own Security Group**.

If you open the EKS console and navigate to the **Networking section**, you will see a **security group automatically created for the cluster**.

This security group is used by several components, such as:

- Worker nodes
- Fargate profiles
- Control plane communication

By default, the rules are very restrictive.  
Initially, the security group is configured to **allow traffic only within itself**.

However, we can **add additional rules** to this security group using Terraform.

The idea of this lesson is to demonstrate **how to manipulate the cluster security group**.

---

# Step 44 — Creating a Security Group Rule

We do **not need to create a new security group**.

Instead, we will create **additional rules attached to the cluster security group**.

Create a new file:

```hcl
sg.tf
```

Create a new resource:

```hcl
resource "aws_security_group_rule" "nodeports" {
  cidr_blocks       = ["0.0.0.0/0"]
  from_port         = 30000
  to_port           = 32768
  protocol          = "tcp"
  description       = "Allow NodePort access to worker nodes"
  type              = "ingress"
  security_group_id = aws_eks_cluster.main.vpc_config[0].cluster_security_group_id
}
```

This resource allows us to define **custom rules for the existing cluster security group**.

---

# Step 45 — Defining the Rule Type

First we define the rule type.

```hcl
type = "ingress"
```

This means we are allowing **incoming traffic**.

---

# Step 46 — Defining the NodePort Range

Kubernetes NodePort services use a specific port range.

The default range is:

```hcl
30000 - 32768
```

So we define the ports:

```hcl
from_port = 30000
to_port   = 32768
```

---

# Step 47 — Defining the Protocol

Now we define the protocol.

```hcl
protocol = "tcp"
```

If necessary, you could also allow all protocols using:

```hcl
protocol = "-1"
```

But in most cases using **TCP is enough**.

---

# Step 48 — Defining the CIDR Block

Now we define who is allowed to access these ports.

For demonstration purposes we will allow access from anywhere.

```hcl
cidr_blocks = ["0.0.0.0/0"]
```

In production environments this should normally be **restricted to specific networks**.

---

# Step 49 — Attaching the Rule to the Cluster Security Group

Now we need the **security group ID of the cluster**.

The EKS cluster resource exposes this information.

```hcl
security_group_id = aws_eks_cluster.main.vpc_config[0].cluster_security_group_id
```

The `vpc_config` attribute returns a list, and inside it we have the field:

```hcl
cluster_security_group_id
```

Using this value we attach our rule to the **cluster security group created by AWS**.

---

# Step 50 — Applying the Configuration

Now run Terraform again.

```bash
terraform apply --auto-approve --var-file=environment/prod/terraform.tfvars
```

Terraform will create the **new rule inside the cluster security group**.

You can confirm this in the AWS console.

Navigate to:

``  
EKS → Cluster → Networking → Security Groups
``

Inside the security group you should now see the **NodePort rule allowing ports 30000–32768**.

---

# Step 51 — Example: Allowing DNS Traffic

We can create additional rules in the same way.

For example, allowing DNS traffic.

```hcl
resource "aws_security_group_rule" "coredns_tcp" {
  cidr_blocks       = ["0.0.0.0/0"]
  from_port         = 53
  to_port           = 53
  protocol          = "tcp"
  description       = "Allow CoreDNS TCP access to worker nodes"
  type              = "ingress"
  security_group_id = aws_eks_cluster.main.vpc_config[0].cluster_security_group_id
}

resource "aws_security_group_rule" "coredns_udp" {
  cidr_blocks       = ["0.0.0.0/0"]
  from_port         = 53
  to_port           = 53
  protocol          = "udp"
  description       = "Allow CoreDNS UDP access to worker nodes"
  type              = "ingress"
  security_group_id = aws_eks_cluster.main.vpc_config[0].cluster_security_group_id
}
```

This rule allows **DNS queries**, which can be useful for services like **CoreDNS**.

---

# Module 2 — Lesson 5  
## Enabling ARC Zonal Shift for the Control Plane

Another extremely important feature related to the **EKS control plane** is the ability to enable **ARC Zonal Shift**.

This feature is part of **AWS Application Recovery Controller (ARC)** and provides an additional mechanism for **handling Availability Zone failures**.

In Kubernetes environments running across multiple AZs, it's possible that **one AZ experiences issues**, such as:

- network degradation  
- infrastructure instability  
- partial outages  

When this happens, workloads running in that AZ may become unhealthy.

The **Zonal Shift feature** helps mitigate this type of problem.

---

# Step 52 — Understanding Zonal Shift

The **Zonal Shift mechanism** allows AWS to **redirect traffic away from an unhealthy Availability Zone**.

For example:

If workloads are running in:

```txt
us-east-1a  
us-east-1b  
us-east-1c  
```

And `us-east-1b` starts experiencing problems, AWS can **shift traffic away from that AZ** and direct it to the **healthy zones**.

This helps maintain **service availability even during partial infrastructure failures**.

---

# Step 53 — Enabling Zonal Shift in the Cluster

Enabling this feature is very simple.

Inside the `aws_eks_cluster` resource we add the **zonal shift configuration block**.

```hcl
  zonal_shift_config {
    enabled = true
  }
```

This activates **ARC Zonal Shift support for the cluster**.

---

# Step 54 — Why This Feature Matters

Even though enabling Zonal Shift requires only a single configuration flag, it can significantly improve **cluster resilience**.

With this feature enabled:

- AWS can automatically react to **Availability Zone failures**
- Traffic can be shifted to **healthy zones**
- Workloads remain accessible during partial outages

This is particularly valuable for **multi-AZ Kubernetes clusters**, which is the standard architecture for production workloads.

---

# Step 55 — Applying the Configuration

Now apply the changes.

```bash
terraform apply --auto-approve --var-file=environment/prod/terraform.tfvars
```

Terraform will update the EKS cluster configuration.

This operation is usually very fast since it is just enabling a feature flag.

---

# Step 56 — Verifying in the AWS Console

After applying the configuration, you can confirm it inside the AWS console.

Navigate to:

```txt
EKS → Cluster → Resilience
```

There you should see:

```txt
ARC Zonal Shift: Enabled
```

This means the cluster is now **capable of performing zonal traffic shifts during AZ failures**.

---

# Module 2 — Lesson 6  
## Configuring AWS Auth (aws-auth ConfigMap) Using Terraform

In this lesson we will configure the **first authentication mechanism used in Amazon EKS**.

This mechanism is called **AWS Auth**, which is implemented through the **aws-auth ConfigMap** inside Kubernetes.

This configuration is responsible for allowing **EC2 worker nodes to authenticate and join the cluster**.

Without this configuration, worker nodes cannot successfully register with the control plane.

---

# Step 57 — Authentication Methods in EKS

Amazon EKS supports two main authentication mechanisms for infrastructure resources:

```txt
1. aws-auth ConfigMap (AWS Auth)
2. Access Entries
```

In this lesson we will start with **AWS Auth**, which is still widely used and important to understand.

Later we will also explore **Access Entries**, which is the more modern approach for managing access to the cluster.

---

# Step 58 — Using the Kubernetes Terraform Provider

To configure the aws-auth ConfigMap we will use the **Kubernetes provider for Terraform**.

This provider allows Terraform to interact directly with the **Kubernetes API**, enabling us to apply Kubernetes resources such as:

```txt
ConfigMaps
Secrets
Deployments
Services
```

This is extremely useful because it allows us to **manage Kubernetes manifests directly through Terraform**.

---

# Step 59 — Authenticating Terraform with the Cluster

Before Terraform can apply Kubernetes manifests, it must authenticate with the Kubernetes API server.

To achieve this, we must configure the **Kubernetes provider** using information from the EKS cluster.

First we add a **data source** that retrieves information about the cluster.

```hcl
data "aws_eks_cluster_auth" "default" {
  name = aws_eks_cluster.main.id
}
```

This data source provides important information such as:

- API endpoint  
- certificate authority data  

---

# Step 60 — Retrieving the Authentication Token

Next, we retrieve an authentication token that Terraform will use to communicate with the cluster.

```hcl
data "aws_caller_identity" "current" {}
```

This token allows Terraform to **authenticate API requests against Kubernetes**.

---

# Step 61 — Configuring the Kubernetes Provider

Now we configure the Kubernetes provider.

```hcl
provider "kubernetes" {
  host                   = aws_eks_cluster.main.endpoint
  cluster_ca_certificate = base64decode(aws_eks_cluster.main.certificate_authority.0.data)
  token                  = data.aws_eks_cluster_auth.default.token
}
```

This configuration provides Terraform with:

- the API server endpoint  
- the cluster certificate authority  
- an authentication token  

With these parameters Terraform can successfully **connect to the Kubernetes API**.

---

# Step 62 — Creating the aws-auth ConfigMap

Now we create a new Terraform file responsible for defining the **aws-auth ConfigMap**.

Example file:

```txt
aws_auth.tf
```

Inside this file we create a Kubernetes ConfigMap resource.

```hcl
resource "kubernetes_config_map_v1" "aws_auth" {
  metadata {
    name      = "aws-auth"
    namespace = "kube-system"
  }
}
```

This ConfigMap **must always be named aws-auth** and must exist in the **kube-system namespace**.

---

# Step 63 — Mapping the Node IAM Role

Inside the ConfigMap we define role mappings that allow EC2 nodes to authenticate with the cluster.

```hcl
  data = {
    mapRoles = <<YAML
- rolearn: ${aws_iam_role.eks_nodes_role.arn}
  username: system:node:{{EC2PrivateDNSName}}
  groups:
  - system:bootstrappers
  - system:nodes
  - system:node-proxier
YAML
  }
```

This mapping allows instances associated with the **node IAM role** to act as Kubernetes worker nodes.

The groups assigned are important:

```hcl
system:bootstrappers
system:nodes
system:node-proxier
```

These permissions allow nodes to:

- bootstrap themselves  
- register with the cluster  
- run kube-proxy networking functions  

---

# Step 64 — Ensuring Proper Deployment Order

Since the cluster must exist before applying Kubernetes resources, we define an explicit dependency.

```hcl
  depends_on = [
    aws_eks_cluster.main
  ]
```

This ensures Terraform **waits until the EKS cluster is fully created** before applying the ConfigMap.

---

# Step 65 — Applying the Configuration

Now we apply the configuration.

```bash
terraform apply --auto-approve --var-file=environment/prod/terraform.tfvars
```

Terraform will now:

1. Connect to the Kubernetes API  
2. Create the aws-auth ConfigMap  
3. Register the node IAM role for cluster access  

---

# Step 66 — Verifying the ConfigMap

After the deployment we can verify the configuration using kubectl.

```bash
kubectl get cm aws-auth -n kube-system -o yaml
```

To inspect its content:

```bash
kubectl describe configmap aws-auth -n kube-system
```

Inside the ConfigMap you should see the **role mapping for the worker nodes**.

---

# Module 2 — Lesson 7  
## Creating the First Managed Node Group

Now that our **EKS control plane is running**, we can add the **first worker nodes to the cluster**.

To do that we will use **EKS Managed Node Groups**, which is the **simplest way to provision nodes in Amazon EKS**.

With managed node groups, AWS handles several operational tasks automatically, such as:

- AMI selection
- node updates
- lifecycle management

This makes it the easiest way to start adding compute capacity to the cluster.

Later we will explore more advanced approaches.

---

# Step 67 — Defining Capacity Variables

Before creating the node group, we will define some variables to control **cluster capacity**.

First, create a variable called **auto_scale_options**.

```hcl
variable "auto_scale_options" {
  type = object({
    min     = number
    max     = number
    desired = number
  })
}
```

This variable will control the **minimum**, **maximum**, and **desired** number of nodes.

Next, create another variable for the instance types used by the nodes.

```hcl
variable "nodes_instance_sizes" {
  type = list(string)
}
```

This allows us to define **multiple instance types** for the node group.

---

# Step 68 — Configuring the Variables

Now configure these variables inside **terraform.tfvars**.

```hcl
auto_scale_options = {
  min     = 2
  max     = 5
  desired = 2
}

nodes_instance_sizes = [
  "t3.large",
  "t3a.large",
]
```

This configuration will create a node group with:

- minimum of **2 nodes**
- maximum of **5 nodes**
- **2 nodes initially running**

Using multiple instance types allows AWS to select instances more flexibly.

---

# Step 69 — Creating the EKS Node Group Resource

Create a new file:

```txt
nodes.tf
```

Now we create the node group using the **aws_eks_node_group** resource.

```hcl
resource "aws_eks_node_group" "main" {
  cluster_name    = aws_eks_cluster.main.id
  node_group_name = aws_eks_cluster.main.id

  node_role_arn = aws_iam_role.eks_nodes_role.arn

  instance_types = var.nodes_instance_sizes

  subnet_ids = data.aws_ssm_parameter.private_subnets[*].value

  tags = {
    "kubernetes.io/cluster/${var.project_name}" = "owned"
  }
}
```

This resource connects the worker nodes to the **EKS cluster control plane**.

The **node role ARN** must be the same role previously authorized in the **aws-auth ConfigMap**.

---

# Step 70 — Configuring Node Scaling

Next we configure the **scaling configuration**.

```hcl
  scaling_config {
    desired_size = lookup(var.auto_scale_options, "desired")
    max_size     = lookup(var.auto_scale_options, "max")
    min_size     = lookup(var.auto_scale_options, "min")
  }
```

This defines how the node group scales.

Terraform will initially create **two nodes**, but the cluster can scale up to **ten nodes** if necessary.

---

# Step 71 — Assigning Subnets

The nodes must be launched inside the **private subnets used by the cluster**.

```hcl
  subnet_ids = data.aws_ssm_parameter.private_subnets[*].value
```

These are the same subnets used during the cluster creation.

---

# Step 72 — Adding Node Labels

Node labels can be applied to every node created in the group.

```hcl
  labels = {
    "ingress/ready" = "true"
  }
```

Labels are extremely useful for:

- scheduling workloads
- defining node placement strategies
- organizing cluster resources

We will explore these strategies later.

---

# Step 73 — Preventing Scaling Drift

If cluster autoscaling changes the desired size dynamically, Terraform could try to revert it.

To avoid that, we configure a lifecycle rule.

```hcl
  lifecycle {
    ignore_changes = [
      scaling_config[0].desired_size
    ]
  }
```

This prevents Terraform from forcing the node group back to the original desired size after autoscaling events.

---

# Step 74 — Ensuring Correct Dependency Order

The node group must only be created **after the aws-auth ConfigMap exists**, otherwise nodes will fail to join the cluster.

```hcl
  depends_on = [
    kubernetes_config_map_v1.aws_auth
  ]
```

This guarantees the correct provisioning order.

---

# Step 75 — Configuring Timeouts

For large clusters, node provisioning or updates can take time.

We can define explicit timeouts.

```hcl
  timeouts {
    create = "1h"
    update = "2h"
    delete = "2h"
  }
```

This is especially useful for **large node groups or production environments**.

---

# Step 76 — Applying the Configuration

Now apply the configuration.

```bash
terraform apply
```

Terraform will create the node group and launch the EC2 instances that will act as worker nodes.

---

# Step 77 — Verifying the Nodes

After deployment, verify the nodes using kubectl.

```bash
kubectl get nodes
```

You should see the newly created nodes registered in the cluster.

Example output:

```bash
ip-10-0-1-12.ec2.internal
ip-10-0-2-18.ec2.internal
```

This confirms that the **nodes successfully joined the cluster**.

---

# Step 78 — Viewing the Node Group in AWS

You can also check the node group in the AWS console.

Navigate to:

```txt
EKS → Cluster → Compute → Node Groups
```

There you will see the newly created node group and its configuration.

---

# Step 79 — Inspecting the EC2 Instances

The worker nodes are **regular EC2 instances** managed by the node group.

You can view them in:

```txt
EC2 → Instances
```

Since they are running inside **private subnets**, they are not accessible through SSH from the internet.

Instead, we can connect using **AWS Systems Manager (SSM)**.

This allows secure access to the instances directly from the AWS console without exposing SSH ports.

---

# Module 2 — Lesson 8  
## Migrating from aws-auth ConfigMap to Access Entries

The **aws-auth ConfigMap** has historically been the most common way to manage authentication in Amazon EKS.

However, it has some limitations.

The configuration is stored as a **YAML string inside a ConfigMap**, which makes it harder to manage with infrastructure-as-code tools like Terraform.

Because of this, AWS introduced a new and more modern mechanism called **Access Entries**.

Access Entries allow us to manage **authentication and authorization for IAM identities directly through the EKS API**, instead of manually editing the aws-auth ConfigMap.

---

# Step 80 — Understanding Access Entries

Access Entries provide a native way to authorize:

```txt
IAM Roles
IAM Users
Node Roles
Fargate Profiles
```

This method integrates directly with the **EKS API**, making it much easier to manage using Terraform.

Compared to the aws-auth ConfigMap, Access Entries are:

- simpler to configure
- easier to maintain
- fully integrated with AWS APIs

---

# Step 81 — Creating an Access Entry for Nodes

Create a new file:

```
access_entries.tf
```

To migrate from aws-auth to Access Entries, we create an **aws_eks_access_entry** resource.

```hcl
resource "aws_eks_access_entry" "nodes" {
  cluster_name  = aws_eks_cluster.main.id
  principal_arn = aws_iam_role.eks_nodes_role.arn
  type          = "EC2_LINUX"
}
```

This resource authorizes the **node IAM role** to access the Kubernetes cluster.

The **type** parameter defines what kind of identity is being authorized.

Some possible values include:

```hcl
STANDARD
EC2_LINUX
EC2_WINDOWS
FARGATE
```

Since our worker nodes are Linux EC2 instances, we use **EC2_LINUX**.

---

# Step 82 — Ensuring Correct Dependency Order

We ensure the access entry is created only after the IAM role exists.

```hcl
  depends_on = [
    # kubernetes_config_map_v1.aws_auth
    aws_eks_access_entry.nodes
  ]
```

This prevents Terraform from attempting to create the access entry before the role is available.

---

# Step 83 — Applying the Configuration

Now apply the Terraform configuration.

```bash
terraform apply --auto-approve --var-file=environment/prod/terraform.tfvars
```

Terraform will create the Access Entry and register the node role with the cluster.

At this point, the **EKS API now manages authentication**, instead of relying on the aws-auth ConfigMap.

---

# Step 84 — Removing the aws-auth ConfigMap

After confirming the Access Entry is working, the old aws-auth configuration can be removed.

You can verify the ConfigMap status using kubectl.

```bash
kubectl get configmap aws-auth -n kube-system
```

Once removed, the cluster will rely entirely on **Access Entries** for authentication.

---

# Step 85 — Verifying Access Entries in AWS

You can check the new access configuration in the AWS console.

Navigate to:

```txt
EKS → Cluster → Access
```

There you will see the **authorized roles and identities** managed through Access Entries.

---

# Step 86 — Testing the Node Authentication

To confirm that Access Entries are working correctly, we can force the nodes to rejoin the cluster.

For example, terminate the EC2 instances belonging to the node group.

The node group will automatically recreate them.

After a few minutes, check the nodes again.

```bash
kubectl get nodes
```

If the nodes successfully rejoin the cluster, the Access Entry configuration is working correctly.

---

# Module 2 — Lesson 9  
## Managing EKS Add-ons with Terraform

Amazon EKS provides a feature called **Add-ons**, which allows AWS to manage important Kubernetes components for the cluster.

These components are considered **core infrastructure services** required for Kubernetes to function properly.

Inside the **EKS console**, there is a dedicated section called **Add-ons**, where we can install and manage these components.

Some common examples include:

```txt
VPC CNI
CoreDNS
kube-proxy
EBS CSI Driver
EFS CSI Driver
Pod Identity Agent
```

These components can also be managed through **Terraform**, allowing us to keep them versioned and controlled as infrastructure code.

---

# Step 87 — Choosing the Core Add-ons

For this lesson we will manage three essential Kubernetes add-ons:

```txt
VPC CNI
CoreDNS
kube-proxy
```

These are fundamental components required for the cluster to operate.

- **VPC CNI** handles networking between pods and the AWS VPC.
- **CoreDNS** provides DNS resolution inside the cluster.
- **kube-proxy** manages service networking rules on each node.

---

# Step 88 — Creating Variables for Add-on Versions

To manage versions through Terraform, we create variables for each add-on.

```hcl
variable "addon_cni_version" {
  type    = string
  default = "v1.21.1-eksbuild.3"
}

variable "addon_coredns_version" {
  type    = string
  default = "v1.13.2-eksbuild.1"
}

variable "addon_kubeproxy_version" {
  type    = string
  default = "v1.34.3-eksbuild.5"
}
```

These variables allow us to easily upgrade or change versions when necessary.

---

# Step 89 — Defining Versions in terraform.tfvars

Now we define the versions inside **terraform.tfvars**.

Example configuration:

```txt
addon_cni_version        = "v1.21.1-eksbuild.3"
addon_coredns_version    = "v1.13.2-eksbuild.1"
addon_kube_proxy_version = "v1.34.3-eksbuild.5"
```

You can obtain the latest versions directly from the **EKS console** when selecting an add-on.

AWS usually shows the **recommended version** for your cluster version.

---

# Step 90 — Creating the VPC CNI Add-on

Create a new file:

```
addons.tf
```

Now we define the Terraform resource for the **VPC CNI add-on**.

```hcl
resource "aws_eks_addon" "cni" {
  cluster_name                = aws_eks_cluster.main.name
  addon_name                  = "vpc-cni"
  addon_version               = var.addon_cni_version
  resolve_conflicts_on_create = "OVERWRITE"
  resolve_conflicts_on_update = "OVERWRITE"

  depends_on = [
    aws_eks_access_entry.nodes
  ]
}
```

The **resolve_conflicts** parameters ensure that Terraform overrides any conflicting configuration.

---

# Step 91 — Creating the CoreDNS Add-on

Next we configure the **CoreDNS add-on**.

```hcl
resource "aws_eks_addon" "coredns" {
  cluster_name                = aws_eks_cluster.main.name
  addon_name                  = "coredns"
  addon_version               = var.addon_coredns_version
  resolve_conflicts_on_create = "OVERWRITE"
  resolve_conflicts_on_update = "OVERWRITE"

  depends_on = [
    aws_eks_access_entry.nodes
  ]
}
```

CoreDNS is responsible for **internal DNS resolution inside the Kubernetes cluster**.

---

# Step 92 — Creating the kube-proxy Add-on

Now we configure **kube-proxy**.

```hcl
resource "aws_eks_addon" "kubeproxy" {
  cluster_name                = aws_eks_cluster.main.name
  addon_name                  = "kube-proxy"
  addon_version               = var.addon_kubeproxy_version
  resolve_conflicts_on_create = "OVERWRITE"
  resolve_conflicts_on_update = "OVERWRITE"

  depends_on = [
    aws_eks_access_entry.nodes
  ]
}
```

kube-proxy manages **network rules and service routing** on every node.

---

# Step 93 — Defining Dependencies

We ensure the add-ons are installed **after the node group is available**.

```hcl
  depends_on = [
    aws_eks_access_entry.nodes
  ]
```

This ensures that the cluster already has nodes available to run these components.

---

# Step 94 — Applying the Configuration

Now apply the Terraform configuration.

```bash
terraform apply --auto-approve --var-file=environment/prod/terraform.tfvars
```

Terraform will install the selected add-ons into the cluster.

AWS will then manage their lifecycle automatically.

---

# Step 95 — Verifying the Add-ons

You can verify the installation using kubectl.

```bash
kubectl get pods -n kube-system
```

You should see pods related to:

```txt
coredns
kube-proxy
aws-node
```

These components will now be running in the **kube-system namespace**.

---

# Step 96 — Viewing Add-ons in the AWS Console

You can also verify the add-ons in the AWS console.

Navigate to:

```txt
EKS → Cluster → Add-ons
```

There you will see the installed add-ons and their versions.

---

# Module 2 — Lesson 10  
## Using the Helm Provider with Terraform

The **Helm provider** is one of the most useful tools when managing Kubernetes clusters as **Infrastructure as Code**.

Helm is the standard package manager for Kubernetes and allows us to install and manage applications using **Helm Charts**.

With the **Terraform Helm provider**, we can:

```txt
helm install
helm upgrade
helm uninstall
```

All of this can be managed directly through Terraform.

This allows us to fully automate the provisioning of a Kubernetes cluster **from infrastructure to applications**.

---

# Step 97 — Configuring the Helm Provider

Before using Helm with Terraform, we must configure the **Helm provider**.

The authentication method is the same used by the Kubernetes provider.

```hcl
provider "helm" {
  kubernetes = {
    host                   = aws_eks_cluster.main.endpoint
    cluster_ca_certificate = base64decode(aws_eks_cluster.main.certificate_authority.0.data)
    token                  = data.aws_eks_cluster_auth.default.token
  }
}
```

This configuration allows Terraform to authenticate with the cluster and perform Helm operations.

After adding the provider, run:

```bash
terraform init --backend-config=environment/prod/backend.tfvars
```

Terraform will download the Helm provider and prepare the environment.

---

# Step 98 — Installing the Metrics Server

One of the most common components installed in Kubernetes clusters is the **Metrics Server**.

This component collects resource metrics such as:

```txt
CPU usage
Memory usage
Node metrics
Pod metrics
```

These metrics are used by features like:

```txt
kubectl top
Horizontal Pod Autoscaler (HPA)
```

Create a new file:

```
helm_metrics_server.tf
```

We can install it using the **helm_release** resource.

```hcl
resource "helm_release" "metrics_server" {
  name       = "metrics-server"
  repository = "https://charts.bitnami.com/bitnami"
  chart      = "metrics-server"
  namespace  = "kube-system"

  wait = false

  set = [
    {
      name  = "apiService.create"
      value = "true"
    }
  ]

  depends_on = [
    aws_eks_cluster.main,
    aws_eks_node_group.main
  ]
}
```

This configuration performs the equivalent of:

```bash
helm install metrics-server metrics-server/metrics-server
```

---

# Step 99 — Applying the Configuration

Now apply the Terraform configuration.

```bash
terraform apply --auto-approve --var-file=environment/prod/terraform.tfvars
```

Terraform will download the Helm chart and deploy the Metrics Server into the cluster.

---

# Step 100 — Verifying the Metrics Server

To confirm that the deployment worked, check the pods inside the **kube-system namespace**.

```bash
kubectl get pods -n kube-system
```

You should see a deployment similar to:

```bash
metrics-server
```

This means the Metrics Server is now running in the cluster.

---

# Step 101 — Installing kube-state-metrics

Another very common component used for monitoring is **kube-state-metrics**.

This service exposes Kubernetes object metrics that can be consumed by monitoring systems such as **Prometheus**.

Examples of exported metrics include:

```txt
node labels
node annotations
deployment status
replica counts
```

Create a new file:

```
helm_kube_state_metrics.tf
```

We can deploy it using Helm.

```hcl
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
    aws_eks_node_group.main
  ]
}
```

This is equivalent to running:

```bash
helm install kube-state-metrics prometheus-community/kube-state-metrics
```

---

# Step 102 — Applying the Deployment

Run Terraform again.

```bash
terraform apply --auto-approve --var-file=environment/prod/terraform.tfvars
```

Terraform will install **kube-state-metrics** using Helm.

---

# Step 103 — Verifying the Installation

You can confirm that the service is running.

```bash
kubectl get pods -n kube-system
```

You should now see pods related to:

```txt
metrics-server
kube-state-metrics
```

These services will expose metrics that can later be scraped by monitoring systems.

---

# Step 104 — Why Helm Is Important

Helm allows us to easily install and manage complex applications in Kubernetes.

Many widely used tools are distributed as Helm charts, including:

```txt
Prometheus
Grafana
NGINX Ingress Controller
Cert Manager
ArgoCD
```

Using Terraform with Helm makes it possible to **automate the full lifecycle of these tools**.

---

# Module 2 — Lesson 11  
## Deploying the First Application in the EKS Cluster

Now that our **EKS cluster is fully configured**, we can deploy our **first application** inside the cluster.

During the course, the Kubernetes manifests used in the labs are available in the **extra resources** section. This avoids the need to manually type large YAML manifests.

In this lesson we will deploy a **simple example application called Chip**.

This lab includes basic Kubernetes resources such as:

```txt
Namespace
Deployment
Service
Horizontal Pod Autoscaler (HPA)
```

---

# Step 105 — Creating the Application Manifest

Create a file for the application deployment.

Example file:

```
assets/chip-first-deploy.yaml
```

Inside this file we define all the Kubernetes resources required for the application.

---

# Step 106 — Creating the Namespace

First we create a dedicated namespace for the application.

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: chip
```

Namespaces help organize workloads and isolate resources inside the cluster.

---

# Step 107 — Creating the Deployment

Next we create a **Deployment** that will run the application.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: chip
  name: chip
  namespace: chip
spec:
  replicas: 2
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

This deployment will create **two pods running the application**.

---

# Step 108 — Creating the Service

Now we expose the deployment using a Kubernetes **Service**.

```yaml
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

The service allows other resources in the cluster to communicate with the application.

---

# Step 109 — Creating the Horizontal Pod Autoscaler

We also configure a simple **Horizontal Pod Autoscaler (HPA)**.

```yaml
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

The HPA monitors **CPU usage** and automatically scales the deployment when necessary.

---

# Step 110 — Applying the Manifest

Now apply the Kubernetes manifest.

```bash
kubectl apply -f chip.yaml
```

Kubernetes will create all defined resources in the cluster.

---

# Step 111 — Verifying the Deployment

Check the pods running in the new namespace.

```bash
kubectl get pods -n chip
```

---

# Step 112 — Checking the Service

You can also verify the service.

```bash
kubectl get svc -n chip
```

This confirms the application is accessible inside the cluster.

---

# Step 113 — Checking the Autoscaler

To verify the HPA configuration:

```bash
kubectl get hpa -n chip
```

The autoscaler will monitor CPU usage and scale the application between:

```bash
2 pods (minimum)
5 pods (maximum)
```

---

# Result

We successfully deployed our **first application inside the EKS cluster**.

The application includes:

```txt
Namespace
Deployment with 2 replicas
Service
Horizontal Pod Autoscaler
```

This confirms that our cluster is fully functional and ready to run workloads.

---

# Conclusion of the Vanilla Cluster

At this stage we built a **fully functional EKS cluster from scratch using Terraform**.

The environment now includes:

```txt
EKS Control Plane
Managed Node Group
Access Entry Authentication
Core Add-ons
Helm Components
Running Application
```

From this point forward, the course will begin exploring **more advanced Kubernetes and EKS concepts**, expanding and improving the cluster architecture.

## Full Code

The complete implementation is available at:

[LuisGustavoBR/linuxtips-eks-vanilla](https://github.com/LuisGustavoBR/linuxtips-eks-vanilla)
