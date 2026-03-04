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

## Contents

The repository is organized into modules, each focused on a specific EKS topic.

### Module 1: EKS Fundamentals and Networking Foundation

- Introduction to EKS  
- Usage models  
- Initial setup (IAM and S3 backend)  
- VPC design  
- Public and private subnets  
- NAT Gateways and Elastic IPs  
- Pod subnets  
- Database subnets and NACLs  
- Parameter Store integration  

### Module 2: Control Plane and Cluster Setup

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

### Module 3: Advanced Node Groups Strategies

- On-Demand and Spot Node Groups  
- Bottlerocket nodes  
- Graviton (ARM64) nodes  
- Workload segregation with Node Selectors  
- Node Affinity  
- Critical workloads segregation  
- Custom Launch Templates  
- Cluster Autoscaler  
- Node Termination Handler  

### Module 4: AWS Fargate

- Fargate fundamentals and Firecracker  
- Fargate profiles  
- Namespace selectors  
- Full Fargate clusters  
- CoreDNS fixes with Lambda  
- Application deployment on Fargate  
- Cloud Native deployment strategies  

### Module 5: Autoscaling with Karpenter (Part 1)

- Introduction to Karpenter  
- Helm installation  
- EC2NodeClasses and NodePools  
- Terraform production setup  
- Spot strategies  
- Workload segregation  
- Interruption queue handling  

### Module 6: Karpenter Groupless Architecture

- Production strategies  
- Fargate main profiles  
- Deleting traditional node groups  
- Groupless architecture model  

### Module 7: AWS Load Balancer Controller

- IAM setup and installation  
- Network Load Balancers  
- Ingress Class  
- HTTPS with ACM  
- External Load Balancer  
- Target Group Binding  
- Observability session  

### Module 8: NGINX Ingress Controller

- Introduction  
- Production hardening  
- Capacity and autoscaling  
- Target Group Binding  
- Multi-service deployments  

### Module 9: Storage in EKS

- CSI drivers overview  
- Pod Identity  
- EBS CSI  
- VolumeClaimTemplates  
- GP3 StorageClasses  
- EFS CSI  
- Shared volumes  
- S3 CSI driver  

### Module 10: External Secrets

- Introduction  
- Installation  
- AWS Secrets Manager integration  
- JSON multi-value secrets  
- AWS Parameter Store integration  
- Service Mesh session  

### Module 11: EKS Auto Mode

- Introduction  
- Automode deployment  
- NodePools in Automode  
- System workload segregation  
- Load Balancer integration  

### Module 12: Observability with Prometheus and Grafana

- kube-prometheus-stack  
- Installation with Karpenter  
- Prometheus deployment  
- Grafana exposure with NGINX  
- ServiceMonitors  
- EFS persistence  
- Retention tuning  
- EKS upgrades  

### Module 13: Service Mesh with Istio

- Introduction  
- Helm installation  
- Ingress Gateway production setup  
- Lab deployments  
- Prometheus monitors  
- Jaeger tracing  
- Kiali integration  
- Resilience strategies  
- External service mapping  

### Module 14: Event-Driven Autoscaling with KEDA

- Introduction  
- Helm installation  
- CPU autoscaling  
- Cron scaling  
- Prometheus TPS scaling  
- SQS consumer scaling  
- Fargate integration  

### Module 15: Progressive Delivery with Argo Rollouts

- Introduction  
- Helm installation  
- Canary releases  
- Manual and timed progression  
- Metric-based promotion (Prometheus)  
- Blue-Green deployments  
- Warm-up strategies  
- Fargate integration  

### Module 16: Helm Advanced Usage

- Helmify  
- Feature toggles  
- Services  
- Rollouts integration  
- TopologySpread  
- Istio integration  
- AnalysisTemplates  
- KEDA triggers  
- Packaging charts  

### Module 17: GitOps with ArgoCD

- Introduction to ArgoCD and ChartMuseum  
- ChartMuseum setup  
- ArgoCD installation  
- Dashboard exposure  
- ApplicationSets  
- Projects  
- Argo Rollouts integration  

### Module 18: Final Project — ArgoCD Multicluster Architecture

- Multicluster foundation  
- Shared Load Balancer (Active-Active)  
- HTTPS with ACM  
- Parameter Store  
- Multiple EKS clusters  
- Multicluster IAM authorization  
- ApplicationSets for components  
- ApplicationSets for workloads  
- ChartMuseum integration  

### Module 19: Final Project — Observability with Grafana Loki

- Introduction to Loki  
- Observability cluster foundation  
- Grafana dashboards  
- Loki setup  
- Network Load Balancer exposure  
- Fluent Bit log shipping  
- Capacity segregation

### Module 20: Final Project — Distributed Tracing with Grafana Tempo

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

### Module 21: Final Project — Centralized Metrics with Grafana Mimir

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

### Module 22: Final Project — Observability Correlation (Metrics, Logs and Traces)

- Introduction to Datasources  
- Metrics and Traces correlation (Service Maps with Tempo)  
- Logs and Traces correlation (Loki and Tempo)  
- Integrated Dashboard (Metrics x Logs x Traces)  
- Metrics Generator configuration  
- Loki configuration  
- Tempo configuration  
- Full Terraform locals setup  
- Example dashboard JSON configuration  

### Module 23: Chaos Engineering with Chaos Mesh

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
