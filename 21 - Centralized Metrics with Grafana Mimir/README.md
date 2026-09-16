# Module 21: Centralized Metrics with Grafana Mimir

## Overview

This module adds the fourth and final piece of the Grafana Stack to the same **observability cluster** built across Modules 19-20: **Grafana Mimir**, for long-term, centralized storage and query of Prometheus metrics. It closes the loop the previous two modules opened — logs (Fluent Bit → Loki), traces (OpenTelemetry Collector → Tempo), and now metrics (Prometheus → Mimir) — so that every workload cluster becomes, in the instructor's words, just "a client of observability itself" rather than an island with its own local Prometheus.

All 7 lessons continue directly in the **same repository and cluster** as Modules 19-20 — `linuxtips-eks-observability-cluster`, branch `main` — not a new repository. As flagged in both prior modules, the Mimir-specific Terraform/Helm files (`helm_mimir.tf`, `iam_mimir.tf`, `s3_mimir.tf`, `lb_mimir.tf`), the `mimir` NodePool entry, the `route53.tf` Mimir record, and the `Mimir` Grafana datasource entry were already sitting in the repository as forward-looking scaffolding; this module is what actually builds, explains, and activates them. The one new repository involved is `linuxtips-eks-multicluster-management` — specifically its `control-plane` stack — which is where the Prometheus `ApplicationSet` (`argo_prometheus.tf`, flagged as scaffolding since Module 18) gets activated, exactly mirroring how `argo_fluentbit.tf` (Module 19) and `argo_otel.tf` (Module 20) were each activated in their turn.

Of every component covered in the observability build so far, the instructor calls Mimir out explicitly as **the most complex to instrument**: far more components, far more parametrization, and far greater scaling potential than Loki or Tempo needed. In exchange, the module itself is comparatively short — the complexity lives in the size of a single Helm values block, not in the number of steps.

> **Note:** the Prometheus servers this module deploys are explicitly **not** the full `kube-prometheus-stack` — just the Prometheus server chart itself, configured to scrape each workload cluster and remote-write to Mimir. Alertmanager is disabled, and the chart's own default-enabled Push Gateway is called out in the lesson as unnecessary and safe to remove.

## Table of Contents

