# Module 14: Event-Driven Autoscaling with KEDA

## Overview

Karpenter and the Cluster Autoscaler, covered in earlier modules, scale infrastructure: they add or remove nodes based on whether existing pods fit. KEDA (Kubernetes Event-Driven Autoscaling) operates one layer up — it scales the pods themselves, based on signals that have nothing to do with a pod's own CPU or memory. A queue depth, a cron schedule, a Prometheus query, a CloudWatch metric: KEDA turns any of these into a Horizontal Pod Autoscaler target, and Karpenter still does its job underneath, provisioning whatever capacity the new pod count needs. This module installs KEDA via Pod Identity and Terraform, fixes a metrics gap left over from the Istio module, and then walks through three real scaling dimensions on the `chip` lab application — CPU, cron, and Prometheus-based TPS — before finishing with a fourth, more advanced example: scaling an SQS consumer that offloads data into DynamoDB for the `nutrition` namespace. It closes by giving KEDA itself a dedicated Fargate profile, the same way Karpenter already has one, so the autoscaler's own control plane doesn't depend on the nodes it's responsible for managing.

## Table of Contents

- [Lesson 1: What KEDA Adds to the Autoscaling Story](#lesson-1-what-keda-adds-to-the-autoscaling-story)
  - [1. Beyond Node-Level Autoscaling](#1-beyond-node-level-autoscaling)
  - [2. The Scaler Ecosystem](#2-the-scaler-ecosystem)
  - [Key Takeaways](#key-takeaways)
- [Lesson 2: Installing KEDA](#lesson-2-installing-keda)
  - [1. Branching from the Istio Lab](#1-branching-from-the-istio-lab)
  - [2. Granting AWS Permissions with Pod Identity](#2-granting-aws-permissions-with-pod-identity)
  - [3. Declaring the KEDA Version](#3-declaring-the-keda-version)
  - [4. Converting the Helm Install into a Terraform helm_release](#4-converting-the-helm-install-into-a-terraform-helm_release)
  - [5. Verifying the Installation](#5-verifying-the-installation)
  - [Key Takeaways](#key-takeaways-1)
- [Lesson 3: Fixing Metrics and Preparing the Lab Application](#lesson-3-fixing-metrics-and-preparing-the-lab-application)
  - [1. Importing a Kubernetes Cluster Dashboard](#1-importing-a-kubernetes-cluster-dashboard)
  - [2. Diagnosing Missing Metrics](#2-diagnosing-missing-metrics)
  - [3. Enabling kube-state-metrics](#3-enabling-kube-state-metrics)
  - [4. Exposing metrics-server Through a ServiceMonitor](#4-exposing-metrics-server-through-a-servicemonitor)
  - [5. Redeploying the chip Lab Application](#5-redeploying-the-chip-lab-application)
  - [6. A CPU-Intensive Endpoint for Testing](#6-a-cpu-intensive-endpoint-for-testing)
  - [Key Takeaways](#key-takeaways-2)
- [Lesson 4: Scaling on CPU Usage](#lesson-4-scaling-on-cpu-usage)
  - [1. Generating a CPU Spike](#1-generating-a-cpu-spike)
  - [2. Visualizing Pods and CPU Usage](#2-visualizing-pods-and-cpu-usage)
  - [3. Anatomy of a ScaledObject](#3-anatomy-of-a-scaledobject)
  - [4. How KEDA Manages the Underlying HPA](#4-how-keda-manages-the-underlying-hpa)
  - [5. Watching the Cluster Scale](#5-watching-the-cluster-scale)
  - [Key Takeaways](#key-takeaways-3)
- [Lesson 5: Scaling on a Schedule with Cron](#lesson-5-scaling-on-a-schedule-with-cron)
  - [1. Why Scale Preventively](#1-why-scale-preventively)
  - [2. Superseding the CPU ScaledObject](#2-superseding-the-cpu-scaledobject)
  - [3. Anatomy of a Cron Trigger](#3-anatomy-of-a-cron-trigger)
  - [4. How KEDA and Karpenter Coordinate on a Schedule](#4-how-keda-and-karpenter-coordinate-on-a-schedule)
  - [Key Takeaways](#key-takeaways-4)
- [Lesson 6: Scaling on Requests Per Second with Prometheus](#lesson-6-scaling-on-requests-per-second-with-prometheus)
  - [1. Generating Realistic Load with k6](#1-generating-realistic-load-with-k6)
  - [2. Visualizing Request Rate](#2-visualizing-request-rate)
  - [3. Anatomy of the Prometheus Trigger](#3-anatomy-of-the-prometheus-trigger)
  - [4. Sizing the Threshold Per Pod](#4-sizing-the-threshold-per-pod)
  - [5. Why TPS-Based Scaling Beats CPU for Synchronous Workloads](#5-why-tps-based-scaling-beats-cpu-for-synchronous-workloads)
  - [Key Takeaways](#key-takeaways-5)
- [Lesson 7: Scaling an SQS Consumer](#lesson-7-scaling-an-sqs-consumer)
  - [1. A New Health API Version](#1-a-new-health-api-version)
  - [2. Provisioning the SQS Queue and DynamoDB Table](#2-provisioning-the-sqs-queue-and-dynamodb-table)
  - [3. Granting AWS Permissions to the nutrition Namespace](#3-granting-aws-permissions-to-the-nutrition-namespace)
  - [4. An Intentionally Jittery Consumer](#4-an-intentionally-jittery-consumer)
  - [5. Authenticating to AWS with a TriggerAuthentication](#5-authenticating-to-aws-with-a-triggerauthentication)
  - [6. Anatomy of the SQS ScaledObject](#6-anatomy-of-the-sqs-scaledobject)
  - [7. Mapping External AWS Calls with ServiceEntry](#7-mapping-external-aws-calls-with-serviceentry)
  - [8. Watching the Queue Drain](#8-watching-the-queue-drain)
  - [Key Takeaways](#key-takeaways-6)
- [Lesson 8: Running KEDA on Fargate](#lesson-8-running-keda-on-fargate)
  - [1. Why KEDA Shouldn't Depend on Karpenter-Managed Nodes](#1-why-keda-shouldnt-depend-on-karpenter-managed-nodes)
  - [2. Adding a Fargate Profile for KEDA](#2-adding-a-fargate-profile-for-keda)
  - [3. Verifying KEDA Runs on Fargate](#3-verifying-keda-runs-on-fargate)
  - [Key Takeaways](#key-takeaways-7)

---

# Lesson 1: What KEDA Adds to the Autoscaling Story

## 1. Beyond Node-Level Autoscaling

Every autoscaling mechanism covered so far in this project — the Cluster Autoscaler, then Karpenter — operates at the node level: it watches for pods that don't fit anywhere and adds capacity, or watches for underused nodes and removes them. None of that decides how many replicas of `chip` or `health-api` should exist in the first place. That decision has, until now, been left to whatever `replicas` count was hardcoded into a Deployment, or to the basic CPU/memory-based HPA behavior that `metrics-server` already enables.

KEDA (Kubernetes Event-Driven Autoscaling) extends that application-level decision beyond CPU and memory. It still ultimately drives a standard Kubernetes HPA under the hood, but it lets that HPA scale on signals that Kubernetes has no native concept of: a message count in a queue, a cron schedule, an arbitrary Prometheus query, a CloudWatch metric, or dozens of other event sources. Karpenter keeps doing exactly what it already does — reacting to unschedulable pods — but now the number of pods reacting to real product traffic is driven by KEDA instead of a static replica count.

## 2. The Scaler Ecosystem

KEDA ships with a large catalog of built-in scalers, each one a pluggable trigger type usable inside a `ScaledObject`. This module works through a representative subset — CPU, cron, Prometheus, and AWS SQS — but the ecosystem covers far more: MySQL, Kafka, InfluxDB, log-based scalers, New Relic, NGINX request metrics, Datadog, and many others. The scalers used in this module were chosen because they compose naturally with infrastructure already built in earlier modules — Prometheus and Istio metrics from Module 13, and Pod Identity from Module 5 — not because they're the only ones that matter. Exploring the remaining scalers is left as further reading once this module's pattern is understood.

## Key Takeaways

- KEDA scales pods (via a standard HPA it creates and manages), not nodes — it complements Karpenter rather than replacing it.
- Its value is in the breadth of event sources it exposes to the HPA: queues, schedules, arbitrary metrics queries, and external APIs, not just CPU/memory.
- This module covers CPU, cron, Prometheus, and SQS scalers as a representative sample of a much larger built-in scaler catalog.

---

# Lesson 2: Installing KEDA

## 1. Branching from the Istio Lab

This module branches directly from the Istio module's final state, rather than starting from `main`, because one of the scalers used later (Lesson 6) reads Prometheus metrics that only exist because Istio's sidecars produce them (`istio_requests_total`). Everything built in Module 13 — the mesh, Kiali, Jaeger, the `chip` and `health-api` Gateways/VirtualServices — is still present and unmodified as the starting point here.

## 2. Granting AWS Permissions with Pod Identity

KEDA's SQS scaler (used in Lesson 7) needs to read queue attributes from AWS. Consistent with how every other AWS-facing controller in this project has been wired — Karpenter included — this uses EKS Pod Identity rather than IRSA/OIDC service-account annotations. A dedicated IAM role is created for KEDA, trusted by the Pod Identity service principal, and associated with the `keda-operator` service account in the `keda` namespace:

```hcl
# iam_keda.tf
data "aws_iam_policy_document" "keda_role" {
  version = "2012-10-17"

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


resource "aws_iam_role" "keda_role" {
  assume_role_policy = data.aws_iam_policy_document.keda_role.json
  name               = format("%s-keda", var.project_name)
}

data "aws_iam_policy_document" "keda_policy" {
  version = "2012-10-17"

  statement {

    effect = "Allow"
    actions = [
      "sqs:Get*",
      "sqs:Describe*",
    ]

    resources = [
      "*"
    ]

  }
}

resource "aws_iam_policy" "keda_policy" {
  name        = format("%s-keda", var.project_name)
  path        = "/"
  description = var.project_name

  policy = data.aws_iam_policy_document.keda_policy.json
}

resource "aws_iam_policy_attachment" "keda" {
  name = "keda"
  roles = [
    aws_iam_role.keda_role.name
  ]

  policy_arn = aws_iam_policy.keda_policy.arn
}

resource "aws_eks_pod_identity_association" "keda" {
  cluster_name    = aws_eks_cluster.main.name
  namespace       = "keda"
  service_account = "keda-operator"
  role_arn        = aws_iam_role.keda_role.arn
}
```

At this stage the policy is deliberately narrow: `sqs:Get*` and `sqs:Describe*` are read-only actions, enough for KEDA's own operator to poll queue attributes for the SQS scaler in Lesson 7. It does not grant `sqs:SendMessage` or `sqs:ReceiveMessage` — KEDA only ever needs to know how many messages are in a queue, never to consume them itself.

## 3. Declaring the KEDA Version

The chart version is pinned through a new Terraform variable, following the same pattern already used for `istio_version` and `kiali_version`:

```hcl
# variables.tf
// Keda

variable "keda_version" {
  type        = string
  description = "value of keda version"
  default     = "2.16.0"

}
```

## 4. Converting the Helm Install into a Terraform helm_release

KEDA's own documentation installs it with a plain `helm install`. As with every other Helm-based component in this project, that gets translated into a `helm_release` resource so the installation is versioned and reproducible through Terraform:

```hcl
# helm_keda.tf
resource "helm_release" "keda" {

  version = var.keda_version

  name             = "keda"
  chart            = "keda"
  repository       = "https://kedacore.github.io/charts"
  namespace        = "keda"
  create_namespace = true

  depends_on = [
    aws_eks_cluster.main,
    helm_release.karpenter,
    helm_release.keda
  ]
}
```

> **Note:** the `depends_on` list includes `helm_release.keda` itself — a resource cannot depend on itself in Terraform, and this is a real self-reference left in the ground-truth file. It has no functional effect (Terraform ignores a resource's dependency on itself when building its graph), but it's very likely a copy-paste leftover from another `helm_release` block, where the intended third dependency was probably `helm_release.metrics_server` — KEDA's own metrics used by its HPA rely on `metrics-server` being present first, and that's the dependency this block is actually missing.

## 5. Verifying the Installation

After `terraform apply`, the KEDA workloads come up in their own namespace:

```bash
kubectl get all -n keda
```

This shows three components: the KEDA operator (the controller reconciling `ScaledObject`/`ScaledJob` resources), a metrics server (a KEDA-specific metrics API server the HPA controller queries), and an admission webhook (validating `ScaledObject`/`ScaledJob` manifests on creation). The install also registers three new CRDs:

```bash
kubectl get crd | grep keda
```

```
scaledjobs.keda.sh
scaledobjects.keda.sh
triggerauthentications.keda.sh
```

`ScaledObject` scales a Deployment or StatefulSet, `ScaledJob` scales Kubernetes Jobs, and `TriggerAuthentication` supplies external credentials — like the Pod Identity association just created — to a scaler that needs to authenticate against something outside the cluster. Before any of these are used for real scaling, though, the cluster's metrics have a gap left over from the Istio module that needs fixing first — that's the subject of the next lesson.

## Key Takeaways

- KEDA's own AWS permissions use Pod Identity, the same pattern as Karpenter, kept deliberately read-only (`Get*`/`Describe*`) since the operator only ever needs to observe queue depth.
- The chart install is expressed as a `helm_release`, matching every other Helm-managed component in this project.
- The real `helm_keda.tf` file contains a self-referential `depends_on` entry — a harmless but clearly unintended leftover, most likely meant to reference `helm_release.metrics_server`.
- Installing KEDA registers three CRDs: `ScaledObject`, `ScaledJob`, and `TriggerAuthentication`.

---

# Lesson 3: Fixing Metrics and Preparing the Lab Application

## 1. Importing a Kubernetes Cluster Dashboard

Rather than hand-building Grafana panels and their backing `ServiceMonitor`/`PodMonitor` resources one at a time, this lesson imports a pre-built "Kubernetes cluster" community dashboard JSON directly into Grafana. It's a fast way to get broad visibility — pod counts, resource usage, node status — without writing PromQL from scratch for every panel this module will need.

## 2. Diagnosing Missing Metrics

Once imported, most of the dashboard's panels come back empty. The dashboard depends heavily on metrics produced by `kube-state-metrics` — a component that exposes the state of Kubernetes objects themselves (pod phases, deployment replica counts, and so on) as Prometheus metrics, distinct from `metrics-server`'s resource-usage metrics (CPU/memory). Module 13's Prometheus values file explicitly disabled it:

```yaml
# helm/prometheus/values.yml (as left at the end of Module 13)
kubeStateMetrics:
  enabled: false
```

With it disabled, the cluster has resource-usage metrics but no object-state metrics — exactly the gap the imported dashboard exposes.

## 3. Enabling kube-state-metrics

The fix is a one-line flip in the same values file:

```yaml
# helm/prometheus/values.yml
kubeStateMetrics:
  enabled: true
```

## 4. Exposing metrics-server Through a ServiceMonitor

While fixing metrics, `metrics-server`'s own `ServiceMonitor` is also enabled, so Prometheus scrapes it directly in addition to using it as the resource-metrics API for the HPA:

```hcl
# helm_metrics_server.tf
resource "helm_release" "metrics_server" {
  name       = "metrics-server"
  repository = "https://charts.bitnami.com/bitnami"
  chart      = "metrics-server"
  namespace  = "kube-system"

  wait = false

  version = "7.2.16"

  set = [
    {
      name  = "apiService.create"
      value = "true"
    },
    {
      name  = "serviceMonitor.enabled"
      value = "true"
    }
  ]

  depends_on = [
    aws_eks_cluster.main,
    aws_eks_fargate_profile.karpenter
  ]
}
```

`kube-state-metrics` and `metrics-server` are complementary, not redundant: after redeploying, Prometheus has both object-state metrics (pod counts, phases) and resource-usage metrics (CPU/memory), and a new `kube-state-metrics` pod appears alongside the existing Prometheus stack. The imported dashboard immediately starts populating.

## 5. Redeploying the chip Lab Application

With metrics fixed, the `chip` application from the Istio module is redeployed unchanged as the lab target for the CPU, cron, and Prometheus scaling examples that follow.

## 6. A CPU-Intensive Endpoint for Testing

To have a reliable way to spike CPU usage on demand, `chip` exposes a `/burncpu` endpoint that intentionally runs a CPU-bound loop when hit. It exists purely as a lab tool for the next lesson, not as production behavior.

## Key Takeaways

- The imported Grafana dashboard exposed a real gap: `kube-state-metrics` had been left disabled since Module 13.
- `kube-state-metrics` (object state) and `metrics-server` (resource usage) are complementary — both are needed for full cluster visibility.
- `metrics-server`'s `ServiceMonitor` is enabled so Prometheus can scrape it directly, independent of its role backing the HPA API.
- `chip`'s `/burncpu` endpoint is a deliberate lab tool for generating CPU spikes on demand.

---

# Lesson 4: Scaling on CPU Usage

## 1. Generating a CPU Spike

Multiple parallel `while true` curl loops are run against `/burncpu` to keep `chip`'s CPU usage elevated:

```bash
while true; do curl -s http://chip.msfidelis.com.br/burncpu > /dev/null; done
```

Running several of these concurrently, from different terminals, produces a sustained CPU spike across the running `chip` pods.

## 2. Visualizing Pods and CPU Usage

A temporary panel is added to Grafana, counting running pods with a `kube_pod_status_phase` query — one of the metrics now available thanks to the `kube-state-metrics` fix from Lesson 3 — placed alongside the imported dashboard's existing compute-resources panels. This makes it possible to watch pod count and CPU usage climb together in the same view once scaling kicks in.

## 3. Anatomy of a ScaledObject

The first and simplest scaler is CPU — conceptually the same signal already used by the Cluster Autoscaler and Karpenter, but now driving pod count instead of node count:

```yaml
# chip.yml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: chip-high-cpu
  namespace: chip
spec:
  scaleTargetRef:
    name: chip
  minReplicaCount: 3
  maxReplicaCount: 20
  pollingInterval: 10
  cooldownPeriod: 30
  advanced:
    horizontalPodAutoscalerConfig:
      behavior:
        scaleDown:
          stabilizationWindowSeconds: 60
        scaleUp:
          stabilizationWindowSeconds: 20
  triggers:
    - type: cpu
      metricType: Utilization
      metadata:
        value: "40"
```

The `scaleUp`/`scaleDown` stabilization windows are deliberately asymmetric — 20 seconds to scale up, 60 to scale down — reacting fast to a spike but cooling down slowly, so the deployment doesn't flap up and down every time CPU dips briefly below the threshold. `pollingInterval` and `cooldownPeriod` control how often KEDA checks the trigger and how long it waits after the last active trigger before allowing a scale-to-zero-style decision; here they're set to 10 and 30 seconds respectively for fast lab feedback.

## 4. How KEDA Manages the Underlying HPA

KEDA doesn't replace the Kubernetes HPA — it creates and manages one on the target's behalf. Applying the `ScaledObject` above produces an HPA visible with:

```bash
kubectl get hpa -n chip
```

The HPA's name is derived from the `ScaledObject`, and its `minReplicas`/`maxReplicas`/target metric all trace directly back to the fields just declared. KEDA's job is to keep that HPA's configuration in sync with the `ScaledObject` spec; the actual scaling decisions are still made by the standard HPA controller.

## 5. Watching the Cluster Scale

With the CPU-burn loops running, `chip`'s pod count climbs toward the 40% utilization threshold's implied replica count, visible in both the temporary Grafana panel and `kubectl get pods -n chip -w`. As pod count rises, Karpenter — still running unmodified from earlier modules — provisions new nodes to schedule the additional replicas, visible via `kubectl get nodeclaims`. Stopping the curl loops eventually brings CPU back down and, after the 60-second scale-down stabilization window, replicas drain back toward the minimum of 3.

## Key Takeaways

- KEDA's `cpu` scaler drives the same kind of signal Karpenter already reacts to at the node level, but now at the pod level via a KEDA-managed HPA.
- Every `ScaledObject` creates and owns a standard Kubernetes HPA object — KEDA doesn't bypass the HPA API, it automates it.
- Asymmetric stabilization windows (fast scale-up, slow scale-down) prevent flapping without slowing down the response to a real spike.
- Scaling `chip`'s pod count up naturally drives Karpenter to provision more nodes underneath — the two layers compose without needing to know about each other.

---

# Lesson 5: Scaling on a Schedule with Cron

## 1. Why Scale Preventively

Reactive scaling — CPU, request rate, queue depth — always lags behind the event that triggers it by at least one polling interval. For traffic spikes that are known in advance (a marketing campaign, a recurring batch job, a predictable daily peak), it's often better to scale preventively: raise the replica floor ahead of the expected spike instead of waiting for it to already be underway.

## 2. Superseding the CPU ScaledObject

The CPU-based `ScaledObject` from Lesson 4 is commented out rather than deleted — preserving it in the file as a reference for the pattern already covered, consistent with how superseded examples have been handled elsewhere in this project (for instance the old NGINX Ingress configuration left commented out in `helm_prometheus.tf` since Module 13):

```yaml
# chip.yml
#apiVersion: keda.sh/v1alpha1
#kind: ScaledObject
#metadata:
#  name: chip-high-cpu
#  namespace: chip
#spec:
#  scaleTargetRef:
#    name: chip
#  minReplicaCount: 3
#  maxReplicaCount: 20
#  pollingInterval: 10
#  cooldownPeriod: 30
#  advanced:
#    horizontalPodAutoscalerConfig:
#      behavior:
#        scaleDown:
#          stabilizationWindowSeconds: 60
#        scaleUp:
#          stabilizationWindowSeconds: 20
#  triggers:
#    - type: cpu
#      metricType: Utilization
#      metadata:
#        value: "40"
```

## 3. Anatomy of a Cron Trigger

In its place, a cron-triggered `ScaledObject` forces a higher replica count during a fixed time window, independent of any live metric:

```yaml
# chip.yml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: chip-scheduled-spike
  namespace: chip
spec:
  scaleTargetRef:
    name: chip
  minReplicaCount: 3
  maxReplicaCount: 20
  triggers:
    - type: cron
      metadata:
        timezone: America/Sao_Paulo
        start: 30 20 * * *
        end: 35 20 * * *
        desiredReplicaCount: "20"
```

`start` and `end` are standard cron expressions defining the active window — here, 20:30 to 20:35 in the `America/Sao_Paulo` timezone, a short window chosen purely so the effect is observable quickly in the lab. Inside that window, KEDA forces the underlying HPA's desired replica count to 20 regardless of CPU, request rate, or any other signal; outside it, the deployment reverts to whatever its normal floor would be (here, the `minReplicaCount` of 3, since no other trigger is currently active).

## 4. How KEDA and Karpenter Coordinate on a Schedule

As the scheduled window opens and `chip` jumps to 20 replicas, Karpenter reacts exactly as it would to any other sudden increase in unschedulable pods — provisioning nodes to fit them. Neither controller is aware of the other's schedule; Karpenter has no concept of "20:30 to 20:35," it simply keeps reacting to whatever pod count KEDA (via the HPA) currently demands. This is the same composition already seen with the CPU scaler in Lesson 4 — the two layers don't need to be scheduled against each other because each does its own job on the signal it owns.

## Key Takeaways

- Cron-based scaling is useful for known, predictable spikes, where reacting after the fact is already too late.
- Superseded `ScaledObject` examples are commented out in place rather than deleted, keeping the module's file history readable.
- A cron trigger's `start`/`end` define an active window during which `desiredReplicaCount` is forced, independent of any live metric.
- Karpenter reacts to the resulting pod count exactly as it would to any other scaling event — no direct coordination between the two controllers is needed.

---

# Lesson 6: Scaling on Requests Per Second with Prometheus

## 1. Generating Realistic Load with k6

Manually looping curl requests, as done in Lesson 4, doesn't produce a realistic traffic pattern. This lesson switches to k6, a load-testing tool, with a script that ramps virtual users up and down in stages against `chip`'s `/system` endpoint:

```javascript
// load.js
import http from 'k6/http';
import { check, sleep } from 'k6';

export let options = {
    stages: [
        { duration: '10s', target: 10 },   
        { duration: '20s', target: 50 },  
        { duration: '30s', target: 100 },
        { duration: '20s', target: 50 },
        { duration: '10s', target: 20 },
        { duration: '60s', target: 40 },
        { duration: '60s', target: 100 },
    ]
};

export default function () {
    let res = http.get('http://chip.msfidelis.com.br/system');

    check(res, {
        'Status 200': (r) => r.status === 200,
        'Tempo de resposta < 500ms': (r) => r.timings.duration < 500,
    });

    sleep(1); // Pequeno tempo de espera entre as requisições
}
```

The staged ramp — up to 10, then 50, then a peak of 100 virtual users, back down, then two longer plateaus at 40 and 100 — produces a request-rate curve that rises and falls the way real traffic does, rather than the on/off spike the CPU-burn loops produced.

## 2. Visualizing Request Rate

A new Grafana panel is added next to the pod-count panel from Lesson 4, graphing request throughput using the same PromQL that Istio's sidecars make available:

```
sum(rate(istio_requests_total{destination_service_name="chip"}[1m]))
```

This is the same `istio_requests_total` metric introduced by the mesh in Module 13, now being read directly by a scaling decision instead of just a dashboard.

## 3. Anatomy of the Prometheus Trigger

The cron `ScaledObject` from Lesson 5 is commented out, and a Prometheus-triggered one takes over — one of KEDA's most powerful scalers, since it accepts an arbitrary PromQL query rather than a fixed metric name:

```yaml
# chip.yml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: chip-high-tps
  namespace: chip
spec:
  scaleTargetRef:
    name: chip
  minReplicaCount: 3
  maxReplicaCount: 30
  pollingInterval: 10
  cooldownPeriod: 30
  advanced:
    horizontalPodAutoscalerConfig:
      behavior:
        scaleDown:
          stabilizationWindowSeconds: 20
        scaleUp:
          stabilizationWindowSeconds: 20
  triggers:
    - type: prometheus
      metadata:
        serverAddress: http://prometheus-kube-prometheus-prometheus.prometheus.svc.cluster.local:9090
        metricName: istio_requests_total
        threshold: "10"
        query: sum(rate(istio_requests_total{destination_service_name="chip"}[1m]))
```

`serverAddress` points at the in-cluster Prometheus Service directly — no ingress or external exposure is needed, since KEDA's operator queries it from inside the cluster. `query` is the exact same PromQL used in the Grafana panel above, so what's driving the scaling decision is literally what the dashboard shows. Unlike Lesson 4's CPU trigger, both `scaleUp` and `scaleDown` stabilization windows are set to 20 seconds here — this trigger reacts quickly in both directions, since request rate changes are a more direct and immediate signal of real load than CPU utilization is.

## 4. Sizing the Threshold Per Pod

`threshold: "10"` is not a global cap — it's interpreted as capacity per replica. KEDA computes the desired replica count as roughly `query result ÷ threshold`, so a threshold of 10 means "one replica handles up to 10 requests per second," and the query result rising to, say, 150 drives the HPA toward 15 replicas (bounded by `minReplicaCount`/`maxReplicaCount`). Getting this number right requires knowing, from load testing, roughly how many requests per second a single `chip` pod can comfortably sustain — a value tuned by observation, not derived automatically.

## 5. Why TPS-Based Scaling Beats CPU for Synchronous Workloads

For synchronous REST or gRPC workloads like `chip`, request-rate-based scaling is a significant improvement over CPU-based scaling. CPU usage typically only rises meaningfully after latency has already started to degrade, or requires setting an artificially conservative utilization threshold to react earlier at the cost of over-provisioning constantly. A direct throughput signal reacts to the thing that actually matters — how much traffic is arriving — before it turns into a CPU or latency problem at all. This is arguably the most valuable pattern in the whole module: it turns metrics Istio was already producing for observability into a first-class autoscaling signal.

## Key Takeaways

- k6's staged VU ramps produce a realistic rising-and-falling load curve, unlike a fixed on/off CPU-burn loop.
- The Prometheus scaler accepts arbitrary PromQL, letting any existing Grafana query become a scaling signal directly.
- A Prometheus trigger's `threshold` is per-replica capacity, not a global ceiling — KEDA divides the query result by it to size the HPA.
- Reusing `istio_requests_total` means Istio's own telemetry, introduced purely for observability in Module 13, now drives real scaling decisions.
- Request-rate scaling reacts to rising traffic directly, instead of waiting for CPU or latency to reflect it indirectly.

---

# Lesson 7: Scaling an SQS Consumer

## 1. A New Health API Version

This lesson moves to a different application and a different kind of trigger: the `health-api` service, from earlier modules, is updated to publish its calculation reports to an SQS queue instead of only returning them synchronously. The relevant environment variables are added directly to its existing Deployment:

```yaml
# health.yml
      - name: health-api
        env:
          - name: MESSAGE_TYPE
            value: "sqs"
          - name: SQS_QUEUE_URL
            value: "<queue-url>"
```

## 2. Provisioning the SQS Queue and DynamoDB Table

The queue itself, along with a DynamoDB table to persist processed results, is provisioned in Terraform:

```hcl
# health_api_deps.tf
resource "aws_sqs_queue" "health" {
  name                       = format("%s-health", var.project_name)
  delay_seconds              = 0
  message_retention_seconds  = 86400
  receive_wait_time_seconds  = 10
  visibility_timeout_seconds = 60
}

resource "aws_sqs_queue_policy" "health" {
  queue_url = aws_sqs_queue.health.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect    = "Allow"
        Principal = "*"
        Action    = "sqs:SendMessage"
        Resource  = aws_sqs_queue.health.arn
      }
    ]
  })
}

resource "aws_dynamodb_table" "health" {
  name           = "health-data"
  hash_key       = "id"
  read_capacity  = 20
  write_capacity = 20

  attribute {
    name = "id"
    type = "S"
  }
}
```

> **Note:** the queue policy's `Principal` is set to `"*"` — allowing any AWS principal, not just this project's own roles, to send messages to the queue. This is real, over-permissive lab-grade configuration: in a production setting the principal should be scoped to the specific role(s) that need to publish (here, `health-api`'s own Pod Identity role), not left open to any authenticated AWS caller.

## 3. Granting AWS Permissions to the nutrition Namespace

A new `nutrition` namespace hosts the consumer that reads from the queue and writes into DynamoDB. It gets its own Pod Identity role, following the same trust pattern as KEDA's own role from Lesson 2, but with broader permissions since this workload actually needs to send and receive messages and read/write table data, not just inspect queue attributes:

```hcl
# health_api_deps.tf
data "aws_iam_policy_document" "nutrition_role" {
  version = "2012-10-17"

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

resource "aws_iam_role" "nutrition_role" {
  assume_role_policy = data.aws_iam_policy_document.nutrition_role.json
  name               = format("%s-nutrition", var.project_name)
}

data "aws_iam_policy_document" "nutrition_policy" {
  version = "2012-10-17"

  statement {
    effect = "Allow"
    actions = [
      "sqs:*",
      "dynamodb:*",
    ]
    resources = ["*"]
  }
}

resource "aws_iam_policy" "nutrition_policy" {
  name        = format("%s-nutrition", var.project_name)
  path        = "/"
  description = var.project_name

  policy = data.aws_iam_policy_document.nutrition_policy.json
}

resource "aws_iam_policy_attachment" "nutrition" {
  name = "nutrition"
  roles = [
    aws_iam_role.nutrition_role.name
  ]

  policy_arn = aws_iam_policy.nutrition_policy.arn
}

resource "aws_eks_pod_identity_association" "nutrition" {
  cluster_name    = aws_eks_cluster.main.name
  namespace       = "nutrition"
  service_account = "nutrition"
  role_arn        = aws_iam_role.nutrition_role.arn
}
```

## 4. An Intentionally Jittery Consumer

A new `health-data-offload` Deployment consumes messages from the queue and writes them into DynamoDB:

```yaml
# health.yml
      - name: health-data-offload
        env:
          - name: WORKERS
            value: "1"
          - name: WORKERS_JITTER_MS
            value: "10000"
          - name: DATABASE_TYPE
            value: "dynamodb"
          - name: DYNAMODB_TABLE
            value: "health-data"
          - name: MESSAGE_TYPE
            value: "sqs"
          - name: SQS_QUEUE_URL
            value: "<queue-url>"
```

`WORKERS_JITTER_MS` adds a random 0-10 second delay to every message this single worker processes. This is a deliberate, artificial slowdown purely for the lab: with only one worker and a random delay on every message, the consumer processes messages slowly enough that a realistic backlog builds up in the queue once load starts arriving — giving KEDA's SQS scaler something meaningful to react to, instead of the queue draining instantly.

## 5. Authenticating to AWS with a TriggerAuthentication

Unlike the CPU, cron, and Prometheus triggers used so far, an SQS trigger needs to authenticate against AWS. KEDA exposes this through a `TriggerAuthentication` resource, and here it's configured to use the same Pod Identity mechanism as everything else in this project — not IRSA or a service-account OIDC annotation:

```yaml
# health.yml
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: nutrition-aws-credentials
  namespace: nutrition
spec:
  podIdentity:
    provider: aws
```

## 6. Anatomy of the SQS ScaledObject

The `ScaledObject` for `health-data-offload` references the `TriggerAuthentication` above and scales based on the queue's approximate message count:

```yaml
# health.yml
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: health-data-offload-sqs
  namespace: nutrition
spec:
  scaleTargetRef:
    name: health-data-offload
  minReplicaCount: 1
  maxReplicaCount: 30
  pollingInterval: 10
  cooldownPeriod: 30
  advanced:
    horizontalPodAutoscalerConfig:
      behavior:
        scaleDown:
          stabilizationWindowSeconds: 60
        scaleUp:
          stabilizationWindowSeconds: 60
  triggers:
    - type: aws-sqs-queue
      authenticationRef:
        name: nutrition-aws-credentials
      metadata:
        queueURL: <queue-url>
        awsRegion: "us-east-1"
        queueLength: "20"
```

`queueLength` plays the same per-replica-capacity role that `threshold` played for the Prometheus trigger in Lesson 6: a value of 20 means "one consumer replica can keep up with roughly 20 queued messages," so a backlog of 200 messages drives the HPA toward roughly 10 replicas. Both stabilization windows here are set to a slower 60 seconds — queue depth is a noisier, more bursty signal than request rate, so this trigger deliberately reacts less aggressively than the TPS-based one from Lesson 6.

## 7. Mapping External AWS Calls with ServiceEntry

Because the mesh only has visibility into traffic between services it manages, calls from `health-api` and `health-data-offload` out to SQS and DynamoDB would otherwise appear as opaque external egress in Kiali. Two `ServiceEntry` resources close that gap, the same way `chip`'s pre-existing `ServiceEntry` for `google.com.br` already did in Module 13:

```yaml
# health.yml
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: sqs-us-east-1
  namespace: nutrition
spec:
  hosts:
    - sqs.us-east-1.amazonaws.com
  location: MESH_EXTERNAL
  ports:
    - number: 443
      name: https
      protocol: TLS
  resolution: DNS
---
apiVersion: networking.istio.io/v1beta1
kind: ServiceEntry
metadata:
  name: dynamodb-us-east-1
  namespace: nutrition
spec:
  hosts:
    - dynamodb.us-east-1.amazonaws.com
  location: MESH_EXTERNAL
  ports:
    - number: 443
      name: https
      protocol: TLS
  resolution: DNS
```

With these in place, Kiali's topology view labels traffic to SQS and DynamoDB explicitly, instead of showing it as unlabeled pass-through traffic leaving the mesh.

## 8. Watching the Queue Drain

Generating load against `health-api` produces thousands of visible messages in the SQS queue, observable via the AWS console or CLI. As the backlog grows, the `health-data-offload-sqs` `ScaledObject` drives `health-data-offload` up toward its maximum of 30 replicas to work through it, and Karpenter provisions whatever node capacity those replicas need — the same composition already seen with `chip` in earlier lessons, now applied to a queue-based consumer instead of a synchronous API.

## Key Takeaways

- `health-api` now publishes to SQS in addition to its existing synchronous behavior, and a new `health-data-offload` Deployment consumes from that queue into DynamoDB.
- The queue's send policy is left open to any AWS principal (`Principal: "*"`) — a real, overly permissive configuration that should be scoped down outside a lab context.
- The `nutrition` namespace gets its own Pod Identity role with broad `sqs:*`/`dynamodb:*` permissions, separate from KEDA's own narrower read-only role.
- The consumer's random jitter is an intentional lab device to let a realistic backlog accumulate, not a defect.
- An SQS trigger needs a `TriggerAuthentication` — here backed by Pod Identity, consistent with every other AWS-facing credential in this project.
- `queueLength` behaves like the Prometheus trigger's `threshold`: per-replica capacity, not a global cap.
- Two new `ServiceEntry` resources make the mesh aware of the external SQS and DynamoDB calls this flow depends on.

---

# Lesson 8: Running KEDA on Fargate

## 1. Why KEDA Shouldn't Depend on Karpenter-Managed Nodes

KEDA's operator is a control-plane component: if it goes down, no `ScaledObject` gets reconciled and the HPAs it manages stop updating. Running it on a Karpenter-managed node means it's exposed to the same churn Karpenter itself causes — consolidation, Spot interruptions, node replacement — for other workloads. Karpenter's own controller avoids this problem by running on a dedicated Fargate profile, isolated from the very node churn it creates; this lesson gives KEDA the same isolation.

## 2. Adding a Fargate Profile for KEDA

A new Fargate profile targets the `keda` namespace, reusing the same execution role and subnets already defined for Karpenter's own profile:

```hcl
# fargate.tf
resource "aws_eks_fargate_profile" "keda" {
  cluster_name           = aws_eks_cluster.main.name
  fargate_profile_name   = "keda"
  pod_execution_role_arn = aws_iam_role.fargate.arn
  subnet_ids             = local.private_subnets

  selector {
    namespace = "keda"
  }
}
```

## 3. Verifying KEDA Runs on Fargate

After applying, the existing KEDA pods are deleted so they get rescheduled under the new profile:

```bash
kubectl delete pods -n keda --all
kubectl get pods -n keda -o wide
```

The `NODE` column for each pod now shows a Fargate-style node name, confirming KEDA's own control plane no longer competes for scheduling with the workloads it's autoscaling.

## Key Takeaways

- KEDA's control plane benefits from the same Fargate isolation already given to Karpenter, for the same reason: it shouldn't be exposed to the node churn it — or Karpenter — causes for other workloads.
- The new Fargate profile reuses Karpenter's existing execution role and subnets, differing only in its namespace selector.
- Deleting the existing KEDA pods after applying is what actually triggers rescheduling onto Fargate — the profile alone doesn't move already-running pods.
