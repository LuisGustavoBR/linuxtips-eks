# LinuxTips EKS Learning Path

Welcome to the repository for the **LinuxTips EKS Learning Path**!
This repository contains practical labs, notes, and configurations based on the *Descomplicando EKS* training by LinuxTips.
The goal is to consolidate my hands-on experience with Amazon Elastic Kubernetes Service, demonstrating real-world production-grade Kubernetes architecture on AWS.

## Overview

Amazon EKS is a fully managed Kubernetes service that simplifies cluster operations while providing high availability, security, and scalability.

This repository follows a modular structure covering:

- Networking architecture
- Control Plane provisioning
- Node management strategies
- Autoscaling with Karpenter
- Ingress and Load Balancing
- Storage integrations
- Secrets management
- Observability stack
- Service Mesh
- GitOps
- Progressive Delivery
- Multicluster architecture
- Logging with Loki

Everything is built using Infrastructure as Code and production best practices.

## Table of Contents

The repository is organized into modules, each focused on a specific EKS topic.

### [Module 01 - EKS Fundamentals and Networking Foundation](./01%20-%20EKS%20Fundamentals%20and%20Networking%20Foundation/README.md)

- Introduction to EKS
- Usage models
- Initial setup (IAM and S3 backend)
- VPC design
- Public and private subnets
- NAT Gateways and Elastic IPs
- Pod subnets
- Database subnets and NACLs
- Parameter Store integration

### [Module 02 - Control Plane and Cluster Setup](./02%20-%20Control%20Plane%20and%20Cluster%20Setup/README.md)

- Initial cluster setup
- IAM roles and KMS
- Control plane deployment
- Security group management
- EKS Zone Shifter
- Terraform Kubernetes provider
- Managed Node Groups
- IAM Access Entry migration
- Core Addons (CoreDNS, Kube-Proxy, VPC-CNI)
- Helm provider setup
- First Kubernetes deployment

### [Module 03 - Advanced Node Groups Strategies](./03%20-%20Advanced%20Node%20Groups%20Strategies/README.md)

- On-Demand and Spot Node Groups
- Bottlerocket nodes
- Graviton (ARM64) nodes
- Workload segregation with Node Selectors
- Node Affinity
- Critical workloads segregation
- Custom Launch Templates
- Cluster Autoscaler
- Node Termination Handler

### [Module 04 - AWS Fargate](./04%20-%20AWS%20Fargate/README.md)

- Fargate fundamentals and Firecracker
- Fargate profiles
- Namespace selectors
- Full Fargate clusters
- CoreDNS fixes with Lambda
- Application deployment on Fargate
- Cloud Native deployment strategies

### [Module 05 - Autoscaling with Karpenter (Part 1)](./05%20-%20Autoscaling%20with%20Karpenter/README.md)

- Introduction to Karpenter
- Helm installation
- EC2NodeClasses and NodePools
- Terraform production setup
- Spot strategies
- Workload segregation
- Interruption queue handling

### [Module 06 - Karpenter Groupless Architecture (Part 2)](./06%20-%20Karpenter%20Groupless%20Architecture/README.md)

- Groupless architecture model
- Fargate profiles for kube-system and karpenter namespaces
- Redeploying the CoreDNS fix Lambda
- Deleting traditional node groups
- Fixing depends_on references after node group removal
- Deploying workloads without a Node Selector

### [Module 07 - AWS Load Balancer Controller](./07%20-%20AWS%20Load%20Balancer%20Controller/README.md)

- IAM setup (IRSA) and Helm installation
- Network Load Balancer via Service annotations
- Application Load Balancer via Ingress and host-based routing
- HTTPS with ACM and DNS validation
- Target Group Binding for Kubernetes-decoupled load balancers

### [Module 08 - NGINX Ingress Controller](./08%20-%20NGINX%20Ingress%20Controller/README.md)

- Shared NLB architecture with Target Group Binding
- Installing NGINX Ingress Controller with Helm
- Capacity and autoscaling for the Ingress Controller
- Parametrizing Helm values and Deployment vs DaemonSet
- Routing multiple services through one shared Ingress Controller

### [Module 09 - Storage in EKS](./09%20-%20Storage%20in%20EKS/README.md)

- CSI drivers overview and Pod Identity as an alternative to IRSA
- EBS CSI driver installation and static PVC provisioning
- Dynamic provisioning with StatefulSets and volumeClaimTemplates
- GP3 StorageClasses
- EFS CSI driver installation and shared, multi-pod volumes
- S3 CSI driver installation and mounting a bucket as a volume