- [Lesson 1: Introduction to Grafana Mimir](#lesson-1-introduction-to-grafana-mimir)
  - [1. Metrics as the Third Pillar of Observability](#1-metrics-as-the-third-pillar-of-observability)
  - [2. Monolithic vs. Microservices: Choosing `mimir-distributed`](#2-monolithic-vs-microservices-choosing-mimir-distributed)
- [Lesson 2: Initial Setup](#lesson-2-initial-setup)
  - [1. A Dedicated `mimir` NodePool](#1-a-dedicated-mimir-nodepool)
  - [2. Two S3 Buckets and IAM Roles for Mimir and Its Ruler](#2-two-s3-buckets-and-iam-roles-for-mimir-and-its-ruler)
- [Lesson 3: Deploying Grafana Mimir](#lesson-3-deploying-grafana-mimir)
  - [1. Disabling Enterprise Features and Multi-Tenancy](#1-disabling-enterprise-features-and-multi-tenancy)
  - [2. Hard Limits on Series and Labels](#2-hard-limits-on-series-and-labels)
  - [3. Backend Storage: Blocks and Ruler](#3-backend-storage-blocks-and-ruler)
  - [4. Mimir's Components at a Glance](#4-mimirs-components-at-a-glance)
  - [5. Installing the `mimir` Helm Release](#5-installing-the-mimir-helm-release)
- [Lesson 4: Exposing Mimir Through an Application Load Balancer](#lesson-4-exposing-mimir-through-an-application-load-balancer)
  - [1. Why an ALB Instead of an NLB](#1-why-an-alb-instead-of-an-nlb)
  - [2. ALB, Target Group Binding, and a Private DNS Record](#2-alb-target-group-binding-and-a-private-dns-record)
- [Lesson 5: Wiring Mimir as a Grafana Data Source](#lesson-5-wiring-mimir-as-a-grafana-data-source)
  - [1. A Prometheus Data Source Pointed at Mimir](#1-a-prometheus-data-source-pointed-at-mimir)
  - [2. Mimir Is Passive: It Needs Prometheus Servers to Push Data In](#2-mimir-is-passive-it-needs-prometheus-servers-to-push-data-in)
- [Lesson 6: Deploying Prometheus Servers Across Workload Clusters](#lesson-6-deploying-prometheus-servers-across-workload-clusters)
  - [1. A New Multicluster `ApplicationSet` for Prometheus](#1-a-new-multicluster-applicationset-for-prometheus)
  - [2. Redeploying the Health API Lab for Traffic](#2-redeploying-the-health-api-lab-for-traffic)
- [Lesson 7: Remote Write to Mimir and Cross-Pillar Correlation](#lesson-7-remote-write-to-mimir-and-cross-pillar-correlation)
  - [1. Configuring `remoteWrite` from Prometheus to Mimir](#1-configuring-remotewrite-from-prometheus-to-mimir)
  - [2. Reloading Prometheus and Confirming Metrics Arrive](#2-reloading-prometheus-and-confirming-metrics-arrive)
  - [3. Correlating Metrics Across Clusters via the `cluster` Label](#3-correlating-metrics-across-clusters-via-the-cluster-label)
  - [4. Extending Scrape Configs Beyond the Chart's Defaults](#4-extending-scrape-configs-beyond-the-charts-defaults)
  - [5. Bringing Traces, Logs, and Metrics Together in One Dashboard](#5-bringing-traces-logs-and-metrics-together-in-one-dashboard)
- [Key Takeaways](#key-takeaways)

---

# Lesson 1: Introduction to Grafana Mimir

## 1. Metrics as the Third Pillar of Observability

Grafana Mimir closes out the three-pillar build started in Module 19: Fluent Bit ships logs into Loki, the OpenTelemetry Collector ships traces into Tempo, and — as of this module — a Prometheus server in each workload cluster ships metrics into Mimir. The pattern is the same shape every time: an agent runs inside each workload cluster, and a centralized backend in the observability cluster receives, stores, and serves the data back out through Grafana.

For metrics specifically, that agent is a full Prometheus server deployed into each cluster, doing the same job a standalone Prometheus normally would — service/pod discovery, scraping — except that instead of only storing data locally, it also performs a **remote write**: forwarding every scraped sample to another Prometheus-compatible endpoint. That endpoint is Mimir's gateway, which ingests the write and persists the underlying chunks to an S3 bucket. Once metrics land in Mimir, a Grafana data source pointed at Mimir lets metrics from every cluster be queried and correlated from one place, the same way Loki already does for logs and Tempo for traces.

## 2. Monolithic vs. Microservices: Choosing `mimir-distributed`

Like Loki and Tempo before it, Mimir ships two deployment shapes: a monolithic single-binary mode, and a microservices mode where each component scales independently. This module deploys the microservices chart, `mimir-distributed`, via Helm.

> **Note:** the instructor explicitly flags Mimir as the most complex component instrumented in the course so far — far more components, far more Helm values, and far greater scaling potential than Loki or Tempo — while also noting this makes for a comparatively shorter lesson sequence, since the complexity is concentrated in one large, already-prepared values block rather than spread across many steps.

---

# Lesson 2: Initial Setup

## 1. A Dedicated `mimir` NodePool

Continuing the "one NodePool per observability workload" pattern from Modules 19-20, a new Karpenter NodePool named `mimir` is added to `karpenter_capacity`, same Bottlerocket/shape convention as `grafana`, `loki`, and `tempo`:

```hcl
# environment/prod/terraform.tfvars (karpenter_capacity, mimir entry)
{
  name               = "mimir"
  workload           = "mimir"
  ami_family         = "Bottlerocket"
  ami_ssm            = "/aws/service/bottlerocket/aws-k8s-1.31/x86_64/latest/image_id"
  instance_family    = ["t3", "t3a", "c6", "c6a", "c7", "c7a"]
  instance_sizes     = ["large", "xlarge", "2xlarge"]
  capacity_type      = ["spot", "on-demand"]
  availability_zones = ["us-east-1a", "us-east-1b", "us-east-1c"]
},
```

## 2. Two S3 Buckets and IAM Roles for Mimir and Its Ruler

Unlike Tempo, which needed only one S3 bucket, Mimir needs **two**: one for the ruler component's rules storage (relevant if the deployment is later extended to multi-tenancy), and one for the actual metric block/chunk storage:

```hcl
# s3_mimir.tf
resource "aws_s3_bucket" "mimir" {
  bucket = format("%s-%s-mimir", var.project_name, data.aws_caller_identity.current.account_id)
}

resource "aws_s3_bucket_ownership_controls" "mimir" {
  bucket = aws_s3_bucket.mimir.id
  rule {
    object_ownership = "BucketOwnerPreferred"
  }
}

resource "aws_s3_bucket_acl" "mimir" {
  bucket = aws_s3_bucket.mimir.id
  acl    = "private"

  depends_on = [
    aws_s3_bucket_ownership_controls.mimir
  ]
}

# Ruler

resource "aws_s3_bucket" "mimir_ruler" {
  bucket = format("%s-%s-mimir-ruler", var.project_name, data.aws_caller_identity.current.account_id)
}

resource "aws_s3_bucket_ownership_controls" "mimir_ruler" {
  bucket = aws_s3_bucket.mimir_ruler.id
  rule {
    object_ownership = "BucketOwnerPreferred"
  }
}

resource "aws_s3_bucket_acl" "mimir_ruler" {
  bucket = aws_s3_bucket.mimir_ruler.id
  acl    = "private"

  depends_on = [
    aws_s3_bucket_ownership_controls.mimir_ruler
  ]
}
```

The IAM side follows the same Pod Identity pattern already used for Loki and Tempo, but this time needs **two separate Pod Identity associations** against the same role — one for the `mimir` service account, one for `mimir-ruler`:

```hcl
# iam_mimir.tf
data "aws_iam_policy_document" "mimir_role" {
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

resource "aws_iam_role" "mimir_role" {
  assume_role_policy = data.aws_iam_policy_document.mimir_role.json
  name               = format("%s-mimir", var.project_name)
}

data "aws_iam_policy_document" "mimir_policy" {
  version = "2012-10-17"

  statement {

    effect = "Allow"
    actions = [
      "s3:*",
    ]

    resources = [
      format("%s/*", aws_s3_bucket.mimir.arn),
      format("%s/*", aws_s3_bucket.mimir_ruler.arn),
      aws_s3_bucket.mimir.arn,
      aws_s3_bucket.mimir_ruler.arn,
    ]

  }
}

resource "aws_iam_policy" "mimir_policy" {
  name        = format("%s-mimir", var.project_name)
  path        = "/"
  description = var.project_name

  policy = data.aws_iam_policy_document.mimir_policy.json
}

resource "aws_iam_policy_attachment" "mimir" {
  name = "mimir"
  roles = [
    aws_iam_role.mimir_role.name
  ]

  policy_arn = aws_iam_policy.mimir_policy.arn

}

resource "aws_eks_pod_identity_association" "mimir" {
  cluster_name    = aws_eks_cluster.main.name
  namespace       = "mimir"
  service_account = "mimir"
  role_arn        = aws_iam_role.mimir_role.arn
}

resource "aws_eks_pod_identity_association" "mimir_ruler" {
  cluster_name    = aws_eks_cluster.main.name
  namespace       = "mimir"
  service_account = "mimir-ruler"
  role_arn        = aws_iam_role.mimir_role.arn
}
```

> **Note:** same course-wide convention already flagged for Loki's and Tempo's IAM roles — the policy grants `s3:*` rather than scoping to only the specific actions Mimir needs, and both Pod Identity associations share a single IAM role rather than two dedicated roles.

---

# Lesson 3: Deploying Grafana Mimir

## 1. Disabling Enterprise Features and Multi-Tenancy

Mimir's Helm values are configured through the same `locals.tf` heredoc pattern used for Grafana, Loki, and Tempo. The block opens by disabling Mimir's commercial Enterprise features and its bundled Graphite-compatible database, since this deployment uses pure, open-source Prometheus:

```hcl
# locals.tf (mimir block, excerpt)
enterprise:
    enabled: false
graphite:
    enabled: false
```

## 2. Hard Limits on Series and Labels

Mimir enforces hard limits on cardinality — how many series and labels a tenant can push:

```hcl
# locals.tf (mimir block, excerpt)
mimir:
  structuredConfig:
    limits:
      max_label_names_per_series: 50
      max_global_series_per_user: 150000000
```

These limits are designed to be per-tenant in a true multi-tenant Mimir deployment; since this setup uses a single generic, unauthenticated user rather than configuring multi-tenancy, the limits become effectively global across every metric this Mimir instance receives. The lesson notes this is the kind of limit that large environments with many custom application metrics can realistically hit.

## 3. Backend Storage: Blocks and Ruler

The `common.storage`, `blocks_storage`, and `ruler_storage` keys point Mimir at the two S3 buckets created in Lesson 2:

```hcl
# locals.tf (mimir block, excerpt)
    common:
      storage:
        backend: s3
        s3:
          endpoint: s3.${var.region}.amazonaws.com
          bucket_name: ${aws_s3_bucket.mimir.id}
          insecure: false
    blocks_storage:
      backend: s3
      s3:
        endpoint: s3.${var.region}.amazonaws.com
        bucket_name: ${aws_s3_bucket.mimir.id}
        insecure: false
    ruler_storage:
      backend: s3
      s3:
        endpoint: s3.${var.region}.amazonaws.com
        bucket_name: ${aws_s3_bucket.mimir_ruler.id}
        insecure: false

alertmanager:
  enabled: false
```

Alertmanager is explicitly disabled, since this deployment isn't using Mimir's alerting features.

## 4. Mimir's Components at a Glance

The remainder of the `mimir` block configures every one of Mimir's many components. Each gets its own replica count, resource requests/limits, and — consistently — a `nodeSelector` pinning it to the new `mimir` NodePool so its capacity doesn't mix with other observability workloads:

```hcl
# locals.tf (mimir block, excerpt: compactor, distributor, ingester)
compactor:
  persistentVolume:
    storageClass: gp3
    size: 20Gi
  resources:
    limits:
      memory: 2Gi
    requests:
      cpu: 1
      memory: 1Gi
  nodeSelector:
    karpenter.sh/nodepool: mimir

distributor:
  replicas: 3
  resources:
    limits:
      memory: 5.7Gi
    requests:
      cpu: 2
      memory: 4Gi
  nodeSelector:
    karpenter.sh/nodepool: mimir
  persistence:
    storageClass: gp3

ingester:
  persistentVolume:
    storageClass: gp3
    size: 50Gi
  replicas: 3
  resources:
    limits:
      memory: 10Gi
    requests:
      cpu: 2
      memory: 4Gi
  nodeSelector:
    karpenter.sh/nodepool: mimir

  zoneAwareReplication:
    enabled: false
```

- **Compactor** — manages TTL and long-term storage housekeeping, backed by a GP3 volume.
- **Distributor** — the write path's entry point, receiving and validating incoming remote-write samples.
- **Ingester** — persists incoming data to a GP3 volume before it's flushed to S3. Zone-aware replication is explicitly disabled here — in production, this component would normally replicate written data across zones, but that isn't necessary for this deployment.

Four caches follow, all backed by GP3 and pinned to the `mimir` NodePool:

```hcl
# locals.tf (mimir block, excerpt: caches)
admin-cache:
  enabled: false
  replicas: 3

chunks-cache:
  enabled: true
  replicas: 3
  nodeSelector:
    karpenter.sh/nodepool: mimir
  persistence:
    storageClass: gp3

index-cache:
  enabled: true
  replicas: 3
  nodeSelector:
    karpenter.sh/nodepool: mimir
  persistence:
    storageClass: gp3

metadata-cache:
  enabled: true
  replicas: 3
  nodeSelector:
    karpenter.sh/nodepool: mimir
  persistence:
    storageClass: gp3

results-cache:
  enabled: true
  replicas: 3
  nodeSelector:
    karpenter.sh/nodepool: mimir
  persistence:
    storageClass: gp3
```

**Admin cache** is disabled (unused in this setup). **Chunks cache**, **index cache**, and **metadata cache** all speed up lookups against label/data indexing and are important for performance given the volume of data Mimir handles. **Results cache** has a shorter TTL than the others but is equally important, since it protects the query backend from being overwhelmed by repeated identical queries.

The read path and remaining components:

```hcl
# locals.tf (mimir block, excerpt: querier, query_frontend, ruler, store_gateway, nginx, gateway)
querier:
  replicas: 1
  resources:
    limits:
      memory: 6Gi
    requests:
      cpu: 2
      memory: 4Gi
  nodeSelector:
    karpenter.sh/nodepool: mimir
query_frontend:
  replicas: 1
  resources:
    limits:
      memory: 3Gi
    requests:
      cpu: 2
      memory: 2Gi
  nodeSelector:
    karpenter.sh/nodepool: mimir
ruler:
  replicas: 1
  serviceAccount:
    create: true
  resources:
    limits:
      memory: 3Gi
    requests:
      cpu: 1
      memory: 2Gi
  nodeSelector:
    karpenter.sh/nodepool: mimir
store_gateway:
  persistentVolume:
    storageClass: gp3
    size: 10Gi
  replicas: 3
  resources:
    limits:
      memory: 2Gi
    requests:
      cpu: 1
      memory: 1Gi
  nodeSelector:
    karpenter.sh/nodepool: mimir
  topologySpreadConstraints: {}
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchExpressions:
              - key: target # support for enterprise.legacyLabels
                operator: In
                values:
                  - store-gateway
          topologyKey: 'kubernetes.io/hostname'
        - labelSelector:
            matchExpressions:
              - key: app.kubernetes.io/component
                operator: In
                values:
                  - store-gateway
          topologyKey: 'kubernetes.io/hostname'
  zoneAwareReplication:
    topologyKey: 'kubernetes.io/hostname'

nginx:
  replicas: 3
  resources:
    limits:
      memory: 1Gi
    requests:
      cpu: 1
      memory: 512Mi
  service:
    type: NodePort
  nodeSelector:
    karpenter.sh/nodepool: mimir

gateway:
  replicas: 3
  resources:
    limits:
      memory: 1Gi
    requests:
      cpu: 1
      memory: 512Mi
  nodeSelector:
    karpenter.sh/nodepool: mimir
```

- **Querier** and **query-frontend** make up the read path, sized larger than most other components since they handle bigger requests and query volumes.
- **Ruler** handles user/tenant routing for the rules feature.
- **Store-gateway** is storage-facing and, per the lesson, needs to be sized larger in a real production deployment. Its `podAntiAffinity` (keyed on `kubernetes.io/hostname`) explicitly prevents multiple store-gateway replicas from landing on the same node — important because this component drives heavy write I/O and colliding on one host would create a bottleneck.
- **Nginx** acts as Mimir's ingress/gateway equivalent — the single Layer 7 entry point handling both the write path (remote-write ingestion) and the read path (queries), exposed as a `NodePort` Service so it can later be bound to a load balancer's target group.

> **Note:** the values block also defines a separate `gateway` key alongside `nginx`, with matching resources and `nodeSelector` but no `service.type` override. The Target Group Binding built in Lesson 4 references the `mimir-nginx` Service by name, confirming `nginx` is the component actually serving as Mimir's real ingress; the `gateway` key appears to be inactive/redundant configuration rather than something this module wires up or explains.

Finally, every component's `nodeSelector` is set to `karpenter.sh/nodepool: mimir`, and pods are spread across hosts by hostname to avoid write-I/O collisions — the same anti-collision reasoning already applied to Loki's write path in Module 19.

## 5. Installing the `mimir` Helm Release

```hcl
# helm_mimir.tf
resource "helm_release" "mimir" {
  name       = "mimir"
  chart      = "mimir-distributed"
  repository = "https://grafana.github.io/helm-charts"
  namespace  = "mimir"

  create_namespace = true

  values = [
    local.mimir["values"]
  ]

  timeout = 900

  depends_on = [
    helm_release.karpenter,
    aws_eks_pod_identity_association.mimir,
    aws_eks_addon.ebs_csi
  ]
}
```

The `timeout` is bumped to 900 seconds — Mimir's sheer number of components makes it noticeably slower to fully roll out than Loki or Tempo. After it finishes, `kubectl get pods,pvc -n mimir` shows every component from Lesson 3 §4 running, several having restarted once or twice during startup (expected), each backed by its own GP3-provisioned PVC at the size configured above.

---

# Lesson 4: Exposing Mimir Through an Application Load Balancer

## 1. Why an ALB Instead of an NLB

Loki and Tempo were both exposed through internal **Network Load Balancers**. Even though Mimir's `nginx` component also operates at Layer 7 like Loki's and Tempo's gateways, Mimir's own documentation recommends exposing it through an **Application Load Balancer** instead — the one deviation from the otherwise identical exposure recipe used for every other Grafana Stack component in this cluster.

## 2. ALB, Target Group Binding, and a Private DNS Record

```hcl
# lb_mimir.tf
resource "aws_lb" "mimir" {

  name = format("%s-mimir", var.project_name)

  internal           = true
  load_balancer_type = "application"

  subnets = data.aws_ssm_parameter.lb_subnets[*].value

  security_groups = [
    aws_security_group.mimir.id
  ]

  enable_cross_zone_load_balancing = true
  enable_deletion_protection       = false

  tags = {
    Name = var.project_name
  }

}

resource "aws_lb_target_group" "mimir" {
  name     = format("%s-mimir", var.project_name)
  port     = 80
  protocol = "HTTP"
  vpc_id   = data.aws_ssm_parameter.vpc.value
}

resource "aws_lb_listener" "mimir" {
  load_balancer_arn = aws_lb.mimir.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.mimir.arn
  }
}

resource "aws_security_group" "mimir" {
  name = format("%s-mimir", var.project_name)

  vpc_id = data.aws_ssm_parameter.vpc.value

  ingress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = var.project_name
  }
}

resource "kubectl_manifest" "mimir" {
  yaml_body = <<YAML
apiVersion: elbv2.k8s.aws/v1beta1
kind: TargetGroupBinding
metadata:
  name: mimir-nginx
  namespace: mimir
spec:
  serviceRef:
    name: mimir-nginx
    port: 80
  targetGroupARN: ${aws_lb_target_group.mimir.arn}
  targetType: instance
YAML
  depends_on = [
    helm_release.alb_ingress_controller,
    helm_release.mimir
  ]
}
```

Like Loki's and Tempo's load balancers, this ALB is `internal = true`, deployed into the private subnets — still not internet-facing, just a different load balancer type than the NLB pattern used so far. The `TargetGroupBinding` associates the ALB's target group with the `mimir-nginx` Service, which is why that Service had to be `NodePort` in Lesson 3's values.

> **Note:** unlike Tempo's and Loki's load balancers (which attach no explicit security group), Mimir's ALB gets its own `aws_security_group` allowing all traffic (`0.0.0.0/0`, all ports/protocols) both inbound and outbound. Since the ALB is internal-only and sits in private subnets, this is broader than strictly necessary — in the same spirit as the course-wide `s3:*` IAM convention already flagged elsewhere — rather than a real exposure risk.

The private DNS record follows the exact same pattern as Loki's and Tempo's, already present in `route53.tf` as of Module 19 as forward-looking scaffolding:

```hcl
# route53.tf
resource "aws_route53_record" "mimir" {
  zone_id = aws_route53_zone.private.zone_id
  name    = format("mimir.%s.local", var.project_name)
  type    = "CNAME"
  ttl     = "30"
  records = [aws_lb.mimir.dns_name]
}
```

With `project_name = "linuxtips-observability"`, this resolves internally at `mimir.linuxtips-observability.local`. The lesson confirms this from inside the cluster with a throwaway `debian`-based shell pod (`kubectl run` + `apt install dnsutils`), resolving the record to three IPs — one per AZ — behind the ALB.

---

# Lesson 5: Wiring Mimir as a Grafana Data Source

## 1. A Prometheus Data Source Pointed at Mimir

Mimir speaks the Prometheus remote-write and query API, so its Grafana data source isn't a dedicated "Mimir" type — it's a standard **Prometheus** data source, just pointed at Mimir's internal endpoint instead of a real Prometheus server:

```yaml
# locals.tf (grafana.values → datasources, Mimir entry)
- name: Mimir
  type: prometheus
  access: proxy
  url: http://mimir-nginx.mimir.svc.cluster.local:80/prometheus
  isDefault: true
  jsonData:
    prometheusType: Mimir
```

This entry already existed in `locals.tf`'s `grafana` block as of Module 19, flagged there and in Module 20 as scaffolding pointing at a Mimir instance that didn't exist yet — this lesson is what finally makes it resolve. `access: proxy` and the internal Kubernetes Service DNS name (`mimir-nginx.mimir.svc.cluster.local`) follow the same convention already used for the Loki and Tempo data sources, reached through the `/prometheus` path suffix that Mimir's `nginx` component exposes. `jsonData.prometheusType: Mimir` is set for future data-source correlation features (e.g. exemplar linking), which this module doesn't configure further.

After a `terraform apply` and a refresh of Grafana's **Connections → Data sources** page, a new `Mimir` data source appears, backed by the Prometheus query type — meaning its **Explore** view uses PromQL, the same query language as any other Prometheus data source.

## 2. Mimir Is Passive: It Needs Prometheus Servers to Push Data In

Opening the new `Mimir` data source in Grafana's **Explore** view returns nothing yet. Unlike Loki or Tempo, Mimir never scrapes anything itself — it only receives whatever gets written to it. The Prometheus servers that will actually collect metrics from each workload cluster and push them into Mimir via remote write haven't been deployed yet; that's the subject of the next two lessons.

---

# Lesson 6: Deploying Prometheus Servers Across Workload Clusters

## 1. A New Multicluster `ApplicationSet` for Prometheus

Continuing the exact deployment model already used for Fluent Bit and the OpenTelemetry Collector, a Prometheus server is deployed to each workload cluster through an Argo CD `ApplicationSet` on the **control-plane** cluster, defined in `linuxtips-eks-multicluster-management`'s `control-plane` stack:

```hcl
# argo_prometheus.tf (linuxtips-eks-multicluster-management, control-plane stack)
resource "kubectl_manifest" "prometheus" {

  yaml_body = <<YAML
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: prometheus
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
      name: prometheus-{{shard}}
    spec:
      project: "system"
      source:
        repoURL: 'https://prometheus-community.github.io/helm-charts'
        chart: prometheus
        targetRevision: 27.11.0
        helm:
          releaseName: prometheus
          valuesObject:
            server:
              global:
                scrape_interval: 15s
                evaluation_interval: 15s
                external_labels:
                    cluster: "{{ cluster }}"
              persistentVolume:
                enabled: false
              remoteWrite:
                - url: "http://mimir.linuxtips-observability.local:80/api/v1/push"
                  queue_config:
                    max_samples_per_send: 1000
                    max_shards: 20
                    capacity: 5000

              extraScrapeConfigs: |
                - job_name: "envoy-stats-monitor"
                  honor_labels: true
                  scrape_interval: 15s
                  kubernetes_sd_configs:
                    - role: pod
                  relabel_configs:
                    - action: drop
                      source_labels: [__meta_kubernetes_pod_label_istio_prometheus_ignore]
                      regex: .+
                    - action: keep
                      source_labels: [__meta_kubernetes_pod_container_name]
                      regex: "istio-proxy"
                    - action: keep
                      source_labels: [__meta_kubernetes_pod_annotationpresent_prometheus_io_scrape]
                      regex: "true"
                    - action: replace
                      source_labels:
                        - __meta_kubernetes_pod_annotation_prometheus_io_port
                        - __meta_kubernetes_pod_ip
                      regex: (\d+);(([A-Fa-f0-9]{1,4}::?){1,7}[A-Fa-f0-9]{1,4})
                      replacement: '[$2]:$1'
                      target_label: __address__
                    - action: replace
                      source_labels:
                        - __meta_kubernetes_pod_annotation_prometheus_io_port
                        - __meta_kubernetes_pod_ip
                      regex: (\d+);((([0-9]+?)(\.|$)){4})
                      replacement: $2:$1
                      target_label: __address__
                    - action: labeldrop
                      regex: "__meta_kubernetes_pod_label_(.+)"
                    - action: replace
                      source_labels: [__meta_kubernetes_namespace]
                      target_label: namespace
                    - action: replace
                      source_labels: [__meta_kubernetes_pod_name]
                      target_label: pod_name
                  metrics_path: /stats/prometheus
            prometheus-node-exporter:
              enabled: true
            alertmanager:
              enabled: false
      destination:
        name: '{{ cluster }}'
        namespace: prometheus
      syncPolicy:
        syncOptions:
          - CreateNamespace=true
        automated: {}
YAML

  depends_on = [
    helm_release.argocd
  ]
}
```

`argo_prometheus.tf` already existed in the control-plane stack as of Module 18, flagged there as forward-looking scaffolding pointing at Mimir/Prometheus endpoints that didn't exist yet. This lesson is the payoff: with Mimir now reachable at `mimir.linuxtips-observability.local` (Lesson 4), applying this `ApplicationSet` for real deploys the Prometheus server chart — **not** the full `kube-prometheus-stack` — to both `linuxtips-cluster-01` and `linuxtips-cluster-02`, using the same `list` generator/shard pattern as Fluent Bit and the OpenTelemetry Collector.

A few configuration choices stand out:

- **`scrape_interval`/`evaluation_interval: 15s`** — the global scrape cadence.
- **`external_labels.cluster: "{{ cluster }}"`** — every metric this Prometheus instance collects gets tagged with its own cluster name, mirroring the `cluster={{cluster}}` label fix already applied to Fluent Bit in Module 19. This is what makes it possible to tell `linuxtips-cluster-01`'s metrics apart from `linuxtips-cluster-02`'s once both land in Mimir.
- **`persistentVolume.enabled: false`** — a deliberate design choice: this Prometheus instance is meant to act only as a stateless relay to Mimir, not a long-term store. The lesson notes that if metric loss were unacceptable, this should be `true` instead, so the instance could recover its own buffered data on restart rather than losing whatever hadn't yet been remote-written.
- **`alertmanager.enabled: false`** — Alertmanager isn't used here, same as on Mimir's own side.

`extraScrapeConfigs` and `remoteWrite` are covered in Lesson 7. After applying, two `Application` resources (`prometheus-01`, `prometheus-02`) appear in Argo CD, and `kubectl get pods -n prometheus` on either workload cluster shows the Prometheus server, a Node Exporter DaemonSet, kube-state-metrics, and a Push Gateway.

> **Note:** the values shown don't explicitly configure `prometheus-node-exporter` beyond `enabled: true`, nor do they reference kube-state-metrics or the Push Gateway at all — both ship enabled by the `prometheus` chart's own defaults. The lesson calls out the Push Gateway specifically as unnecessary for this setup and safe to remove if desired; kube-state-metrics is kept, since it aggregates useful additional cluster-level metrics.

## 2. Redeploying the Health API Lab for Traffic

To have something generating metrics worth looking at, the lesson re-applies the Health API `ApplicationSet` from Module 20 (`health-api.yml`) from the control-plane cluster — the same multi-service gRPC lab used to generate distributed traces in the previous module, reused here unchanged as the metrics workload. Once every service is healthy again across both workload clusters, the same infinite request loop against the public entry point used in Module 20 is restarted, to keep producing request traffic while the remote-write pipeline is wired up in Lesson 7.

---

# Lesson 7: Remote Write to Mimir and Cross-Pillar Correlation

## 1. Configuring `remoteWrite` from Prometheus to Mimir

The key that makes each workload cluster's Prometheus forward its collected metrics into Mimir is `remoteWrite`, under the Helm chart's `server` values (already shown in full in Lesson 6):

```yaml
# argo_prometheus.tf (server.remoteWrite, excerpt)
remoteWrite:
  - url: "http://mimir.linuxtips-observability.local:80/api/v1/push"
    queue_config:
      max_samples_per_send: 1000
      max_shards: 20
      capacity: 5000
```

Setting this key alone is enough for Prometheus to self-configure remote write — no additional wiring is needed. `queue_config.max_samples_per_send` caps how many samples get batched per send, avoiding overwhelming Mimir's ingestion endpoint, which the lesson notes already receives a large volume of requests under normal operation.

> **Note:** this URL is the **external** Route 53 record behind Mimir's ALB (`mimir.linuxtips-observability.local`, from Lesson 4) — not the internal Kubernetes Service DNS name (`mimir-nginx.mimir.svc.cluster.local`) that Tempo's own `metricsGenerator.config.storage.remote_write` uses (Module 20). The difference is where each writer runs: Tempo's metrics generator runs inside the same cluster as Mimir, so it can reach the internal Service directly, while these Prometheus servers run in the separate workload clusters (`linuxtips-cluster-01`/`02`) and can only reach Mimir through its externally-resolvable (but still VPC-internal) load balancer address.

## 2. Reloading Prometheus and Confirming Metrics Arrive

After re-applying the control-plane `ApplicationSet` with the `remoteWrite` key added, the lesson inspects the Prometheus server's `config-reloader` sidecar logs on `linuxtips-cluster-02` first: it detects the ConfigMap change but doesn't reload the running Prometheus server process automatically, requiring a manual `kubectl delete pod` on the Prometheus server pod to pick up the new configuration. Its logs then confirm the remote-write target is `linuxtips-observability.local`. Checking the same thing on `linuxtips-cluster-01`, the config reload had already happened on its own.

> **Note:** the lesson observes this inconsistency in passing — one cluster's Prometheus needed a manual pod restart to pick up the new remote-write config, the other reloaded automatically — without identifying a root cause. It's presented as an observed quirk of that particular run, not a bug to fix.

Back in Grafana's **Explore** view against the `Mimir` data source, metrics are now arriving from both clusters.

## 3. Correlating Metrics Across Clusters via the `cluster` Label

Because every metric carries the `external_labels.cluster` tag set in Lesson 6, the same metric name can now be filtered by the `cluster` label to distinguish data coming from `linuxtips-cluster-01` versus `linuxtips-cluster-02` — both feeding the same Mimir instance simultaneously. The lesson frames this as the real payoff of the whole build: with this label-based correlation in place, the same approach scales to 10, 15, or 20 clusters all hanging off one central observability location, with every cluster's metrics comparable and cross-referenceable from a single pane of glass. Observability is no longer scoped per-cluster — each cluster is simply a client of the shared observability stack.

## 4. Extending Scrape Configs Beyond the Chart's Defaults

The simplified Prometheus Helm chart deployment used here doesn't support creating a `PodMonitor` custom resource directly the way a full `kube-prometheus-stack` deployment would. To reach beyond the chart's built-in scrape targets, the lesson instead uses `extraScrapeConfigs` (shown in full in Lesson 6), reusing the exact same scrape configuration as the Envoy-sidecar `PodMonitor` created earlier in the course — the same job name, relabeling rules, and `/stats/prometheus` metrics path, just expressed as a raw scrape config block instead of a CRD. This demonstrates the general mechanism for extending what a chart-managed Prometheus scrapes without needing operator-style CRDs.

## 5. Bringing Traces, Logs, and Metrics Together in One Dashboard

The lesson closes the three-module observability build by adding panels from all three pillars into a single Grafana dashboard:

- A **Tempo**-backed traces panel, reusing the Health API's OTel-instrumented trace data from Module 20 (`health-api`/`nutrition-cal-service`).
- A **Loki**-backed logs panel, filtered to `namespace=nutrition` with `container != proxy` to exclude Istio sidecar noise.
- A **Mimir**-backed metrics panel, showing a request-rate/TPS query (`istio_requests_total`) against the same service.

Having all three inside one dashboard is framed as the payoff of the whole observability build across Modules 19-21: logs, traces, and metrics, correlated in one place, for any of the fleet's clusters. The lesson explicitly defers further refinement — linking data sources together, dashboard visual polish, and defining SLOs/SLIs — to a following lesson on dashboard aesthetics and service-level metrics.

> **Note:** only 7 lesson files exist for this module (`lesson01.txt` through `lesson07.txt`), so the "next lesson" the instructor references for dashboard polish and SLOs/SLIs isn't part of the source material currently available — this closing point is where Module 21's transcripts end, not a sign of missing content within what exists.

---

## Key Takeaways

- **This module continues the same repository and cluster as Modules 19-20** — `linuxtips-eks-observability-cluster` on `main` — not a new one. Every Mimir file flagged as forward-looking scaffolding since Module 19 (`helm_mimir.tf`, `iam_mimir.tf`, `s3_mimir.tf`, `lb_mimir.tf`, the `route53.tf` Mimir record, the `grafana.values` Mimir datasource, the `mimir` NodePool, and Tempo's `metricsGenerator` remote-write target) is real, active content here — as is `argo_prometheus.tf` in `linuxtips-eks-multicluster-management`, flagged as scaffolding since Module 18.
- **Mimir is the most complex component in the observability build.** Its `mimir-distributed` Helm chart spans a dozen-plus independently-scaled components (compactor, distributor, ingester, four caches, querier, query-frontend, ruler, store-gateway, nginx), two S3 buckets, and two Pod Identity associations against one shared role — far more surface area than Loki or Tempo needed.
- **Mimir is purely passive** — it never scrapes anything itself. Every metric it holds arrives via remote write from a Prometheus server running elsewhere, which is why Lessons 6-7 exist: without a Prometheus in each workload cluster configured with `remoteWrite`, the `Mimir` Grafana data source (wired up in Lesson 5) has nothing to show.
- **Mimir's Grafana data source is a `prometheus`-type data source, not a dedicated "Mimir" type** — pointed at Mimir's internal `nginx` Service at a `/prometheus` path, and queried with the same PromQL as any other Prometheus source.
- **Mimir is the one exception to this cluster's NLB exposure pattern.** Despite its `nginx` component also operating at Layer 7, Mimir's documentation recommends — and this module uses — an Application Load Balancer instead of the internal Network Load Balancer used for Loki's and Tempo's gateways.
- **Each workload cluster's Prometheus is a lightweight, stateless relay, not a full monitoring stack.** Only the Prometheus server chart is deployed (not `kube-prometheus-stack`), with `persistentVolume: false` and Alertmanager disabled — deliberately trading local durability for simplicity, since the real long-term store is Mimir.
- **The `cluster` external label is what makes centralized metrics useful.** Tagging every scraped metric with its source cluster's name is what lets Grafana Explore (and any future dashboard/alert) distinguish and correlate data from an arbitrary number of clusters feeding into one Mimir instance — the same `cluster={{cluster}}` labeling idea already used for Fluent Bit's logs in Module 19.
- **The three-module observability build (Modules 19-21) ends with all three pillars correlated in one Grafana dashboard** — traces from Tempo, logs from Loki, and metrics from Mimir, all sourced from the same Health API lab traffic — with dashboard polish and SLOs/SLIs explicitly deferred beyond what this module's source material covers.
