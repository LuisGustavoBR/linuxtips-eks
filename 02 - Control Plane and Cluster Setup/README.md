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
variable "ssm_pods_subnets" {
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

ssm_pods_subnets = [
  "/linuxtips-vpc/subnets/private/us-east-1a/linuxtips-private-1a",
  "/linuxtips-vpc/subnets/private/us-east-1b/linuxtips-private-1b",
  "/linuxtips-vpc/subnets/private/us-east-1c/linuxtips-private-1c",
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
