# Module 15: Progressive Delivery with Argo Rollouts

## Overview

Every deployment strategy used so far in this project has been Kubernetes' native rolling update: replace pods a few at a time, trust the readiness probe, and hope for the best. Argo Rollouts replaces that all-or-nothing rollout with two controlled, reversible strategies — canary releases and blue-green deployments — built around a single CRD, `Rollout`, that is nearly a drop-in replacement for `Deployment`. This module installs Argo Rollouts standalone (it can also run as part of the wider Argo stack, alongside Argo CD, but that integration is left for a later module) via Helm and Terraform, exposes its dashboard through Istio, and then walks the `chip` lab application through both strategies end to end: manual and time-based canary promotion, automated promotion driven by Prometheus-backed `AnalysisTemplate` checks, manual and automatic blue-green promotion, containerized pod warm-up before a green version goes live, and metric-based pre-promotion analysis. It closes, as with Karpenter and KEDA before it, by giving the Argo Rollouts controller its own Fargate profile so it isn't affected by the node churn it has nothing to do with.

## Table of Contents

- [Lesson 1: From Deployment to Rollout](#lesson-1-from-deployment-to-rollout)
  - [1. The Rollout CRD](#1-the-rollout-crd)
  - [2. Deployment vs. Rollout: A Diff](#2-deployment-vs-rollout-a-diff)
  - [Key Takeaways](#key-takeaways)
- [Lesson 2: Installing Argo Rollouts](#lesson-2-installing-argo-rollouts)
  - [1. Branching from the KEDA Lab](#1-branching-from-the-keda-lab)
  - [2. Declaring the Argo Rollouts Version and Host](#2-declaring-the-argo-rollouts-version-and-host)
  - [3. Installing the Chart with Terraform](#3-installing-the-chart-with-terraform)
  - [4. Exposing the Dashboard Through Istio](#4-exposing-the-dashboard-through-istio)
  - [5. Deploying the First Rollout](#5-deploying-the-first-rollout)
  - [Key Takeaways](#key-takeaways-1)
- [Lesson 3: Understanding Canary Releases](#lesson-3-understanding-canary-releases)
  - [1. Why "Gradually" Is the Key Word](#1-why-gradually-is-the-key-word)
  - [2. Two Ways to Describe a Canary Step](#2-two-ways-to-describe-a-canary-step)
  - [Key Takeaways](#key-takeaways-2)
- [Lesson 4: Manual Canary Promotion](#lesson-4-manual-canary-promotion)
  - [1. Building a Canary with Indefinite Pauses](#1-building-a-canary-with-indefinite-pauses)
  - [2. Watching Traffic Shift Pod by Pod](#2-watching-traffic-shift-pod-by-pod)
  - [3. Promoting, Aborting, and Rolling Back from the Dashboard](#3-promoting-aborting-and-rolling-back-from-the-dashboard)
  - [Key Takeaways](#key-takeaways-3)
- [Lesson 5: Time-Based Canary Promotion](#lesson-5-time-based-canary-promotion)
  - [1. Replacing Manual Pauses with a Duration](#1-replacing-manual-pauses-with-a-duration)
  - [2. Mixing Manual and Automatic Steps](#2-mixing-manual-and-automatic-steps)
  - [3. Starting a Rollout Paused](#3-starting-a-rollout-paused)
  - [Key Takeaways](#key-takeaways-4)
- [Lesson 6: Automated Promotion with AnalysisTemplates](#lesson-6-automated-promotion-with-analysistemplates)
  - [1. What an AnalysisTemplate Does](#1-what-an-analysistemplate-does)
  - [2. Anatomy of a Prometheus-Based Analysis](#2-anatomy-of-a-prometheus-based-analysis)
  - [3. Wiring Analysis into Canary Steps](#3-wiring-analysis-into-canary-steps)
  - [4. Watching an Analysis Force a Rollback](#4-watching-an-analysis-force-a-rollback)
  - [Key Takeaways](#key-takeaways-5)
- [Lesson 7: Introduction to Blue-Green Deployments](#lesson-7-introduction-to-blue-green-deployments)
  - [1. Canary vs. Blue-Green](#1-canary-vs-blue-green)
  - [2. A Second Service for the Preview Version](#2-a-second-service-for-the-preview-version)
  - [3. Anatomy of a Blue-Green Strategy](#3-anatomy-of-a-blue-green-strategy)
  - [Key Takeaways](#key-takeaways-6)
- [Lesson 8: Manual Blue-Green Promotion](#lesson-8-manual-blue-green-promotion)
  - [1. Validating the Preview Version Before It's Public](#1-validating-the-preview-version-before-its-public)
  - [2. Promoting and Rolling Back](#2-promoting-and-rolling-back)
  - [3. Controlling How Long the Old Version Stays Alive](#3-controlling-how-long-the-old-version-stays-alive)
  - [Key Takeaways](#key-takeaways-7)
- [Lesson 9: Automatic Blue-Green Promotion](#lesson-9-automatic-blue-green-promotion)
  - [1. Flipping autoPromotionEnabled](#1-flipping-autopromotionenabled)
  - [2. Why Automatic Promotion Alone Isn't Enough](#2-why-automatic-promotion-alone-isnt-enough)
  - [Key Takeaways](#key-takeaways-8)
- [Lesson 10: Warming Up Pods Before Promotion](#lesson-10-warming-up-pods-before-promotion)
  - [1. Why Warm-Up Matters for JVM-Style Workloads](#1-why-warm-up-matters-for-jvm-style-workloads)
  - [2. Containerizing a k6 Load Test](#2-containerizing-a-k6-load-test)
  - [3. Anatomy of a Job-Based AnalysisTemplate](#3-anatomy-of-a-job-based-analysistemplate)
  - [4. Wiring the Warm-Up into prePromotionAnalysis](#4-wiring-the-warm-up-into-prepromotionanalysis)
  - [Key Takeaways](#key-takeaways-9)
- [Lesson 11: Metric-Based Pre-Promotion Analysis](#lesson-11-metric-based-pre-promotion-analysis)
  - [1. Pointing the Success-Rate Query at the Preview Version](#1-pointing-the-success-rate-query-at-the-preview-version)
  - [2. Running Both Analyses Before Every Promotion](#2-running-both-analyses-before-every-promotion)
  - [Key Takeaways](#key-takeaways-10)
- [Lesson 12: Running Argo Rollouts on Fargate](#lesson-12-running-argo-rollouts-on-fargate)
  - [1. Why the Rollouts Controller Needs Its Own Fargate Profile](#1-why-the-rollouts-controller-needs-its-own-fargate-profile)
  - [2. Adding the Fargate Profile](#2-adding-the-fargate-profile)
  - [3. Verifying the Controller Runs on Fargate](#3-verifying-the-controller-runs-on-fargate)
  - [Key Takeaways](#key-takeaways-11)

---

# Lesson 1: From Deployment to Rollout

## 1. The Rollout CRD

Argo Rollouts is the component of the wider Argo stack responsible for canary releases and blue-green deployments in Kubernetes. It can run as part of Argo CD or, as in this module, entirely standalone — the two integrate later, but nothing here depends on Argo CD being installed. Its central resource is a CRD called `Rollout`. A `Rollout` is intentionally very close to a `Deployment`: the pod template, the selector, the replica count all work exactly the same way. From this lesson forward, deploying `chip` means replacing its `Deployment` with a `Rollout` and stops using the core Kubernetes resource entirely — everything that would previously have gone into a `Deployment` still goes into the `Rollout`'s `spec`, with one addition: a `strategy` field that declares, up front, whether this workload uses `canary` or `blueGreen`.

## 2. Deployment vs. Rollout: A Diff

Comparing the lab's old plain `Deployment` against the `Rollout` that replaces it later in this module shows exactly how small that surface change is:

```yaml
# chip.yml (before — Deployment)
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: chip
  name: chip
  namespace: chip
spec:
  replicas: 3
  selector:
    matchLabels:
      app: chip
  template:
    ...
```

```yaml
# chip-rollouts.yml (after — Rollout)
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  labels:
    app: chip
  name: chip
  namespace: chip
spec:
  # Canary Release Manual
  revisionHistoryLimit: 3
  strategy:
    blueGreen:
      activeService: chip
      previewService: chip-green
      autoPromotionEnabled: true
      scaleDownDelaySeconds: 30
      prePromotionAnalysis:
        templates:
        - templateName: chip-warm-up-operation
        - templateName: chip-success-rate
  replicas: 5
  selector:
    matchLabels:
      app: chip
  template:
    ...
```

Only three things change: `apiVersion` moves from the core `apps/v1` to `argoproj.io/v1alpha1`, `kind` moves from `Deployment` to `Rollout`, and a `strategy` block appears. `revisionHistoryLimit` (how many old ReplicaSets to keep around for a rollback) is available on both resources, but it becomes far more relevant here, since rollback is a first-class operation in this module. Everything else — probes, resources, env vars, `topologySpreadConstraints` — carries over unchanged, which is exactly the point: `Rollout` extends `Deployment`, it doesn't replace its feature set.

> **Note:** `chip.yml`, the pre-Rollouts file kept around purely for this comparison, also carries a stray `ScaledObject` named `nginx-istio-tps` at the bottom, targeting a `Rollout` called `nginx` in a `nginx` namespace that doesn't exist anywhere else in this lab. Its `pollingInterval` and `cooldownPeriod` are left blank, and its Prometheus `serverAddress` is wrapped in literal angle brackets (`<http://...>`), which would make it invalid the moment it was applied. None of the 12 lessons in this module reference `nginx` at all — this looks like a leftover copy-paste artifact bundled into the commit, not part of the taught curriculum, and is called out here so it isn't mistaken for something the lessons actually cover.

## Key Takeaways

- `Rollout` (CRD `argoproj.io/v1alpha1`) is a near drop-in replacement for `Deployment`; the pod template, selector, and replica count work identically.
- The only structural addition is `strategy`, which declares `canary` or `blueGreen` up front.
- From this module forward, `chip` is deployed as a `Rollout`, and rollback becomes a routine, first-class operation rather than an emergency `kubectl rollout undo`.

---

# Lesson 2: Installing Argo Rollouts

## 1. Branching from the KEDA Lab

This module continues directly from Module 14's final state rather than starting from `main`. Istio's ingress gateway (from Module 13) is reused to expose the Argo Rollouts dashboard, and Prometheus (also installed alongside Istio) is reused later in this module to back `AnalysisTemplate` checks — none of that infrastructure needs to be rebuilt here.

## 2. Declaring the Argo Rollouts Version and Host

As with every other Helm-based install in this project, the chart version and the public hostname are declared as Terraform variables rather than hardcoded into the `helm_release` resource:

```hcl
# variables.tf
// Argo Rollouts

variable "argo_rollouts_version" {
  type        = string
  default     = "2.34.1"
  description = "value of argo rollouts version"
}

variable "argo_rollouts_host" {
  type        = string
  default     = "argo-rollouts.msfidelis.com.br"
  description = "Host do Argo Rollouts"
}
```

`2.34.1` was pinned deliberately: at the time this lab was recorded, versions up to `2.39` had bugs affecting the dashboard specifically, and `2.34.1` was the latest release confirmed to work cleanly. Newer versions are worth trying, but should be validated against the dashboard before relying on them.

## 3. Installing the Chart with Terraform

The Helm release installs two things: the Rollouts controller (its control plane) and, via `dashboard.enabled`, the web dashboard used throughout this module to watch and manually intervene on rollouts:

```hcl
# helm_argo_rollouts.tf
resource "helm_release" "argo_rollouts" {

  name       = "argo-rollouts"
  chart      = "argo-rollouts"
  repository = "https://argoproj.github.io/argo-helm"
  namespace  = "argo-rollouts"

  version = var.argo_rollouts_version

  create_namespace = true

  set = [
    {
      name  = "dashboard.enabled"
      value = "true"
    },
    {
      name  = "controller.metrics.enabled"
      value = "true"
    },
    {
      name  = "controller.metrics.serviceMonitor.enabled"
      value = "true"
    }
  ]

  depends_on = [
    helm_release.karpenter,
    helm_release.istio_ingress
  ]
}
```

`controller.metrics.enabled` and `controller.metrics.serviceMonitor.enabled` expose the controller's own metrics to the Prometheus already running in the cluster — the same `kube-prometheus-stack` ServiceMonitor pattern used for KEDA and Istio in earlier modules. Applying this creates three pods in the `argo-rollouts` namespace: two for the controller and one for the dashboard, each with its own Service.

## 4. Exposing the Dashboard Through Istio

To reach the dashboard from outside the cluster, it needs the same Gateway/VirtualService pair used for every other exposed service in this project. Rather than writing one from scratch, the lab reuses the existing Jaeger `Gateway`/`VirtualService` as a template and does a find-and-replace on the names, namespace, host, and destination:

```hcl
# helm_argo_rollouts.tf
resource "kubectl_manifest" "rollouts_gateway" {
  yaml_body = <<YAML
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: argo-rollouts
  namespace: argo-rollouts
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "${var.argo_rollouts_host}"
YAML

  depends_on = [
    helm_release.argo_rollouts,
    helm_release.istio_ingress
  ]

}

resource "kubectl_manifest" "rollouts_virtual_service" {
  yaml_body = <<YAML
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: argo-rollouts
  namespace: argo-rollouts
spec:
  hosts:
  - "${var.argo_rollouts_host}"
  gateways:
  - argo-rollouts
  http:
  - route:
    - destination:
        host: argo-rollouts-dashboard
        port:
          number: 3100 
YAML

  depends_on = [
    helm_release.jaeger,
    helm_release.istio_ingress
  ]

}
```

The `VirtualService` routes to the `argo-rollouts-dashboard` Service on port `3100`, the port the chart's dashboard component listens on.

> **Note:** the `rollouts_virtual_service` block's `depends_on` still lists `helm_release.jaeger` instead of `helm_release.argo_rollouts`. This is a real leftover from copying Jaeger's manifest as the starting template: the Gateway's `depends_on` was correctly updated to `helm_release.argo_rollouts`, but the equivalent change on the VirtualService was missed. Functionally this only means Terraform could attempt to create the VirtualService slightly earlier than ideal relative to the Helm release it actually routes to (`argo_rollouts`, not `jaeger`) — worth fixing to `helm_release.argo_rollouts` if reusing this pattern.

## 5. Deploying the First Rollout

With the dashboard reachable but showing no rollouts yet, the lab applies its first `Rollout` for `chip` — no canary or blue-green customization yet, just enough to see it appear in the dashboard and confirm the controller is managing it correctly:

```yaml
# chip-rollouts.yml (illustrative — initial state, later customized through Lessons 3-6)
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  labels:
    app: chip
  name: chip
  namespace: chip
spec:
  revisionHistoryLimit: 3
  replicas: 10
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
        ...
```

With no `strategy.canary.steps` declared, the `Rollout` behaves as a canary that jumps straight to 100% — effectively a plain rolling update, just running through the Rollout controller instead of the Deployment controller. Once the 10 replicas stabilize, the dashboard shows a `chip` Rollout with a single revision, all pods healthy, and a canary sitting at 100% with no active steps — the starting point for everything the next four lessons build on top of.

## Key Takeaways

- Argo Rollouts installs via Helm like any other chart in this project; `dashboard.enabled` turns on the web UI used for the rest of the module.
- The dashboard is exposed through Istio using the same Gateway/VirtualService pattern as every other service, copied from an existing manifest (Jaeger's) and adapted.
- A `Rollout` with no `strategy.canary.steps` still works — it just behaves like an ordinary rolling update, progressing straight to 100%.

---

# Lesson 3: Understanding Canary Releases

## 1. Why "Gradually" Is the Key Word

A canary release keeps two versions live at once — a `stable` version already serving traffic, and a `canary` version being promoted — and shifts a percentage of real, production traffic from one to the other in controlled increments rather than all at once. That promotion can be driven by time, by metrics, by manual approval, or by alerts, but the defining trait is always the same: it is gradual. With 10 pods running `chip`, moving from 0% to 10% canary traffic means one pod runs the new version and nine still run the old one; the percentage climbs — based on whatever condition is chosen — until the canary becomes the new stable version and makes room for the next release.

## 2. Two Ways to Describe a Canary Step

The Rollout CRD's `strategy.canary.steps` list supports two different ways of expressing "how much of the new version should be live right now":

- **`setWeight`** — an explicit traffic percentage (e.g. `setWeight: 25`), paired with `pause` steps to hold at that percentage.
- **`setCanaryScale`** — an explicit replica count (e.g. `{replicas: 3}`) or an explicit weight (e.g. `{weight: 25}`) applied to the canary ReplicaSet specifically, useful when the canary's own scale needs to be controlled independently of the traffic split.

This module uses `setWeight` and `pause` throughout, since it maps directly onto "what percentage of traffic is the new version receiving right now" — the question a canary release exists to answer.

## Key Takeaways

- A canary release is defined by two things: percentage-based traffic control, and gradual, incremental promotion.
- `strategy.canary.steps` accepts `setWeight`/`pause` for percentage-and-time-based control, or `setCanaryScale` for direct control over the canary ReplicaSet's own scale.
- Promotion conditions (manual, time, metric-based) and rollback are independent of which of the two step styles is used.

---

# Lesson 4: Manual Canary Promotion

## 1. Building a Canary with Indefinite Pauses

The simplest way to drive a canary is to pause indefinitely after each step and use the dashboard's **Promote** button as the trigger to continue — effectively handing control to a human:

```yaml
# chip-rollouts.yml (illustrative — Canary strategy, Lesson 4 stage)
spec:
  strategy:
    canary:
      steps:
      - setWeight: 10
      - pause: {}
      - setWeight: 20
      - pause: {}
      - setWeight: 30
      - pause: {}
      - setWeight: 40
      - pause: {}
      - setWeight: 80
      - pause: {}
      - setWeight: 100
```

An empty `pause: {}` (no `duration`) pauses indefinitely — the Rollout will not progress to the next step until something external tells it to, which is exactly what the dashboard's Promote button does.

> This block reconstructs the Canary-stage configuration described in the lesson narration; the real `chip-rollouts.yml` file in the lab repository was later fully rewritten for the Blue-Green strategy (Lesson 7), so no intermediate Canary-stage file survives in the repository to quote verbatim.

## 2. Watching Traffic Shift Pod by Pod

Bumping the `chip` pod's `VERSION` environment variable (surfaced through a debug endpoint that prints the running version) and reapplying the Rollout with this new image starts a canary rollout. With 10 replicas and a first step of `setWeight: 10`, the controller brings up exactly one new-version pod and leaves nine running the old version, then pauses. Requests hitting the app show mostly the old version with an occasional new one — a live, observable reflection of the current step's weight.

## 3. Promoting, Aborting, and Rolling Back from the Dashboard

The dashboard exposes the controls needed to drive (or stop) this process by hand:

- **Promote** — advance to the next step in the list.
- **Promote Full** — skip every remaining step and jump straight to 100%.
- **Abort** — stop the rollout and revert immediately to the previous stable version.
- **Rollback** — return to a previously stable revision, usable during or after a rollout.

Argo Rollouts keeps recent ReplicaSets around (bounded by `revisionHistoryLimit`) specifically so this kind of rollback is instant — switching which ReplicaSet is scaled up, not rebuilding anything from scratch.

## Key Takeaways

- `pause: {}` with no duration pauses a canary step indefinitely, waiting for a manual Promote.
- Promote, Promote Full, Abort, and Rollback are all available directly from the dashboard, with no `kubectl` needed.
- Because old ReplicaSets are retained (`revisionHistoryLimit`), rollback is close to instant, at any point during or after a rollout.

---

# Lesson 5: Time-Based Canary Promotion

## 1. Replacing Manual Pauses with a Duration

Giving `pause` a `duration` turns each step from "wait for a human" into "wait this long, then continue automatically" — seconds, minutes, or days, whatever the step actually needs to build confidence:

```yaml
# chip-rollouts.yml (illustrative — Canary strategy, Lesson 5 stage)
spec:
  strategy:
    canary:
      steps:
      - setWeight: 10
      - pause: {duration: 30s}
      - setWeight: 20
      - pause: {duration: 30s}
      - setWeight: 40
      - pause: {duration: 30s}
      - setWeight: 80
      - pause: {duration: 30s}
      - setWeight: 100
```

With a 30-second pause at each step, a new version (bumped to `v3` for this test) rolls out to 20%, 40%, 80%, and 100% automatically, one interval at a time, with no manual intervention required. **Abort** still works exactly as before — issuing it mid-rollout immediately reverts to the previous stable version.

## 2. Mixing Manual and Automatic Steps

Duration-based and indefinite pauses can be combined in the same `steps` list — for example, progressing automatically every 15 minutes up to 50%, then switching to manual `pause: {}` for the remainder, so a human makes the final call once the canary has already proven itself at moderate traffic.

## 3. Starting a Rollout Paused

Setting the very first step to `setWeight: 0` followed by an indefinite `pause: {}` starts the Rollout in a paused state from the moment it's applied — nothing shifts to the new version until that first pause is manually cleared, giving full control over exactly when a rollout begins, not just how it progresses once started.

## Key Takeaways

- `pause: {duration: "30s"}` (or any duration) turns a step into a timed, automatic promotion instead of a manual one.
- Manual (`pause: {}`) and timed (`pause: {duration: ...}`) steps can be freely mixed within the same canary.
- A leading `setWeight: 0` plus an indefinite pause starts a Rollout paused, deferring even the first traffic shift until manually released.

---

# Lesson 6: Automated Promotion with AnalysisTemplates

## 1. What an AnalysisTemplate Does

An `AnalysisTemplate` lets a Rollout run small experiments mid-release against external metrics — Datadog, New Relic, CloudWatch, or, as used here, Prometheus — and use the result to decide automatically whether to keep progressing or cancel the rollout. Instead of trusting a fixed timer, the promotion decision becomes data-driven: did this canary step meet its error-rate target or not.

## 2. Anatomy of a Prometheus-Based Analysis

The `AnalysisTemplate` used here checks the ratio of non-5xx requests to total requests for `chip` over a short window, and only continues if that ratio holds above `0.95`:

```yaml
# chip-rollouts.yml (illustrative — Canary strategy, Lesson 6 stage, checking the chip service directly)
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: chip-success-rate
  namespace: chip
spec:
  metrics:
    - name: success-rate
      interval: 2m
      successCondition: result[0] >= 0.95
      failureLimit: 0
      count: 1
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
```

`successCondition` is evaluated against `result[0]`, the query's own return value; `failureLimit: 0` means a single failed check is enough to fail the whole analysis; `count: 1` controls how many times the check runs per invocation, and `interval` controls how often within that. This particular query and template are reconstructed from the lesson narration at the stage where the canary strategy still points the check at the plain `chip` service — Lesson 11 revisits this same template pointed at `chip-green` instead, which is the version that survives in the final repository state.

## 3. Wiring Analysis into Canary Steps

An `analysis.templates` entry inside a step tells the Rollout to run that `AnalysisTemplate` before continuing past that point:

```yaml
# chip-rollouts.yml (illustrative — Canary strategy, Lesson 6 stage)
spec:
  strategy:
    canary:
      steps:
      - setWeight: 10
      - pause: {duration: 60s}
      - analysis:
          templates:
          - templateName: chip-success-rate
      - setWeight: 40
      - pause: {duration: 30s}
      - analysis:
          templates:
          - templateName: chip-success-rate
      - setWeight: 80
      - pause: {duration: 30s}
      - analysis:
          templates:
          - templateName: chip-success-rate
      - setWeight: 100
```

At each analysis point, the controller only advances to the next `setWeight` if `chip-success-rate` reports success; a failed check stops the rollout right there.

## 4. Watching an Analysis Force a Rollback

To see the failure path, the lab enables `chip`'s built-in Chaos Monkey (`CHAOS_MONKEY_ENABLED: "true"`) partway through this same rollout and temporarily reduces the `VirtualService`'s retry budget (`retries.attempts: 1`) so that induced failures aren't masked by Istio's own retries. As traffic shifts toward the new, chaos-affected version, the success-rate query drops below `0.95`; the next scheduled analysis run reports failure, and the Rollout controller immediately terminates the failing revision's pods and reverts to the previous stable revision — no manual `abort` required. This is the same mechanism that later backs automated blue-green promotion in Lessons 10-11, just applied to a canary instead.

## Key Takeaways

- An `AnalysisTemplate` turns a canary step from "wait N seconds" into "wait until this metric proves it's safe," using Prometheus (or Datadog, New Relic, CloudWatch) as the source of truth.
- `analysis.templates` inside a step blocks progression until the referenced template succeeds; `failureLimit: 0` means the first failure is final.
- A failed analysis rolls back automatically, with no operator intervention — the same chaos-injection test used here to prove it out reappears for blue-green in Lessons 10-11.

---

# Lesson 7: Introduction to Blue-Green Deployments

## 1. Canary vs. Blue-Green

Where canary releases shift a percentage of live production traffic gradually, blue-green deployments aim for the opposite trade-off: as close to zero downtime as possible, with a fast, clean rollback path. Both versions — conventionally called blue (stable) and green (preview) — run fully in parallel; the new version can be tested and validated before it receives any real traffic at all, and the old version is kept running for a defined window after promotion specifically so a rollback is instant if needed.

## 2. A Second Service for the Preview Version

A canary needs only one Service, since it routes a percentage of traffic to the same pool of pods regardless of version. Blue-green needs two: one pointing at whichever version is currently active, and one dedicated to the preview version so it can be reached (and tested) independently before promotion. The lab adds a `chip-green` `VirtualService`/`Service` pair alongside the existing `chip` ones, identical in shape but suffixed `-green` and routing to a `chip-green` destination:

```yaml
# chip-rollouts.yml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: chip-green
  namespace: chip
spec:
  hosts:
  - "chip-green.chip.svc.cluster.local"
  - "chip-green.msfidelis.com.br"
  gateways:
  - chip
  http:
  - route:
    - destination:
        host: chip-green
        port:
          number: 8080 
    retries:
      attempts: 3
      perTryTimeout: 10500ms
      retryOn: 5xx
---
apiVersion: v1
kind: Service
metadata:
  name: chip-green
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
```

Both Services share the same pod selector (`app: chip`) — which pods each one actually reaches is entirely controlled by the Rollout controller relabeling pods as active or preview, not by any difference in the Service spec itself.

## 3. Anatomy of a Blue-Green Strategy

With the canary `steps` removed, `strategy.blueGreen` replaces it, naming which Service is active, which is the preview, and how promotion behaves:

```yaml
# chip-rollouts.yml
spec:
  revisionHistoryLimit: 3
  strategy:
    blueGreen: 
      activeService: chip
      previewService: chip-green
      autoPromotionEnabled: false
      scaleDownDelaySeconds: 30
  replicas: 5
```

`activeService`/`previewService` point at the Service pair from the previous section; `autoPromotionEnabled: false` means a new revision stays parked in preview until manually promoted; `scaleDownDelaySeconds` controls how long the previous stable version's pods are kept alive after a promotion, purely as a fast-rollback window. This lesson also reduces `chip` to 5 replicas (down from 10) and resets Chaos Monkey to disabled, giving a clean baseline before exploring blue-green's own promotion behavior in the next two lessons.

## Key Takeaways

- Blue-green keeps both versions fully running in parallel behind two Services (`activeService`/`previewService`), rather than splitting traffic within a single pool of pods.
- `scaleDownDelaySeconds` is what makes rollback fast — the previous version isn't torn down the moment promotion happens.
- `autoPromotionEnabled: false` is blue-green's manual mode; the next two lessons cover flipping it to automatic.

---

# Lesson 8: Manual Blue-Green Promotion

## 1. Validating the Preview Version Before It's Public

With `autoPromotionEnabled: false`, applying a new `chip` version brings up a full second set of pods (five, matching the existing replica count) running the new version, reachable only through the `chip-green` internal service — the public `chip` Service still routes entirely to the old, stable version. The lab validates this from inside the cluster before promoting anything public-facing, running a throwaway debug pod and curling the preview service directly:

```bash
kubectl run bash --rm -it --image=debian -- bash
# from inside the debug pod:
curl chip-green.chip.svc.cluster.local:8080/version
```

This lets internal testing — pipeline checks, manual QA — hit the exact new version running in the cluster without it ever being exposed to real customer traffic.

## 2. Promoting and Rolling Back

Once the preview version has been validated, the dashboard's **Promote** button flips `chip`'s active Service over to the new version's pods immediately — no gradual traffic shift, since blue-green has no concept of partial traffic like canary does. If a problem surfaces afterward, **Rollback** reverts the active Service back to the previous stable version just as immediately.

## 3. Controlling How Long the Old Version Stays Alive

`scaleDownDelaySeconds` governs the window between promotion and the old version's pods actually being terminated. The lab walks this value from 30 seconds up to 120 and then 240 (4 minutes), demonstrating the trade-off directly: a longer delay means a longer window in which rollback is instant (the old pods are still warm and running), at the cost of running double the pod count for that much longer after every promotion.

## Key Takeaways

- With manual promotion, the new version is fully live and testable internally (via its own `-green` Service) before any customer ever reaches it.
- Promotion and rollback both take effect immediately — there's no partial-traffic state to wait through, unlike canary.
- `scaleDownDelaySeconds` is a direct trade-off between rollback speed and how long double the pod count stays running after each promotion.

---

# Lesson 9: Automatic Blue-Green Promotion

## 1. Flipping autoPromotionEnabled

Setting `autoPromotionEnabled: true` removes the manual approval step entirely: as soon as the new version's pods pass their health checks and stabilize, the Rollout controller promotes them to active on its own, with no dashboard click required. Applying a new version now results in the active Service switching over automatically the moment the new pods are healthy, and the old version is kept around for the configured `scaleDownDelaySeconds` purely as a rollback window — a human can still intervene during that window, but nothing is required for the promotion itself to happen.

## 2. Why Automatic Promotion Alone Isn't Enough

Automatic promotion based purely on pod health is a meaningfully weaker guarantee than the health checks it relies on: a pod can be `Ready` and still be serving a broken feature, a regression that only shows up under real load, or a version with a functional bug no readiness probe would ever catch. Promoting automatically the instant probes pass, with no further validation, trades safety for convenience. The next two lessons build exactly that missing validation back in — first as a pre-promotion warm-up and smoke test, then as a metric-based check — so automatic promotion can be trusted rather than merely convenient.

## Key Takeaways

- `autoPromotionEnabled: true` promotes as soon as the new version's pods are healthy — no manual click needed.
- Health-check-based automatic promotion alone doesn't validate anything about correctness, only that the pod started successfully.
- Lessons 10 and 11 add pre-promotion validation (a warm-up smoke test, then a Prometheus-based check) so automatic promotion has real evidence behind it, not just a passing readiness probe.

---

# Lesson 10: Warming Up Pods Before Promotion

## 1. Why Warm-Up Matters for JVM-Style Workloads

Workloads with a JVM-style cold start — `chip` included — tend to serve their first requests noticeably slower than steady-state ones, whether from JIT warm-up, connection pool initialization, or lazy class loading. Promoting a brand-new set of pods straight into production traffic means the very first real users hit that cold-start penalty. Blue-green's preview window makes it possible to avoid this entirely: since the new version is already running and reachable before promotion, it can be pre-warmed with synthetic load first.

## 2. Containerizing a k6 Load Test

The lab builds a small, purpose-specific load test with [k6](https://k6.io/), hitting the preview version's own endpoints (`/version`, `/system`, `/system/warmup`) repeatedly with a ramping number of virtual users over about 30 seconds, then packages it as a container image:

```dockerfile
# Dockerfile (warmup test)
FROM grafana/k6:latest
ADD load.js /load.js
CMD ["run", "/load.js"]
```

The image is built and pushed to Docker Hub as `fidelissauro/chip-test-green:latest` — the same image referenced by the `AnalysisTemplate` in the next section. Its actual requests don't need to exercise meaningful business logic; the goal is purely to drive enough traffic through the new pods to warm up whatever needs warming before real customers do.

## 3. Anatomy of a Job-Based AnalysisTemplate

Argo Rollouts' `AnalysisTemplate` isn't limited to querying a metrics backend — its `job` provider runs an arbitrary Kubernetes `Job` and treats the Job's success or failure as the analysis result, which is exactly what's needed to run this warm-up container as a gating step:

```yaml
# chip-rollouts.yml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: chip-warm-up-operation
  namespace: chip
spec:
  metrics:
  - name: k6-stack-warm-up
    failureLimit: 0
    provider:
      job:
        spec:
          backoffLimit: 1
          template:
            metadata:
              labels:
                istio-injection: disabled
                sidecar.istio.io/inject: "false"
            spec:
              containers:
              - name: k6-stack-warm-up
                image: fidelissauro/chip-test-green:latest
              restartPolicy: Never
```

The pod template is explicitly annotated to skip Istio sidecar injection (`istio-injection: disabled`, `sidecar.istio.io/inject: "false"`). A short-lived Job doesn't need to participate in the mesh, and injecting a sidecar into it only adds startup latency — or, in the worst case, can hang the Job waiting on a sidecar that never needs to be there in the first place.

## 4. Wiring the Warm-Up into prePromotionAnalysis

`prePromotionAnalysis` is where blue-green's promotion gate lives: a list of `AnalysisTemplate`s that must all succeed before the controller is allowed to promote the preview version to active, even with `autoPromotionEnabled: true`:

```yaml
# chip-rollouts.yml
spec:
  strategy:
    blueGreen: 
      activeService: chip
      previewService: chip-green
      autoPromotionEnabled: true
      scaleDownDelaySeconds: 30
      prePromotionAnalysis:
        templates:
        - templateName: chip-warm-up-operation
```

Applying a new version now brings up the preview pods, immediately kicks off the `chip-warm-up-operation` Job against them, and only promotes once that Job exits successfully — turning "automatic" promotion back into something that has actually proven the new pods work under load, not just that they passed a readiness probe.

## Key Takeaways

- A `job`-provider `AnalysisTemplate` runs an arbitrary Kubernetes Job and uses its exit status as the analysis result — useful for anything a metrics query can't express, like a synthetic warm-up load test.
- Disabling Istio sidecar injection on short-lived Job pods (`istio-injection: disabled`) avoids unnecessary startup overhead or a Job hanging on a sidecar it doesn't need.
- `prePromotionAnalysis.templates` gates promotion — even with `autoPromotionEnabled: true` — behind every listed `AnalysisTemplate` succeeding first.

---

# Lesson 11: Metric-Based Pre-Promotion Analysis

## 1. Pointing the Success-Rate Query at the Preview Version

The same success-rate `AnalysisTemplate` idea used for canary in Lesson 6 applies just as well before a blue-green promotion — with one necessary change: since the preview version isn't public yet, the query has to check the `chip-green` destination specifically, not `chip`:

```yaml
# chip-rollouts.yml
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: chip-success-rate
  namespace: chip
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
              istio_requests_total{destination_service=~"chip-green.chip.svc.cluster.local",response_code!~"5.*"}[1m]
            )) /
            sum(irate(
              istio_requests_total{destination_service=~"chip-green.chip.svc.cluster.local"}[1m]
            ))
      count: 1
```

With the warm-up load test from Lesson 10 already driving traffic at `chip-green`, this query has real request data to evaluate even before any production traffic reaches the preview version — the warm-up test doubles as the very traffic this check measures.

## 2. Running Both Analyses Before Every Promotion

Both templates now sit in `prePromotionAnalysis`, and both must succeed before promotion happens:

```yaml
# chip-rollouts.yml
spec:
  strategy:
    blueGreen: 
      activeService: chip
      previewService: chip-green
      autoPromotionEnabled: true
      scaleDownDelaySeconds: 30
      prePromotionAnalysis:
        templates:
        - templateName: chip-warm-up-operation
        - templateName: chip-success-rate
```

Applying a new version now runs the warm-up Job first, then the success-rate check against the traffic that warm-up generated, and only then promotes automatically — the same combination of "prove it under load" and "prove it's not erroring" used for canary in Lesson 6, adapted to blue-green's all-or-nothing traffic model. This is the exact configuration that ships in the final state of `chip-rollouts.yml` in this lab's repository.

## Key Takeaways

- The same success-rate check used for canary works for blue-green pre-promotion, with the query simply repointed at the preview Service (`chip-green`) instead of the active one.
- Running the warm-up Job and the metric-based check together means the metric check has real traffic to evaluate, generated by the warm-up itself.
- `prePromotionAnalysis.templates` accepts multiple templates; all must succeed for promotion to proceed, whether triggered automatically or manually.

---

# Lesson 12: Running Argo Rollouts on Fargate

## 1. Why the Rollouts Controller Needs Its Own Fargate Profile

Karpenter, by design, constantly reprovisions nodes as workloads change — exactly the kind of node churn that shouldn't be allowed to affect the controller responsible for managing every Rollout in the cluster. Consistent with the same reasoning already applied to Karpenter's own controller and to KEDA (Module 14), the Argo Rollouts controller gets a dedicated Fargate profile, isolating it from any instability in the Karpenter-managed node pool it has nothing to do with.

## 2. Adding the Fargate Profile

The new profile follows the exact same shape as the existing `karpenter` and `keda` profiles — same IAM role, same subnets, just a different namespace selector:

```hcl
# fargate.tf
resource "aws_eks_fargate_profile" "rollouts" {
  cluster_name         = aws_eks_cluster.main.name
  fargate_profile_name = "argo-rollouts"

  pod_execution_role_arn = aws_iam_role.fargate.arn

  subnet_ids = data.aws_ssm_parameter.pod_subnets[*].value

  selector {
    namespace = "argo-rollouts"
  }
}
```

## 3. Verifying the Controller Runs on Fargate

Applying this profile alone doesn't move already-running pods — Fargate profiles only affect where a pod schedules at creation time, so the existing Argo Rollouts pods (still running on EC2, in `us-west-2b` at the time of recording) need to be deleted so the scheduler places their replacements on Fargate instead:

```bash
kubectl delete pods --all -n argo-rollouts
```

Their replacements come up `Pending` momentarily while a Fargate node provisions (roughly 50-60 seconds), then move through `ContainerCreating` and into `Running`. Once every pod passes its health checks on the new Fargate node, the Argo Rollouts control plane is exactly as isolated from Karpenter-driven node churn as Karpenter's own controller and KEDA already are.

## Key Takeaways

- A Fargate profile only affects pod placement at creation time — moving already-running pods onto Fargate requires deleting them so the scheduler places the replacements there.
- The `rollouts` Fargate profile is structurally identical to the existing `karpenter` and `keda` profiles: same IAM role, same subnets, different namespace selector.
- With this in place, Karpenter, KEDA, and Argo Rollouts all have their own control planes running independently of the node churn they each manage or react to.
