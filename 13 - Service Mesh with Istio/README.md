# Module 13: Service Mesh with Istio

## Overview

This module retires the NGINX Ingress Controller in favor of Istio, moving traffic management, security, and observability out of application code and into a dedicated infrastructure layer. Istio is installed via Helm in three pieces — `base` (CRDs), `istiod` (control plane), and the ingress `gateway` chart — and the Ingress Gateway takes over the NodePort + AWS `TargetGroupBinding` pattern already used by the NLB from earlier modules. A multi-service lab application (`health-api`, with six internal gRPC dependencies, plus a separate `chip` service) is deployed with automatic sidecar injection to exercise the mesh under realistic east-west traffic. From there, the module wires Istio into the observability stack built in Module 12 — exposing Grafana through an Istio `Gateway`/`VirtualService` instead of NGINX, scraping every Envoy sidecar with a dedicated `PodMonitor`, and installing Jaeger for distributed tracing. Kiali ties tracing, metrics, and topology together into a single mesh dashboard. The module closes with two resilience patterns enforced entirely at the mesh level — automatic retries and circuit breaking via `VirtualService`/`DestinationRule` — and a `ServiceEntry` that brings external, out-of-mesh traffic under the same policies.

## Table of Contents

- [Lesson 1: Introduction to Istio and Service Mesh](#lesson-1-introduction-to-istio-and-service-mesh)
  - [1. What a Service Mesh Actually Does](#1-what-a-service-mesh-actually-does)
  - [2. The Sidecar Proxy Pattern](#2-the-sidecar-proxy-pattern)
  - [3. mTLS and Service-to-Service Communication](#3-mtls-and-service-to-service-communication)
  - [Key Takeaways](#key-takeaways)
- [Lesson 2: Installing Istio with Helm](#lesson-2-installing-istio-with-helm)
  - [1. Retiring the NGINX Ingress Controller](#1-retiring-the-nginx-ingress-controller)
  - [2. Declaring the Istio Version](#2-declaring-the-istio-version)
  - [3. Installing istio-base: CRDs and Cluster-Wide Resources](#3-installing-istio-base-crds-and-cluster-wide-resources)
  - [4. Installing istiod: the Control Plane](#4-installing-istiod-the-control-plane)
  - [Key Takeaways](#key-takeaways-1)
- [Lesson 3: Productionizing the Ingress Gateway](#lesson-3-productionizing-the-ingress-gateway)
  - [1. Installing the Ingress Gateway Chart](#1-installing-the-ingress-gateway-chart)
  - [2. Exposing NodePort Service and Autoscaling](#2-exposing-nodeport-service-and-autoscaling)
  - [3. Binding the Gateway to the Existing Network Load Balancer](#3-binding-the-gateway-to-the-existing-network-load-balancer)
  - [Key Takeaways](#key-takeaways-2)
- [Lesson 4: Deploying a Sidecar-Injected Lab Application](#lesson-4-deploying-a-sidecar-injected-lab-application)
  - [1. Enabling Automatic Sidecar Injection](#1-enabling-automatic-sidecar-injection)
  - [2. Exposing health-api with a Gateway and VirtualService](#2-exposing-health-api-with-a-gateway-and-virtualservice)
  - [3. Internal gRPC Services Without a Gateway](#3-internal-grpc-services-without-a-gateway)
  - [4. Deploying the chip Service](#4-deploying-the-chip-service)
  - [Key Takeaways](#key-takeaways-3)
- [Lesson 5: Migrating Grafana and Monitoring the Envoy Sidecars](#lesson-5-migrating-grafana-and-monitoring-the-envoy-sidecars)
  - [1. Exposing Grafana Through the Istio Gateway](#1-exposing-grafana-through-the-istio-gateway)
  - [2. Scraping Envoy Metrics with a PodMonitor](#2-scraping-envoy-metrics-with-a-podmonitor)
  - [Key Takeaways](#key-takeaways-4)
- [Lesson 6: Distributed Tracing with Jaeger](#lesson-6-distributed-tracing-with-jaeger)
  - [1. Installing Jaeger as an All-in-One Deployment](#1-installing-jaeger-as-an-all-in-one-deployment)
  - [2. Enabling Mesh-Wide Tracing on istiod](#2-enabling-mesh-wide-tracing-on-istiod)
  - [3. Exposing the Jaeger Query UI](#3-exposing-the-jaeger-query-ui)
  - [Key Takeaways](#key-takeaways-5)
- [Lesson 7: Installing and Configuring Kiali](#lesson-7-installing-and-configuring-kiali)
  - [1. Installing the Kiali Server](#1-installing-the-kiali-server)
  - [2. Integrating Kiali with Jaeger, Prometheus, and Grafana](#2-integrating-kiali-with-jaeger-prometheus-and-grafana)
  - [3. Exposing Kiali](#3-exposing-kiali)
  - [Key Takeaways](#key-takeaways-6)
- [Lesson 8: Resilience with Retries and Circuit Breaking](#lesson-8-resilience-with-retries-and-circuit-breaking)
  - [1. Automatic Retries with a VirtualService](#1-automatic-retries-with-a-virtualservice)
  - [2. Circuit Breaking with a DestinationRule](#2-circuit-breaking-with-a-destinationrule)
  - [Key Takeaways](#key-takeaways-7)
- [Lesson 9: Mapping External Services](#lesson-9-mapping-external-services)
  - [1. The Problem with Out-of-Mesh Traffic](#1-the-problem-with-out-of-mesh-traffic)
  - [2. Registering an External Host with a ServiceEntry](#2-registering-an-external-host-with-a-serviceentry)
  - [Key Takeaways](#key-takeaways-8)

---

# Lesson 1: Introduction to Istio and Service Mesh

## 1. What a Service Mesh Actually Does

A service mesh moves cross-cutting network concerns — routing, retries, timeouts, circuit breaking, mutual TLS, and observability — out of application code and into infrastructure. None of this requires touching the application itself: routing rules, resilience policies, and telemetry are all declared as Kubernetes-native resources and enforced by the mesh's own data plane.

## 2. The Sidecar Proxy Pattern

Istio implements this by injecting an Envoy proxy as a sidecar container into every pod that participates in the mesh. All inbound and outbound traffic for the pod is transparently intercepted by this proxy. Envoy doesn't make routing decisions on its own — it continuously receives configuration from the mesh's control plane, `istiod`, which is what turns `Gateway`, `VirtualService`, and `DestinationRule` objects into live proxy configuration.

## 3. mTLS and Service-to-Service Communication

Because every service-to-service call passes through a pair of Envoy sidecars, Istio can transparently encrypt that traffic with mutual TLS, without either application being aware it's happening. Combined with the mesh's built-in telemetry, this is what makes it possible to secure and observe traffic between services without changing a single line of application code.

## Key Takeaways

- A service mesh externalizes routing, resilience, security, and observability from application code into the infrastructure layer.
- Istio enforces this through Envoy sidecar proxies that transparently intercept all pod traffic.
- The control plane, `istiod`, is what turns Istio's Kubernetes-native resources into live Envoy configuration.
- mTLS between sidecars is possible precisely because every call already passes through a pair of Envoy proxies.

---

# Lesson 2: Installing Istio with Helm

## 1. Retiring the NGINX Ingress Controller

The NGINX Ingress Controller used since earlier modules is retired in favor of the Istio Ingress Gateway. Its Helm release and the `TargetGroupBinding` that pointed the shared Network Load Balancer at it are commented out rather than deleted, preserving the history of how the cluster's edge traffic was previously handled:

```hcl
# helm_nginx.tf
# resource "helm_release" "nginx_controller" {
#   name       = "ingress-nginx"
#   namespace  = "ingress-nginx"
#   chart      = "ingress-nginx"
#   repository = "https://kubernetes.github.io/ingress-nginx"
#   version    = "4.11.3"
#   ...
# }

# resource "kubectl_manifest" "target_binding_80" {
#   yaml_body = <<YAML
# apiVersion: elbv2.k8s.aws/v1beta1
# kind: TargetGroupBinding
# metadata:
#   name: ingress-nginx
#   namespace: ingress-nginx
# spec:
#   serviceRef:
#     name: ingress-nginx-controller
#     port: 80
#   targetGroupARN: ${aws_lb_target_group.main.arn}
#   targetType: instance
# YAML
#   depends_on = [
#     helm_release.nginx_controller
#   ]
# }
```

The Network Load Balancer, its target group, and its listener stay exactly as they are — only what gets bound to the target group changes, from the NGINX controller's Service to the Istio Ingress Gateway's Service later in this module.

## 2. Declaring the Istio Version

A single variable pins the Istio version used across all three Helm releases installed in this module:

```hcl
# variables.tf
// Istio

variable "istio_version" {
  type        = string
  description = "Versão do Istio"
  default     = "1.25.0"
}
```

## 3. Installing istio-base: CRDs and Cluster-Wide Resources

Istio's Helm installation is split across multiple charts. The first, `base`, installs the CustomResourceDefinitions (`Gateway`, `VirtualService`, `DestinationRule`, `ServiceEntry`, and the rest) and other cluster-wide resources that every other Istio component depends on:

```hcl
# helm_istio.tf
resource "helm_release" "istio_base" {
  name       = "istio-base"
  chart      = "base"
  repository = "https://istio-release.storage.googleapis.com/charts"
  namespace  = "istio-system"

  create_namespace = true

  version = var.istio_version

  depends_on = [
    aws_eks_cluster.main,
    helm_release.karpenter
  ]
}
```

## 4. Installing istiod: the Control Plane

With the CRDs in place, `istiod` — the control plane — is installed next. It depends on `istio_base` and configures two behaviors that matter for the rest of the module: it disables `sidecarInjectorWebhook.rewriteAppHTTPProbe` (so injected sidecars don't rewrite application health-check probes), and it turns on mesh-wide tracing, pointing it at the Zipkin-compatible endpoint that Jaeger will expose in Lesson 6:

```hcl
# helm_istio.tf
resource "helm_release" "istiod" {
  name       = "istio"
  chart      = "istiod"
  repository = "https://istio-release.storage.googleapis.com/charts"
  namespace  = "istio-system"

  create_namespace = true

  version = var.istio_version

  set {
    name  = "sidecarInjectorWebhook.rewriteAppHTTPProbe"
    value = "false"
  }

  set {
    name  = "meshConfig.enableTracing"
    value = "true"
  }

  set {
    name  = "meshConfig.defaultConfig.tracing.zipkin.address"
    value = "jaeger-collector.tracing.svc.cluster.local:9411"
  }

  depends_on = [
    helm_release.istio_base
  ]
}
```

The tracing address already points at `jaeger-collector.tracing.svc.cluster.local:9411` even though Jaeger itself is only installed in Lesson 6 — `istiod` simply won't have anywhere to send spans until that Helm release exists.

## Key Takeaways

- Istio's Helm installation is split into independent charts (`base`, `istiod`, `gateway`), each with its own release and its own `depends_on` chain.
- `base` must be installed first: it owns the CRDs every other Istio resource, and every other chart, depends on.
- `istiod` is where mesh-wide behavior is configured — including tracing, which is wired up here even though the tracing backend isn't installed until later.
- Retiring an old ingress path is done by commenting it out, not deleting it, preserving how the cluster's edge traffic evolved.

---

# Lesson 3: Productionizing the Ingress Gateway

## 1. Installing the Ingress Gateway Chart

The third and final Istio Helm chart, `gateway`, installs the Ingress Gateway itself — the mesh's actual entry point for external traffic, running as its own Envoy proxy deployment:

```hcl
# helm_istio.tf
resource "helm_release" "istio_ingress" {
  name             = "istio-ingressgateway"
  chart            = "gateway"
  repository       = "https://istio-release.storage.googleapis.com/charts"
  namespace        = "istio-system"
  create_namespace = true

  version = var.istio_version
```

## 2. Exposing NodePort Service and Autoscaling

The Ingress Gateway's Service is set to `NodePort`, mirroring the exposure pattern already used for the NGINX Ingress Controller in earlier modules, and its autoscaling behavior is parametrized through two new variables so it can react to real traffic load:

```hcl
# helm_istio.tf (continued)
  set {
    name  = "service.type"
    value = "NodePort"
  }

  set {
    name  = "autoscaling.minReplicas"
    value = var.istio_min_replicas
  }

  set {
    name  = "autoscaling.targetCPUUtilizationPercentage"
    value = var.istio_cpu_threshold
  }

  depends_on = [
    helm_release.istio_base,
    helm_release.istiod
  ]
}
```

```hcl
# variables.tf
variable "istio_min_replicas" {
  type        = string
  description = "value of min replicas"
  default     = "3"
}

variable "istio_cpu_threshold" {
  type        = string
  description = "value of cpu threshold"
  default     = "60"
}
```

## 3. Binding the Gateway to the Existing Network Load Balancer

Just like the NGINX controller before it, the Ingress Gateway's Service is bound to the same pre-existing `aws_lb_target_group.main` NLB target group with a `TargetGroupBinding`. The Network Load Balancer itself, its listener, and its target group — all created back when the NGINX Ingress Controller was first set up — are untouched; only the Service being bound to port 80 changes:

```hcl
# helm_istio.tf
resource "kubectl_manifest" "target_binding_80" {
  yaml_body = <<YAML
apiVersion: elbv2.k8s.aws/v1beta1
kind: TargetGroupBinding
metadata:
  name: istio-ingress
  namespace: istio-system
spec:
  serviceRef:
    name: istio-ingressgateway
    port: 80
  targetGroupARN: ${aws_lb_target_group.main.arn}
  targetType: instance
YAML
  depends_on = [
    helm_release.istio_ingress
  ]
}
```

```hcl
# nlb.tf
resource "aws_lb" "ingress" {

  name = var.project_name

  internal           = false
  load_balancer_type = "network"

  subnets = data.aws_ssm_parameter.public_subnets[*].value

  enable_cross_zone_load_balancing = true
  enable_deletion_protection       = false

  tags = {
    Name = var.project_name
  }

}

resource "aws_lb_target_group" "main" {
  name     = format("%s-http", var.project_name)
  port     = 8080
  protocol = "TCP"
  vpc_id   = data.aws_ssm_parameter.vpc.value
}

resource "aws_lb_listener" "main" {
  load_balancer_arn = aws_lb.ingress.arn
  port              = 80
  protocol          = "TCP"

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.main.arn
  }
}
```

## Key Takeaways

- The Ingress Gateway is a normal Istio data-plane proxy dedicated to handling external traffic — it's installed and scaled the same way any other Envoy-backed workload would be.
- Reusing the existing NodePort + `TargetGroupBinding` pattern means the AWS-side Network Load Balancer never needs to change when the ingress technology behind it does.
- Gateway autoscaling is parametrized (`istio_min_replicas`, `istio_cpu_threshold`) rather than hardcoded, so capacity can be tuned per environment.

---

# Lesson 4: Deploying a Sidecar-Injected Lab Application

## 1. Enabling Automatic Sidecar Injection

Sidecar injection is opt-in per namespace, controlled by a single label. Every namespace hosting workloads that should participate in the mesh carries `istio-injection: enabled`:

```yaml
# health-api.yml
apiVersion: v1
kind: Namespace
metadata:
  labels:
    istio-injection: enabled
  name: nutrition
```

```yaml
# chip.yml
apiVersion: v1
kind: Namespace
metadata:
  name: chip
  labels:
    istio-injection: enabled
```

With this label in place, any pod created in the namespace automatically gets an Envoy sidecar container injected alongside its application container — no changes to the Deployment manifests themselves are required.

## 2. Exposing health-api with a Gateway and VirtualService

`health-api` is the entry point into a small nutrition-tracking lab application. It gets its own `Gateway`, bound to the shared `istio-ingressgateway` Service via the `istio: ingressgateway` selector, and a `VirtualService` routing traffic for its external host to the `health-api` Service on port 8080:

```yaml
# health-api.yml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: health-api
  namespace: nutrition
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "health.msfidelis.com.br"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: health-api
  namespace: nutrition
spec:
  hosts:
  - "health-api.nutrition.svc.cluster.local"
  - "health.msfidelis.com.br"
  gateways:
  - health-api
  http:
  - route:
    - destination:
        host: health-api
        port:
          number: 8080
```

Its `Deployment` sends spans directly to the Zipkin-compatible collector endpoint that `istiod` was already configured to trace against back in Lesson 2, and depends on six other internal services reachable only inside the mesh:

```yaml
# health-api.yml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: health-api
  name: health-api
  namespace: nutrition
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
        env:
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
        - name: NATS_URI
          value: "nats://nats.nutrition.svc.cluster.local:4222"
        ports:
        - containerPort: 8080
          name: http
        startupProbe:
          failureThreshold: 10
          httpGet:
            path: /healthcheck
            port: 8080
          periodSeconds: 10
        livenessProbe:
          failureThreshold: 10
          httpGet:
            path: /healthcheck
            port: 8080
      terminationGracePeriodSeconds: 60
```

## 3. Internal gRPC Services Without a Gateway

`health-api` fans out to six internal gRPC services (`recommendations-grpc`, `bmr-grpc`, `imc-grpc`, `calories-grpc`, `proteins-grpc`, `water-grpc`). None of them is reachable from outside the mesh, so none of them needs a `Gateway` — each gets only a `VirtualService`, a `Deployment`, and a `ClusterIP` `Service`. `recommendations-grpc` is representative of the pattern, itself fanning out further to `proteins-grpc`, `water-grpc`, and `calories-grpc`:

```yaml
# health-api.yml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: recommendations-grpc
  namespace: nutrition
spec:
  hosts:
  - "recommendations-grpc.nutrition.svc.cluster.local"
  http:
  - route:
    - destination:
        host: recommendations-grpc
        port:
          number: 30000
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: recommendations-grpc
  name: recommendations-grpc
  namespace: nutrition
spec:
  replicas: 2
  selector:
    matchLabels:
      app: recommendations-grpc
  template:
    metadata:
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "30000"
        policy.cilium.io/proxy-visibility: "<Egress/53/UDP/DNS>,<Egress/30000/TCP/HTTP>"
      labels:
        app: recommendations-grpc
        name: recommendations-grpc
        version: v1
    spec:
      containers:
      - image: fidelissauro/recommendations-grpc-service:latest
        name: recommendations-grpc
        env:
        - name: ENVIRONMENT
          value: "dev"
        - name: ZIPKIN_COLLECTOR_ENDPOINT
          value: http://jaeger-collector.tracing.svc.cluster.local:9411/api/v2/spans
        - name: PROTEINS_SERVICE_ENDPOINT
          value: "proteins-grpc.nutrition.svc.cluster.local:30000"
        - name: WATER_SERVICE_ENDPOINT
          value: "water-grpc.nutrition.svc.cluster.local:30000"
        - name: CALORIES_SERVICE_ENDPOINT
          value: "calories-grpc.nutrition.svc.cluster.local:30000"
        ports:
        - containerPort: 30000
          name: http
      terminationGracePeriodSeconds: 60
---
apiVersion: v1
kind: Service
metadata:
  name: recommendations-grpc
  namespace: nutrition
  annotations:
    prometheus.io/scrape: "true"
    prometheus.io/port: "30000"
```

The remaining five gRPC services (`bmr-grpc`, `imc-grpc`, `calories-grpc`, `proteins-grpc`, `water-grpc`) follow this exact same three-object shape.

## 4. Deploying the chip Service

A second, independent lab application, `chip`, is deployed in its own namespace with its own `Gateway` on `chip.msfidelis.com.br`. It's used throughout the rest of this module — including the retry, circuit-breaking, and `ServiceEntry` examples in Lessons 8 and 9 — and carries a `CHAOS_MONKEY_*` set of environment variables used to deliberately inject latency and failures for testing the mesh's resilience features:

```yaml
# chip.yml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: chip
  namespace: chip
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "chip.msfidelis.com.br"
---
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
    metadata:
      labels:
        app: chip
        name: chip
        version: v1
    spec:
      topologySpreadConstraints:
      - maxSkew: 1
        topologyKey: "topology.kubernetes.io/zone"
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: chip

      - maxSkew: 1
        topologyKey: "node.kubernetes.io/instance-type"
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: chip

      - maxSkew: 1
        topologyKey: "karpenter.sh/capacity-type"
        whenUnsatisfiable: ScheduleAnyway
        labelSelector:
          matchLabels:
            app: chip

      containers:
      - name: chip
        image: fidelissauro/chip:v1
        ports:
        - containerPort: 8080
          name: http
        resources:
          requests:
            cpu: 250m
            memory: 512Mi
          limits:
            cpu: 250m
            memory: 512Mi
        startupProbe:
          failureThreshold: 10
          httpGet:
            path: /readiness
            port: 8080
          periodSeconds: 10
        livenessProbe:
          failureThreshold: 10
          httpGet:
            httpHeaders:
            - name: Custom-Header
              value: Awesome
            path: /liveness
            port: 8080
          periodSeconds: 10
        env:
        - name: CHAOS_MONKEY_ENABLED
          value: "false"
        - name: CHAOS_MONKEY_MODE
          value: "critical"
        - name: CHAOS_MONKEY_LATENCY
          value: "true"
        - name: CHAOS_MONKEY_EXCEPTION
          value: "true"
        - name: CHAOS_MONKEY_APP_KILLER
          value: "true"
        - name: CHAOS_MONKEY_MEMORY
          value: "true"
      terminationGracePeriodSeconds: 60
---
apiVersion: v1
kind: Service
metadata:
  name: chip
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

The `topologySpreadConstraints` on `chip` predate this module — they were added back in Module 5 to spread replicas across AZs, instance types, and capacity types — and keep working unmodified once the pod also carries an injected Envoy sidecar.

## Key Takeaways

- Sidecar injection is entirely opt-in and namespace-scoped, controlled by the single `istio-injection: enabled` label — no Deployment manifest needs to change to join the mesh.
- Only services that need to be reachable from outside the mesh get a `Gateway`; purely internal services only need a `VirtualService`, `Deployment`, and `Service`.
- Deploying a multi-service application under the mesh doesn't change anything about how the application itself is written — routing and connectivity are handled by the sidecars, not the app.

---

# Lesson 5: Migrating Grafana and Monitoring the Envoy Sidecars

## 1. Exposing Grafana Through the Istio Gateway

With NGINX retired, Grafana's `Ingress` resource is replaced by an Istio `Gateway` and `VirtualService`, routing `var.grafana_host` to the same `prometheus-grafana` Service installed back in Module 12. The old NGINX-based `Ingress` is kept as a comment for reference rather than deleted:

```hcl
# helm_prometheus.tf
resource "kubectl_manifest" "grafana_gateway" {
  yaml_body = <<YAML
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: grafana
  namespace: prometheus
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "${var.grafana_host}"
YAML
}

resource "kubectl_manifest" "grafana_virtual_service" {
  yaml_body = <<YAML
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: grafana
  namespace: prometheus
spec:
  hosts:
  - "${var.grafana_host}"
  gateways:
  - grafana
  http:
  - route:
    - destination:
        host: prometheus-grafana
        port:
          number: 80
YAML
}

# resource "kubectl_manifest" "grafana_host" {
#   yaml_body = <<YAML
# apiVersion: networking.k8s.io/v1
# kind: Ingress
# metadata:
#   name: grafana-ingress
#   namespace: prometheus
# spec:
#   ingressClassName: nginx
#   rules:
#     - host: ${var.grafana_host}
#       http:
#         paths:
#           - path: /
#             pathType: Prefix
#             backend:
#               service:
#                 name: prometheus-grafana
#                 port:
#                   number: 80
# YAML
#   depends_on = [
#     helm_release.prometheus,
#     helm_release.nginx_controller
#   ]
# }
```

## 2. Scraping Envoy Metrics with a PodMonitor

Every Envoy sidecar exposes its own Prometheus metrics on `/stats/prometheus`. A dedicated `PodMonitor` targets any pod running an `istio-proxy` container, across every namespace, and relabels the scraped series so they carry usable `namespace` and `pod_name` labels in Prometheus:

```yaml
# pod_monitor.tf
resource "kubectl_manifest" "envoy_pod_monitor" {
  yaml_body = <<YAML
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: envoy-stats-monitor
  namespace: istio-system
  labels:
    monitoring: istio-proxies
    release: istio
spec:
  selector:
    matchExpressions:
    - {key: istio-prometheus-ignore, operator: DoesNotExist}
  namespaceSelector:
    any: true
  jobLabel: envoy-stats
  podMetricsEndpoints:
  - path: /stats/prometheus
    interval: 15s
    relabelings:
    - action: keep
      sourceLabels: [__meta_kubernetes_pod_container_name]
      regex: "istio-proxy"
    - action: keep
      sourceLabels: [__meta_kubernetes_pod_annotationpresent_prometheus_io_scrape]
    - action: replace
      regex: (\d+);(([A-Fa-f0-9]{1,4}::?){1,7}[A-Fa-f0-9]{1,4})
      replacement: '[$2]:$1'
      sourceLabels:
      - __meta_kubernetes_pod_annotation_prometheus_io_port
      - __meta_kubernetes_pod_ip
      targetLabel: __address__
    - action: replace
      regex: (\d+);((([0-9]+?)(\.|$)){4})
      replacement: $2:$1
      sourceLabels:
      - __meta_kubernetes_pod_annotation_prometheus_io_port
      - __meta_kubernetes_pod_ip
      targetLabel: __address__
    - action: labeldrop
      regex: "__meta_kubernetes_pod_label_(.+)"
    - sourceLabels: [__meta_kubernetes_namespace]
      action: replace
      targetLabel: namespace
    - sourceLabels: [__meta_kubernetes_pod_name]
      action: replace
      targetLabel: pod_name
YAML

  depends_on = [
    helm_release.prometheus,
    helm_release.istiod
  ]

}
```

With this `PodMonitor` in place, Istio's own community Grafana dashboards (mesh, service, and workload-level) have real data to render from every sidecar in the cluster.

## Key Takeaways

- Migrating an existing Ingress-based route to Istio is a matter of adding a `Gateway` + `VirtualService` pair pointed at the same Service — the backend doesn't change.
- Envoy sidecars are just another Prometheus scrape target; a `PodMonitor` selecting on the `istio-proxy` container name picks them up across every namespace at once.
- Keeping the previous ingress definition as a comment (rather than deleting it) preserves the migration history directly in the Terraform code.

---

# Lesson 6: Distributed Tracing with Jaeger

## 1. Installing Jaeger as an All-in-One Deployment

Jaeger is installed with its `allInOne` mode enabled and every other component (agent, collector, query, and the Cassandra/Kafka/Elasticsearch storage provisioners) explicitly disabled — an intentionally minimal, in-memory setup suited for a lab environment rather than production-grade trace retention:

```hcl
# helm_jaeger.tf
resource "helm_release" "jaeger" {
  name             = "jaeger"
  repository       = "https://jaegertracing.github.io/helm-charts"
  chart            = "jaeger"
  namespace        = "tracing"
  create_namespace = true

  set {
    name  = "allInOne.enabled"
    value = "true"
  }

  set {
    name  = "storage.type"
    value = "memory"
  }

  set {
    name  = "agent.enabled"
    value = "false"
  }

  set {
    name  = "collector.enabled"
    value = "false"
  }

  set {
    name  = "query.enabled"
    value = "false"
  }

  set {
    name  = "provisionDataStore.cassandra"
    value = "false"
  }

  set {
    name  = "provisionDataStore.kafka"
    value = "false"
  }

  set {
    name  = "provisionDataStore.elasticsearch"
    value = "false"
  }

  set {
    name  = "collector.service.zipkin.port"
    value = "9411"
  }

  depends_on = [
    aws_eks_cluster.main,
    helm_release.istiod
  ]
}
```

## 2. Enabling Mesh-Wide Tracing on istiod

The `collector.service.zipkin.port` set above is what makes the `jaeger-collector.tracing.svc.cluster.local:9411` address configured on `istiod` back in Lesson 2 actually resolve to something. From this point on, both application-level spans (like the ones `health-api` sends explicitly via `ZIPKIN_COLLECTOR_ENDPOINT`) and Istio's own mesh-level auto-tracing land in the same Jaeger backend.

## 3. Exposing the Jaeger Query UI

The Jaeger Query UI is exposed through the mesh the same way every other service in this module is — a `Gateway` plus a `VirtualService` routing `var.jaeger_host` to the `jaeger-query` Service on port 16686:

```hcl
# helm_jaeger.tf
resource "kubectl_manifest" "jaeger_gateway" {
  yaml_body = <<YAML
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: jaeger-query
  namespace: tracing
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "${var.jaeger_host}"
YAML

  depends_on = [
    helm_release.jaeger,
    helm_release.istio_ingress
  ]

}

resource "kubectl_manifest" "jaeger_virtual_service" {
  yaml_body = <<YAML
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: jaeger-query
  namespace: tracing
spec:
  hosts:
  - "${var.jaeger_host}"
  gateways:
  - jaeger-query
  http:
  - route:
    - destination:
        host: jaeger-query
        port:
          number: 16686
YAML

  depends_on = [
    helm_release.jaeger,
    helm_release.istio_ingress
  ]

}
```

```hcl
# variables.tf
// Jaeger

variable "jaeger_host" {
  type        = string
  description = "Host do Jaeger"
  default     = "jaeger.msfidelis.com.br"
}
```

## Key Takeaways

- Jaeger's all-in-one, in-memory mode is enough to demonstrate distributed tracing without standing up a dedicated storage backend.
- Istio's Zipkin-compatible tracing integration means both application-instrumented spans and automatic mesh-level spans converge on the same backend.
- Exposing Jaeger's UI reuses the exact same `Gateway`/`VirtualService` pattern as every other host in this module — there's nothing tracing-specific about how it's routed.

---

# Lesson 7: Installing and Configuring Kiali

## 1. Installing the Kiali Server

Kiali is installed via its own Helm chart, pinned to a dedicated version variable, and depends on all three core Istio releases since it needs the mesh, the control plane, and the ingress gateway already running:

```hcl
# variables.tf
// Kiali

variable "kiali_host" {
  type        = string
  description = "Host do Kiali"
  default     = "kiali.msfidelis.com.br"
}

variable "kiali_version" {
  type        = string
  description = "value of kiali version"
  default     = "2.5"
}
```

```hcl
# helm_kiali.tf
resource "helm_release" "kiali-server" {
  name       = "kiali-server"
  chart      = "kiali-server"
  repository = "https://kiali.org/helm-charts"
  namespace  = "istio-system"

  create_namespace = true

  version = var.kiali_version

  set = [
    {
      name  = "server.web_fqdn"
      value = var.kiali_host
    },
    {
      name  = "auth.strategy"
      value = "anonymous"
    },
  ]

  depends_on = [
    helm_release.istio_base,
    helm_release.istiod,
    helm_release.istio_ingress
  ]
}
```

Authentication is set to `anonymous`, appropriate for a lab environment where Kiali sits behind the same network boundary as the rest of the mesh's tooling.

## 2. Integrating Kiali with Jaeger, Prometheus, and Grafana

Kiali's real value comes from tying together the three other observability tools installed earlier in this module. Its `external_services` configuration points at Jaeger for trace data, Prometheus for mesh metrics, and Grafana for dashboard deep-links — including three of Istio's own community dashboards (Mesh, Service, and Workload):

```hcl
# helm_kiali.tf (continued)
  set = [
    # ...
    {
      name  = "external_services.tracing.use_grpc"
      value = "false"
    },
    {
      name  = "external_services.tracing.enabled"
      value = "true"
    },
    {
      name  = "external_services.tracing.internal_url"
      value = "http://jaeger-query.tracing.svc.cluster.local:16686"
    },
    {
      name  = "external_services.tracing.external_url"
      value = format("http://%s", var.jaeger_host)
    },
    {
      name  = "external_services.prometheus.url"
      value = "http://prometheus-kube-prometheus-prometheus.prometheus.svc.cluster.local:9090"
    },
    {
      name  = "external_services.grafana.enabled"
      value = "true"
    },
    {
      name  = "external_services.grafana.external_url"
      value = format("http://%s", var.grafana_host)
    },
    {
      name  = "external_services.grafana.internal_url"
      value = "http://prometheus-grafana.prometheus.svc.cluster.local:80"
    },
    {
      name  = "external_services.grafana.auth.type"
      value = "basic"
    },
    {
      name  = "external_services.grafana.auth.insecure_skip_verify"
      value = "true"
    },
    {
      name  = "external_services.grafana.auth.username"
      value = "admin"
    },
    {
      name  = "external_services.grafana.auth.password"
      value = "linuxtips"
    },
    {
      name  = "external_services.grafana.dashboards[0].name"
      value = "Istio Mesh Dashboard"
    },
    {
      name  = "external_services.grafana.dashboards[1].name"
      value = "Istio Service Dashboard"
    },
    {
      name  = "external_services.grafana.dashboards[1].variables.namespace"
      value = "var-namespace"
    },
    {
      name  = "external_services.grafana.dashboards[1].variables.service"
      value = "var-service"
    },
    {
      name  = "external_services.grafana.dashboards[2].name"
      value = "Istio Workload Dashboard"
    },
    {
      name  = "external_services.grafana.dashboards[2].variables.namespace"
      value = "var-namespace"
    },
    {
      name  = "external_services.grafana.dashboards[2].variables.workload"
      value = "var-workload"
    },
    {
      name  = "external_services.grafana.dashboards[3].name"
      value = "Istio Performance Dashboard"
    },
  ]
```

Note that `set` here is a single list-of-objects attribute rather than repeated `set { }` blocks, matching the Helm provider v3 syntax this file was written against — unlike `helm_istio.tf` and `helm_jaeger.tf` in this same module, which still use the older repeatable-block form.

## 3. Exposing Kiali

Kiali gets the same `Gateway`/`VirtualService` treatment as every other UI in this module, routing `var.kiali_host` to Kiali's own Service on port 20001:

```hcl
# helm_kiali.tf
resource "kubectl_manifest" "kiali_gateway" {
  yaml_body = <<YAML
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: kiali-gateway
  namespace: istio-system
spec:
  selector:
    istio: ingressgateway 
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - ${var.kiali_host}
YAML

  depends_on = [
    helm_release.kiali-server,
  ]

}

resource "kubectl_manifest" "kiali_virtual_service" {
  yaml_body = <<YAML
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: kiali
  namespace: istio-system
spec:
  hosts:
  - ${var.kiali_host}
  gateways:
  - kiali-gateway
  http:
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        host: kiali
        port:
          number: 20001
YAML

  depends_on = [
    helm_release.kiali-server,
  ]

}
```

## Key Takeaways

- Kiali doesn't collect its own data — it's a UI that queries Jaeger, Prometheus, and Grafana, so it depends on all of them being installed and reachable first.
- Wiring in Grafana dashboard links lets Kiali deep-link straight into the Istio Mesh/Service/Workload/Performance dashboards instead of duplicating that visualization itself.
- `helm_kiali.tf` uses the newer `set = [ {...}, ... ]` list-attribute syntax, while the other Helm releases in this module still use the older repeatable `set { }` block form — both are valid, but they reflect different points in the Helm provider's own evolution.

---

# Lesson 8: Resilience with Retries and Circuit Breaking

## 1. Automatic Retries with a VirtualService

Resilience policies in Istio live entirely in `VirtualService` and `DestinationRule` objects — no application code changes are involved. The `chip` service's `VirtualService` adds a `retries` policy on top of its existing routing rule: up to 3 attempts, each with a 500ms per-try timeout, retried only when the upstream responds with a 5xx:

```yaml
# chip.yml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: chip
  namespace: chip
spec:
  hosts:
  - "chip.chip.svc.cluster.local"
  - "chip.msfidelis.com.br"
  gateways:
  - chip
  http:
  - route:
    - destination:
        host: chip
        port:
          number: 8080
    retries:
      attempts: 3
      perTryTimeout: 500ms
      retryOn: 5xx
```

`chip`'s `CHAOS_MONKEY_*` environment variables (introduced in Lesson 4) make it straightforward to trigger the failure conditions these retries are meant to absorb.

## 2. Circuit Breaking with a DestinationRule

A `DestinationRule` on the same host adds a connection pool and outlier detection policy, capping concurrent TCP connections and pending HTTP requests, then ejecting any endpoint that returns 100 consecutive 5xx errors within a 300ms evaluation interval for 60 seconds, up to a maximum of 50% of endpoints ejected at once:

```yaml
# chip.yml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: chip
  namespace: chip
spec:
  host: chip.chip.svc.cluster.local
  trafficPolicy:
    connectionPool:
      tcp:
        maxConnections: 2
      http:
        http1MaxPendingRequests: 10
        maxRequestsPerConnection: 10
    outlierDetection:
      consecutive5xxErrors: 100
      interval: 300ms
      baseEjectionTime: 60s
      maxEjectionPercent: 50
```

## Key Takeaways

- Retries and circuit breaking are declared entirely through `VirtualService`/`DestinationRule` objects — the application never needs its own retry or backoff logic.
- A `retries` policy and an `outlierDetection` policy solve different problems: retries mask transient individual-request failures, while outlier detection removes a consistently failing endpoint from the load-balancing pool altogether.
- `chip`'s built-in chaos-injection environment variables make it a convenient target for exercising both policies under controlled failure conditions.

---

# Lesson 9: Mapping External Services

## 1. The Problem with Out-of-Mesh Traffic

By default, Istio's Envoy sidecars only have registry information about services inside the mesh. A pod's outbound call to a host that lives entirely outside the cluster still passes through its sidecar, but the sidecar treats it as generic, unregistered egress traffic that none of the mesh's routing, retry, or observability features apply to.

## 2. Registering an External Host with a ServiceEntry

A `ServiceEntry` closes this gap by adding an external host to the mesh's internal service registry. Once registered, the same policy and telemetry machinery used for in-mesh traffic — sidecar-level metrics, and any matching `DestinationRule` — applies to calls made to that host as well. `chip`'s `ServiceEntry` registers Google's public domains as an external, DNS-resolved, TLS-only destination:

```yaml
# chip.yml
apiVersion: networking.istio.io/v1
kind: ServiceEntry
metadata:
  name: google.com.br
  namespace: chip
spec:
  hosts:
  - google.com.br
  - www.google.com.br
  - google.com
  location: MESH_EXTERNAL
  ports:
  - number: 443
    name: https
    protocol: TLS
  resolution: DNS
```

`location: MESH_EXTERNAL` tells Istio this host is not part of the mesh itself, and `resolution: DNS` tells the sidecar to resolve it the normal way rather than expecting a static endpoint list.

## Key Takeaways

- Without a `ServiceEntry`, traffic to hosts outside the mesh is invisible to Istio's routing and telemetry layer — it's passed through, not managed.
- `location: MESH_EXTERNAL` is what distinguishes a `ServiceEntry` describing an outside dependency from one describing a workload that actually runs inside the mesh.
- Registering an external service is what makes it possible to apply the same kind of mesh-level policy to third-party dependencies as to internal services.
