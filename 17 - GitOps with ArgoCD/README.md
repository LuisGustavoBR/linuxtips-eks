# Module 17: GitOps with ArgoCD

## Overview

Module 16 ended with a single artifact: `linuxtips-0.1.0.tgz`, a Helm chart that encodes every platform requirement built up over the previous modules — Rollouts, Istio, KEDA, ServiceMonitors — behind a set of `values.yaml` toggles. This module puts that chart to work. It installs Argo CD to own continuous delivery inside the cluster, and ChartMuseum — a deliberately minimalist, community-run Helm chart registry — to host the `linuxtips` chart internally, backed by an S3 bucket instead of a heavier registry like Harbor or JFrog. From the very first deploy, every workload is delivered through an `ApplicationSet` rather than a plain `Application`, even though only one cluster exists today: the same generator-based model is what makes multicluster, active-active delivery possible later in the course without changing how a workload is described. Two labs prove the pattern out — the `chip` canary from Module 15/16, and a much larger multi-service "nutrition" health-API stack — before the module closes by installing the community Rollout extension so that Argo CD's own dashboard can drive canary promotions directly, with no separate Argo Rollouts dashboard needed.

## Table of Contents

- [Lesson 1: GitOps, Argo CD, and ChartMuseum](#lesson-1-gitops-argo-cd-and-chartmuseum)
  - [1. Continuous Delivery Inside the Cluster](#1-continuous-delivery-inside-the-cluster)
  - [2. Application vs. ApplicationSet](#2-application-vs-applicationset)
  - [3. Why ApplicationSets From Day One](#3-why-applicationsets-from-day-one)
  - [Key Takeaways](#key-takeaways)
- [Lesson 2: Deploying ChartMuseum on S3](#lesson-2-deploying-chartmuseum-on-s3)
  - [1. The S3 Bucket](#1-the-s3-bucket)
  - [2. A Pod Identity Role Scoped to the Bucket](#2-a-pod-identity-role-scoped-to-the-bucket)
  - [3. Installing ChartMuseum With an S3 Backend](#3-installing-chartmuseum-with-an-s3-backend)
  - [4. Uploading the Chart Directly to S3](#4-uploading-the-chart-directly-to-s3)
  - [Key Takeaways](#key-takeaways-1)
- [Lesson 3: Installing Argo CD](#lesson-3-installing-argo-cd)
  - [1. A Minimal `helm_release`](#1-a-minimal-helm_release)
  - [2. What Each Argo CD Component Does](#2-what-each-argo-cd-component-does)
  - [Key Takeaways](#key-takeaways-2)
- [Lesson 4: Exposing the Dashboard](#lesson-4-exposing-the-dashboard)
  - [1. An Istio Gateway and VirtualService for Argo CD](#1-an-istio-gateway-and-virtualservice-for-argo-cd)
  - [2. Retrieving and Changing the Initial Admin Password](#2-retrieving-and-changing-the-initial-admin-password)
  - [Key Takeaways](#key-takeaways-3)
- [Lesson 5: Deploying `chip` as an ApplicationSet](#lesson-5-deploying-chip-as-an-applicationset)
  - [1. Pointing an ApplicationSet at ChartMuseum](#1-pointing-an-applicationset-at-chartmuseum)
  - [2. `valuesObject` Instead of a `values.yaml` File](#2-valuesobject-instead-of-a-valuesyaml-file)
  - [3. Watching the Sync and the Rendered Resources](#3-watching-the-sync-and-the-rendered-resources)
  - [4. Enabling Pod Exec From the Dashboard](#4-enabling-pod-exec-from-the-dashboard)
  - [Key Takeaways](#key-takeaways-4)
- [Lesson 6: An AppProject and a Multi-Service Lab](#lesson-6-an-appproject-and-a-multi-service-lab)
  - [1. AppProjects for Context Segregation](#1-appprojects-for-context-segregation)
  - [2. Six ApplicationSets, One Chart](#2-six-applicationsets-one-chart)
  - [3. Filtering the Dashboard by Project](#3-filtering-the-dashboard-by-project)
  - [Key Takeaways](#key-takeaways-5)
- [Lesson 7: The Rollout Extension](#lesson-7-the-rollout-extension)
  - [1. Why Extend Argo CD Instead of Using the Rollouts Dashboard](#1-why-extend-argo-cd-instead-of-using-the-rollouts-dashboard)
  - [2. Enabling Extensions on the `helm_release`](#2-enabling-extensions-on-the-helm_release)
  - [3. Watching a Canary Promote From Inside Argo CD](#3-watching-a-canary-promote-from-inside-argo-cd)
  - [Key Takeaways](#key-takeaways-6)

---

# Lesson 1: GitOps, Argo CD, and ChartMuseum

## 1. Continuous Delivery Inside the Cluster

Argo CD's job is continuous delivery *inside* Kubernetes clusters — deliberately plural, because multicluster support is one of its core strengths and something this course returns to in a later module. The most common way to run it is Git-driven: Argo CD watches a Git repository, keeps track of the desired state described there, and continuously reconciles the live cluster state against it. If someone changes a resource by hand, Argo CD's control loop detects the drift and self-heals it back to the declared state after a configurable interval.

This module puts Argo CD to work on top of what was built in Module 16: the `linuxtips` Helm chart, which already standardizes Istio exposure, KEDA-based autoscaling, canary Rollouts, and ServiceAccounts for every workload. Rather than pointing Argo CD at a Git repo full of raw manifests, this module hosts that chart in ChartMuseum — a simplified, community-maintained Helm chart registry — and has Argo CD consume it from there. ChartMuseum is a lighter alternative to a full artifact registry like Harbor or JFrog Artifactory for teams that only need an internal Helm chart store and don't want to run a much larger installation just for that.

## 2. Application vs. ApplicationSet

Argo CD offers two ways to describe a deploy: an `Application`, which manages a single deploy, and an `ApplicationSet`, which uses generators to template out many `Application`s from one definition — useful for deploying the same workload across many similar destinations at once.

## 3. Why ApplicationSets From Day One

Even though this module only targets a single cluster, every deploy from this point on uses an `ApplicationSet`. The reasoning is forward-looking: a later module in this course builds an active-active, multicluster architecture, where the same application (potentially with per-cluster overrides like a different database endpoint) needs to be mirrored across several clusters and managed as a kind of federation. An `ApplicationSet`'s list generator already supports enumerating multiple cluster destinations — today's list just happens to contain one element — so adopting the pattern now avoids a rewrite later.

> **Note:** this lesson is intentionally introductory. Argo CD has far more capability than what's covered here — the instructor explicitly points to LinuxTips' own dedicated Argo CD course for a deeper treatment, and frames this module as a platform-integration pass rather than a full tour of the tool.

## Key Takeaways

- Argo CD manages continuous delivery inside one or more Kubernetes clusters, most commonly by reconciling against a Git repository, with self-healing drift correction built in.
- ChartMuseum is used here as a minimalist, S3-backed internal Helm chart registry — an alternative to heavier options like Harbor or JFrog.
- `Application` deploys one workload; `ApplicationSet` uses generators to template many `Application`s from one definition.
- Every deploy in this module goes through an `ApplicationSet`, even with a single cluster today, because the list generator is what makes multicluster/active-active delivery possible later without changing the deploy shape.

---

# Lesson 2: Deploying ChartMuseum on S3

## 1. The S3 Bucket

ChartMuseum's persistence layer for this lab is an S3 bucket, created with the same minimal three-resource pattern used for other buckets earlier in the course: the bucket itself, ownership controls, and an ACL.

```hcl
# s3_chartmuseum.tf
resource "aws_s3_bucket" "chartmuseum" {
  bucket = format("%s-%s-chartmuseum", var.project_name, data.aws_caller_identity.current.account_id)
}

resource "aws_s3_bucket_ownership_controls" "chartmuseum" {
  bucket = aws_s3_bucket.chartmuseum.id
  rule {
    object_ownership = "BucketOwnerPreferred"
  }
}

resource "aws_s3_bucket_acl" "chartmuseum" {
  bucket = aws_s3_bucket.chartmuseum.id
  acl    = "private"

  depends_on = [
    aws_s3_bucket_ownership_controls.chartmuseum
  ]
}
```

## 2. A Pod Identity Role Scoped to the Bucket

ChartMuseum authenticates to that bucket the same way every other component in this course has since Module 9: EKS Pod Identity rather than IRSA. A trust policy lets the Pod Identity service assume the role, a permission policy scopes `s3:*` to just the ChartMuseum bucket and its objects, and a Pod Identity association binds the role to a `chartmuseum` ServiceAccount in a `chartmuseum` namespace.

```hcl
# iam_chartmuseum.tf
data "aws_iam_policy_document" "chartmuseum_role" {
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

resource "aws_iam_role" "chartmuseum_role" {
  assume_role_policy = data.aws_iam_policy_document.chartmuseum_role.json
  name               = format("%s-chartmuseum", var.project_name)
}

data "aws_iam_policy_document" "chartmuseum_policy" {
  version = "2012-10-17"

  statement {

    effect = "Allow"
    actions = [
      "s3:*",
    ]

    resources = [
      format("%s/*", aws_s3_bucket.chartmuseum.arn),
      aws_s3_bucket.chartmuseum.arn,
    ]

  }
}

resource "aws_iam_policy" "chartmuseum_policy" {
  name        = format("%s-chartmuseum", var.project_name)
  path        = "/"
  description = var.project_name

  policy = data.aws_iam_policy_document.chartmuseum_policy.json
}

resource "aws_iam_policy_attachment" "chartmuseum" {
  name = "chartmuseum"
  roles = [
    aws_iam_role.chartmuseum_role.name
  ]

  policy_arn = aws_iam_policy.chartmuseum_policy.arn
}

resource "aws_eks_pod_identity_association" "chartmuseum" {
  cluster_name    = aws_eks_cluster.main.name
  namespace       = "chartmuseum"
  service_account = "chartmuseum"
  role_arn        = aws_iam_role.chartmuseum_role.arn
}
```

## 3. Installing ChartMuseum With an S3 Backend

ChartMuseum ships its own Helm chart. The `helm_release` here follows the chart's own documentation for pointing it at S3: creating its own ServiceAccount, enabling the write API (`DISABLE_API=false`), selecting `amazon` as the storage backend, and disabling ChartMuseum's local `index-cache.yaml` state file — since the workflow used in this lesson to upload charts bypasses ChartMuseum's own API entirely, that cached state file isn't needed.

```hcl
# helm_chartmuseum.tf
resource "helm_release" "chartmuseum" {
  name       = "chartmuseum"
  repository = "https://chartmuseum.github.io/charts"
  chart      = "chartmuseum"
  namespace  = "chartmuseum"

  create_namespace = true

  set = [
    {
      name  = "serviceAccount.create"
      value = "true"
    },
    {
      name  = "env.open.AWS_SDK_LOAD_CONFIG"
      value = "true"
    },
    {
      name  = "env.open.DISABLE_API"
      value = "false"
    },
    {
      name  = "env.open.STORAGE"
      value = "amazon"
    },
    {
      name  = "env.open.DISABLE_STATEFILES"
      value = "true"
    },
    {
      name  = "env.open.STORAGE_AMAZON_BUCKET"
      value = aws_s3_bucket.chartmuseum.id
    },
    {
      name  = "env.open.STORAGE_AMAZON_REGION"
      value = var.region
    }
  ]

  depends_on = [
    aws_eks_cluster.main,
    aws_eks_fargate_profile.karpenter
  ]
}
```

Once deployed, `kubectl get namespace` shows a new `chartmuseum` namespace with its pod and ServiceAccount running, and the ChartMuseum service exposes a `GET /api/charts` endpoint over its ClusterIP address — empty at this point, since nothing has been uploaded yet.

## 4. Uploading the Chart Directly to S3

ChartMuseum's HTTP API could upload the chart, but that would mean exposing that API outside the cluster. Instead, since ChartMuseum's storage backend *is* the S3 bucket, the chart is uploaded straight to S3 with an `aws_s3_object` resource — after first running `helm package linuxtips` inside the Module 16 chart directory to produce the `.tgz` archive.

```hcl
# helm_chartmuseum.tf
resource "aws_s3_object" "linuxtips" {
  bucket = aws_s3_bucket.chartmuseum.id
  key    = "linuxtips-0.1.0.tgz"
  source = "${path.module}/helm/linuxtips-0.1.0.tgz"
  etag   = filemd5("${path.module}/helm/linuxtips-0.1.0.tgz")
}
```

The `etag` argument, computed from the file's MD5 hash, is what lets Terraform detect when the packaged chart has actually changed and re-upload it only then, instead of re-uploading on every apply. After the apply, `GET /api/charts` on ChartMuseum now returns the `linuxtips` chart with its version and download URL, and the same object is visible directly in the S3 console.

## Key Takeaways

- ChartMuseum's persistence is a private S3 bucket, created with the bucket/ownership-controls/ACL pattern used elsewhere in the course.
- Authentication to the bucket uses EKS Pod Identity, scoped narrowly to `s3:*` on just that bucket.
- The `helm_release` for ChartMuseum sets `STORAGE=amazon`, points `STORAGE_AMAZON_BUCKET` at the new bucket, and disables ChartMuseum's local state files since the upload path bypasses its API.
- The chart itself is uploaded with `helm package` plus an `aws_s3_object` resource — not through ChartMuseum's HTTP API — with the S3 object's `etag` driven off `filemd5()` so Terraform only re-uploads when the packaged `.tgz` actually changes.

---

# Lesson 3: Installing Argo CD

## 1. A Minimal `helm_release`

Argo CD is installed the same way most other cluster components have been in this course: via its official Helm chart, kept intentionally minimal for now and expanded in later lessons. The only non-default setting at this stage is `--insecure` on the server, since TLS termination is handled by the Istio Gateway that fronts it, not by the Argo CD server itself.

```hcl
# helm_argocd.tf
resource "helm_release" "argocd" {

  name             = "argocd"
  namespace        = "argocd"
  create_namespace = true

  repository = "https://argoproj.github.io/argo-helm"
  chart      = "argo-cd"

  set = [
    {
      name  = "server.extraArgs[0]"
      value = "--insecure"
    }
  ]

  depends_on = [
    aws_eks_cluster.main,
    aws_eks_fargate_profile.karpenter
  ]

}
```

> **Note:** this listing shows the chart's final state, which also carries the Rollout-extension settings covered in [Lesson 7](#lesson-7-the-rollout-extension) — `lesson/argocd` is a single squashed commit on the vanilla companion repo, so no intermediate snapshot of `helm_argocd.tf` survives from before that extension was added. This pattern — quoting the real final file at the point in the lessons where its fields are actually explained — repeats through this module wherever a file is touched by more than one lesson.

## 2. What Each Argo CD Component Does

After the apply, `kubectl get pods -n argocd` shows several distinct components, each with a specific role in the distributed system:

- **Application Controller** — watches the state of every deployed `Application` and continuously tries to correct drift back to the declared state.
- **ApplicationSet Controller** — manages the generators behind `ApplicationSet`s, splitting one definition into the individual `Application`s it templates out.
- **Dex** — the authentication server, used to integrate Argo CD with an identity provider such as LDAP, GitHub, or SAML. It isn't configured in this module.
- **Notifications Controller** — sends external notifications, e.g. via webhooks, on application events.
- **Redis** — an internal cache shared by the other components.
- **Repo Server** — responsible for rendering Helm charts (and other manifest sources); this is the component that will talk to ChartMuseum.
- **Argo CD Server** — the control plane and UI for all of the above.

## Key Takeaways

- Argo CD is installed via its official Helm chart with a deliberately minimal starting configuration, expanded across the rest of this module.
- `--insecure` on the server is used because TLS termination happens at the Istio Gateway, not the Argo CD server.
- The installation is a small distributed system: an Application Controller and ApplicationSet Controller reconcile state, Dex handles auth (unused here), Notifications handles webhooks, Redis caches, the Repo Server renders charts, and the Server component exposes the control plane and UI.

---

# Lesson 4: Exposing the Dashboard

## 1. An Istio Gateway and VirtualService for Argo CD

Following the same pattern used for every other dashboard exposed through the mesh (Grafana, Jaeger, Kiali, Argo Rollouts), a new `argocd_host` variable backs an Istio `Gateway` and `VirtualService` that route external traffic to the `argocd-server` Service.

```hcl
# variables.tf
variable "argocd_host" {
  type        = string
  default     = "argocd.msfidelis.com.br"
  description = "Host do ArgoCD"
}
```

```hcl
# helm_argocd.tf
resource "kubectl_manifest" "argocd_gateway" {
  yaml_body = <<YAML
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: argocd
  namespace: argocd
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "${var.argocd_host}"
YAML

  depends_on = [
    helm_release.argocd,
    helm_release.istio_ingress
  ]

}

resource "kubectl_manifest" "argocd_virtual_service" {
  yaml_body = <<YAML
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: argocd
  namespace: argocd
spec:
  hosts:
  - "${var.argocd_host}"
  gateways:
  - argocd
  http:
  - route:
    - destination:
        host: argocd-server
        port:
          number: 80 
YAML

  depends_on = [
    helm_release.argocd,
    helm_release.istio_ingress
  ]

}
```

Anyone following along without a real Route 53 hosted zone pointed at the ingress NLB can instead resolve `argocd_host` locally through `/etc/hosts`, pointed at the load balancer's IP — the same workaround suggested for every other dashboard host earlier in the course.

## 2. Retrieving and Changing the Initial Admin Password

Argo CD provisions a default `admin` user on first install, with a random password stored in a Kubernetes Secret. It's retrieved with `kubectl` and decoded from base64:

```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath='{.data.password}' | base64 -d
```

That password logs into the dashboard as `admin`; from there, the **User Info** screen's **Update Password** action sets a memorable one for the rest of the module. With the dashboard reachable, the **Repositories**, **Projects**, and **Clusters** panels show an empty state (aside from the built-in `local` cluster) — the next lessons populate all three.

## Key Takeaways

- The Argo CD dashboard is exposed through an Istio `Gateway`/`VirtualService` pair pointed at `argocd-server`, the same pattern used for every other in-mesh dashboard in this course.
- The initial `admin` password lives in the `argocd-initial-admin-secret` Kubernetes Secret, base64-encoded, and should be rotated from the UI after first login.

---

# Lesson 5: Deploying `chip` as an ApplicationSet

## 1. Pointing an ApplicationSet at ChartMuseum

Argo CD can install a Helm chart directly — not only manifests from a Git repository. An `Application` (or `ApplicationSet` template) `source` block can name a chart, its repository URL, and a target revision directly, which is exactly how `chip` — the canary lab application from Modules 14-16 — gets its first GitOps-managed deploy: pointed at the `linuxtips` chart now sitting in ChartMuseum.

```yaml
# chip-applicationset.yml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: chip
  namespace: argocd
spec:
  generators:
    - list:
        elements:
        - cluster: https://kubernetes.default.svc
  template:
    metadata:
      name: chip
    spec:
      project: "default"
      source:
        repoURL: 'http://chartmuseum.chartmuseum.svc.cluster.local:8080'
        chart: linuxtips
        targetRevision: 0.1.0
        helm:
          releaseName: chip
          valuesObject:
            app:
              name: chip
              namespace: chip
              image:
                repository: fidelissauro/chip
                tag: latest
                pullPolicy: IfNotPresent
              createNamespace: true
              iam: ""

              type: ClusterIP
              ports:
                - name: http
                  port: 8080
                  targetPort: 8080
      destination:
        server: '{{ cluster }}'
        namespace: argocd
      syncPolicy:
        syncOptions:
          - CreateNamespace=true
        automated: {}
```

The `generators.list.elements` block is what makes this an `ApplicationSet` rather than a plain `Application`: today it enumerates a single cluster (`https://kubernetes.default.svc`, the in-cluster API server), but the same list could hold several cluster URLs to fan the same deploy out to every one of them, once additional clusters are registered with Argo CD.

## 2. `valuesObject` Instead of a `values.yaml` File

Everything that used to live in a `values.yaml` file for the `linuxtips` chart is inlined instead under `helm.valuesObject`, right inside the `ApplicationSet` manifest. The full block for `chip` reuses every toggle built in Module 16: capacity and autoscaling bounds, the `general` Karpenter NodePool, environment variables, probes on `/healthcheck`, a `ServiceMonitor`, a canary `Rollout` with an `AnalysisTemplate`, and an Istio `VirtualService` with retries — with KEDA left disabled for this particular deploy.

```yaml
# chip-applicationset.yml
              capacity:
                requests:
                  cpu: 100m
                  memory: 128Mi
                limits:
                  cpu: 200m
                  memory: 256Mi
                autoscaling:
                  min: 10
                  max: 30

                nodepool:
                  enabled: true
                  name: general

              envs:
                - name: ENV
                  value: "dev"
                - name: FOO
                  value: "basr"
                - name: VERSION
                  value: v2

              rollout:
                revisionHistoryLimit: 3
                version: v1
                strategy:
                  canary:
                    enabled: true
                    steps:
                      - setWeight: 20
                      - pause: {duration: 30s}
                      - analysis:
                          templates:
                            - templateName: chip-success
                      - setWeight: 40
                      - pause: {duration: 30s}
                      - analysis:
                          templates:
                            - templateName: chip-success
                      - setWeight: 60
                      - pause: {duration: 30s}
                      - analysis:
                          templates:
                            - templateName: chip-success
                      - setWeight: 80
                      - pause: {duration: 30s}
                      - analysis:
                          templates:
                            - templateName: chip-success
                      - setWeight: 100
                analysisTemplates:
                  - name: chip-success
                    spec:
                      metrics:
                        - name: success-rate
                          interval: 2m
                          successCondition: result[0] >= 0.95
                          failureLimit: 0
                          provider:
                            prometheus:
                              address: http://prometheus-kube-prometheus-prometheus.prometheus.svc.cluster.local:9090
                              query: |
                                sum(irate(
                                  istio_requests_total{destination_service=~"chip.chip.svc.cluster.local",response_code!~"5.*"}[1m]
                                )) /
                                sum(irate(
                                  istio_requests_total{destination_service=~"chip.chip.svc.cluster.local"}[1m]
                                ))
                          count: 1

              istio:
                host: chip.msfidelis.com.br
                virtualService:
                  enabled: true
                  http:
                    enabled: true
                    port: 8080
                    retries:
                      attempts: 1
                      perTryTimeout: 2s
                      retryOn: 5xx

              keda:
                enabled: false
```

## 3. Watching the Sync and the Rendered Resources

Applying the `ApplicationSet` produces a `chip` `Application` that starts in a **Degraded** state until Argo CD's sync loop catches up, or a manual **Synchronize** is triggered from the dashboard. Once it syncs, the Repo Server pulls the chart from ChartMuseum, renders it with the inlined `valuesObject`, and every resulting resource — the `Rollout`, its Pods, the `TriggerAuthentication`, the `ServiceMonitor`, the `VirtualService`, the `Gateway` — is listed and tracked in the Argo CD dashboard, along with the NodeClaims Karpenter provisions to run them. From there, individual pods' logs are reachable straight from the dashboard for quick troubleshooting (not a substitute for a real log aggregator), and resources can be inspected, synced, or deleted from the same UI.

## 4. Enabling Pod Exec From the Dashboard

Argo CD's `argocd-cm` ConfigMap has an `admin.enabled` and `exec.enabled` pair of flags that, once turned on, add an **Exec** tab to a pod's detail view for opening a terminal directly inside it from the dashboard.

```bash
kubectl edit configmap argocd-cm -n argocd
```

`chip` itself can't demonstrate this — it's a distroless image with no shell — but the feature works against any pod that does carry `bash` or `sh`. This flag flip is made directly against the live `ConfigMap` with `kubectl edit`, not through any Terraform resource, so no file in the repository captures it; it's reconstructed here from the transcript rather than quoted from a source file.

## Key Takeaways

- An Argo CD `Application`/`ApplicationSet` can source directly from a Helm chart repository — here, ChartMuseum — with no Git repo of raw manifests involved at all.
- `helm.valuesObject` inlines everything a `values.yaml` file would normally hold, directly inside the `ApplicationSet` manifest.
- Once synced, every resource the chart renders is visible, inspectable, and manageable from the Argo CD dashboard, including logs and (for shell-capable images) an exec terminal enabled via the `argocd-cm` ConfigMap.

---

# Lesson 6: An AppProject and a Multi-Service Lab

## 1. AppProjects for Context Segregation

Everything deployed so far falls into Argo CD's built-in `default` `AppProject` unless told otherwise. An `AppProject` is a lightweight way to segregate applications into separate visual and (potentially) permission contexts — restricting which repositories, destination clusters/namespaces, or resource kinds a group of applications is allowed to use. This lesson creates a dedicated `nutrition` project, left permissive for now, purely to demonstrate the grouping:

```yaml
# nutrition-applicationset.yml
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

## 2. Six ApplicationSets, One Chart

The lab for this lesson is the "nutrition" health-API stack seen in earlier modules — proteins, water, calories, IMC (BMI), BMR, and recommendations microservices, plus a `health-api` gateway in front of them — with every one of the six deployed as its own `ApplicationSet`, all sourcing the same `linuxtips` chart from ChartMuseum and all assigned to the new `nutrition` project instead of `default`. Each follows the same shape already established for `chip`, adjusted per service: gRPC ports instead of HTTP where applicable, a two-step canary (`setWeight: 50` → pause → `setWeight: 100`) instead of `chip`'s five-step one, and Jaeger tracing wired in through a `ZIPKIN_COLLECTOR_ENDPOINT` environment variable.

```yaml
# nutrition-applicationset.yml
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: nutrition-proteins
  namespace: argocd
spec:
  generators:
    - list:
        elements:
        - cluster: https://kubernetes.default.svc
  template:
    metadata:
      name: nutrition-proteins
    spec:
      project: nutrition
      source:
        repoURL: 'http://chartmuseum.chartmuseum.svc.cluster.local:8080'
        chart: linuxtips
        targetRevision: 0.1.0
        helm:
          releaseName: chip
          valuesObject:
            app:
              name: proteins-grpc
              namespace: nutrition
              image:
                repository: fidelissauro/proteins-grpc-service
                tag: latest
                pullPolicy: IfNotPresent
              createNamespace: true
              iam: ""

              type: ClusterIP
              ports:
                - name: grpc
                  port: 30000
                  targetPort: 30000

              envs:
                - name: ENVIRONMENT
                  value: "dev"
                - name: ZIPKIN_COLLECTOR_ENDPOINT
                  value: http://jaeger-collector.tracing.svc.cluster.local:9411/api/v2/spans

              rollout:
                revisionHistoryLimit: 3
                version: v1
                strategy:
                  canary:
                    enabled: true
                    steps:
                      - setWeight: 50
                      - pause: {duration: 30s}
                      - setWeight: 100

      destination:
        server: '{{ cluster }}'
        namespace: argocd
      syncPolicy:
        syncOptions:
          - CreateNamespace=true
        automated: {}
```

The `health-api` `ApplicationSet` sits in front of the other five, wiring their in-cluster Service DNS names together as environment variables so it can call them directly:

```yaml
# nutrition-applicationset.yml
              envs:
                - name: ENVIRONMENT
                  value: "dev"
                - name: ZIPKIN_COLLECTOR_ENDPOINT
                  value: http://jaeger-collector.tracing.svc.cluster.local:9411/api/v2/spans
                - name: BMR_SERVICE_ENDPOINT
                  value: "bmr-grpc.nutrition.svc.cluster.local:30000"
                - name: IMC_SERVICE_ENDPOINT
                  value: "imc-grpc.nutrition.svc.cluster.local:30000"
                - name: RECOMMENDATIONS_SERVICE_ENDPOINT
                  value: "recommendations-grpc.nutrition.svc.cluster.local:30000"
                # Configs do Health-data-offload
                # - name: MESSAGE_TYPE
                #   value: "sqs"
                # - name: SQS_QUEUE_URL
                #   value: "URL DA QUEUE"
```

> **Note:** those last two commented-out environment variables (`MESSAGE_TYPE`/`SQS_QUEUE_URL`, labeled "Health-data-offload" in the file) aren't wired up or explained anywhere in this module's lessons — they're left in place, disabled, as a real artifact in the manifest pointing at integration work this module doesn't cover.

## 3. Filtering the Dashboard by Project

With six applications now spread across two projects, the Argo CD dashboard's project filter narrows the view down to just the `nutrition` group — one of the direct payoffs of using `AppProject`s for segregation. Individual applications' Istio-mesh graphs (response times, traffic rate, and traffic animation between services) are also visible per-application, the same visualizations Kiali introduced back in Module 13, now surfaced from inside Argo CD.

## Key Takeaways

- An `AppProject` groups applications into a shared visual (and, when configured, permission) context — this module creates one (`nutrition`) left permissive, purely to demonstrate the grouping.
- The nutrition health-API stack deploys as six separate `ApplicationSet`s, all sourcing the same `linuxtips` chart from ChartMuseum, differing only in their `valuesObject` (ports, image, canary steps, environment variables).
- Service-to-service calls between the six are wired entirely through in-cluster Service DNS names passed as environment variables — no service mesh routing tricks needed for that part.

---

# Lesson 7: The Rollout Extension

## 1. Why Extend Argo CD Instead of Using the Rollouts Dashboard

The standalone Argo Rollouts dashboard (exposed back in Module 15) has no authentication and no audit trail — there's no way to tell which user triggered a rollback or a promotion. Argo CD, by contrast, already has an auth story (Dex, worth extending to LDAP or another IdP) built in. The community-maintained **Rollout Extension** closes that gap by rendering the same Rollout visualization — steps, AnalysisRuns, promote/abort/restart actions — directly inside the Argo CD dashboard, so canary and blue-green deploys can be operated from the same authenticated surface as everything else.

## 2. Enabling Extensions on the `helm_release`

Enabling it means adding a handful of extra arguments to the Argo CD `helm_release`: turning on extensions for both the UI and the proxy, pointing at the `argocd-extension-installer` sidecar image, and listing the extension itself by name and download URL.

```hcl
# helm_argocd.tf
resource "helm_release" "argocd" {

  name             = "argocd"
  namespace        = "argocd"
  create_namespace = true

  repository = "https://argoproj.github.io/argo-helm"
  chart      = "argo-cd"

  set = [
    {
      name  = "server.extraArgs[0]"
      value = "--insecure"
    },
    {
      name  = "server.extensions.enabled"
      value = "true"
    },
    {
      name  = "server.enable.proxy.extension"
      value = "true"
    },
    {
      name  = "server.extensions.image.repository"
      value = "quay.io/argoprojlabs/argocd-extension-installer"
    },
    {
      name  = "server.extensions.extensionList[0].name"
      value = "rollout-extension"
    },
    {
      name  = "server.extensions.extensionList[0].env[0].name"
      value = "EXTENSION_URL"
    },
    {
      name  = "server.extensions.extensionList[0].env[0].value"
      value = "https://github.com/argoproj-labs/rollout-extension/releases/download/v0.3.6/extension.tar"
    }
  ]

  depends_on = [
    aws_eks_cluster.main,
    aws_eks_fargate_profile.karpenter
  ]

}
```

Before this change, a `Rollout`'s detail view in Argo CD only offered a generic **Summary**, **Events**, and **Logs** — no steps, no AnalysisRun results. After the extension installs, a dedicated **Rollout** tab appears with exactly that information.

## 3. Watching a Canary Promote From Inside Argo CD

To exercise it, `chip`'s `ApplicationSet` gets its `VERSION` environment variable bumped to `v2` and is re-applied. The rollout kicks off visibly in both the `Application`'s resource tree and the new **Rollout** tab: `setWeight: 20` fires, the Rollout sits in a **Paused** state for its 30-second pause, then the `chip-success` `AnalysisTemplate` runs an `AnalysisRun` against Prometheus and reports pass/fail against its `result[0] >= 0.95` condition before the next step proceeds. From the same tab, an operator can `resume`, `restart`, or `promote-full` (skip straight to 100%) the Rollout — every capability the standalone Argo Rollouts dashboard offered, now available with Argo CD's authentication and resource visibility on top.

## Key Takeaways

- The Rollout Extension is a community plugin, installed via the Argo CD `helm_release`'s `server.extensions.*` settings and a `server.extensions.extensionList` entry pointing at its release URL.
- Once installed, a Rollout's Argo CD detail view gains a dedicated tab showing canary steps, AnalysisRun results, and promote/abort/restart controls — capability that previously required the separate Argo Rollouts dashboard.
- This closes the module: every capability built since Module 13 (mesh exposure, KEDA autoscaling, canary/blue-green Rollouts) is now delivered and operated through one GitOps-managed, chart-based, authenticated pipeline.
