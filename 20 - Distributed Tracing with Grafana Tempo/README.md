# Module 20: Distributed Tracing with Grafana Tempo

## Overview

This module adds the third piece of the Grafana Stack to the same **observability cluster** built in Module 19: **Grafana Tempo**, for distributed trace storage and query. It's the direct successor to the Jaeger setup from the Istio module — same underlying idea (capture and visualize distributed traces across microservices), but now integrated natively into the Grafana Stack instead of running its own separate UI, and following the same label-based, cheap-and-performant philosophy as Loki and Prometheus, with a smaller feature set traded for tight Grafana integration.

All 6 lessons continue directly in the **same repository and cluster** as Module 19 — `linuxtips-eks-observability-cluster`, branch `main` — not a new repository. As already flagged in Module 19's Overview, this repo is written once for the whole three-module observability build, so the Tempo-specific Terraform/Helm files (`helm_tempo.tf`, `iam_tempo.tf`, `s3_tempo.tf`, `lb_tempo.tf`) and the standalone `otel.yml` `ApplicationSet` were already sitting in the repository as forward-looking scaffolding; this module is what actually builds, explains, and activates them.

The module's centerpiece is deploying a much larger lab than anything used so far: a multi-service "Health API" (`health-api.yml`) — six gRPC microservices (BMR, IMC, calories, proteins, water, recommendations) plus a public HTTP entry point — deployed active-active across both workload clusters, each instrumented to send Zipkin-format spans to a new OpenTelemetry Collector. This is the same `health-api.yml` file that Module 19's README flagged as an unrelated, stray file using a different domain (`msfidelis.com.br`) than the course's own (`luisgustavo.com.br`); it's confirmed here to be real, intentional Module 20 content, not an anomaly.

> **Note:** `locals.tf`'s `grafana` and `tempo` blocks, and `environment/prod/terraform.tfvars`'s `karpenter_capacity` list, already carry Mimir-related entries in the repository's current state (a `metricsGenerator.config.storage.remote_write` pointing at `mimir-nginx`, `tracesToMetrics`/`serviceMap` datasource UIDs referencing a `Mimir` data source, and a `mimir` NodePool) — none of it built, wired to a working Mimir instance, or explained by this module. That's Module 21 content, following the exact same forward-looking-scaffolding pattern already seen with Tempo/OTel in Module 19.

## Table of Contents

