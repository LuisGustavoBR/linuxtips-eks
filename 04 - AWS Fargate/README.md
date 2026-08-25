# Module 4: AWS Fargate

## Overview

In this module we introduce **AWS Fargate in EKS**, a serverless compute option that removes the need to manage nodes.

Unlike traditional Kubernetes setups where you manage EC2 instances (nodes), Fargate allows you to run workloads without provisioning or maintaining infrastructure.

Key idea:

```txt
1 Pod = 1 dedicated Fargate microVM (node)
```

## Table of Contents

- [Lesson 1: Introduction to AWS Fargate in EKS](#lesson-1-introduction-to-aws-fargate-in-eks)
  - [1. What is Fargate?](#1-what-is-fargate)
  - [2. How Fargate Works in EKS](#2-how-fargate-works-in-eks)
  - [3. Comparing EKS Models](#3-comparing-eks-models)
    - [Self-managed Kubernetes](#self-managed-kubernetes)
    - [EKS (Default)](#eks-default)
    - [EKS with Fargate](#eks-with-fargate)
  - [4. Advantages of Fargate](#4-advantages-of-fargate)
  - [5. Limitations of Fargate](#5-limitations-of-fargate)
  - [6. When to Use Fargate](#6-when-to-use-fargate)
  - [7. What We Will Do Next](#7-what-we-will-do-next)
- [Lesson 2: Setting Up Fargate Profiles in EKS](#lesson-2-setting-up-fargate-profiles-in-eks)
  - [1. Creating the Fargate IAM Role](#1-creating-the-fargate-iam-role)
  - [2. Attaching Required Policy](#2-attaching-required-policy)
  - [3. Creating Access Entry](#3-creating-access-entry)
  - [4. Creating the Fargate Profile](#4-creating-the-fargate-profile)
  - [5. Defining the Selector](#5-defining-the-selector)
  - [6. Applying the Configuration](#6-applying-the-configuration)
  - [7. Deploying a Workload on Fargate](#7-deploying-a-workload-on-fargate)
  - [8. Understanding Provisioning Behavior](#8-understanding-provisioning-behavior)
  - [9. Validating Fargate Nodes](#9-validating-fargate-nodes)
  - [10. Scaling the Application](#10-scaling-the-application)
  - [Key Takeaways](#key-takeaways)
- [Lesson 3: Building a Fully Fargate EKS Cluster](#lesson-3-building-a-fully-fargate-eks-cluster)
  - [1. Creating a Dedicated Branch](#1-creating-a-dedicated-branch)
  - [2. Removing the Existing Fargate Profile](#2-removing-the-existing-fargate-profile)
  - [3. Creating a Wildcard Fargate Profile](#3-creating-a-wildcard-fargate-profile)
  - [4. Understanding Wildcard Profiles](#4-understanding-wildcard-profiles)
  - [5. Redeploying Workloads](#5-redeploying-workloads)
  - [6. Understanding DaemonSet Limitations](#6-understanding-daemonset-limitations)
  - [7. Observing Fargate Scheduling](#7-observing-fargate-scheduling)
  - [8. Provisioning Time](#8-provisioning-time)
  - [9. Removing Managed Node Groups](#9-removing-managed-node-groups)
  - [10. Updating Terraform Dependencies](#10-updating-terraform-dependencies)
  - [11. Understanding Security Group Changes](#11-understanding-security-group-changes)
  - [12. Opening Required Ports](#12-opening-required-ports)
  - [13. Applying the Changes](#13-applying-the-changes)
  - [14. Validating the Fargate Cluster](#14-validating-the-fargate-cluster)
  - [15. Verifying Node Types](#15-verifying-node-types)
  - [16. Metrics Server Considerations](#16-metrics-server-considerations)
  - [17. Deciding Whether to Keep Metrics Server](#17-deciding-whether-to-keep-metrics-server)
  - [18. Confirming a Fully Fargate Cluster](#18-confirming-a-fully-fargate-cluster)
  - [19. Understanding CoreDNS Challenges](#19-understanding-coredns-challenges)
  - [20. Historical CoreDNS Limitation](#20-historical-coredns-limitation)
  - [21. Current CoreDNS Behavior](#21-current-coredns-behavior)
  - [22. Preparing for the CoreDNS Workaround](#22-preparing-for-the-coredns-workaround)
- [Lesson 4: Fixing CoreDNS in Full Fargate Clusters](#lesson-4-fixing-coredns-in-full-fargate-clusters)
  - [1. Understanding the CoreDNS Problem](#1-understanding-the-coredns-problem)
  - [2. Reviewing the Lambda Code](#2-reviewing-the-lambda-code)
  - [3. Creating the Lambda IAM Role](#3-creating-the-lambda-iam-role)
  - [4. Creating the Lambda Security Group](#4-creating-the-lambda-security-group)
  - [5. Packaging the Lambda Code](#5-packaging-the-lambda-code)
  - [6. Initializing Terraform](#6-initializing-terraform)
  - [7. Creating the Lambda Function](#7-creating-the-lambda-function)
  - [8. Invoking the Lambda Automatically](#8-invoking-the-lambda-automatically)
  - [9. Passing Authentication Information](#9-passing-authentication-information)
  - [10. Triggering the CoreDNS Rollout](#10-triggering-the-coredns-rollout)
  - [11. Validating the Rollout](#11-validating-the-rollout)
  - [12. Confirming CoreDNS Health](#12-confirming-coredns-health)
  - [13. Why This Fix Is Useful](#13-why-this-fix-is-useful)
  - [Key Takeaways](#key-takeaways-1)
- [Lesson 5: Understanding Fargate Capacity and Sizing](#lesson-5-understanding-fargate-capacity-and-sizing)
  - [1. Understanding Fargate Capacity Provisioning](#1-understanding-fargate-capacity-provisioning)
  - [2. Requests and Limits Define the Size](#2-requests-and-limits-define-the-size)
  - [3. Fargate Uses Fixed Capacity Profiles](#3-fargate-uses-fixed-capacity-profiles)
  - [4. Understanding Resource Rounding](#4-understanding-resource-rounding)
  - [5. Testing Larger Resource Requests](#5-testing-larger-resource-requests)
  - [6. Inspecting Provisioned Capacity](#6-inspecting-provisioned-capacity)
  - [7. Understanding Fargate Billing](#7-understanding-fargate-billing)
  - [8. Capacity Planning Considerations](#8-capacity-planning-considerations)
  - [9. When Fargate Makes Sense](#9-when-fargate-makes-sense)
  - [Key Takeaways](#key-takeaways-2)

---

# Lesson 1: Introduction to AWS Fargate in EKS

---

## 1. What is Fargate?

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

## 2. How Fargate Works in EKS

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

## 3. Comparing EKS Models

We now have three main models:

### Self-managed Kubernetes
```txt
You manage everything  
(Control Plane + Nodes)
```

### EKS (Default)
```txt
AWS manages Control Plane  
You manage Nodes
```

### EKS with Fargate
```txt
AWS manages Control Plane + Nodes  
You manage only Pods
```

---

## 4. Advantages of Fargate

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

## 5. Limitations of Fargate

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

## 6. When to Use Fargate

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

## 7. What We Will Do Next

In this lesson, we will:

```txt
Run Fargate alongside EC2 node groups  
Create a full Fargate-based cluster  
Compare both approaches in practice
```

---

# Lesson 2: Setting Up Fargate Profiles in EKS

In this lesson we will **configure Fargate from scratch** inside our EKS cluster.

The goal is to:

```txt
Create IAM role for Fargate  
Create a Fargate Profile  
Deploy workloads directly on Fargate  
```

We will use a **clean (vanilla) cluster** as base to keep things simple.

---

## 1. Creating the Fargate IAM Role

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

## 2. Attaching Required Policy

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

## 3. Creating Access Entry

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

## 4. Creating the Fargate Profile

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

> **Note:** the real repo's own `data.tf` names this data source `pod_subnets`, not `ssm_pod_subnets` — this is a genuine bug in the reference Terraform (would fail `terraform plan`/`apply` as written), faithfully reproduced here.

Required fields:

```txt
cluster_name  
fargate_profile_name  
pod_execution_role_arn  
subnet_ids  
```

---

## 5. Defining the Selector

Fargate works using **selectors**.

You define which pods will run on Fargate based on namespace.

Example:

```hcl
selector {
  namespace = "chip"
}
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

## 6. Applying the Configuration

Run:

```bash
terraform apply --auto-approve --var-file=environment/prod/terraform.tfvars
```

After creation, you can verify in EKS console:

```txt
Fargate Profile - Active
```

---

## 7. Deploying a Workload on Fargate

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

## 8. Understanding Provisioning Behavior

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

## 9. Validating Fargate Nodes

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

## 10. Scaling the Application

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

## Key Takeaways

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

# Lesson 3: Building a Fully Fargate EKS Cluster

In the previous lesson, we configured a namespace-specific Fargate Profile and deployed workloads directly on Fargate.

Now we'll take things a step further by creating a **fully Fargate-based EKS cluster**, removing EC2 worker nodes entirely and allowing all workloads to run on Fargate.

We'll also explore some of the operational challenges that appear when running EKS without traditional nodes.

---

## 1. Creating a Dedicated Branch

Since this lesson introduces significant infrastructure changes, create a dedicated branch.

Example:

```bash
git checkout -b lesson/fargate-full
```

This allows us to experiment without affecting previous configurations.

---

## 2. Removing the Existing Fargate Profile

Previously, we created a Fargate Profile only for the `chip` namespace.

Comment out that profile because we will replace it with a cluster-wide profile — the reference implementation leaves the old `chip` profile commented out in `fargate.tf` rather than deleting it outright.

Current profile:

```txt
fargate.tf
```

---

## 3. Creating a Wildcard Fargate Profile

Instead of targeting a specific namespace, create a profile that matches all namespaces.

Example:

```hcl
resource "aws_eks_fargate_profile" "wildcard" {
  cluster_name         = aws_eks_cluster.main.name
  fargate_profile_name = "wildcard"

  pod_execution_role_arn = aws_iam_role.fargate.arn

  subnet_ids = data.aws_ssm_parameter.ssm_pod_subnets[*].value

  selector {
    namespace = "*"
  }
}
```

> **Note:** as with the earlier `chip` profile, `ssm_pod_subnets` doesn't match the real data source name (`pod_subnets`) declared in `data.tf` — a genuine bug in the reference Terraform, faithfully reproduced here.

This profile will be responsible for scheduling workloads from any namespace.

---

## 4. Understanding Wildcard Profiles

Using:

```txt
namespace = "*"
```

means:

```txt
All namespaces become eligible for Fargate scheduling
```

This greatly simplifies cluster management because individual namespace profiles are no longer required.

---

## 5. Redeploying Workloads

After creating the wildcard profile, existing workloads may still be running on EC2 nodes.

Delete and recreate pods so they can be rescheduled onto Fargate.

Example:

```bash
kubectl delete pods -A --all
```

As pods restart, EKS will evaluate them against the new profile.

---

## 6. Understanding DaemonSet Limitations

Not every workload can run on Fargate.

Important limitation:

```txt
DaemonSets are not supported on Fargate
```

Examples:

```txt
Node monitoring agents
Security agents
Infrastructure DaemonSets
```

These workloads require traditional Kubernetes nodes.

---

## 7. Observing Fargate Scheduling

After workloads are recreated, you may notice pods entering a pending state.

Example event:

```txt
Nominated Node
```

This is expected.

Fargate must provision the underlying infrastructure before the pod starts.

---

## 8. Provisioning Time

Unlike traditional node groups, Fargate creates infrastructure on demand.

Typical startup time:

```txt
40 seconds to 1 minute
```

During this period AWS:

```txt
Creates the microVM
Registers the node
Deploys the pod
```

---

## 9. Removing Managed Node Groups

To build a true Fargate-only cluster, remove all EC2 node groups.

Delete:

```txt
nodes.tf
```

After removal, all compute capacity will come from Fargate.

---

## 10. Updating Terraform Dependencies

Resources that previously depended on node groups should now depend on the Fargate Profile.

Example:

```hcl
depends_on = [
  aws_eks_cluster.main,
  aws_eks_fargate_profile.wildcard
]
```

This ensures workloads are only deployed after Fargate is available.

---

## 11. Understanding Security Group Changes

Fargate networking differs from EC2 node groups.

Fargate workloads use the cluster security group directly.

Because of that, some ports must be explicitly allowed.

---

## 12. Opening Required Ports

Common ports to allow include:

```txt
sg.tf
```

```hcl
resource "aws_security_group_rule" "port_80" {
  cidr_blocks       = ["0.0.0.0/0"]
  from_port         = 80
  to_port           = 80
  protocol          = "tcp"
  type              = "ingress"
  security_group_id = aws_eks_cluster.main.vpc_config[0].cluster_security_group_id
}

resource "aws_security_group_rule" "port_443" {
  cidr_blocks       = ["0.0.0.0/0"]
  from_port         = 443
  to_port           = 443
  protocol          = "tcp"
  type              = "ingress"
  security_group_id = aws_eks_cluster.main.vpc_config[0].cluster_security_group_id
}

resource "aws_security_group_rule" "port_8080" {
  cidr_blocks       = ["0.0.0.0/0"]
  from_port         = 8080
  to_port           = 8080
  protocol          = "tcp"
  type              = "ingress"
  security_group_id = aws_eks_cluster.main.vpc_config[0].cluster_security_group_id
}

resource "aws_security_group_rule" "port_8443" {
  cidr_blocks       = ["0.0.0.0/0"]
  from_port         = 8443
  to_port           = 8443
  protocol          = "tcp"
  type              = "ingress"
  security_group_id = aws_eks_cluster.main.vpc_config[0].cluster_security_group_id
}
```

These ports are commonly used by:

```txt
Applications
Ingress Controllers
Admission Controllers
```

---

## 13. Applying the Changes

Deploy all modifications.

```bash
terraform apply --auto-approve --var-file=environment/prod/terraform.tfvars
```

This updates:

```txt
Fargate Profiles
Security Groups
Cluster Configuration
```

---

## 14. Validating the Fargate Cluster

Check running workloads:

```bash
kubectl get pods -A
```

Most workloads should now be scheduled on Fargate infrastructure.

---

## 15. Verifying Node Types

Check nodes:

```bash
kubectl get nodes
```

You should now see Fargate nodes replacing traditional EC2 worker nodes.

Remember:

```txt
1 Pod = 1 Fargate Node
```

---

## 16. Metrics Server Considerations

Metrics Server may not behave as expected in a fully Fargate cluster.

Reason:

```txt
Metrics Server expects Kubernetes nodes for scraping metrics
```

Without traditional worker nodes, it may remain unhealthy.

---

## 17. Deciding Whether to Keep Metrics Server

For Fargate-only clusters, consider:

```txt
Removing Metrics Server
Using alternative monitoring solutions
Running a hybrid EC2 + Fargate architecture
```

The correct approach depends on your operational requirements.

---

## 18. Confirming a Fully Fargate Cluster

At this point:

```txt
No EC2 worker nodes are required
All workloads run on Fargate
AWS manages compute capacity
Infrastructure management is minimized
```

The cluster is now operating entirely on Fargate.

---

## 19. Understanding CoreDNS Challenges

One common challenge in fully Fargate clusters involves CoreDNS.

Symptoms may include:

```txt
Slow startup
Scheduling delays
Provisioning timeouts
```

This is more common during initial cluster provisioning.

---

## 20. Historical CoreDNS Limitation

Older EKS versions could not schedule CoreDNS directly on Fargate.

A workaround was required to move CoreDNS onto Fargate infrastructure.

---

## 21. Current CoreDNS Behavior

Modern EKS versions support CoreDNS on Fargate.

However, during cluster bootstrap, CoreDNS may still require intervention to complete startup successfully.

---

## 22. Preparing for the CoreDNS Workaround

AWS provides a Lambda-based workaround commonly used to restart CoreDNS and force proper scheduling.

In the next lesson, we will deploy this solution and see how it helps CoreDNS initialize correctly in fully Fargate environments.

---

# Lesson 4: Fixing CoreDNS in Full Fargate Clusters

In this lesson we will solve one of the most common challenges when building a **fully Fargate-based EKS cluster**.

The component involved is:

```txt
CoreDNS
```

Although newer EKS versions support CoreDNS running on Fargate, during cluster provisioning it is still common to encounter situations where CoreDNS becomes stuck or fails to start correctly.

To solve this, we will create a simple Lambda function that forces a new CoreDNS rollout whenever the cluster is provisioned.

The goal is:

```txt
Package a Lambda function
Deploy it with Terraform
Invoke it automatically
Force a CoreDNS rollout
Ensure CoreDNS starts correctly on Fargate
```

---

## 1. Understanding the CoreDNS Problem

In fully Fargate-based clusters, CoreDNS may fail to become healthy during initial provisioning.

Typical symptoms:

```txt
CoreDNS pods remain Pending
CoreDNS rollout gets stuck
Cluster DNS becomes unavailable
```

Although newer EKS versions improved this behavior, a rollout restart is still commonly used as a workaround.

---

## 2. Reviewing the Lambda Code

The course materials include a pre-built Lambda function called:

```txt
coredns_fix.tf
```

Create a new directory:

```txt
lambda/coredns/
```

And create:

```txt
main.py
```

```python
import json
import logging
import os
import ssl
import time
import urllib.parse
import urllib.request
import urllib.response
from datetime import datetime
from typing import Dict
 
# configure logging parameters
debug = os.environ.get("DEBUG")
logging_level = logging.DEBUG if debug else logging.INFO
request_debuglevel = 5 if debug else 0
 
 
# configure logger
logger = logging.getLogger()
logger.setLevel(logging.DEBUG)
 
# configure request handler
request_context = ssl._create_unverified_context()
request_handler = urllib.request.HTTPSHandler(
    context=request_context, debuglevel=request_debuglevel
)
 
 
def build_patch_payload(annotations: Dict) -> Dict:
    return json.dumps(
        {"spec": {"template": {"metadata": {"annotations": annotations}}}}
    )
 
 
def build_patch_body(annotations: Dict) -> Dict:
    return json.dumps(
        {"spec": {"template": {"metadata": {"annotations": annotations}}}}
    )
 
 
def patch_coredns_service(url: str, headers: Dict[str, str], data: str) -> None:
    request = urllib.request.Request(
        url, headers=headers, data=bytes(data.encode("utf-8")), method="PATCH"
    )
    opener = urllib.request.build_opener(request_handler)
    with opener.open(request) as response:
        return response.read().decode()
 
def fix(endpoint, token, stepback):
    url = f"{endpoint}/apis/apps/v1/namespaces/kube-system/deployments/coredns"
    headers = {
        "Authorization": f"Bearer {token}",
        "Accept": "application/json",
        "Content-Type": "application/strategic-merge-patch+json",
    }
    
    logger.error("Waiting stepback for: %s", stepback)
    time.sleep(stepback)

    try:
        patch_payload = build_patch_payload(
            {"$patch": "delete", "eks.amazonaws.com/compute-type": "ec2"}
        )
        logging.info("Patch Request: %s", patch_payload)
        patch_response = patch_coredns_service(url, headers, patch_payload)
        logging.info("Patch Response: %s", patch_response)
 
        restart_payload = build_patch_body(
            {"kubectl.kubernetes.io/restartedAt": datetime.utcnow().isoformat()}
        )
        logging.info("Restart Request: %s", restart_payload)
        restart_response = patch_coredns_service(url, headers, restart_payload)
        logging.info("Restart Response: %s", restart_response)
    except urllib.error.HTTPError as e:
        logger.error("Request Error: %s", e)
        fix(endpoint, token, stepback * 2)


def handler(event, _):
    logger.info(event)
 
    token = event.get("token")
    endpoint = event.get("endpoint")

    fix(endpoint, token, 5)
```

This Lambda is intentionally simple.

Its purpose is to:

```txt
Connect to the cluster
Patch CoreDNS
Force a rollout restart
```

Nothing more.

---

## 3. Creating the Lambda IAM Role

Create:

```txt
coredns_fix.tf
```

```hcl
data "aws_iam_policy_document" "coredns_fix" {

  version = "2012-10-17"

  statement {

    actions = ["sts:AssumeRole"]

    principals {
      type = "Service"
      identifiers = [
        "lambda.amazonaws.com"
      ]
    }
  }
}

resource "aws_iam_role" "coredns_fix" {
  name_prefix        = format("%s-coredns-fix", var.project_name)
  assume_role_policy = data.aws_iam_policy_document.coredns_fix.json
}

resource "aws_iam_role_policy_attachment" "coredns_fix" {
  role       = aws_iam_role.coredns_fix.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole"
}
```

First create the Lambda execution role.

The trust relationship is the standard Lambda configuration:

```txt
lambda.amazonaws.com
```

This Lambda does not require custom IAM permissions.

We only need the AWS managed policy:

```txt
AWSLambdaVPCAccessExecutionRole
```

This allows the Lambda to:

```txt
Create ENIs
Run inside the VPC
Access private cluster resources
```

---

## 4. Creating the Lambda Security Group

Next create a Security Group for the Lambda.

Requirements:

```txt
Outbound traffic allowed
No inbound rules required
```

```hcl
resource "aws_security_group" "coredns_fix" {
  name   = format("%s-coredns-fix", var.project_name)
  vpc_id = data.aws_ssm_parameter.vpc.value

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

The Lambda only needs to reach the EKS API endpoint.

---

## 5. Packaging the Lambda Code

Terraform provides a useful provider for packaging files.

Add:

```txt
archive provider
```

Create an archive resource:

```hcl
data "archive_file" "coredns_archive" {
  type        = "zip"
  source_dir  = "lambda/coredns"
  output_path = "lambda/coredns.zip"
}
```

This will generate:

```txt
coredns.zip
```

containing the Lambda source code.

---

## 6. Initializing Terraform

Because we introduced a new provider, run:

```bash
terraform init -backend-config=environment/prod/backend.tfvars
```

Terraform will download:

```txt
archive provider
```

and make it available to the project.

---

## 7. Creating the Lambda Function

Now create the Lambda resource.

Required configuration:

```txt
Runtime
Handler
Execution Role
Subnets
Security Group
Deployment Package
```

The ZIP file generated previously will be used as the deployment package.

Apply the configuration:

```bash
terraform apply --auto-approve --var-file=environment/prod/terraform.tfvars
```

After a few moments the Lambda should be available.

---

## 8. Invoking the Lambda Automatically

Now we want Terraform to execute the Lambda automatically.

Terraform provides:

```txt
aws_lambda_invocation
```

This data source invokes a Lambda function during execution.

Example workflow:

```txt
Terraform Apply
        ↓
Lambda Invocation
        ↓
CoreDNS Patch
        ↓
New Rollout
```

---

## 9. Passing Authentication Information

The Lambda needs cluster access.

Pass the required payload:

```txt
Cluster Endpoint
Authentication Token
```

These values allow the Lambda to connect to Kubernetes and execute the rollout.

---

## 10. Triggering the CoreDNS Rollout

When Terraform invokes the Lambda:

```txt
CoreDNS Deployment
        ↓
Patch Applied
        ↓
New ReplicaSet Created
        ↓
Pods Restarted
```

This forces Kubernetes to recreate the CoreDNS pods.

---

## 11. Validating the Rollout

You can observe the rollout using:

```bash
kubectl get pods -n kube-system -w
```

You should see:

```txt
Old CoreDNS pods terminating
New CoreDNS pods starting
```

The rollout occurs gradually, avoiding downtime.

---

## 12. Confirming CoreDNS Health

After the rollout completes, verify:

```bash
kubectl get pods -n kube-system
```

Expected result:

```txt
CoreDNS pods Running
```

At this point DNS resolution should be functioning normally again.

---

## 13. Why This Fix Is Useful

This approach helps when:

```txt
Provisioning a brand-new Fargate cluster
CoreDNS becomes stuck during initialization
Cluster bootstrap requires DNS recovery
```

```txt
coredns_fix.tf
```

```hcl
data "aws_iam_policy_document" "coredns_fix" {

  version = "2012-10-17"

  statement {

    actions = ["sts:AssumeRole"]

    principals {
      type = "Service"
      identifiers = [
        "lambda.amazonaws.com"
      ]
    }
  }
}

resource "aws_iam_role" "coredns_fix" {
  name_prefix        = format("%s-coredns-fix", var.project_name)
  assume_role_policy = data.aws_iam_policy_document.coredns_fix.json
}

resource "aws_iam_role_policy_attachment" "coredns_fix" {
  role       = aws_iam_role.coredns_fix.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaVPCAccessExecutionRole"
}

resource "aws_security_group" "coredns_fix" {
  name   = format("%s-coredns-fix", var.project_name)
  vpc_id = data.aws_ssm_parameter.vpc.value

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}

data "archive_file" "coredns_archive" {
  type        = "zip"
  source_dir  = "lambda/coredns"
  output_path = "lambda/coredns.zip"
}

resource "aws_lambda_function" "coredns_fix" {
  function_name = format("%s-coredns-fix", var.project_name)
  runtime       = "python3.13"

  handler          = "main.handler"
  role             = aws_iam_role.coredns_fix.arn
  filename         = data.archive_file.coredns_archive.output_path
  source_code_hash = data.archive_file.coredns_archive.output_base64sha256
  timeout          = 120

  vpc_config {
    subnet_ids         = data.aws_ssm_parameter.private_subnets[*].value
    security_group_ids = [aws_security_group.coredns_fix.id]
  }
}

data "aws_lambda_invocation" "coredns_fix" {
  function_name = aws_lambda_function.coredns_fix.function_name
  input         = <<JSON
{
  "endpoint": "${aws_eks_cluster.main.endpoint}",
  "token": "${data.aws_eks_cluster_auth.default.token}"
}
JSON

  depends_on = [
    aws_lambda_function.coredns_fix,
    aws_eks_fargate_profile.wildcard
  ]
}
```

The Lambda acts as an automated recovery mechanism.

---

## Key Takeaways

In this lesson we:

```txt
Created a Lambda function
Packaged it using Terraform
Deployed it inside the VPC
Invoked it automatically
Forced a CoreDNS rollout
Recovered CoreDNS on Fargate
```

With CoreDNS working correctly, the cluster is now ready for the next step: understanding how Fargate handles CPU, memory, and capacity planning.

---

# Lesson 5: Understanding Fargate Capacity and Sizing

In this lesson we will explore how **AWS Fargate allocates CPU and memory** for Kubernetes workloads.

Unlike EC2-based node groups, where you choose instance sizes directly, Fargate provisions resources automatically based on the requests and limits defined in your pods.

Understanding these sizing rules is extremely important because:

```txt
Fargate billing is based on provisioned capacity
Not necessarily on requested capacity
```

This means a small configuration mistake can increase costs significantly.

---

## 1. Understanding Fargate Capacity Provisioning

When a pod runs on Fargate, AWS automatically provisions a dedicated node for that pod.

You can inspect the pod using:

```bash
kubectl describe pod <pod-name>
```

Inside the pod metadata you will find annotations similar to:

```txt
CapacityProvisioned:
0.25 vCPU
1 GB Memory
```

This annotation shows the actual capacity allocated by Fargate.

---

## 2. Requests and Limits Define the Size

Fargate sizing is determined by the resource requests and limits configured in the pod specification.

Example:

```yaml
resources:
  requests:
    cpu: 250m
    memory: 512Mi
```

Based on these values, Fargate selects the closest supported profile.

---

## 3. Fargate Uses Fixed Capacity Profiles

Fargate does not allow arbitrary CPU and memory combinations.

Instead, AWS provides predefined sizing ranges.

For example:

```txt
0.25 vCPU
  512 MB
  1 GB
  2 GB
```

```txt
0.5 vCPU
  1 GB
  2 GB
  3 GB
  4 GB
```

```txt
1 vCPU
  2 GB
  3 GB
  4 GB
  5 GB
  6 GB
  7 GB
  8 GB
```

Fargate always selects one of these supported combinations.

---

## 4. Understanding Resource Rounding

A very important behavior is that Fargate may provision more resources than requested.

Example request:

```txt
CPU: 500m
Memory: 512Mi
```

Fargate may allocate:

```txt
0.5 vCPU
1 GB Memory
```

because that is the nearest valid profile.

The allocated capacity can be larger than the requested capacity.

---

## 5. Testing Larger Resource Requests

Let's increase the pod resources.

Example:

```yaml
resources:
  requests:
    cpu: 1024m
    memory: 2048Mi
  limits:
    cpu: 1024m
    memory: 2048Mi
```

This requests:

```txt
1 vCPU
2 GB Memory
```

After redeploying the workload, inspect the pod again.

---

## 6. Inspecting Provisioned Capacity

Run:

```bash
kubectl describe pod <pod-name>
```

Check the annotation:

```txt
CapacityProvisioned
```

You may notice something similar to:

```txt
1 vCPU
2 GB Memory
```

This matches exactly what was requested. No rounding happens this time, because `1 vCPU / 2 GB` already matches one of the supported profiles from the table in Section 3 — unlike the rounding example in Section 4.

---

## 7. Understanding Fargate Billing

One of the most important details about Fargate is billing.

AWS charges based on:

```txt
Provisioned Capacity
```

Not:

```txt
Requested Capacity
```

Example:

```txt
Requested:
0.5 vCPU
0.5 GB RAM

Provisioned:
0.5 vCPU
1 GB RAM
```

Billing will be calculated using:

```txt
0.5 vCPU
1 GB RAM
```

---

## 8. Capacity Planning Considerations

Because of the sizing model, capacity planning becomes very important.

Always evaluate:

```txt
CPU requests
Memory requests
Supported Fargate profiles
Actual provisioned capacity
```

Small adjustments can significantly reduce costs.

---

## 9. When Fargate Makes Sense

Fargate works particularly well for:

```txt
Small environments
Short-lived clusters
Development environments
Low operational overhead
```

Main advantages:

```txt
No node management
Strong isolation
Simple operations
```

---

## Key Takeaways

In this lesson we learned:

```txt
How Fargate provisions resources
How requests and limits affect sizing
How CapacityProvisioned works
How Fargate billing is calculated
```

Most importantly:

```txt
Fargate bills based on provisioned capacity,
not necessarily on the resources you requested.
```

Understanding this behavior is essential for both performance planning and cost optimization when running EKS on Fargate.

---