### [Module 10 - External Secrets](./10%20-%20External%20Secrets/README.md)

- External Secrets Operator overview and Pod Identity setup
- IAM setup and installation via Helm
- SecretStore and ExternalSecret CRDs, wired into a Deployment
- AWS Secrets Manager integration, including JSON multi-value secrets
- AWS Parameter Store integration

### [Module 11 - EKS Auto Mode](./11%20-%20EKS%20Auto%20Mode/README.md)

- EKS Auto Mode overview: what it manages, node lifecycle, pros and cons
- Enabling Auto Mode: cluster IAM permissions and the `aws_eks_cluster` config
- First deployment and the built-in `general-purpose`/`system` NodePools
- Scheduling system components onto the `system` NodePool
- Ingress and Load Balancer integration on Auto Mode

### [Module 12 - Observability with Prometheus and Grafana](./12%20-%20Observability%20with%20Prometheus%20and%20Grafana/README.md)

- Introduction to the kube-prometheus-stack umbrella Helm chart and its components
- Preparing the cluster: Pod Identity, EFS CSI, and a temporary CoreDNS/Karpenter workaround
- Installing Prometheus and exposing Grafana behind an NGINX Ingress
- Scraping real metrics with ServiceMonitors and importing Grafana Labs dashboards
- Persisting Prometheus and Grafana data on EFS so pod restarts don't lose it
- Segregating capacity onto a dedicated Karpenter NodePool and tuning metric retention

### [Module 13 - Service Mesh with Istio](./13%20-%20Service%20Mesh%20with%20Istio/README.md)

- Introduction to service meshes, the Envoy sidecar pattern, and mTLS
- Retiring NGINX and installing Istio's base, control plane, and ingress gateway charts with Helm
- Productionizing the Ingress Gateway: NodePort exposure, autoscaling, and binding it to the existing NLB
- Deploying a multi-service lab application with automatic sidecar injection
- Migrating Grafana onto the Istio Gateway and scraping every Envoy sidecar with a PodMonitor
- Distributed tracing with Jaeger, wired into istiod's mesh-wide auto-tracing
- Installing Kiali and integrating it with Jaeger, Prometheus, and Grafana
- Resilience with VirtualService retries and DestinationRule circuit breaking
- Mapping external, out-of-mesh traffic with a ServiceEntry

### [Module 14 - Event-Driven Autoscaling with KEDA](./14%20-%20Event-Driven%20Autoscaling%20with%20KEDA/README.md)

- What KEDA adds beyond Karpenter's node-level autoscaling, and its scaler ecosystem
- Installing KEDA with Pod Identity and a Terraform helm_release
- Scaling on CPU usage with a ScaledObject and its underlying HPA
- Scaling on a schedule with a cron trigger
- Scaling on requests per second with a Prometheus trigger and k6 load testing
- Scaling an SQS consumer into DynamoDB with a TriggerAuthentication and ServiceEntry mapping
- Running KEDA itself on a dedicated Fargate profile

### [Module 15 - Progressive Delivery with Argo Rollouts](./15%20-%20Progressive%20Delivery%20with%20Argo%20Rollouts/README.md)

- From Deployment to the Rollout CRD, and installing Argo Rollouts with a Terraform helm_release
- Exposing the Argo Rollouts dashboard through Istio
- Manual, time-based, and metric-driven canary promotion with AnalysisTemplates
- Blue-Green deployments: manual and automatic promotion, scaleDownDelaySeconds
- Warming up pods pre-promotion with a containerized k6 load test and a job-based AnalysisTemplate
- Metric-based pre-promotion analysis against the preview service
- Running Argo Rollouts itself on a dedicated Fargate profile

### [Module 16 - Helm Advanced Usage](./16%20-%20Helm%20Advanced%20Usage/README.md)

- Packaging every platform capability built so far (Rollouts, Istio, KEDA) into one reusable Helm chart
- Building the chart from scratch, resource by resource, against `helm create` and Helmfy reference scaffolds
- Feature-toggled Namespace, Gateway, VirtualService, ServiceMonitor, and KEDA `ScaledObject`
- A canary-only Rollout template with configurable steps, probes, capacity, and topology spread
- Parameterized `AnalysisTemplate`s and five templated KEDA trigger types (Prometheus, cron, memory, CPU, SQS)
- Packaging the finished chart with `helm package` for the next module's Argo CD hand-off