- [Lesson 1: Introduction to Grafana Tempo](#lesson-1-introduction-to-grafana-tempo)
  - [1. From Jaeger to Tempo](#1-from-jaeger-to-tempo)
  - [2. Prerequisites](#2-prerequisites)
- [Lesson 2: Deploying Grafana Tempo](#lesson-2-deploying-grafana-tempo)
  - [1. An S3 Bucket and IAM Role for Tempo](#1-an-s3-bucket-and-iam-role-for-tempo)
  - [2. A Dedicated `tempo` NodePool](#2-a-dedicated-tempo-nodepool)
  - [3. Monolithic vs. Distributed: Choosing the Helm Chart](#3-monolithic-vs-distributed-choosing-the-helm-chart)
  - [4. Tempo's Helm Values via a `locals` Block](#4-tempos-helm-values-via-a-locals-block)
  - [5. Installing the `tempo` Helm Release](#5-installing-the-tempo-helm-release)
  - [6. Tempo's Components at a Glance](#6-tempos-components-at-a-glance)
- [Lesson 3: Exposing Tempo Through a Network Load Balancer](#lesson-3-exposing-tempo-through-a-network-load-balancer)
  - [1. Internal NLB and Target Group Binding for the Gateway](#1-internal-nlb-and-target-group-binding-for-the-gateway)
  - [2. A Private DNS Record for the Tempo Gateway](#2-a-private-dns-record-for-the-tempo-gateway)
- [Lesson 4: Wiring Tempo as a Grafana Data Source](#lesson-4-wiring-tempo-as-a-grafana-data-source)
  - [1. Adding Tempo to Grafana's `datasources` via `locals`](#1-adding-tempo-to-grafanas-datasources-via-locals)
- [Lesson 5: The OpenTelemetry Collector](#lesson-5-the-opentelemetry-collector)
  - [1. A New Multicluster `ApplicationSet`](#1-a-new-multicluster-applicationset)
  - [2. Activating It for Real via Terraform](#2-activating-it-for-real-via-terraform)
  - [3. Why Applications Still Need Instrumentation](#3-why-applications-still-need-instrumentation)
- [Lesson 6: Deploying the Health API Lab and Exploring Traces](#lesson-6-deploying-the-health-api-lab-and-exploring-traces)
  - [1. A Multicluster, Multi-Service Lab](#1-a-multicluster-multi-service-lab)
  - [2. From Jaeger to the OpenTelemetry Collector](#2-from-jaeger-to-the-opentelemetry-collector)
  - [3. Generating Traffic and Exploring Traces](#3-generating-traffic-and-exploring-traces)
  - [4. Adding a Tempo Panel to the Grafana Dashboard](#4-adding-a-tempo-panel-to-the-grafana-dashboard)
- [Key Takeaways](#key-takeaways)

---

# Lesson 1: Introduction to Grafana Tempo

## 1. From Jaeger to Tempo

Grafana Tempo is a distributed tracing backend, filling the same role Jaeger filled in the Istio module: storing and querying spans that trace a request as it flows across multiple microservices. The difference is where it lives — instead of its own dedicated UI, Tempo plugs directly into Grafana as a data source, sitting alongside Loki (Module 19) and, later, Mimir (Module 21) in one correlated observability front end. Architecturally, Tempo follows the same design philosophy as Loki and Prometheus: index cheaply (by trace ID, not full content), store the bulk of the data in object storage, and accept a reduced feature set in exchange for being lightweight and horizontally scalable.

Tempo is fully compatible with **OpenTelemetry (OTel)** for trace ingestion, which is the format this module standardizes on — Lesson 5 introduces an OpenTelemetry Collector to receive spans from applications and forward them into Tempo.

## 2. Prerequisites

This module assumes Module 19's observability cluster is already up and running: Grafana and Loki deployed, the private Route 53 zone (`linuxtips-observability.local`) in place, and the two workload clusters (`linuxtips-cluster-01`, `linuxtips-cluster-02`) still federated to the control-plane cluster's Argo CD from Module 18. All new work in this module happens in the same `linuxtips-eks-observability-cluster` repository Module 19 built, continuing to add files alongside the existing Grafana/Loki ones.

---

# Lesson 2: Deploying Grafana Tempo

## 1. An S3 Bucket and IAM Role for Tempo

Tempo's data path is deliberately similar to Loki's: a **distributor** receives and validates incoming spans and hands them to an **ingester**, which buffers and groups them into chunks in memory before flushing them to an object store — S3, in this deployment. Unlike Loki (which used three separate buckets for chunks, admin data, and the ruler), Tempo needs only one:

```hcl
# s3_tempo.tf
resource "aws_s3_bucket" "tempo" {
  bucket = format("%s-%s-tempo", var.project_name, data.aws_caller_identity.current.account_id)
}

resource "aws_s3_bucket_ownership_controls" "tempo" {
  bucket = aws_s3_bucket.tempo.id
  rule {
    object_ownership = "BucketOwnerPreferred"
  }
}

resource "aws_s3_bucket_acl" "tempo" {
  bucket = aws_s3_bucket.tempo.id
  acl    = "private"

  depends_on = [
    aws_s3_bucket_ownership_controls.tempo
  ]
}
```

The IAM role follows the exact same Pod Identity pattern as Loki's, scoped this time to the single Tempo bucket:

```hcl
# iam_tempo.tf
data "aws_iam_policy_document" "tempo_role" {
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

resource "aws_iam_role" "tempo_role" {
  assume_role_policy = data.aws_iam_policy_document.tempo_role.json
  name               = format("%s-tempo", var.project_name)
}

data "aws_iam_policy_document" "tempo_policy" {
  version = "2012-10-17"

  statement {
    effect = "Allow"
    actions = [
      "s3:*",
    ]

    resources = [
      format("%s/*", aws_s3_bucket.tempo.arn),
      aws_s3_bucket.tempo.arn,
    ]
  }
}

resource "aws_iam_policy" "tempo_policy" {
  name        = format("%s-tempo", var.project_name)
  path        = "/"
  description = var.project_name

  policy = data.aws_iam_policy_document.tempo_policy.json
}

resource "aws_iam_policy_attachment" "tempo" {
  name = "tempo"
  roles = [
    aws_iam_role.tempo_role.name
  ]

  policy_arn = aws_iam_policy.tempo_policy.arn
}

resource "aws_eks_pod_identity_association" "tempo" {
  cluster_name    = aws_eks_cluster.main.name
  namespace       = "tempo"
  service_account = "tempo"
  role_arn        = aws_iam_role.tempo_role.arn
}
```

> **Note:** as with Loki's IAM role in Module 19, the policy grants `s3:*` — full S3 API access — rather than being scoped to only the actions Tempo actually needs (`GetObject`/`PutObject`/`ListBucket`/etc.). This is the same course-wide convention already flagged for `iam_chartmuseum.tf` and `iam_loki.tf`.

## 2. A Dedicated `tempo` NodePool

Before deploying Tempo itself, a new Karpenter NodePool is added — continuing the "one NodePool per observability workload" pattern Module 19 established in its own Lesson 9:

```hcl
# environment/prod/terraform.tfvars (karpenter_capacity, tempo entry)
{
  name               = "tempo"
  workload           = "tempo"
  ami_family         = "Bottlerocket"
  ami_ssm            = "/aws/service/bottlerocket/aws-k8s-1.31/x86_64/latest/image_id"
  instance_family    = ["t3", "t3a", "c6", "c6a", "c7", "c7a"]
  instance_sizes     = ["large", "xlarge", "2xlarge"]
  capacity_type      = ["spot", "on-demand"]
  availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]
},
```

This entry was already present in `terraform.tfvars` as of Module 19 (flagged there as scaffolding provisioning a NodePool with no workload targeting it yet) — this lesson is what finally schedules pods onto it.

## 3. Monolithic vs. Distributed: Choosing the Helm Chart

Tempo's own documentation offers the same two deployment shapes already seen with Loki:

- **`tempo`** — the single-binary chart, all components (distributor, ingester, querier, compactor, etc.) running in one process. The equivalent of Loki's monolithic mode.
- **`tempo-distributed`** — the microservices chart, each component as its own independently-scalable Deployment. The equivalent of Loki's simple-scalable/microservice modes.

This lesson deploys `tempo-distributed`, for the same reason Loki was deployed in simple-scalable mode: independent, granular scaling per component instead of one monolithic binary.

## 4. Tempo's Helm Values via a `locals` Block

Same `locals` heredoc pattern as Grafana and Loki, configuring the S3 backend, one `nodeSelector` per component pinning everything to the new `tempo` NodePool, and which trace protocols the gateway accepts:

```hcl
# locals.tf (tempo block)
locals {
  tempo = {
    values : <<-VALUES
storage:
    trace:
        backend: s3
        s3:
            bucket: ${aws_s3_bucket.tempo.id}
            region: ${var.region}
            endpoint: s3.amazonaws.com
            forcepathstyle: false
gateway:
    enabled: true
    replicas: 3
    service:
        type: NodePort
    nodeSelector:
        karpenter.sh/nodepool: tempo
queryFrontend:
    replicas: 3
    query:
        enabled: false
    nodeSelector:
        karpenter.sh/nodepool: tempo       
querier:
    replicas: 3
    nodeSelector:
        karpenter.sh/nodepool: tempo       
distributor:
    enabled: true
    replicas: 3
    nodeSelector:
        karpenter.sh/nodepool: tempo           
ingester:
    replicas: 3
    nodeSelector:
        karpenter.sh/nodepool: tempo     
compactor:
    replicas: 3
    nodeSelector:
        karpenter.sh/nodepool: tempo       
traces:
    otlp:
        http:
            enabled: true
    grpc:
        enabled: true

metricsGenerator:
  enabled: true
  registry:
    external_labels:
      source: tempo
  config: 
    storage:
      remote_write: 
      - url: "http://mimir-nginx.mimir.svc.cluster.local:80/api/v1/push"
        send_exemplars: true

overrides:
  defaults:
    metrics_generator:
      processors: [service-graphs, span-metrics, local-blocks]

global_overrides:
  defaults:
    metrics_generator:
      processors: [service-graphs, span-metrics, local-blocks]
    VALUES
  }
}
```

> **Note:** `metricsGenerator`, `overrides`, and `global_overrides` configure Tempo to derive span-metrics/service-graphs and remote-write them to `mimir-nginx.mimir.svc.cluster.local` — a Mimir instance that doesn't exist until Module 21. This block is the repository's cumulative final state, same as Module 19's `grafana`/`loki` blocks; this lesson's own scope is only the `storage`, `gateway`, `queryFrontend`, `querier`, `distributor`, `ingester`, `compactor`, and `traces` sections.

`traces.otlp.http.enabled` and `traces.grpc.enabled` open Tempo's gateway to OTLP over both HTTP and gRPC — the protocols the OpenTelemetry Collector in Lesson 5 uses to push spans in.

## 5. Installing the `tempo` Helm Release

```hcl
# helm_tempo.tf
resource "helm_release" "tempo" {
  name       = "tempo"
  chart      = "tempo-distributed"
  repository = "https://grafana.github.io/helm-charts"
  namespace  = "tempo"
  create_namespace = true
  values = [
    local.tempo["values"]
  ]
  depends_on = [
    helm_release.karpenter,
    aws_eks_pod_identity_association.tempo,
    aws_eks_addon.ebs_csi
  ]
}
```

## 6. Tempo's Components at a Glance

After `terraform apply`, `kubectl get all -n tempo` shows the microservices shape of `tempo-distributed`: `gateway`, `distributor`, `ingester`, `querier`, `query-frontend`, `compactor`, and a `memcached`-style cache.

- **Gateway** — exposes Tempo's external APIs (HTTP, OTLP, Zipkin) as a single entry point, acting as a small reverse proxy that translates incoming calls to either the distributor or the query-frontends.
- **Distributor** — the first stop on the write path. Receives spans from the gateway, validates and normalizes them with the right labels, and hands them to the ingesters.
- **Ingester** — buffers incoming spans in memory, grouping them into chunks before flushing to the S3 backend.
- **Cache** — a `memcached`-compatible cache keyed by trace ID, speeding up repeated lookups of the same trace.
- **Compactor** — TTL-based cleanup of old objects in the backend, analogous to Loki's compactor.
- **Query-frontend** — receives incoming trace search requests.
- **Querier** — the component that actually queries the S3 object store and returns results back through the query-frontend.

`kubectl describe node` on a node backing the `tempo` NodePool confirms Tempo's pods scheduled there as expected.

> **Note:** the instructor calls out, while inspecting nodes, that one Loki pod also landed on a `tempo`-labeled node — a minor known quirk of how that particular pod's own scheduling constraints resolved, mentioned in passing rather than as a bug to fix.

---

# Lesson 3: Exposing Tempo Through a Network Load Balancer

## 1. Internal NLB and Target Group Binding for the Gateway

Identical recipe to Loki's exposure in Module 19: an internal Network Load Balancer, target-group-bound to the `tempo-gateway` service:

```hcl
# lb_tempo.tf
resource "aws_lb" "tempo" {
  name = format("%s-tempo", var.project_name)

  internal           = true
  load_balancer_type = "network"

  subnets = data.aws_ssm_parameter.lb_subnets[*].value

  enable_cross_zone_load_balancing = true
  enable_deletion_protection       = false

  tags = {
    Name = var.project_name
  }
}

resource "aws_lb_target_group" "tempo" {
  name     = format("%s-tempo", var.project_name)
  port     = 80
  protocol = "TCP"
  vpc_id   = data.aws_ssm_parameter.vpc.value
}

resource "aws_lb_listener" "tempo" {
  load_balancer_arn = aws_lb.tempo.arn
  port              = 80
  protocol          = "TCP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.tempo.arn
  }
}

resource "kubectl_manifest" "tempo" {
  yaml_body = <<YAML
apiVersion: elbv2.k8s.aws/v1beta1
kind: TargetGroupBinding
metadata:
  name: tempo-gateway
  namespace: tempo
spec:
  serviceRef:
    name: tempo-gateway
    port: 80
  targetGroupARN: ${aws_lb_target_group.tempo.arn}
  targetType: instance
YAML
  depends_on = [
    helm_release.alb_ingress_controller,
    helm_release.tempo
  ]
}
```

Same shape as `lb_loki.tf` almost line-for-line — the lesson itself describes this step as close to copy-paste, since the exposure pattern for every Grafana Stack component in this cluster is identical.

## 2. A Private DNS Record for the Tempo Gateway

`aws_route53_record.tempo` — already present in `route53.tf` as of Module 19, flagged there as scaffolding pointing at a load balancer that didn't exist yet — now resolves for real, once `aws_lb.tempo` exists:

```hcl
# route53.tf
resource "aws_route53_record" "tempo" {
  zone_id = aws_route53_zone.private.zone_id
  name    = format("tempo.%s.local", var.project_name)
  type    = "CNAME"
  ttl     = "30"
  records = [aws_lb.tempo.dns_name]
}
```

With `project_name = "linuxtips-observability"`, this resolves internally at `tempo.linuxtips-observability.local` — the same internal address the OpenTelemetry Collector in Lesson 5 forwards spans to.

---

# Lesson 4: Wiring Tempo as a Grafana Data Source

## 1. Adding Tempo to Grafana's `datasources` via `locals`

Configuring Tempo as a Grafana data source follows the exact same pattern as Loki in Module 19 — `access: proxy`, pointed at the internal Kubernetes Service DNS name rather than the NLB:

```yaml
# locals.tf (grafana.values → datasources, Tempo entry)
- name: Tempo
  type: tempo
  access: proxy
  url: http://tempo-gateway.tempo.svc.cluster.local
  basicAuth: false
  jsonData:
    tracesToMetrics:
      datasourceUid: 'Mimir'
    serviceMap:
      datasourceUid: 'Mimir'
    nodeGraph:
      enabled: true
    tracesToLogs:
      datasourceUid: 'Loki'
```

`tracesToLogs.datasourceUid: 'Loki'` is functional right away, since the `Loki` data source already exists from Module 19 — clicking a trace in Tempo can jump straight to its correlated logs. `tracesToMetrics` and `serviceMap` both reference a `Mimir` data source UID that doesn't exist yet.

> **Note:** as flagged in the Overview, `tracesToMetrics`/`serviceMap`'s `Mimir` references are Module 21 scaffolding, not something this lesson builds or explains — only `tracesToLogs` is real, working Module 20 content.

After a `terraform apply` and a refresh of Grafana's **Connections → Data sources** page, the new `Tempo` data source appears, and its **Explore** view is queryable (returning nothing yet, since no traces have been sent to Tempo). The next two lessons build the pipeline that actually gets traces flowing in.

---

# Lesson 5: The OpenTelemetry Collector

## 1. A New Multicluster `ApplicationSet`

A new component, deployed to both workload clusters as an Argo CD `ApplicationSet` using the same `list` generator pattern as Fluent Bit in Module 19: the **OpenTelemetry Collector**, configured to receive spans over OTLP (both gRPC and HTTP) and Zipkin, batch them, and forward them to Tempo:

```yaml
# otel.yml
---
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: opentelemetry-collector
  namespace: argocd
spec:
  generators:
    - list:
        elements:
          - cluster: linuxtips-cluster-01
            shard: "01"
          - cluster: linuxtips-cluster-02
            shard: "02"
  template:
    metadata:
      name: opentelemetry-collector-{{shard}}
    spec:
      project: "system"
      source:
        repoURL: 'https://open-telemetry.github.io/opentelemetry-helm-charts'
        chart: opentelemetry-collector
        targetRevision: "0.122.0"
        helm:
          releaseName: opentelemetry-collector
          valuesObject:
            mode: deployment
            image:
              repository: otel/opentelemetry-collector-k8s
              pullPolicy: IfNotPresent
              tag: 0.122.0
            config:
              receivers:
                otlp:
                  protocols:
                    grpc:
                      endpoint: 0.0.0.0:4317
                    http:
                      endpoint: 0.0.0.0:4318
                zipkin: {}
              processors:
                batch: {}
              exporters:
                otlphttp:
                  endpoint: "http://tempo.linuxtips-observability.local"
                  tls:
                    insecure: true
              service:
                pipelines:
                  traces:
                    receivers:
                      - otlp
                      - zipkin
                    processors:
                      - batch
                    exporters:
                      - otlphttp
      destination:
        name: '{{ cluster }}'
        namespace: opentelemetry
      syncPolicy:
        syncOptions:
          - CreateNamespace=true
        automated: {}
```

`mode: deployment` runs the collector as a `Deployment`, not a `DaemonSet` — every application on the cluster sends spans to it over the network (via its Kubernetes Service), rather than each node running its own local collector instance. The exporter points at `tempo.linuxtips-observability.local` — the private DNS record from Lesson 3 — with `tls.insecure: true`, since traffic never leaves the VPC.

> **Note:** the lesson narration states the chart version as "0.12.0," but the real `targetRevision` in this manifest is `"0.122.0"` — the file is the source of truth. The narration also describes the collector as potentially running "via daemon," which reads as an ambiguous reference to a DaemonSet; the actual, applied value is `mode: deployment`, confirmed both here and in the Terraform-managed copy in Lesson 5 §2 below.

After applying, `kubectl get pods,svc -n opentelemetry` on either workload cluster shows the collector running and its Service ready to receive spans.

## 2. Activating It for Real via Terraform

Exactly like Fluent Bit in Module 19, this `ApplicationSet` also exists as a Terraform-managed resource on the **control-plane** cluster, applied from `linuxtips-eks-multicluster-management` so it survives as part of the cluster's own infrastructure-as-code rather than a manually-applied one-off manifest:

```hcl
# argo_otel.tf (linuxtips-eks-multicluster-management, control-plane stack)
resource "kubectl_manifest" "otel" {

  yaml_body = <<YAML
# ... identical ApplicationSet content to otel.yml above
YAML

  depends_on = [
    helm_release.argocd
  ]

}
```

> **Note:** `argo_otel.tf` already existed in `linuxtips-eks-multicluster-management`'s control-plane stack as of Module 18, flagged there as forward-looking scaffolding pointing at a Tempo endpoint that didn't exist yet. This module is the payoff — `tempo.linuxtips-observability.local` now exists (Lesson 3), so this is the Terraform-managed `ApplicationSet` that's actually running, byte-for-byte identical in content to the standalone `otel.yml` shown above, exactly mirroring how `argo_fluentbit.tf` activated in Module 19.

Applying it from the control-plane cluster changes nothing observable — no new signature, just the same `ApplicationSet` now managed from a different, more durable place — which the lesson confirms by checking that the resulting collector pods on both workload clusters are unchanged.

## 3. Why Applications Still Need Instrumentation

Unlike logs (where Fluent Bit transparently tails container stdout with no application changes needed), trace collection isn't fully transparent. An application has to actively create spans and export them — either through OpenTelemetry SDK libraries wired into the code, or (as in this module's lab) an existing Zipkin-compatible tracing client already built into the application, the same approach already seen in the Jaeger/Istio module. Some auto-instrumentation options exist, but the lesson notes they aren't as mature or convenient as explicit instrumentation. The Health API lab in Lesson 6 uses the explicit approach: it already emits Zipkin-format spans, so the change needed is only to point it at the new OpenTelemetry Collector.

---

# Lesson 6: Deploying the Health API Lab and Exploring Traces

## 1. A Multicluster, Multi-Service Lab

`health-api.yml` deploys a much larger lab than anything used earlier in the course: an `AppProject` named `nutrition`, and one Argo CD `ApplicationSet` per microservice, each replicated across both workload clusters via the same `list` generator pattern used throughout:

```yaml
# health-api.yml (AppProject)
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: nutrition
  namespace: argocd
spec:
  sourceRepos:
  - '*'
  destinations:
  - namespace: '*'
    server: '*'
  clusterResourceWhitelist:
  - group: '*'
    kind: '*'
```

The individual services — `nutrition-bmr`, `nutrition-imc`, `nutrition-calories`, `nutrition-proteins`, `nutrition-water`, `nutrition-recommendations`, and the public-facing `nutrition-health-api` — are all gRPC microservices sharing one Helm chart (`chartmuseum`'s `linuxtips` chart, the same one used for `chip` throughout the ArgoCD modules), each with its own canary rollout, Istio `VirtualService`, and probes. `nutrition-recommendations` calls `proteins`/`water`/`calories` internally, and `nutrition-health-api` — the entry point — calls `bmr`/`imc`/`recommendations`, forming the multi-hop call chain traced in this lesson.

## 2. From Jaeger to the OpenTelemetry Collector

Every one of the Health API's microservices carries a `ZIPKIN_COLLECTOR_ENDPOINT` environment variable, pointed at the new OpenTelemetry Collector's internal cluster address on the standard Zipkin port:

```yaml
# health-api.yml (repeated across every microservice's envs)
- name: ZIPKIN_COLLECTOR_ENDPOINT
  value: http://opentelemetry-collector.opentelemetry.svc.cluster.local:9411/api/v2/spans
```

This is the one line that changed from an earlier module: previously this same environment variable pointed at an internal Jaeger address; now it points at the OpenTelemetry Collector deployed in Lesson 5, on the collector's Zipkin receiver port (`9411`). Everything else about how these services emit spans is unchanged.

`nutrition-health-api`, `nutrition-recommendations`, and `nutrition-bmr`/`nutrition-imc` additionally wire internal service endpoints for the call chain, e.g.:

```yaml
# health-api.yml (nutrition-health-api envs)
- name: BMR_SERVICE_ENDPOINT
  value: "bmr-grpc.nutrition.svc.cluster.local:30000"
- name: IMC_SERVICE_ENDPOINT
  value: "imc-grpc.nutrition.svc.cluster.local:30000"
- name: RECOMMENDATIONS_SERVICE_ENDPOINT
  value: "recommendations-grpc.nutrition.svc.cluster.local:30000"
```

After applying `health-api.yml` from the control-plane cluster, many `ApplicationSet`s and their per-cluster Applications appear at once — BMR, IMC, the calorie calculator, proteins, water, recommendations, and the health-api entry point, each with a `-01`/`-02` shard per workload cluster. The lesson waits for all of them to reach a healthy state, noting that the workload clusters may need to scale up additional nodes (via Karpenter, on the `general` NodePool) to fit everything.

> **Note:** as flagged in Module 19, this file's Istio hosts use the domain `msfidelis.com.br` rather than the course's own `luisgustavo.com.br` used elsewhere. That framing was already correct: this is genuine lab content the instructor reuses from their own prior material, now confirmed as real, intentional Module 20 content — not a stray or unrelated file.

## 3. Generating Traffic and Exploring Traces

With the Health API reachable through the shared ingress ALB (same `Host` header-based routing pattern used throughout the ArgoCD modules — `host.msfidelis.com.br` for the entry point), the lesson repeatedly calls the macro-calculation endpoint to generate traces, using a load-generation approach referred to informally in the recording without enough detail to identify a specific tool by name.

In Grafana's **Explore** view against the `Tempo` data source, filtering by **service name** `health-api` and **span name** shows spans arriving as requests land. Opening one shows the full trace waterfall: `health-api` calls the BMR service (creating a client, calling it, receiving a response), then the IMC service, then `recommendations` — which in turn fans out to `water`, `proteins`, and `calories` before returning. Each span in the waterfall shows its own duration, making it straightforward to see which downstream service is the slowest contributor to the overall request time and which calls are sequential versus independent.

## 4. Adding a Tempo Panel to the Grafana Dashboard

Building on the dashboard created in Module 19, the lesson adds a new panel backed by the `Tempo` data source — a table filtered to the `nutrition` namespace, showing trace/span data (e.g. `health-api`'s logs and a `nutrition-cal-service`-filtered span view) alongside the existing Loki-based panels. The lesson explicitly defers polishing the dashboard's layout and visual organization to a later point, focusing this module only on proving the ingestion and correlation pipeline works end-to-end.

---

## Key Takeaways

- **This module continues the same repository and cluster as Module 19** — `linuxtips-eks-observability-cluster` on `main` — not a new one. Every Tempo/OTel file flagged as forward-looking scaffolding in Modules 18-19 (`helm_tempo.tf`, `iam_tempo.tf`, `s3_tempo.tf`, `lb_tempo.tf`, `otel.yml`, `argo_otel.tf`, the `route53.tf` Tempo record, the `grafana.values` Tempo datasource, the `tempo` NodePool) is real, active content here.
- **Tempo mirrors Loki's architecture almost exactly**, just for traces instead of logs: a gateway fronting distributor/ingester (write path) and query-frontend/querier (read path), S3 as the object store, a trace-ID-keyed cache, and a `tempo-distributed` (microservices) chart chosen over the single-binary `tempo` chart — the same monolithic-vs-distributed choice Loki offered in Module 19.
- **The OpenTelemetry Collector is the new bridge between applications and Tempo.** It runs as a `Deployment` (not a DaemonSet) on both workload clusters, accepting OTLP and Zipkin spans and exporting them to Tempo over OTLP/HTTP — and, exactly like Fluent Bit, its real activated deployment is a Terraform-managed `ApplicationSet` (`argo_otel.tf`) on the control-plane cluster, not the standalone `otel.yml` shown as the "before" example.
- **Trace instrumentation isn't as transparent as log shipping.** Unlike Fluent Bit tailing stdout with zero app changes, every Health API microservice needed its `ZIPKIN_COLLECTOR_ENDPOINT` pointed at the new collector instead of the old Jaeger address — the application itself still does the work of creating and exporting spans.
- **`health-api.yml`, previously flagged as a stray/unrelated file in Module 19, is confirmed as genuine Module 20 content** — a large multicluster lab (`AppProject` + per-microservice `ApplicationSet`s) purpose-built to generate real, multi-hop distributed traces for this module.
- **Forward-looking Mimir scaffolding is already baked into this lesson's own files** — `locals.tf`'s `tempo` block remote-writes metrics to a not-yet-existing Mimir, and the Tempo Grafana datasource's `tracesToMetrics`/`serviceMap` reference a Mimir UID that doesn't resolve until Module 21 — consistent with the same repository-written-once pattern seen throughout.
