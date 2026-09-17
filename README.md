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

### [Module 19 - Observability with Grafana Loki](./19%20-%20Observability%20with%20Grafana%20Loki/README.md)

- Bootstrapping a fourth, dedicated observability cluster by copying Module 18's control-plane stack
- Grafana with EFS-backed dashboard persistence, exposed through an internet-facing ALB
- Grafana Loki in simple-scalable mode, backed by S3 chunks and a GP3-backed write/backend path
- An internal NLB and private Route 53 zone exposing the Loki gateway inside the VPC
- Wiring Loki as a Grafana data source and querying logs with LogQL
- Shipping logs from both workload clusters into Loki with a multicluster Fluent Bit `ApplicationSet`
- Dedicated Karpenter NodePools per observability workload (`grafana`, `loki`)

### [Module 20 - Distributed Tracing with Grafana Tempo](./20%20-%20Distributed%20Tracing%20with%20Grafana%20Tempo/README.md)

- Grafana Tempo, deployed via the `tempo-distributed` Helm chart into the same observability cluster built in Module 19
- A dedicated S3 bucket, IAM Pod Identity role, and Karpenter NodePool for Tempo
- An internal NLB and private Route 53 record exposing the Tempo gateway inside the VPC
- Wiring Tempo as a Grafana data source, correlated with the existing Loki data source
- A multicluster OpenTelemetry Collector `ApplicationSet`, activated for real via Terraform on the control-plane cluster
- Deploying a large multi-service "Health API" lab, instrumented to send Zipkin-format traces through the collector into Tempo
- Exploring end-to-end distributed traces and adding a Tempo panel to the Module 19 Grafana dashboard

### [Module 21 - Centralized Metrics with Grafana Mimir](./21%20-%20Centralized%20Metrics%20with%20Grafana%20Mimir/README.md)

- Grafana Mimir, the fourth and most complex Grafana Stack component, deployed via the `mimir-distributed` Helm chart into the same observability cluster built in Modules 19-20
- Two dedicated S3 buckets, two Pod Identity associations, and a dedicated Karpenter NodePool for Mimir
- Mimir's many components at a glance: compactor, distributor, ingester, four caches, querier, query-frontend, ruler, and store-gateway
- Exposing Mimir through an internal Application Load Balancer — the one exception to this cluster's NLB exposure pattern
- Wiring Mimir as a `prometheus`-type Grafana data source
- Deploying a lightweight, stateless Prometheus server to each workload cluster via a multicluster `ApplicationSet`, activated for real via Terraform on the control-plane cluster
- Remote-writing metrics from each cluster's Prometheus into Mimir, tagged with a `cluster` external label for cross-cluster correlation
- Bringing traces (Tempo), logs (Loki), and metrics (Mimir) together in one Grafana dashboard

### [Module 22 - Observability Correlation (Metrics, Logs and Traces)](./22%20-%20Observability%20Correlation%20%28Metrics%2C%20Logs%20and%20Traces%29/README.md)

- Tying together the three pillars built in Modules 19-21 — Loki, Tempo, and Mimir — into one correlated Grafana experience, with no new infrastructure
- Enabling Tempo's metrics generator to derive service-graph and span metrics from trace data, remote-written into Mimir
- Wiring the `Tempo` datasource's `serviceMap`/`nodeGraph`/`tracesToMetrics` fields for a live, interactive service map
- Correlating logs and traces both ways via Loki's derived fields and an application-logged trace ID
- Building a dashboard that combines a service graph, an outlier-traces table, application logs, and RED-method (Rate, Errors, Duration) metrics from Istio
- Demonstrating the "single pane of glass" payoff: narrowing a metric anomaly down to the exact trace and failing downstream call

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