### [Module 17 - GitOps with ArgoCD](./17%20-%20GitOps%20with%20ArgoCD/README.md)

- GitOps concepts, Application vs. ApplicationSet, and why ApplicationSets are used from day one
- Deploying ChartMuseum as an internal, S3-backed Helm chart registry via Pod Identity
- Installing Argo CD and exposing its dashboard through an Istio Gateway/VirtualService
- Deploying the `chip` canary as an ApplicationSet sourced directly from ChartMuseum
- AppProjects and a six-service "nutrition" health-API lab for context segregation
- The community Rollout Extension, promoting canaries from inside the Argo CD dashboard

### [Module 18 - ArgoCD Multicluster Architecture](./18%20-%20ArgoCD%20Multicluster%20Architecture/README.md)

- Bootstrapping a new three-stack repo: shared `ingress`, reusable `clusters`, and a dedicated `control-plane` GitOps cluster
- A shared ALB with weighted active-active routing between two workload clusters, plus optional HTTPS via ACM
- Provisioning both workload clusters (Karpenter, AWS Load Balancer Controller, Istio) with EKS Pod Identity throughout
- Federating clusters into Argo CD via a two-role IAM chain and per-cluster EKS access entries
- Managing shared add-ons (Argo Rollouts, Metrics Server, KEDA) as multicluster `ApplicationSet`s
- Deploying the `chip` canary active-active across both clusters, with independent per-cluster rollouts and ALB-based failover

### Module 19 - Final Project — Observability with Grafana Loki

- Introduction to Loki
- Observability cluster foundation
- Grafana dashboards
- Loki setup
- Network Load Balancer exposure
- Fluent Bit log shipping
- Capacity segregation

### Module 20 - Final Project — Distributed Tracing with Grafana Tempo

- Introduction to Grafana Tempo
- Setup and Installation of Grafana Tempo
- Tempo Exposure
- Grafana Datasource Configuration
- OpenTelemetry Collector Setup for Trace Collection
- Health API Lab Deployment
- ArgoCD integration for OpenTelemetry
- Helm deployment for Tempo
- IAM and Pod Identity configuration
- Load Balancer and Target Group Binding
- S3 backend configuration for Tempo
- Route53 integration

### Module 21 - Final Project — Centralized Metrics with Grafana Mimir

- Introduction to Grafana Mimir
- Initial Setup
- Grafana Mimir Installation
- Mimir Exposure
- Datasource Configuration
- Prometheus Server installation across clusters
- Prometheus scrape configs
- Remote Write from Prometheus to Grafana Mimir
- ArgoCD integration (standard, complete and remote write setups)
- IAM and Pod Identity configuration
- S3 backend configuration for Mimir
- Load Balancer and Route53 integration
- Target Group Binding

### Module 22 - Final Project — Observability Correlation (Metrics, Logs and Traces)

- Introduction to Datasources
- Metrics and Traces correlation (Service Maps with Tempo)
- Logs and Traces correlation (Loki and Tempo)
- Integrated Dashboard (Metrics x Logs x Traces)
- Metrics Generator configuration
- Loki configuration
- Tempo configuration
- Full Terraform locals setup
- Example dashboard JSON configuration

### Module 23 - Chaos Engineering with Chaos Mesh

- Introduction to Chaos Mesh
- Installation of Chaos Mesh
- Pod Kill and Pod Failure tests
- Network Delay, Partition and Bandwidth tests
- CPU Stress and Memory Stress tests
- DNS Error and DNS Random IP tests
- Chaos Mesh Dashboards
- Chaos Workflows
- Scheduling Chaos Experiments
- Helm deployment of Chaos Mesh
- Exposure configuration
- Lab deployment for chaos testing
- Variables and infrastructure setup

## Usage

Each module folder contains its own README with objectives, configuration files, and step-by-step exercises.
You can clone the repository and follow along:

```bash
git clone https://github.com/LuisGustavoBR/linuxtips-eks.git
```

## How to Contribute

Contributions are welcome!
If you find issues or have improvements, feel free to open a pull request.
Please maintain consistent formatting and clear explanations in your submissions.

## Disclaimer

This repository is for educational purposes only.
While it follows best practices, it may not reflect production-grade configurations.
Always validate your setup and consult the official EKS documentation for deployment-critical environments.

## Credits

Original training by **LinuxTips**
Hands-on notes and exercises compiled by **Luis Gustavo Bordon**
