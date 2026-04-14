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

# Module 4 - Lesson 2  
## Setting Up Fargate Profiles in EKS

In this lesson we will **configure Fargate from scratch** inside our EKS cluster.

The goal is to:

```txt
Create IAM role for Fargate  
Create a Fargate Profile  
Deploy workloads directly on Fargate  
```

We will use a **clean (vanilla) cluster** as base to keep things simple.

---

# Step 8 - Creating the Fargate IAM Role

First, we need an IAM Role that Fargate will use.

Create a policy document `iam_fargate.tf`:

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

---

# Step 9 - Attaching Required Policy

Now attach the AWS managed policy:

```txt
AmazonEKSFargatePodExecutionRolePolicy
```

This policy allows Fargate to:

```txt
Pull container images  
Write logs  
Integrate with AWS services
```

---

# Step 10 - Creating Access Entry

We must allow this role to interact with the cluster.

Create an access entry similar to node groups:

```txt
access_entries.tf
```

```hcl
resource "aws_eks_access_entry" "fargate" {
  cluster_name  = aws_eks_cluster.main.id
  principal_arn = aws_iam_role.fargate.arn
  type          = "FARGATE_LINUX"
}
```

This authorizes Fargate to join the cluster.

---

# Step 11 - Creating the Fargate Profile

Now we create the **Fargate Profile**.

This is the key resource.

```txt
fargate.tf
```

```hcl
resource "aws_eks_fargate_profile" "chip" {
  cluster_name         = aws_eks_cluster.main.name
  fargate_profile_name = "chip"

  pod_execution_role_arn = aws_iam_role.fargate.arn

  subnet_ids = data.aws_ssm_parameter.ssm_pod_subnets[*].value

  selector {
    namespace = "chip"
  }
}
```

Required fields:

```txt
cluster_name  
fargate_profile_name  
pod_execution_role_arn  
subnet_ids  
```

---

# Step 12 - Defining the Selector

Fargate works using **selectors**.

You define which pods will run on Fargate based on namespace.

Example:

```hcl
selector:
  namespace = "chip"
```

Meaning:

```txt
Any pod in this namespace will run on Fargate
```

Deploy:

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
            cpu: 250m
            memory: 512Mi
          limits:
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
---
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

---

# Step 13 - Applying the Configuration

Run:

```bash
terraform apply --auto-approve --var-file=environment/prod/terraform.tfvars
```

After creation, you can verify in EKS console:

```txt
Fargate Profile - Active
```

---

# Step 14 - Deploying a Workload on Fargate

Now deploy the same application (chip).

Important:

```txt
No changes required in the deployment
```

Only requirement:

```txt
Namespace must match the Fargate profile
```

---

# Step 15 - Understanding Provisioning Behavior

When deploying, you will notice:

```txt
Pods stay in Pending state
```

This happens because:

```txt
Fargate needs to provision a node first
```

Provisioning time:

```txt
~40s to 1 minute per pod
```

---

# Step 16 - Validating Fargate Nodes

Check nodes:

```bash
kubectl get nodes
```

You will see:

```txt
EC2 nodes (node groups)  
Fargate nodes
```

Key behavior:

```txt
Each pod runs in its own node
```

Example:

```txt
2 pods - 2 Fargate nodes  
4 pods - 4 Fargate nodes
```

---

# Step 17 - Scaling the Application

Scale your deployment:

```bash
kubectl scale deployment chip --replicas=4
```

Observe:

```txt
New nodes are created automatically  
Each pod gets its own environment
```

---

# Step 18 - Key Takeaways

Fargate setup flow:

```txt
Create IAM Role  
Attach execution policy  
Create Fargate Profile  
Define namespace selector  
Deploy workloads
```

Important behavior:

```txt
1 Pod = 1 Node  
No node management  
Slower startup compared to EC2  
```

---
