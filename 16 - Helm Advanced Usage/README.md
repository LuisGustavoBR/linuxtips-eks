# Module 16: Helm Advanced Usage

## Overview

Every module up to this point has grown the same lab application, `chip`, one capability at a time — Rollouts, Istio, KEDA, AnalysisTemplates — until its plain manifests turned into a sprawling, hard-to-read YAML file. This module steps back from adding cluster capabilities and instead asks a platform question: how does a team hand all of that complexity to developers without making them hand-write a Rollout, a Gateway, a VirtualService, a ScaledObject, and an AnalysisTemplate every time they ship an app? The answer built here is a from-scratch Helm chart, `linuxtips`, that encodes every minimum requirement a workload on this cluster is expected to meet — its own namespace, its own ServiceAccount, requests/limits, health probes, multi-AZ spread, autoscaling, mesh exposure, and progressive delivery — behind a small set of `values.yaml` toggles and lists. The chart is built resource by resource, testing each one with `helm template` before moving to the next, starting from a `helmfy`-compiled reference and a `helm create` scaffold but keeping almost nothing from either. By the end it's packaged with `helm package`, ready to be consumed by Argo CD in the next module.

## Table of Contents

- [Lesson 1: Why Package the Platform Into a Helm Chart](#lesson-1-why-package-the-platform-into-a-helm-chart)
  - [1. One Simple App, One Enormous Manifest](#1-one-simple-app-one-enormous-manifest)
  - [2. Two Starting Points: `helm create` and Helmfy](#2-two-starting-points-helm-create-and-helmfy)
  - [3. This Module's Approach: A Hybrid With Feature Toggles](#3-this-modules-approach-a-hybrid-with-feature-toggles)
  - [Key Takeaways](#key-takeaways)
- [Lesson 2: Namespace and Service Account](#lesson-2-namespace-and-service-account)
  - [1. Gutting the Helmfy Scaffold](#1-gutting-the-helmfy-scaffold)
  - [2. The `app` Values Block](#2-the-app-values-block)
  - [3. A Toggled Namespace](#3-a-toggled-namespace)
  - [4. A Mandatory Service Account for IRSA](#4-a-mandatory-service-account-for-irsa)
  - [Key Takeaways](#key-takeaways-1)
- [Lesson 3: The Service Template](#lesson-3-the-service-template)
  - [1. A Minimal Service Spec: Type and Ports](#1-a-minimal-service-spec-type-and-ports)
  - [2. Rendering and Deploying](#2-rendering-and-deploying)
  - [Key Takeaways](#key-takeaways-2)
- [Lesson 4: Building the Rollout Template, Piece by Piece](#lesson-4-building-the-rollout-template-piece-by-piece)
  - [1. Why a Complex Resource Needs Baby Steps](#1-why-a-complex-resource-needs-baby-steps)
  - [2. Capacity: Requests, Limits, and Autoscaling Bounds](#2-capacity-requests-limits-and-autoscaling-bounds)
  - [3. Container Ports via `range`](#3-container-ports-via-range)
  - [4. Feature-Toggled Probes](#4-feature-toggled-probes)
  - [5. Environment Variables and the Image](#5-environment-variables-and-the-image)
  - [Key Takeaways](#key-takeaways-3)
- [Lesson 5: Canary Strategy and Version Labels](#lesson-5-canary-strategy-and-version-labels)
  - [1. Choosing Canary Over Blue-Green](#1-choosing-canary-over-blue-green)
  - [2. Passing the Whole Steps List Through `toYaml`](#2-passing-the-whole-steps-list-through-toyaml)
  - [3. Version Labels for Traceability](#3-version-labels-for-traceability)
  - [Key Takeaways](#key-takeaways-4)
- [Lesson 6: Spreading Pods Across Zones and Nodes](#lesson-6-spreading-pods-across-zones-and-nodes)
  - [1. Two `topologySpreadConstraints` by Default](#1-two-topologyspreadconstraints-by-default)
  - [2. Targeting a Karpenter NodePool Through a Toggle](#2-targeting-a-karpenter-nodepool-through-a-toggle)
  - [3. An Anti-Affinity Experiment That Didn't Survive](#3-an-anti-affinity-experiment-that-didnt-survive)
  - [Key Takeaways](#key-takeaways-5)
- [Lesson 7: Exposing the Application Through Istio](#lesson-7-exposing-the-application-through-istio)
  - [1. Enabling Sidecar Injection at the Namespace Level](#1-enabling-sidecar-injection-at-the-namespace-level)
  - [2. A Toggled Gateway](#2-a-toggled-gateway)
  - [3. A Toggled VirtualService With Retries](#3-a-toggled-virtualservice-with-retries)
  - [Key Takeaways](#key-takeaways-6)
- [Lesson 8: Parameterizing AnalysisTemplates](#lesson-8-parameterizing-analysistemplates)
  - [1. Why AnalysisTemplates Are Hard to Templatize](#1-why-analysistemplates-are-hard-to-templatize)
  - [2. Looping Over a List of Templates](#2-looping-over-a-list-of-templates)
  - [3. Wiring an Analysis Step Into the Canary](#3-wiring-an-analysis-step-into-the-canary)
  - [Key Takeaways](#key-takeaways-7)
- [Lesson 9: A Templated KEDA ScaledObject for Rollouts](#lesson-9-a-templated-keda-scaledobject-for-rollouts)
  - [1. One ScaledObject per Workload](#1-one-scaledobject-per-workload)
  - [2. Pointing Autoscaling at a Rollout Instead of a Deployment](#2-pointing-autoscaling-at-a-rollout-instead-of-a-deployment)
  - [3. Reusing HPA Behavior and Capacity Bounds](#3-reusing-hpa-behavior-and-capacity-bounds)
  - [Key Takeaways](#key-takeaways-8)
- [Lesson 10: Productizing KEDA Triggers](#lesson-10-productizing-keda-triggers)
  - [1. Prometheus Triggers as a List](#1-prometheus-triggers-as-a-list)
  - [2. Scheduled Scaling With Cron Triggers](#2-scheduled-scaling-with-cron-triggers)
  - [3. Memory and CPU Triggers](#3-memory-and-cpu-triggers)
  - [4. SQS Triggers and a Global TriggerAuthentication](#4-sqs-triggers-and-a-global-triggerauthentication)
  - [5. Keeping Only One Trigger Active](#5-keeping-only-one-trigger-active)
  - [Key Takeaways](#key-takeaways-9)
- [Lesson 11: Automating Metric Collection With a ServiceMonitor](#lesson-11-automating-metric-collection-with-a-servicemonitor)
  - [1. A Toggled ServiceMonitor Template](#1-a-toggled-servicemonitor-template)
  - [2. A Scrape Timeout That Must Be a String](#2-a-scrape-timeout-that-must-be-a-string)
  - [Key Takeaways](#key-takeaways-10)
- [Lesson 12: Packaging the Chart](#lesson-12-packaging-the-chart)
  - [1. Cleaning Up the Feature-Toggle Debris](#1-cleaning-up-the-feature-toggle-debris)
  - [2. `helm package`](#2-helm-package)
  - [Key Takeaways](#key-takeaways-11)

---

# Lesson 1: Why Package the Platform Into a Helm Chart

## 1. One Simple App, One Enormous Manifest

`chip` was introduced back in Module 3 as a deliberately minimal lab application, but every module since has bolted more onto it: a `Rollout` instead of a `Deployment`, canary steps, `AnalysisTemplate` references, a `Gateway` and `VirtualService` for Istio, `ScaledObject`s for KEDA. None of that is wrong on its own — each piece was necessary for what its module taught — but the cumulative result is that a "simple" application now requires a developer to correctly hand-write a large, interdependent stack of Kubernetes and CRD manifests just to ship a change. Every one of those manifests is also an opportunity for a typo, a missed field, or an inconsistent choice between two developers who happen to configure the same thing slightly differently.

## 2. Two Starting Points: `helm create` and Helmfy

Helm's own `helm create <name>` scaffolds a chart with the bare minimum a generic app might need — a `Deployment`, `Service`, `ServiceAccount`, `HorizontalPodAutoscaler`, and an `Ingress`, all wired to a handful of `values.yaml` fields. It's a reasonable starting point for a brand-new chart, but it knows nothing about Rollouts, Istio, or KEDA, and most of what it generates isn't relevant here.

The second tool is [Helmfy](https://github.com/isuruceanu/helmfy), an open-source CLI that does the opposite: instead of scaffolding a chart from nothing, it takes a directory of existing plain manifests — `chip-rollouts.yml`, with all of its Rollouts, Gateways, VirtualServices, and AnalysisTemplates — and compiles them into a chart automatically, splitting resources into template files and lifting hardcoded values into a starter `values.yaml`. Run against the lab's manifests it produces two `Service` templates (recognizing `chip` and its blue-green `chip-green` pair as belonging to the same release) and dumps everything it can't confidently categorize into a generic `templates/service` folder.

## 3. This Module's Approach: A Hybrid With Feature Toggles

Neither tool's output is used as-is. The chart built in this module keeps only the two directories `helm create` and Helmfy produce as loose references — thrown into a scratch `lixo` ("trash") folder to compare against while building — and starts every real template from an empty file, added resource by resource: namespace, then service account, then service, then the much more complex Rollout, Istio objects, KEDA `ScaledObject`, and `AnalysisTemplate`s. Every optional capability is gated behind a boolean in `values.yaml` (`createNamespace`, `istio.gateway.enabled`, `keda.enabled`, and so on) so a chart consumer can request exactly the platform features their app needs with a handful of values instead of a wall of YAML — the same feature-toggle philosophy used earlier in the LinuxTips training line for SaaS deployment modules, applied here to Helm instead of a homegrown templating layer.

## Key Takeaways

- The lab application `chip` accumulated a large, interdependent manifest across Modules 1-15; this module packages that complexity into a reusable Helm chart instead of asking every developer to hand-write it.
- `helm create` scaffolds a generic starter chart; Helmfy compiles existing plain manifests into one automatically. Both are used only as loose reference material, not as the chart's real templates.
- The chart, named `linuxtips`, is built from scratch, one resource at a time, with every optional platform capability gated behind a `values.yaml` toggle.

---

# Lesson 2: Namespace and Service Account

## 1. Gutting the Helmfy Scaffold

Work starts from the Helmfy-generated chart (renamed from its default output to `linuxtips`), but almost nothing in it is kept. Its default `values.yaml`, its auto-generated `templates/`, and its `_helpers.tpl` are all emptied out or deleted, leaving just the `Chart.yaml` metadata and an empty template directory to build into from scratch.

## 2. The `app` Values Block

Every value the chart needs lives under a single top-level `app` key in `values.yaml`, so that the whole chart can be reasoned about — and eventually documented — as one coherent object rather than a flat, unrelated bag of settings:

```yaml
# helm/linuxtips/values.yaml
app:
  name: linuxtips
  namespace: linuxtips
  image:
    repository: nginx
    tag: latest
    pullPolicy: IfNotPresent
  createNamespace: true
  iam: ""
```

`helm template debug linuxtips` (aliased as a quick way to compile the chart without applying it) and `kubectl create ns linuxtips --dry-run=client -o yaml` are both used repeatedly through this lesson as scratch tools — the dry-run output gives a real manifest to copy into a template file as a starting point, and `helm template` is re-run after almost every single edit to catch YAML and templating mistakes immediately rather than after a failed `helm upgrade`.

## 3. A Toggled Namespace

The first template created is the `Namespace` itself, gated by `app.createNamespace` so that a consumer deploying into a namespace someone else already owns can opt out of creating it:

```yaml
# helm/linuxtips/templates/namespace.yaml
{{- if .Values.app.createNamespace }}
apiVersion: v1
kind: Namespace
metadata:
  name: {{ .Values.app.namespace }}
  labels:
    istio-injection: enabled
{{- end}}
```

> **Note:** The `istio-injection: enabled` label shown above is part of the chart's final state, but it isn't added until [Lesson 7](#lesson-7-exposing-the-application-through-istio), when Istio mesh integration is introduced. Because this chart lives in a single Git commit on the `lesson/helm` branch (no intermediate commit-per-lesson history survives), the same real file is quoted at the point each of its fields is actually explained in the transcript, rather than only once.

## 4. A Mandatory Service Account for IRSA

Unlike the namespace, the `ServiceAccount` has no toggle — every application on this platform is required to have its own, so that IAM permissions (via IRSA) are never accidentally shared between unrelated workloads:

```yaml
# helm/linuxtips/templates/service-account.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: {{ .Values.app.name }}
  namespace: {{ .Values.app.namespace }}
  annotations:
    eks.amazonaws.com/role-arn: {{ .Values.app.iam }}
```

`app.iam` takes the ARN of whichever IAM role the application's pods should assume — the same `eks.amazonaws.com/role-arn` annotation pattern used for Pod Identity/IRSA throughout every module since Module 9.

## Key Takeaways

- The Helmfy-compiled scaffold is used only as a rename target and a source of copy-paste reference snippets; every real template is rebuilt from an empty file.
- All chart values live under one `app` key in `values.yaml`, and `helm template` is run after nearly every edit to catch mistakes immediately.
- The `Namespace` is optional (`createNamespace` toggle); the `ServiceAccount` is not — every workload on the platform must have its own, annotated with an IAM role ARN for IRSA.

---

# Lesson 3: The Service Template

## 1. A Minimal Service Spec: Type and Ports

The `Service` is reduced to the two pieces of information that actually vary between applications — its `type` and its list of ports — with everything else derived from the application's name:

```yaml
# helm/linuxtips/values.yaml (app block, continued)
  type: ClusterIP
  ports:
    - name: http
      port: 80
      targetPort: 80
    - name: https
      port: 443
      targetPort: 443
```

```yaml
# helm/linuxtips/templates/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: {{ .Values.app.name }}
  namespace: {{ .Values.app.namespace }}
  labels:
    version: {{ .Values.app.rollout.version }}
spec:
  type: {{ .Values.app.type }}
  selector:
    app: {{ .Values.app.name }}
  ports:
  {{- .Values.app.ports | toYaml | nindent 2 }}
```

Piping `.Values.app.ports` straight through `toYaml` means the ports list in `values.yaml` is rendered verbatim into the manifest — a consumer can add a third port, or change `targetPort`, without the template itself ever needing to change.

## 2. Rendering and Deploying

`helm template` catches an early YAML indentation mistake (a `toYaml` pipe with the dash not immediately touching the following bracket, which Helm's templating engine refuses to parse) before anything is ever applied to the cluster. Once it renders cleanly, `helm upgrade --install linuxtips ./linuxtips -n linuxtips` — always `upgrade --install` rather than a plain `install`, so the same command is idempotent whether or not a release already exists — creates the namespace and the `ClusterIP` Service exposing ports 80 and 443, confirmed with `kubectl get services -n linuxtips`.

## Key Takeaways

- A `Service` only needs its `type` and a `ports` list to vary between applications; both are pushed into `values.yaml` and piped through `toYaml` rather than hand-mapped field by field.
- `helm upgrade --install` is used everywhere in this module instead of `helm install`, so the same command works whether the release already exists or not.
- `helm template` is the fastest feedback loop for catching templating mistakes — it's used before every `helm upgrade` from this point forward.

---

# Lesson 4: Building the Rollout Template, Piece by Piece

## 1. Why a Complex Resource Needs Baby Steps

The `Rollout` is, by a wide margin, the most complex resource in the chart — it needs a progressive-delivery strategy, environment variables, probes, requests/limits, and port mappings, all in one object. Templating it in one pass risks producing something so rigid that a consumer has to override half of it just to get a working deployment, which defeats the point of a platform chart. Instead it's built the same way the rest of the chart is: one field group at a time, re-rendering with `helm template` after each addition to catch mistakes before they compound.

## 2. Capacity: Requests, Limits, and Autoscaling Bounds

A new `capacity` key groups every resource-sizing concern the Rollout (and, from Lesson 9 onward, KEDA) will need — requests, limits, and the min/max replica bounds autoscaling is allowed to operate within:

```yaml
# helm/linuxtips/values.yaml (app block, continued)
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
```

```yaml
# helm/linuxtips/templates/rollouts.yaml (excerpt)
spec:
  replicas: {{ .Values.app.capacity.autoscaling.min }}
  revisionHistoryLimit: {{ .Values.app.rollout.revisionHistoryLimit }}
  ...
        resources:
          requests:
            cpu: {{ .Values.app.capacity.requests.cpu }}
            memory: {{ .Values.app.capacity.requests.memory }}
          limits:
            cpu: {{ .Values.app.capacity.limits.cpu }}
            memory: {{ .Values.app.capacity.limits.memory }}
```

The Rollout's `replicas` field is driven by `capacity.autoscaling.min` rather than its own independent value — once KEDA is wired in during Lesson 9, this keeps the Rollout's starting replica count and the `ScaledObject`'s floor from silently drifting apart.

## 3. Container Ports via `range`

The same `app.ports` list introduced for the `Service` in Lesson 3 is reused here, looped with `range` to emit one `containerPort` per entry instead of hardcoding two:

```yaml
# helm/linuxtips/templates/rollouts.yaml (excerpt)
        ports:
{{- range .Values.app.ports}}
        - containerPort: {{ .targetPort}}
          name: {{ .name}}
{{- end}}
```

## 4. Feature-Toggled Probes

All three probe types — startup, readiness, and liveness — follow the exact same shape and the exact same toggle pattern, deliberately kept identical for now even though a real application would likely want different paths or thresholds per probe type:

```yaml
# helm/linuxtips/values.yaml (app block, continued)
  probes:
    startupProbe:
      enabled: true
      failureThreshold: 10
      periodSeconds: 10
      httpGet:
        path: /
        port: 80
    readinessProbe:
      enabled: true
      failureThreshold: 10
      periodSeconds: 10
      httpGet:
        path: /
        port: 80
    livenessProbe:
      enabled: true
      failureThreshold: 10
      periodSeconds: 10
      httpGet:
        path: /
        port: 80
```

```yaml
# helm/linuxtips/templates/rollouts.yaml (excerpt)
        {{- if .Values.app.probes.startupProbe.enabled }}
        startupProbe:
          failureThreshold: {{ .Values.app.probes.startupProbe.failureThreshold }}
          httpGet:
            path: {{ .Values.app.probes.startupProbe.httpGet.path }}
            port: {{ .Values.app.probes.startupProbe.httpGet.port }}
          periodSeconds: {{ .Values.app.probes.startupProbe.periodSeconds }}
        {{- end }}
```

The `readinessProbe` and `livenessProbe` blocks in the real template repeat this same structure verbatim, each gated by its own `.enabled` flag.

## 5. Environment Variables and the Image

Environment variables are passed through as a raw list, exactly like the ports, rather than being templated field by field:

```yaml
# helm/linuxtips/values.yaml (app block, continued)
  envs:
    - name: ENV
      value: "dev"
    - name: FOO
      value: "bar"
    - name: VERSION
      value: v3
```

```yaml
# helm/linuxtips/templates/rollouts.yaml (excerpt)
      containers:
      - name: {{ .Values.app.name }}
        image: {{ .Values.app.image.repository }}:{{ .Values.app.image.tag }}
        imagePullPolicy: {{ .Values.app.image.pullPolicy }}

        env:
          {{- toYaml .Values.app.envs | nindent 8 }}
```

The image itself is assembled from the three fields already declared in Lesson 2's `app.image` block (`repository`, `tag`, `pullPolicy`), so a consumer only ever edits `values.yaml` to change what's running, never the template.

## Key Takeaways

- The Rollout is the chart's most complex resource, so it's built and re-rendered field group by field group instead of all at once: capacity, then ports, then probes, then envs and image.
- `capacity.autoscaling.min` drives both the Rollout's starting `replicas` and, from Lesson 9 on, the KEDA `ScaledObject`'s floor — one number instead of two that can drift apart.
- Ports and environment variables are both passed through as raw lists (`range` for ports, `toYaml` for envs) rather than hand-mapped, so a consumer can add either without touching the template.

---

# Lesson 5: Canary Strategy and Version Labels

## 1. Choosing Canary Over Blue-Green

An AI-assisted suggestion for the `strategy` block defaults to a `blueGreen` shape — a reasonable guess, since blue-green was the most recently covered strategy in Module 15, but not what this chart standardizes on. The `blueGreen` suggestion is discarded in favor of `canary`, gated by its own toggle so a future iteration of the chart could add blue-green back as an alternative:

```yaml
# helm/linuxtips/templates/rollouts.yaml (excerpt)
  strategy:
    {{- if .Values.app.rollout.strategy.canary.enabled }}
    canary:
      steps:
      {{ toYaml .Values.app.rollout.strategy.canary.steps | nindent 6 }}
    {{- end}}
```

> **Note:** No file on the `lesson/helm` branch preserves the discarded `blueGreen` suggestion — the chart's single commit only contains the final, canary-only `strategy` block quoted above. The `blueGreen` shape is described here from the lesson's own narration for context, not reproduced from a real file.

## 2. Passing the Whole Steps List Through `toYaml`

Rather than templating each canary step's fields individually, the entire `steps` list is declared as plain data in `values.yaml` and piped through `toYaml` as a single block — the same pattern already used for `ports` and `envs`:

```yaml
# helm/linuxtips/values.yaml (app.rollout block)
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
                - templateName: istio-success
          - setWeight: 40
          - pause: {duration: 30s}
          - setWeight: 60
          - pause: {duration: 30s}
          - setWeight: 80
          - pause: {duration: 30s}
          - setWeight: 100
```

This makes the canary progression itself fully consumer-configurable — a step count, weight schedule, or pause duration can be changed from `values.yaml` alone — while keeping the `analysis` step (wired up properly in [Lesson 8](#lesson-8-parameterizing-analysistemplates)) as a first-class part of the same list.

## 3. Version Labels for Traceability

A `version` label, driven by `app.rollout.version`, is added to both the Rollout's pod template and the plain `Service` from Lesson 3, so the currently running version is visible directly on `kubectl get pods --show-labels` and on the Service itself without needing to inspect the Rollout's status:

```yaml
# helm/linuxtips/templates/rollouts.yaml (excerpt)
  template:
    metadata:
      labels:
        app: {{ .Values.app.name }}
        name: {{ .Values.app.name }}
        version: {{ .Values.app.rollout.version }}
```

## Key Takeaways

- The chart standardizes on a canary strategy, gated by its own toggle; an initially suggested `blueGreen` shape was discarded and isn't part of the final chart.
- The full canary `steps` list is passed straight from `values.yaml` through `toYaml`, so the progression itself is fully consumer-configurable without touching the template.
- A `version` label (from `app.rollout.version`) is stamped on both the Rollout's pods and its Service, making the running version visible without inspecting Rollout status directly.

---

# Lesson 6: Spreading Pods Across Zones and Nodes

## 1. Two `topologySpreadConstraints` by Default

Every workload deployed through this chart gets multi-AZ and multi-node spread by default, with no toggle and no opt-out — a consumer can't accidentally ship an application that piles every replica onto a single zone or a single node:

```yaml
# helm/linuxtips/templates/rollouts.yaml (excerpt)
      topologySpreadConstraints:
      - labelSelector:
          matchLabels:
            app: {{ .Values.app.name }}
        maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: ScheduleAnyway

      - labelSelector:
          matchLabels:
            app: {{ .Values.app.name }}
        maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: ScheduleAnyway
```

The first constraint spreads pods across availability zones (`topology.kubernetes.io/zone`); the second spreads them across individual nodes (`kubernetes.io/hostname`), so that even within a single zone, replicas don't stack up on the same underlying node.

## 2. Targeting a Karpenter NodePool Through a Toggle

Unlike the spread constraints, which node the workload can land on at all is optional and consumer-controlled, reusing the same `capacity` key introduced in Lesson 4:

```yaml
# helm/linuxtips/templates/rollouts.yaml (excerpt)
      {{- if .Values.app.capacity.nodepool.enabled }}
      nodeSelector:
        karpenter.sh/nodepool: {{ .Values.app.capacity.nodepool.name }}
      {{- end }}
```

With `nodepool.enabled: true` and `nodepool.name: general` in `values.yaml`, the Rollout's pods are pinned to Karpenter's `general` NodePool; setting it to `false` lets Karpenter schedule the workload onto any NodePool available in the cluster.

## 3. An Anti-Affinity Experiment That Didn't Survive

Before settling on the two `topologySpreadConstraints` above as the chart's final spreading mechanism, a `podAntiAffinity` block with `requiredDuringSchedulingIgnoredDuringExecution` (keyed on the same `topology.kubernetes.io/zone` label) is tried as a way to *force* zone spread rather than merely prefer it. It's removed again at the start of [Lesson 7](#lesson-7-exposing-the-application-through-istio) in favor of keeping the topology constraints alone.

> **Note:** As with the discarded `blueGreen` strategy in Lesson 5, the anti-affinity experiment isn't preserved in any file on the `lesson/helm` branch — the final `rollouts.yaml` contains only the two `topologySpreadConstraints` shown above, with no `affinity` block at all. It's described here for context, not reproduced from a real file.

## Key Takeaways

- Two `topologySpreadConstraints` — one for AZ, one for node — are unconditional on every workload the chart deploys; there's no toggle to opt out of basic spread.
- Which Karpenter NodePool a workload can land on is optional (`capacity.nodepool.enabled`), reusing the same `capacity` values key as requests, limits, and autoscaling bounds.
- A stricter `podAntiAffinity` alternative was tried and then discarded in favor of the two spread constraints alone — it doesn't appear anywhere in the chart's final state.

---

# Lesson 7: Exposing the Application Through Istio

## 1. Enabling Sidecar Injection at the Namespace Level

Before any Istio resource can do anything useful, the namespace itself needs to opt into automatic sidecar injection — the same `istio-injection: enabled` label used everywhere else in this project since Module 13, added here to the `Namespace` template already built in Lesson 2:

```yaml
# helm/linuxtips/templates/namespace.yaml
{{- if .Values.app.createNamespace }}
apiVersion: v1
kind: Namespace
metadata:
  name: {{ .Values.app.namespace }}
  labels:
    istio-injection: enabled
{{- end}}
```

Existing pods have to be recycled (deleted, so the Rollout's controller recreates them) before they pick up the sidecar — the label only affects pods scheduled after it's applied.

## 2. A Toggled Gateway

The `Gateway` is built directly from the loose Helmfy reference, stripped down to just its `host`, `port`, and `protocol`:

```yaml
# helm/linuxtips/values.yaml (app.istio block)
  istio:
    host: linuxtips.msfidelis.com.br
    gateway:
      enabled: true
      protocol: HTTP
      port: 80
```

```yaml
# helm/linuxtips/templates/istio-gateway.yaml
{{- if .Values.app.istio.gateway.enabled}}
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: {{ .Values.app.name }}
  namespace: {{ .Values.app.namespace }}
spec:
  selector:
    istio: ingressgateway
  servers:
  - hosts:
    - {{ .Values.app.istio.host }}
    port:
      name: http
      number: {{ .Values.app.istio.gateway.port }}
      protocol: {{ .Values.app.istio.gateway.protocol }}
{{- end}}
```

## 3. A Toggled VirtualService With Retries

The `VirtualService` follows the same pattern, adding its own retry policy as consumer-configurable fields rather than a fixed default buried in the template:

```yaml
# helm/linuxtips/values.yaml (app.istio block, continued)
    virtualService:
      enabled: true
      http:
        enabled: true
        port: 80
        retries:
          attempts: 1
          perTryTimeout: 2s
          retryOn: 5xx
```

```yaml
# helm/linuxtips/templates/istio-virtualservices.yaml
{{- if .Values.app.istio.virtualService.enabled}}
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: {{ .Values.app.name }}
  namespace: {{ .Values.app.namespace }}
spec:
  gateways:
  - {{ .Values.app.name }}
  hosts:
  - {{ .Values.app.name }}.{{ .Values.app.namespace }}.svc.cluster.local
  - {{ .Values.app.istio.host }}
  http:
  - retries:
      attempts: {{ .Values.app.istio.virtualService.http.retries.attempts }}
      perTryTimeout: {{ .Values.app.istio.virtualService.http.retries.perTryTimeout }}
      retryOn: {{ .Values.app.istio.virtualService.http.retries.retryOn }}
    route:
    - destination:
        host: {{ .Values.app.name }}
        port:
          number: {{ .Values.app.istio.virtualService.http.port }}
{{- end}}
```

The `hosts` list covers both the external hostname and the Service's own in-mesh DNS name, so the `VirtualService` routes traffic arriving from the Gateway and traffic arriving from other in-mesh services through the same retry policy.

## Key Takeaways

- Sidecar injection is enabled at the namespace level via the same `istio-injection: enabled` label used since Module 13; existing pods must be recycled to pick it up.
- Both the `Gateway` and `VirtualService` are optional (`istio.gateway.enabled`, `istio.virtualService.enabled`) and reduced to the handful of fields that actually vary: host, port, protocol, and retry policy.
- The `VirtualService`'s `hosts` list covers both the external hostname and the Service's in-mesh DNS name, so one retry policy applies to both ingress and internal mesh traffic.

---

# Lesson 8: Parameterizing AnalysisTemplates

## 1. Why AnalysisTemplates Are Hard to Templatize

`AnalysisTemplate`s are the hardest resource in the chart to make generic: their `spec` shape varies significantly depending on the analysis provider (Prometheus, a Job, or something else), and a chart consumer might reasonably need more than one of them. Rather than trying to enumerate every possible provider shape as explicit `values.yaml` fields, the chart takes the opposite approach and keeps the `spec` itself as free-form, pass-through data.

## 2. Looping Over a List of Templates

`app.rollout.analysisTemplates` is a list, so the chart can render as many `AnalysisTemplate`s as a consumer declares, each with a `name` and an arbitrary `spec` blob:

```yaml
# helm/linuxtips/values.yaml (app.rollout block, continued)
    analysisTemplates:
    - name: 'istio-success'
      spec:
        metrics:
        - name: success-rate
          interval: 2m
          failureLimit: 0
          successCondition: result[0] >= 0.95
          count: 1
          provider:
            prometheus:
              address: http://prometheus-kube-prometheus-prometheus.prometheus.svc.cluster.local:9090
              query: |
                sum(irate(
                  istio_requests_total{destination_service=~"linuxtips.linuxtips.svc.cluster.local",response_code!~"5.*"}[1m]
                )) /
                sum(irate(
                  istio_requests_total{destination_service=~"linuxtips.linuxtips.svc.cluster.local"}[1m]
                ))
```

```yaml
# helm/linuxtips/templates/analysistemplates.yaml
{{- range .Values.app.rollout.analysisTemplates }}
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: {{ .name }}
  namespace: {{ $.Values.app.namespace }}
spec:
  {{ .spec | toYaml | nindent 2 }}
{{- end}}
```

Because `range` changes the meaning of `.` to the current list item, the namespace has to be reached through `$.Values.app.namespace` (the `$` referring back to the template's root context) rather than the usual `.Values.app.namespace`.

## 3. Wiring an Analysis Step Into the Canary

With the `AnalysisTemplate` rendering successfully, the same `analysis` step already present in Lesson 5's `steps` list becomes fully functional — `templateName: istio-success` now resolves to a real, chart-managed resource instead of a static reference to a manifest that would otherwise have to be applied separately. A quick round trip through the dashboard (dropping `pause` durations down to something short, like 30 seconds) confirms the `AnalysisRun` triggers and completes successfully even with no real traffic yet flowing through `istio-success`'s query — a smoke test of the wiring itself, not of the metric's correctness. Once confirmed, the extra debugging environment variable used to trigger the test rollout is removed again.

## Key Takeaways

- `AnalysisTemplate` specs are kept as free-form pass-through data (`spec | toYaml`) rather than broken into explicit fields, since their shape varies too much by provider to templatize field by field.
- `app.rollout.analysisTemplates` is a list, so the chart can render as many `AnalysisTemplate`s as a consumer needs, each rendered inside a `range` loop.
- Inside a `range` block, the root values have to be reached via `$.Values...` instead of `.Values...`, since `range` rebinds `.` to the current loop item.

---

# Lesson 9: A Templated KEDA ScaledObject for Rollouts

## 1. One ScaledObject per Workload

KEDA only allows a single `ScaledObject` per scale target, so unlike triggers (which can be a list), the `ScaledObject` itself is a single resource per application, gated by one top-level toggle:

```yaml
# helm/linuxtips/values.yaml (app.keda block)
  keda:
    enabled: true
```

```yaml
# helm/linuxtips/templates/keda.yaml (excerpt)
{{- if .Values.app.keda.enabled}}

apiVersion: keda.sh/v1alpha1
kind: ScaledObject
metadata:
  name: {{ .Values.app.name }}
  namespace: {{ .Values.app.namespace }}
spec:
```

## 2. Pointing Autoscaling at a Rollout Instead of a Deployment

Every KEDA `ScaledObject` seen through Module 14 targeted a plain `Deployment`, which is what KEDA assumes by default. Since Module 15 replaced `chip`'s `Deployment` with a `Rollout`, the `scaleTargetRef` here has to be explicit about both the target's `kind` and its `apiVersion` — without it, KEDA would look for a `Deployment` that no longer exists:

```yaml
# helm/linuxtips/templates/keda.yaml (excerpt)
  scaleTargetRef:
    apiVersion: argoproj.io/v1alpha1
    kind: Rollout
    name: {{ .Values.app.name }}
```

## 3. Reusing HPA Behavior and Capacity Bounds

The scale-down/scale-up stabilization windows, cooldown period, and polling interval are lifted directly from the KEDA `ScaledObject`s built in Module 14, made consumer-configurable, while the min/max replica bounds are pulled from the same `capacity.autoscaling` values already driving the Rollout's own `replicas` field:

```yaml
# helm/linuxtips/values.yaml (app.keda block, continued)
    hpaConfig:
      behavior:
        scaleDown:
          stabilizationWindowSeconds: 20
        scaleUp:
          stabilizationWindowSeconds: 20
      cooldownPeriod: 30
      pollingInterval: 10
```

```yaml
# helm/linuxtips/templates/keda.yaml (excerpt)
  advanced:
    horizontalPodAutoscalerConfig:
      behavior:
        scaleDown:
          stabilizationWindowSeconds:  {{ .Values.app.keda.hpaConfig.behavior.scaleDown.stabilizationWindowSeconds}}
        scaleUp:
          stabilizationWindowSeconds: {{ .Values.app.keda.hpaConfig.behavior.scaleUp.stabilizationWindowSeconds}}
  cooldownPeriod: {{ .Values.app.keda.hpaConfig.cooldownPeriod }}
  pollingInterval: {{ .Values.app.keda.hpaConfig.pollingInterval }}
  maxReplicaCount:  {{ .Values.app.capacity.autoscaling.max }}
  minReplicaCount: {{ .Values.app.capacity.autoscaling.min }}
```

At this point the `ScaledObject` still has no `triggers` — attempting `helm upgrade` fails validation, since KEDA requires at least one trigger, which is exactly what the next lesson builds out.

## Key Takeaways

- A `ScaledObject` is singular per workload (unlike triggers), so it's gated by one `keda.enabled` toggle rather than rendered as a list.
- Because `chip` runs as a `Rollout`, not a `Deployment`, `scaleTargetRef` must explicitly declare `apiVersion: argoproj.io/v1alpha1` and `kind: Rollout` — KEDA defaults to targeting a `Deployment` otherwise.
- HPA behavior settings are lifted from Module 14's KEDA lab and made configurable; min/max replica bounds are reused from the same `capacity.autoscaling` key that drives the Rollout's `replicas`, avoiding two independent sources of truth.

---

# Lesson 10: Productizing KEDA Triggers

## 1. Prometheus Triggers as a List

Unlike the singular `ScaledObject`, triggers can be a list, so `app.keda.prometheus` holds zero or more Prometheus-based trigger definitions, each rendered by a `range` loop directly into the `ScaledObject`'s `triggers` array:

```yaml
# helm/linuxtips/values.yaml (app.keda block, continued — commented out by default)
    prometheus:
    # - name: keda-scale-tps
    #   metricName: istio_requests_total
    #   query: "sum(rate(istio_requests_total{destination_service_name=\"linuxtips\"}[1m]))"
    #   serverAddress: http://prometheus-kube-prometheus-prometheus.prometheus.svc.cluster.local:9090
    #   threshold: "10"
```

```yaml
# helm/linuxtips/templates/keda.yaml (excerpt)
  triggers:
{{- range .Values.app.keda.prometheus}}
  - type: prometheus
    metadata:
      serverAddress: {{ .serverAddress }}
      metricName: {{ .metricName }}
      query: {{ .query }}
      threshold: {{ .threshold | quote }}
{{- end}}
```

## 2. Scheduled Scaling With Cron Triggers

A `cron` list follows the same shape, letting a consumer declare one or more scheduled scale-ups without touching the template:

```yaml
# helm/linuxtips/values.yaml (app.keda block, continued — commented out by default)
    cron:
    # - name: scale-up
    #   start: 40 14 * * *
    #   end: 40 18 * * *
    #   timezone: "America/Sao_Paulo"
    #   desiredReplicas: 20
```

```yaml
# helm/linuxtips/templates/keda.yaml (excerpt)
{{- range .Values.app.keda.cron }}
  - type: cron
    metadata:
      timezone: {{ .timezone }}
      start: {{ .start }}
      end: {{ .end }}
      desiredReplicas: "{{ .desiredReplicas }}"
{{- end}}
```

A live test with real timestamps a few minutes in the future confirms the schedule works end to end, but the KEDA `cron` trigger only exposes a `desiredReplicas` value, not independent min/max bounds — unlike the `ScaledObject`'s own `min`/`maxReplicaCount`, there's no way to express a cron-driven range, only a single target replica count during the window.

## 3. Memory and CPU Triggers

Memory and CPU triggers share the same two fields — a `metricType` (`Utilization` or `AverageValue`) and a `threshold`:

```yaml
# helm/linuxtips/templates/keda.yaml (excerpt)
{{- range .Values.app.keda.memory }}
  - type: memory
    metricType: {{ .metricType }}
    metadata:
      value: {{ .threshold | quote }}
{{- end}}

{{- range .Values.app.keda.cpu }}
  - type: cpu
    metricType: {{ .metricType }}
    metadata:
      value: {{ .threshold | quote }}
{{- end}}
```

## 4. SQS Triggers and a Global TriggerAuthentication

The SQS trigger needs an AWS region and queue URL, and — unlike Prometheus, cron, memory, or CPU — depends on a `TriggerAuthentication` so KEDA can actually reach AWS. Since every workload on the platform already gets an IAM-annotated ServiceAccount (Lesson 2), that same identity is reused via Pod Identity rather than static AWS credentials, and the `TriggerAuthentication` itself is created unconditionally (not gated by a toggle) since it costs nothing to have present even when unused:

```yaml
# helm/linuxtips/templates/keda.yaml (excerpt)
{{- range .Values.app.keda.sqs }}
  - type: aws-sqs-queue
    authenticationRef:
      name: {{ $.Values.app.name }}
    metadata:
      queueURL: {{ .queueURL }}
      queueLength: {{ .queueLength | quote }}
      region: {{ .region }}
{{- end}}
```

```yaml
# helm/linuxtips/templates/trigger-authentication.yaml
apiVersion: keda.sh/v1alpha1
kind: TriggerAuthentication
metadata:
  name: {{ .Values.app.name }}
  namespace: {{ .Values.app.namespace }}
spec:
  podIdentity:
    provider: aws
```

## 5. Keeping Only One Trigger Active

With five trigger types now templated, only one is left enabled in the chart's shipped `values.yaml` — the rest are commented out as reference examples rather than deleted, since KEDA supports combining trigger types but doing so isn't this chart's default behavior:

```yaml
# helm/linuxtips/values.yaml (app.keda block, final state)
    cpu:
    - name: cpu-high
      metricType: Utilization
      threshold: 80
```

## Key Takeaways

- Five KEDA trigger types — Prometheus, cron, memory, CPU, and SQS — are all templated as `range`-driven lists, so a consumer can mix and match by uncommenting entries in `values.yaml` rather than editing templates.
- The `cron` trigger only supports a single `desiredReplicas` target during its window, not an independent min/max range like the `ScaledObject` itself.
- SQS scaling needs a `TriggerAuthentication` using the same Pod Identity already wired up for the ServiceAccount; it's rendered unconditionally since it costs nothing when no SQS trigger is active. The chart ships with only the CPU trigger enabled by default.

---

# Lesson 11: Automating Metric Collection With a ServiceMonitor

## 1. A Toggled ServiceMonitor Template

Every workload that wants its metrics scraped by the shared `kube-prometheus-stack` from Module 12 needs a matching `ServiceMonitor`, gated by its own toggle so applications that don't expose Prometheus metrics don't get an orphaned scrape target:

```yaml
# helm/linuxtips/values.yaml (app.prometheus block)
  prometheus:
    serviceMonitor:
      enabled: true
      interval: 30s
      scrapeTimeout: 10s
      port: http
      path: /metrics
```

```yaml
# helm/linuxtips/templates/servicemonitor.yml
{{- if .Values.app.prometheus.serviceMonitor.enabled}}

apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: {{ .Values.app.name }}-monitor
  namespace: {{ .Values.app.namespace }}
  labels:
    release: prometheus
spec:
  selector:
    matchLabels:
      app: {{ .Values.app.name }}

  endpoints:
    - port: {{ .Values.app.prometheus.serviceMonitor.port }}
      path: {{ .Values.app.prometheus.serviceMonitor.path }}
      interval: {{ .Values.app.prometheus.serviceMonitor.interval }}
      scrapeTimeout: "{{ .Values.app.prometheus.serviceMonitor.scrapeTimeout }}"
  namespaceSelector:
    matchNames:
      - {{ .Values.app.namespace }}

{{- end}}
```

The `release: prometheus` label matches the `kube-prometheus-stack` release name from Module 12, which is what makes the Prometheus Operator pick this `ServiceMonitor` up automatically without any manual wiring.

## 2. A Scrape Timeout That Must Be a String

The first `helm upgrade` attempt with this template fails Kubernetes API validation because `scrapeTimeout` is a duration-shaped field that the OpenAPI schema requires as a `string`, not a bare Helm duration value — the fix is the explicit `"{{ ... }}"` quoting visible on that field above, while `interval` (declared identically in `values.yaml`) doesn't need the same treatment.

> **Note:** `port`, `path`, and `interval` are all interpolated without quotes in the real template, while `scrapeTimeout` is the one field wrapped in explicit quotes — a real, deliberate asymmetry in the shipped file caused by the schema validation error described above, not an inconsistency introduced by this write-up.

## Key Takeaways

- The `ServiceMonitor` is optional per workload (`prometheus.serviceMonitor.enabled`) and relies on the `release: prometheus` label matching the `kube-prometheus-stack` Helm release name for auto-discovery.
- `scrapeTimeout` has to be explicitly quoted as a string in the template — the Kubernetes API rejects it as a bare duration value, unlike the otherwise-identical `interval` field.

---

# Lesson 12: Packaging the Chart

## 1. Cleaning Up the Feature-Toggle Debris

With every resource templated and working, the last step before packaging is a pass to remove leftover debugging artifacts accumulated across the previous eleven lessons — stray environment variables added only to force a test rollout, commented-out trigger examples left in place deliberately as documentation (Lesson 10), and any values left toggled on purely for testing that shouldn't ship enabled by default.

## 2. `helm package`

`helm package ./linuxtips` produces the chart's first versioned archive, matching the version already declared in `Chart.yaml`:

```yaml
# helm/linuxtips/Chart.yaml
apiVersion: v2
name: linuxtips
description: A Helm chart for Kubernetes
type: application
version: 0.1.0
appVersion: "0.1.0"
```

The resulting `linuxtips-0.1.0.tgz` is the artifact the next module consumes: rather than every application team writing and maintaining their own Rollout, Istio, and KEDA manifests, Argo CD deploys this one chart, parameterized per application through its own `values.yaml`, across every lab built so far in this project.

## Key Takeaways

- The chart is packaged with `helm package`, producing a versioned `.tgz` archive from the `version` already declared in `Chart.yaml`.
- A cleanup pass removes debugging leftovers accumulated while building each resource, but keeps disabled trigger examples in `values.yaml` intentionally, as inline documentation for chart consumers.
- The packaged chart is the hand-off point to the next module, where Argo CD consumes it to deploy every lab application built across this project through one standardized, parameterized template.
