# Module 12: Observability with Prometheus and Grafana

## Overview

Every module so far has been about getting workloads running and reachable. This module is about seeing what's actually happening inside the cluster: CPU and memory per node, request rates on the Ingress, and everything in between. The tool of choice is **Prometheus** — a metrics-scraping and time-series storage engine that has become one of the pillars of observability in the Kubernetes ecosystem — installed through **kube-prometheus-stack**, a community-maintained "umbrella" Helm chart that bundles Prometheus itself alongside the Prometheus Operator, kube-state-metrics, Alertmanager, Grafana, and Node Exporter as a single, cloud-native-friendly deployment.

The path through this module goes from a bare Helm install all the way to a genuinely production-shaped setup: scraping real application metrics via `ServiceMonitor` CRDs, exposing Grafana behind an Ingress with a real host, persisting both Prometheus's time-series data and Grafana's dashboards on EFS so a pod restart doesn't wipe them out, and finally segregating this whole observability stack onto its own dedicated Karpenter capacity so it never competes with application workloads for CPU and memory.

## Table of Contents

- [Lesson 1: Introduction to the Prometheus Stack](#lesson-1-introduction-to-the-prometheus-stack)
  - [1. What Prometheus Actually Does](#1-what-prometheus-actually-does)
  - [2. kube-prometheus-stack: an Umbrella Chart](#2-kube-prometheus-stack-an-umbrella-chart)
  - [3. Meet the Components](#3-meet-the-components)
  - [4. Service Discovery and the ServiceMonitor/PodMonitor CRDs](#4-service-discovery-and-the-servicemonitorpodmonitor-crds)
  - [Key Takeaways](#key-takeaways)
- [Lesson 2: Preparing the Cluster for Prometheus](#lesson-2-preparing-the-cluster-for-prometheus)
  - [1. Starting From the NGINX Controller Branch](#1-starting-from-the-nginx-controller-branch)
  - [2. Installing the Pod Identity Add-on](#2-installing-the-pod-identity-add-on)
  - [3. Removing the kube-system Fargate Profile](#3-removing-the-kube-system-fargate-profile)
  - [4. Working Around the CoreDNS/Karpenter Chicken-and-Egg Problem](#4-working-around-the-corednskarpenter-chicken-and-egg-problem)
  - [5. Installing the EFS CSI Driver and Its IAM Role](#5-installing-the-efs-csi-driver-and-its-iam-role)
  - [Key Takeaways](#key-takeaways-1)
- [Lesson 3: Installing kube-prometheus-stack with Helm](#lesson-3-installing-kube-prometheus-stack-with-helm)
  - [1. Why a Dedicated Values File Instead of set Blocks](#1-why-a-dedicated-values-file-instead-of-set-blocks)
  - [2. Enabling Only the Prometheus Server and Operator](#2-enabling-only-the-prometheus-server-and-operator)
  - [3. The helm_release Resource](#3-the-helm_release-resource)
  - [4. The Node Exporter Fargate Affinity Gotcha](#4-the-node-exporter-fargate-affinity-gotcha)
  - [Key Takeaways](#key-takeaways-2)
- [Lesson 4: Exposing Grafana](#lesson-4-exposing-grafana)
  - [1. Enabling Grafana in the Values File](#1-enabling-grafana-in-the-values-file)
  - [2. Default Credentials: admin / prom-operator](#2-default-credentials-admin--prom-operator)
  - [3. A First, Throwaway Ingress](#3-a-first-throwaway-ingress)
  - [4. Overriding the Admin Password](#4-overriding-the-admin-password)
  - [5. Productizing the Ingress with a Variable](#5-productizing-the-ingress-with-a-variable)
  - [Key Takeaways](#key-takeaways-3)
- [Lesson 5: Populating Dashboards with Real Metrics](#lesson-5-populating-dashboards-with-real-metrics)
  - [1. Deploying the health-api Lab](#1-deploying-the-health-api-lab)
  - [2. Importing Community Dashboards from Grafana Labs](#2-importing-community-dashboards-from-grafana-labs)
  - [3. The Missing Piece: No ServiceMonitor, No Metrics](#3-the-missing-piece-no-servicemonitor-no-metrics)
  - [4. Enabling a ServiceMonitor for the NGINX Ingress Controller](#4-enabling-a-servicemonitor-for-the-nginx-ingress-controller)
  - [5. Watching Metrics Flow In](#5-watching-metrics-flow-in)
  - [Key Takeaways](#key-takeaways-4)
- [Lesson 6: The Ephemeral Storage Problem](#lesson-6-the-ephemeral-storage-problem)
  - [1. Why Prometheus and Grafana Lose Data on Restart](#1-why-prometheus-and-grafana-lose-data-on-restart)
  - [2. Choosing EFS for Persistent Storage](#2-choosing-efs-for-persistent-storage)
  - [3. A Security Group for EFS Traffic](#3-a-security-group-for-efs-traffic)
  - [4. Filesystems and Mount Targets for Prometheus and Grafana](#4-filesystems-and-mount-targets-for-prometheus-and-grafana)
  - [5. StorageClasses Backed by the EFS CSI Driver](#5-storageclasses-backed-by-the-efs-csi-driver)
  - [Key Takeaways](#key-takeaways-5)
- [Lesson 7: Mounting Persistent Storage for Prometheus](#lesson-7-mounting-persistent-storage-for-prometheus)
  - [1. The storageSpec.volumeClaimTemplate Key](#1-the-storagespecvolumeclaimtemplate-key)
  - [2. Applying and Verifying the PVC](#2-applying-and-verifying-the-pvc)
  - [3. Testing Metric Persistence Across Pod Restarts](#3-testing-metric-persistence-across-pod-restarts)
  - [Key Takeaways](#key-takeaways-6)
- [Lesson 8: Mounting Persistent Storage for Grafana](#lesson-8-mounting-persistent-storage-for-grafana)
  - [1. The grafana.persistence Keys](#1-the-grafanapersistence-keys)
  - [2. Disabling initChownData](#2-disabling-initchowndata)
  - [3. Testing Dashboard Persistence Across Pod Restarts](#3-testing-dashboard-persistence-across-pod-restarts)
  - [Key Takeaways](#key-takeaways-7)
- [Lesson 9: Segregating Capacity with a Dedicated NodePool](#lesson-9-segregating-capacity-with-a-dedicated-nodepool)
  - [1. Why Prometheus and Grafana Need Their Own Capacity](#1-why-prometheus-and-grafana-need-their-own-capacity)
  - [2. Creating a Spot-Only prometheus NodePool](#2-creating-a-spot-only-prometheus-nodepool)
  - [3. Targeting the NodePool via nodeSelector](#3-targeting-the-nodepool-via-nodeselector)
  - [4. Why the Node Exporter Is Left Out](#4-why-the-node-exporter-is-left-out)
  - [Key Takeaways](#key-takeaways-8)
- [Lesson 10: Tuning Metric Retention](#lesson-10-tuning-metric-retention)
  - [1. The prometheusSpec.retention Key](#1-the-prometheusspecretention-key)
  - [Key Takeaways](#key-takeaways-9)

# Lesson 1: Introduction to the Prometheus Stack

## 1. What Prometheus Actually Does

Prometheus is a monitoring and observability tool built around one core job: **scraping metrics**. Applications, services, and cluster components expose metrics at regular intervals on an HTTP endpoint; Prometheus periodically fetches ("scrapes") those endpoints and stores the results in a time-series database, indexed by time. Once collected, that data becomes queryable — the foundation for dashboards, alerts, and everything else built on top of it.

By itself, Prometheus only collects and stores. Its real strength comes from the ecosystem built around it, which is exactly what this module explores.

## 2. kube-prometheus-stack: an Umbrella Chart

**kube-prometheus-stack** is the Helm chart this module uses to install the Prometheus ecosystem in a cloud-native, Kubernetes-friendly way. It's maintained by the Prometheus community and works as an "umbrella" chart — a chart of charts — that installs and wires together several specialized sub-charts instead of being a single monolithic install.

## 3. Meet the Components

kube-prometheus-stack manages six components, each toggled independently:

- **Prometheus Server** — the core component: scrapes and stores metrics as time series.
- **Prometheus Operator** — manages the Prometheus/Alertmanager lifecycle and introduces the `ServiceMonitor`/`PodMonitor` CRDs used to declare what gets scraped.
- **kube-state-metrics** — an additional metrics aggregator that exposes cluster-object state (Deployments, nodes, etc.) as Prometheus metrics.
- **Alertmanager** — connects to Prometheus, evaluates alerting thresholds, and dispatches notifications (Slack, webhooks, and other integrations).
- **Grafana** — connects to Prometheus as a data source and turns its time series into dashboards and visualizations.
- **Node Exporter** — a per-node exporter that reports CPU, memory, network, and disk metrics for every node in the cluster, which Prometheus in turn scrapes.

More generally, **exporters** are the pattern behind Node Exporter: small dedicated agents that expose a service's internal metrics — CPU, memory, disk, or something domain-specific like MongoDB, Elasticsearch, Kafka, or JVM/JMX metrics for Java applications — on an endpoint Prometheus knows how to scrape.

## 4. Service Discovery and the ServiceMonitor/PodMonitor CRDs

In a container environment, workloads come and go constantly, so hardcoding scrape targets doesn't scale. Prometheus solves this with **service discovery**: rules that tell it which pods expose metrics, and on which endpoint and port to fetch them.

The Prometheus Operator formalizes this into two CRDs:

- **ServiceMonitor** — the standard way to describe how a Service (and the pods behind it) should be scraped. This covers the overwhelming majority of cases.
- **PodMonitor** — for scraping a pod directly when there's no intermediate Service/Deployment to attach a ServiceMonitor to.

Both are, in effect, recipes that tell Prometheus how a given workload wants to be monitored — no manual target configuration required.

## Key Takeaways

- Prometheus scrapes metrics from exposed endpoints on a schedule and stores them as time series for later querying.
- **kube-prometheus-stack** is an umbrella Helm chart bundling Prometheus Server, Prometheus Operator, kube-state-metrics, Alertmanager, Grafana, and Node Exporter.
- Exporters (like Node Exporter) are dedicated agents that expose a service's internal state as scrapeable metrics.
- `ServiceMonitor` and `PodMonitor`, introduced by the Prometheus Operator, declare what to scrape without hardcoding targets — `ServiceMonitor` covers nearly every case.

---

# Lesson 2: Preparing the Cluster for Prometheus

## 1. Starting From the NGINX Controller Branch

Exposing Grafana requires an Ingress, so this module builds directly on top of the NGINX Ingress Controller branch rather than starting fresh — Karpenter, the AWS Load Balancer Controller, and the Fargate profiles from that lesson are all already in place and get reused as-is.

## 2. Installing the Pod Identity Add-on

Two components from the earlier CSI lesson need to be brought over, since this branch didn't have them yet: the **Pod Identity** add-on and its version variable.

```hcl
# addons.tf
resource "aws_eks_addon" "pod_identity" {
  cluster_name = aws_eks_cluster.main.name
  addon_name   = "eks-pod-identity-agent"

  addon_version               = var.addon_pod_identity_version
  resolve_conflicts_on_create = "OVERWRITE"
  resolve_conflicts_on_update = "OVERWRITE"

  depends_on = [
    aws_eks_access_entry.nodes
  ]
}
```

## 3. Removing the kube-system Fargate Profile

The branch this module inherits from provisions two Fargate profiles: one for `kube-system` and one for `karpenter`. The EFS CSI driver's components can't run on Fargate, so the `kube-system` profile is removed, keeping only the `karpenter` one — along with the CoreDNS-on-Fargate annotation-stripping Lambda from the CSI lesson, which is no longer needed once `kube-system` moves off Fargate.

## 4. Working Around the CoreDNS/Karpenter Chicken-and-Egg Problem

Removing the `kube-system` Fargate profile introduces a circular dependency: Karpenter depends on CoreDNS to resolve AWS API names, but CoreDNS's pods now need *some* node to run on, and without CoreDNS resolving names, Karpenter can't reliably come up to provision that node. With two separate Fargate profiles this wasn't an issue, but consolidating onto Karpenter-only capacity creates a bootstrap loop.

The workaround is temporary and simple: stand up a minimal, plain `aws_eks_node_group` (the same self-managed node group pattern used in earlier modules, with a single node) purely to give `kube-system` somewhere to boot from. Once the cluster is up and Karpenter is healthy, that node group is disposable and can be deleted.

## 5. Installing the EFS CSI Driver and Its IAM Role

With capacity issues resolved, the EFS CSI driver is installed the same way it was in the dedicated CSI module: a Pod Identity-based IAM role, and the `aws-efs-csi-driver` EKS add-on itself.

```hcl
# iam_efs_csi.tf
data "aws_iam_policy_document" "efs_role" {
  statement {
    effect = "Allow"

    principals {
      type        = "Service"
      identifiers = ["pods.eks.amazonaws.com"]
    }

    actions = [
      "sts:AssumeRole",
      "sts:TagSession"
    ]
  }
}

resource "aws_iam_role" "efs_role" {
  assume_role_policy = data.aws_iam_policy_document.efs_role.json
  name               = format("%s-efs-csi-role", var.project_name)
}

resource "aws_iam_role_policy_attachment" "efs_csi_role" {
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonEFSCSIDriverPolicy"
  role       = aws_iam_role.efs_role.name
}

resource "aws_eks_pod_identity_association" "efs_csi" {
  cluster_name    = aws_eks_cluster.main.name
  namespace       = "kube-system"
  service_account = "efs-csi-controller-sa"
  role_arn        = aws_iam_role.efs_role.arn
}
```

```hcl
# addons.tf
resource "aws_eks_addon" "efs_csi" {
  cluster_name = aws_eks_cluster.main.name
  addon_name   = "aws-efs-csi-driver"

  addon_version               = var.addon_efs_csi_version
  resolve_conflicts_on_create = "OVERWRITE"
  resolve_conflicts_on_update = "OVERWRITE"

  depends_on = [
    aws_eks_access_entry.nodes
  ]
}
```

Applying this brings up the EFS CSI controller and node DaemonSet inside `kube-system`, which is the storage foundation the later lessons on persistence depend on.

## Key Takeaways

- This module's branch is built on top of the NGINX Ingress Controller branch, reusing its Karpenter, ALB Controller, and Fargate setup.
- The Pod Identity add-on is brought forward from the CSI module, since it wasn't part of the NGINX branch.
- The `kube-system` Fargate profile is removed because the EFS CSI driver can't run on Fargate — this creates a temporary CoreDNS/Karpenter bootstrap loop, solved with a disposable plain node group.
- The EFS CSI driver is installed exactly as in the dedicated CSI module: a Pod Identity IAM role plus the `aws-efs-csi-driver` EKS add-on.

---

# Lesson 3: Installing kube-prometheus-stack with Helm

## 1. Why a Dedicated Values File Instead of set Blocks

Every Helm install so far in this course has used inline `set` blocks inside the `helm_release` resource. kube-prometheus-stack's configuration surface is large and deeply nested enough that this stops being practical — so instead, its configuration is read from a dedicated values file that doesn't change between environments.

## 2. Enabling Only the Prometheus Server and Operator

kube-prometheus-stack's values work like feature toggles: every sub-component has its own `enabled` key. The install starts minimal — only the Prometheus server and its Operator — with kube-state-metrics, Alertmanager, and Grafana all switched off, to be enabled one at a time in later lessons:

```yaml
# helm/prometheus/values.yml
prometheus:
  enabled: true
  prometheusSpec:

    podMonitorSelector: {}
    podMonitorSelectorNilUsesHelmValues: false
    ruleSelector: {}
    ruleSelectorNilUsesHelmValues: false
    serviceMonitorSelector: {}
    serviceMonitorSelectorNilUsesHelmValues: false
    scrapeConfigSelectorNilUsesHelmValues: false

kubeStateMetrics:
  enabled: false

alertmanager:
  enabled: false

grafana:
  enabled: false

prometheusOperator:
  enabled: true
```

The four `*SelectorNilUsesHelmValues: false` keys (plus the empty `podMonitorSelector`/`ruleSelector`/`serviceMonitorSelector` maps) come straight from kube-prometheus-stack's own documented recommendations, and tell the Operator to pick up `ServiceMonitor`/`PodMonitor`/`PrometheusRule` objects from **any** namespace, not just the release namespace — otherwise it would only discover monitors it created itself.

## 3. The helm_release Resource

```hcl
# helm_prometheus.tf
resource "helm_release" "prometheus" {

  name             = "prometheus"
  chart            = "kube-prometheus-stack"
  repository       = "https://prometheus-community.github.io/helm-charts"
  namespace        = "prometheus"
  create_namespace = true

  version = "69.3.2"

  values = [
    "${file("./helm/prometheus/values.yml")}"
  ]

  depends_on = [
    aws_eks_cluster.main,
    helm_release.karpenter
  ]
}
```

`values` reads the file created in the previous step instead of building a long list of `set` entries, and `depends_on` ensures Karpenter is already installed so the Prometheus workloads have somewhere to schedule.

## 4. The Node Exporter Fargate Affinity Gotcha

Node Exporter runs as a DaemonSet, which Fargate can't run at all. Depending on the chart version, kube-prometheus-stack's Node Exporter pods may still attempt to schedule on Fargate-backed capacity and fail. The fix is a `nodeAffinity` rule excluding any node annotated as a Fargate compute type:

```yaml
# helm/prometheus/values.yml
prometheus-node-exporter:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
          - matchExpressions:
              - key: "eks.amazonaws.com/compute-type"
                operator: "NotIn"
                values:
                  - "fargate"
```

Newer chart versions don't always hit this, but it's worth adding proactively if the Node Exporter DaemonSet gets stuck pending.

## Key Takeaways

- kube-prometheus-stack's configuration is large enough that a dedicated values file (read via `file()`) is more practical than inline Helm `set` blocks.
- The install starts with only `prometheus` and `prometheusOperator` enabled; `kubeStateMetrics`, `alertmanager`, and `grafana` are switched on individually in later lessons.
- The `*SelectorNilUsesHelmValues: false` keys are needed so the Operator discovers `ServiceMonitor`/`PodMonitor`/`PrometheusRule` objects across all namespaces, not just its own.
- Node Exporter, being a DaemonSet, can't run on Fargate — a `nodeAffinity` excluding `eks.amazonaws.com/compute-type: fargate` nodes avoids pods stuck pending on some chart versions.

---

# Lesson 4: Exposing Grafana

## 1. Enabling Grafana in the Values File

Grafana is switched on the same way every other component was — flipping its `enabled` feature toggle to `true` and applying:

```yaml
# helm/prometheus/values.yml
grafana:
  enabled: true
```

## 2. Default Credentials: admin / prom-operator

Out of the box, without any credentials configured in the values file, kube-prometheus-stack's Grafana ships with the login `admin` / `prom-operator`.

## 3. A First, Throwaway Ingress

kube-prometheus-stack does support configuring an Ingress directly inside its own values, but — consistent with how Ingress has been handled in every other module — it's created as a separate, plain Kubernetes manifest instead, pointing at the `prometheus-grafana` Service on port 80:

```yaml
# grafana-ingress-temp.yml
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana-ingress
  namespace: prometheus
  annotations:
spec:
  ingressClassName: nginx
  rules:
    - host: grafana.msfidelis.com.br
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: prometheus-grafana
                port:
                  number: 80
---
```

Since this branch inherits the NGINX lesson's Network Load Balancer, resolving the chosen hostname to that NLB's address (a local `/etc/hosts` entry works for testing; a real DNS zone is the production equivalent) is enough to reach Grafana's login page.

## 4. Overriding the Admin Password

The default `admin` / `prom-operator` login can be replaced by setting explicit values under the `grafana` key:

```yaml
# helm/prometheus/values.yml
grafana:
  enabled: true
  adminUser: admin
  adminPassword: linuxtips
```

## 5. Productizing the Ingress with a Variable

The throwaway Ingress from earlier in this lesson is deleted and replaced with a `kubectl_manifest` resource defined alongside the Helm release itself, with the hostname parametrized through a new `grafana_host` variable instead of hardcoded:

```hcl
# variables.tf
variable "grafana_host" {
  type        = string
  default     = "grafana.msfidelis.com.br"
  description = "Host do Grafana"
}
```

```hcl
# helm_prometheus.tf
resource "kubectl_manifest" "grafana_host" {
  yaml_body = <<YAML
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: grafana-ingress
  namespace: prometheus
  annotations:
spec:
  ingressClassName: nginx
  rules:
    - host: ${var.grafana_host}
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: prometheus-grafana
                port:
                  number: 80
YAML
  depends_on = [
    helm_release.prometheus,
    helm_release.nginx_controller
  ]

}
```

## Key Takeaways

- Grafana is enabled the same way as every other kube-prometheus-stack component: an `enabled: true` feature toggle.
- Without an explicit `adminUser`/`adminPassword`, Grafana's default login is `admin` / `prom-operator`.
- The Ingress is a plain, standalone Kubernetes manifest rather than the chart's built-in Ingress support — consistent with how Ingress is handled everywhere else in the course.
- The final version parametrizes the hostname through a `grafana_host` Terraform variable instead of a hardcoded value, and depends on both the Prometheus release and the NGINX controller.

---

# Lesson 5: Populating Dashboards with Real Metrics

## 1. Deploying the health-api Lab

To generate real traffic and metrics, this lesson reuses the exact same lab application from the AWS Load Balancer Controller Ingress module: `health-api` and its downstream gRPC services (`recommendations`, `bmr`, `imc`, `calories`, `proteins`, `water`), each already annotated for Prometheus scraping:

```yaml
# health-api.yml
apiVersion: v1
kind: Namespace
metadata:
  name: health-api
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: health-api
  namespace: health-api
spec:
  ingressClassName: nginx
  rules:
    - host: health.msfidelis.com.br
      http:
        paths:
          - pathType: Prefix
            backend:
              service:
                name: health-api
                port:
                  number: 8080
            path: /
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: health-api
  name: health-api
  namespace: health-api
spec:
  replicas: 2
  selector:
    matchLabels:
      app: health-api
  template:
    metadata:
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
      labels:
        app: health-api
        name: health-api
        version: v1
    spec:
      containers:
      - image: fidelissauro/health-api:latest
        name: health-api
        ports:
        - containerPort: 8080
          name: http
      terminationGracePeriodSeconds: 60
---
apiVersion: v1
kind: Service
metadata:
  name: health-api
  namespace: health-api
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "8080"
  labels:
    app.kubernetes.io/name: health-api
    app.kubernetes.io/instance: health-api
spec:
  ports:
  - name: web
    port: 8080
    protocol: TCP
  selector:
    app: health-api
  type: ClusterIP
```

The rest of the manifest — the six downstream gRPC microservices — follows the identical Namespace/Deployment/Service pattern and is unchanged from the Ingress module; it's only reused here as background load, not new content for this lesson.

Once applied and resolvable via the same `/etc/hosts` trick used for Grafana, a `curl` loop against `health.msfidelis.com.br` keeps the application generating steady traffic to observe.

## 2. Importing Community Dashboards from Grafana Labs

Rather than building dashboards from scratch, **Grafana Labs** hosts a large public library of community dashboards, importable by ID or by uploading their JSON definition — for example, the "Kubernetes / NGINX Ingress Controller" dashboard. Grafana's **New → Import** screen accepts either the raw JSON or a direct upload, then asks which Prometheus data source to bind the dashboard to.

## 3. The Missing Piece: No ServiceMonitor, No Metrics

Importing the NGINX dashboard at this point shows no data at all. Prometheus already auto-discovers `ServiceMonitor`s for components it installed itself — Node Exporter, its own server, CoreDNS — which is why those already have populated dashboards. NGINX has none yet, because nothing has told Prometheus where to scrape it from.

## 4. Enabling a ServiceMonitor for the NGINX Ingress Controller

The ingress-nginx Helm chart has its own values for exposing a metrics endpoint and registering a `ServiceMonitor` for it — `controller.metrics.enabled`, `controller.metrics.serviceMonitor.enabled`, and a `podAnnotations` block matching the community-recommended scrape annotations:

```hcl
# helm_nginx.tf
set = [
  {
    name  = "controller.metrics.enabled"
    value = "true"
  },
  {
    name  = "controller.metrics.serviceMonitor.enabled"
    value = "true"
  },
  {
    name  = "controller.podAnnotations.prometheus\\.io/scrape"
    value = "true"
  },
  {
    name  = "controller.podAnnotations.prometheus\\.io/port"
    value = "10254"
  }
]
```

Applying this creates a new `ingress-nginx-controller-metrics` Service on port `10254` and a matching `ingress-controller` `ServiceMonitor` object that tells Prometheus exactly where to scrape NGINX metrics from.

## 5. Watching Metrics Flow In

After the new `ServiceMonitor` is picked up (recycling the Prometheus server pod speeds this up), NGINX metrics start showing up in Prometheus's own metrics explorer, and the previously-empty NGINX dashboard in Grafana starts filling in — request counts, success rates, and connection counts all populate within a few minutes. The same exercise works for any dashboard on Grafana Labs, including general-purpose ones like "Kubernetes cluster monitoring," which is worth exploring for ideas on what else is worth tracking.

## Key Takeaways

- The `health-api` lab reused here is identical to the one from the ALB Controller Ingress module — it exists purely to generate traffic and Prometheus-annotated metrics.
- Grafana Labs hosts a large public library of importable community dashboards, bound to a chosen Prometheus data source at import time.
- A dashboard with no matching `ServiceMonitor` shows no data — Prometheus only auto-discovers monitors for components it installed itself.
- ingress-nginx's own chart exposes `controller.metrics.enabled` / `controller.metrics.serviceMonitor.enabled` / `podAnnotations` values to register itself for scraping — after which its dashboards populate within minutes.

---

# Lesson 6: The Ephemeral Storage Problem

## 1. Why Prometheus and Grafana Lose Data on Restart

Everything installed so far stores data effectively in memory: deleting the Prometheus server pod brings it back with an empty local store, so previously-scraped metrics are gone; deleting the Grafana pod brings it back with the default dashboards only, and any imported or hand-built dashboard disappears. Neither component has any persistence configured yet.

## 2. Choosing EFS for Persistent Storage

The fix is mounting a persistent volume into both components. **EFS** is the choice made here — the same CSI driver installed back in Lesson 2 — because it mounts cleanly across every Availability Zone and is simple to manage, though any other CSI (EBS, S3-compatible options, etc.) would work just as well for this same purpose.

## 3. A Security Group for EFS Traffic

A dedicated security group opens NFS traffic (port 2049) for the EFS mount targets:

```hcl
# sg_efs.tf
resource "aws_security_group" "efs" {
  name   = format("%s-efs", var.project_name)
  vpc_id = data.aws_ssm_parameter.vpc.value

  ingress {
    from_port   = 2049
    to_port     = 2049
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

## 4. Filesystems and Mount Targets for Prometheus and Grafana

Two separate EFS filesystems are created — one per component, so their data stays independent — each mounted into every pod subnet:

```hcl
# efs_prometheus.tf
resource "aws_efs_file_system" "prometheus" {
  creation_token   = format("%s-efs-prometheus", var.project_name)
  performance_mode = "generalPurpose"

  tags = {
    Name = format("%s-efs-prometheus", var.project_name)
  }
}

resource "aws_efs_mount_target" "prometheus" {
  count = length(data.aws_ssm_parameter.pod_subnets)

  file_system_id = aws_efs_file_system.prometheus.id
  subnet_id      = data.aws_ssm_parameter.pod_subnets[count.index].value
  security_groups = [
    aws_security_group.efs.id
  ]
}
```

```hcl
# efs_grafana.tf
resource "aws_efs_file_system" "grafana" {
  creation_token   = format("%s-efs-grafana", var.project_name)
  performance_mode = "generalPurpose"

  tags = {
    Name = format("%s-efs-grafana", var.project_name)
  }
}

resource "aws_efs_mount_target" "grafana" {
  count = length(data.aws_ssm_parameter.pod_subnets)

  file_system_id = aws_efs_file_system.grafana.id
  subnet_id      = data.aws_ssm_parameter.pod_subnets[count.index].value
  security_groups = [
    aws_security_group.efs.id
  ]
}
```

## 5. StorageClasses Backed by the EFS CSI Driver

Each filesystem gets its own `StorageClass`, matching the pattern established in the dedicated CSI module, with a `Retain` reclaim policy so the underlying EFS data survives even if the `PersistentVolumeClaim` is deleted:

```hcl
# efs_prometheus.tf
resource "kubectl_manifest" "prometheus_efs_storage_class" {
  yaml_body = <<YAML
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-prometheus
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: ${aws_efs_file_system.prometheus.id}
  directoryPerms: "777"
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
YAML

  depends_on = [
    aws_eks_cluster.main,
    helm_release.karpenter
  ]
}
```

```hcl
# efs_grafana.tf
resource "kubectl_manifest" "grafana_efs_storage_class" {
  yaml_body = <<YAML
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: efs-grafana
provisioner: efs.csi.aws.com
parameters:
  provisioningMode: efs-ap
  fileSystemId: ${aws_efs_file_system.grafana.id}
  directoryPerms: "777"
reclaimPolicy: Retain
volumeBindingMode: WaitForFirstConsumer
YAML

  depends_on = [
    aws_eks_cluster.main,
    helm_release.karpenter
  ]
}
```

With `efs-prometheus` and `efs-grafana` both showing up under `kubectl get storageclass`, the cluster is ready to wire actual persistence into each Helm-managed component.

## Key Takeaways

- Without persistence, both the Prometheus server and Grafana lose their data (metrics and dashboards, respectively) on every pod restart.
- Two independent EFS filesystems are used — one for Prometheus, one for Grafana — each with mount targets in every pod subnet, sharing one security group opened on port 2049.
- Each filesystem gets its own `StorageClass` (`efs-prometheus`, `efs-grafana`) with `reclaimPolicy: Retain`, so data survives PVC deletion.
- This is the same EFS CSI pattern used in the dedicated CSI module, just applied twice for two independent volumes.

---

# Lesson 7: Mounting Persistent Storage for Prometheus

## 1. The storageSpec.volumeClaimTemplate Key

`prometheusSpec` exposes a `storageSpec.volumeClaimTemplate` key that lets the chart provision a `PersistentVolumeClaim` using any `StorageClass` already available in the cluster — in this case, `efs-prometheus`:

```yaml
# helm/prometheus/values.yml
prometheus:
  enabled: true
  prometheusSpec:

    # Storage
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: "efs-prometheus"
          accessModes:
            - ReadWriteMany
          resources:
            requests:
              storage: 50Gi
```

`ReadWriteMany` is required (rather than `ReadWriteOnce`) because EFS is the backing filesystem — the same access mode used throughout the CSI module.

## 2. Applying and Verifying the PVC

Applying this recycles the Prometheus server pod and mounts the new volume. `kubectl describe pod` on the Prometheus server shows the volume mounted with a claim name referencing a PVC in the same namespace, and `kubectl get pvc` confirms it's bound to the `efs-prometheus` StorageClass.

## 3. Testing Metric Persistence Across Pod Restarts

With the volume mounted, deleting the Prometheus server pod and waiting for it to come back no longer wipes its data — re-opening a previously-imported dashboard (like the NGINX Ingress one from Lesson 5) shows the same metrics still there, because they're now being written to a persistent EFS-backed volume instead of ephemeral local storage.

## Key Takeaways

- `prometheusSpec.storageSpec.volumeClaimTemplate` provisions a PVC against any existing `StorageClass`, here `efs-prometheus`.
- `ReadWriteMany` is required for an EFS-backed volume, unlike the `ReadWriteOnce` typical of EBS.
- After applying, deleting the Prometheus server pod no longer loses previously-scraped metrics — they now live on a persistent volume.

---

# Lesson 8: Mounting Persistent Storage for Grafana

## 1. The grafana.persistence Keys

Grafana's persistence is configured under its own `persistence` key: enabling it, pointing at the `efs-grafana` StorageClass, and setting a size:

```yaml
# helm/prometheus/values.yml
grafana:
  enabled: true
  adminUser: admin
  adminPassword: linuxtips

  persistence:
    enabled: true
    storageClassName: "efs-grafana"
    accessModes:
      - ReadWriteMany
    size: 10Gi
```

## 2. Disabling initChownData

Grafana's chart normally runs an init container that `chown`s the mounted storage path so the Grafana process can write to it — a step that fails against EFS, since the pod doesn't have permission to change ownership across the whole filesystem this way. Disabling it avoids the failure:

```yaml
# helm/prometheus/values.yml
grafana:
  initChownData:
    enabled: false
```

## 3. Testing Dashboard Persistence Across Pod Restarts

Applying this recycles the Grafana pod, and the first restart still loses the previously-imported dashboards — expected, since they were only ever in the old ephemeral storage. Re-adding the data source and re-importing the dashboards once populates the new EFS-backed volume; from that point on, deleting the Grafana pod (or even deleting every Prometheus-namespace pod at once, Prometheus included) and waiting for everything to come back leaves both the dashboards and the underlying metrics intact.

## Key Takeaways

- `grafana.persistence` (`enabled`, `storageClassName: efs-grafana`, `accessModes`, `size`) wires Grafana's dashboards onto a persistent EFS-backed volume, mirroring `prometheusSpec.storageSpec` from the previous lesson.
- `grafana.initChownData.enabled` must be set to `false` — the chart's default ownership-fixing init container isn't compatible with EFS permissions.
- The very first restart after enabling persistence still loses old dashboards (they were never on the new volume); every restart after that preserves them.

---

# Lesson 9: Segregating Capacity with a Dedicated NodePool

## 1. Why Prometheus and Grafana Need Their Own Capacity

The Prometheus stack — particularly the Prometheus server itself — tends to carry meaningful CPU and memory overhead. Left unsegregated, it competes for capacity with application workloads on shared NodePools, which is worth avoiding given how many example NodePools this course has already created via Karpenter.

## 2. Creating a Spot-Only prometheus NodePool

A new NodePool/EC2NodeClass pair named `prometheus` is added through the same `karpenter_capacity` Terraform mechanism used throughout the Karpenter modules — restricted to the `c7a` instance family, `large` size only, and Spot capacity exclusively, since this observability capacity doesn't need On-Demand guarantees. Applying it produces a dedicated `prometheus` NodePool, visible alongside every other example NodePool already in the cluster.

## 3. Targeting the NodePool via nodeSelector

Segregation is enforced the same way it has been in every other Karpenter module: a `nodeSelector` on `karpenter.sh/nodepool` pointing at the new NodePool's name. It's added to `prometheusSpec`, `prometheusOperator`, and `grafana` — every component of the stack except Node Exporter:

```yaml
# helm/prometheus/values.yml
prometheus:
  enabled: true
  prometheusSpec:
    nodeSelector:
      karpenter.sh/nodepool: "prometheus"

grafana:
  enabled: true
  nodeSelector:
    karpenter.sh/nodepool: "prometheus"

prometheusOperator:
  enabled: true
  nodeSelector:
    karpenter.sh/nodepool: "prometheus"
```

Applying this triggers a new `c7a.large` Spot NodeClaim on the `prometheus` NodePool; once it's ready, `kubectl describe node` on it shows exactly the Prometheus server, Prometheus Operator, and Grafana pods scheduled there — and nothing else.

## 4. Why the Node Exporter Is Left Out

Node Exporter is deliberately **not** pointed at the `prometheus` NodePool. As a DaemonSet, it needs to run on every node in the cluster to report per-node metrics — pinning it to a single dedicated NodePool would defeat its entire purpose.

## Key Takeaways

- A dedicated `prometheus` NodePool (`c7a.large`, Spot-only) is created through the same Terraform mechanism used for every other Karpenter NodePool in this course.
- `nodeSelector: {karpenter.sh/nodepool: "prometheus"}` is applied to `prometheusSpec`, `prometheusOperator`, and `grafana` to pin them off of shared application capacity.
- Node Exporter is intentionally excluded from this segregation — as a DaemonSet, it must run on every node to report per-node metrics.

---

# Lesson 10: Tuning Metric Retention

## 1. The prometheusSpec.retention Key

By default, Prometheus retains scraped metrics for 10 days before discarding older data, which keeps storage usage from growing indefinitely. That window is configurable via `prometheusSpec.retention`:

```yaml
# helm/prometheus/values.yml
prometheus:
  enabled: true
  prometheusSpec:
    retention: 15d
```

Raising or lowering this value directly controls how far back metrics stay queryable inside the cluster, trading off against the storage capacity reserved in the `volumeClaimTemplate` from Lesson 7.

## Key Takeaways

- Prometheus retains metrics for 10 days by default; `prometheusSpec.retention` (e.g. `15d`) adjusts that window directly.
- Longer retention trades directly against the storage size reserved for the Prometheus server's persistent volume.
