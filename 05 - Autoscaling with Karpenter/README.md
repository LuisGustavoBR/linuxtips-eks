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

# Module 5 - Lesson 2
## Installing Karpenter on EKS

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

# Step 17 - Starting from a Vanilla Cluster

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

# Step 18 - Creating the Karpenter IAM Resources

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

resource "aws_iam_policy_attachment" "karpenter" {
  name = "karpenter"
  roles = [
    aws_iam_role.karpenter.name
  ]

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

# Step 19 - Required Permissions

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

# Step 20 - Understanding SQS Permissions

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

Without requiring an additional Node Termination Handler deployment.

---

# Step 21 - Creating the IAM Role

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

# Step 22 - Applying the IAM Configuration

Deploy the IAM resources:

```bash
terraform apply --auto-approve --var-file=environment/prod/terraform.tfvars
```

After the deployment completes, Karpenter will have the permissions required to manage AWS infrastructure.

---

# Step 23 - Installing Karpenter with Helm

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
    },

    {
      name  = "aws.defaultInstanceProfile"
      value = aws_iam_instance_profile.nodes.name
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

# Step 24 - Using the Official Repository

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

# Step 25 - Configuring IRSA

Because we are still using IRSA, we must annotate the service account.

The annotation links:

```txt
Service Account
↓
IAM Role
```

This allows Karpenter to authenticate securely against AWS APIs without static credentials.

---

# Step 26 - Required Helm Parameters

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

```txt
defaultInstanceProfile
```

Defines the IAM Instance Profile that new EC2 nodes will receive.

---

# Step 27 - Understanding the Instance Profile

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

Without this configuration, nodes would be created but could not register in Kubernetes.

---

# Step 28 - Deploying Karpenter

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

# Step 29 - Verifying the Installation

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

# Step 30 - Monitoring Karpenter Logs

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

# Step 31 - What Has Been Installed?

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

# Step 32 - Key Takeaways

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

# Module 5 - Lesson 3
## Understanding NodePools and EC2NodeClasses in Karpenter

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

# Step 33 - Creating an EC2NodeClass

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

# Step 34 - Understanding Each EC2NodeClass Setting

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
Windows
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

# Step 35 - Creating a NodePool

With the infrastructure definition ready, we can create a NodePool.

The NodePool defines the provisioning rules that Karpenter will follow.

Add the following resource to the same file.

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

# Step 36 - Defining Instance Families

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

# Step 37 - Choosing Spot or On-Demand

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

# Step 38 - Understanding Consolidation

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

# Step 39 - Applying the Configuration

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

# Step 40 - Deploying a Test Application

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

      # Topology Spread da zona de disponibilidade
      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: "topology.kubernetes.io/zone"
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: chip
              
        # Topology Spread do tipo de instância
        - maxSkew: 1
          topologyKey: "node.kubernetes.io/instance-type"
          whenUnsatisfiable: ScheduleAnyway
          labelSelector:
            matchLabels:
              app: chip
              
        # Topology Spread do tipo de capacity-type
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

# Step 41 - Triggering Node Provisioning

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

# Step 42 - Watching Karpenter Create Nodes

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

# Step 43 - Scaling Back Down

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

# Step 44 - Watching Consolidation in Action

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

# Step 45 - Why Karpenter Is More Powerful Than Cluster Autoscaler

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

# Step 46 - What We Will Improve Next

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

# Module 5 - Lesson 4
## Automating Karpenter NodePools with Terraform

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

# Step 47 - Installing the Kubectl Manifest Provider

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

After adding the provider configuration, run:

```bash
terraform init -backend-config=environment/prod/backend.tfvars
```

Terraform will download the required provider and make it available for use.

---

# Step 48 - Defining the Capacity Configuration

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

# Step 49 - Creating the Capacity Object

A single entry in the list might look like:

```hcl
karpenter_capacity = [
  {
    name               = "linux-apps"
    workload           = "linux"
    ami_family         = "AL2023"

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

# Step 50 - Retrieving the Latest AMI Automatically

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

# Step 51 - Creating Template Files

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

# Step 52 - Building the EC2NodeClass Template

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
```

Terraform will replace these placeholders automatically during execution.

---

# Step 53 - Handling Dynamic Subnets

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

# Step 54 - Creating the EC2NodeClass Resource

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

# Step 55 - Validating the EC2NodeClass

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

# Step 56 - Building the NodePool Template

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

# Step 57 - Generating Instance Families

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

# Step 58 - Generating Instance Sizes

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

# Step 59 - Generating Capacity Types

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

# Step 60 - Creating Dynamic Workload Labels

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

# Step 61 - Creating the NodePool Resource

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

# Step 62 - Applying the Configuration

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

# Step 63 - Benefits of This Approach

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

# Step 64 - Preparing for Workload Segregation

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
