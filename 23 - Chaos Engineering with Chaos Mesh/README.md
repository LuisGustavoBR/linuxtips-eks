# Module 23: Chaos Engineering with Chaos Mesh

## Overview

This module is framed by the instructor as a bonus lesson — "aula extra" — recorded outside the main sequence of the course to cover a topic that never found a dedicated slot in the earlier modules: **Chaos Engineering**, the practice of intentionally injecting failures or unusual conditions into a running workload to see how it actually behaves, rather than assuming it will hold up. The tool used throughout is **Chaos Mesh**, a Kubernetes-native chaos engineering platform installed as a suite of CRDs and controllers that let failure scenarios be declared and run against a cluster in a controlled, repeatable way.

Unlike Modules 18-22, which built on the dedicated `linuxtips-eks-multicluster-management` and `linuxtips-eks-observability-cluster` repositories, this module returns to **`linuxtips-eks-vanilla`** — the "final project" codebase used since the beginning of the course — but on a dedicated branch, `lesson/chaos-mesh`, rather than `main`. This repository follows a different ground-truth pattern than the observability-cluster repos: instead of a single flat final-state commit, it carries one commit per lesson topic on its own branch. The `lesson/chaos-mesh` branch's single commit on top of the ArgoCD/ChartMuseum baseline (`add chaos mesh configurations`) gives an exact, authoritative manifest of every file this module adds or changes, which is what this README is grounded in, alongside the module's 9 lesson transcripts.

The lab target throughout is the same multicluster "Health API" gRPC demo used in earlier modules — `health-api`, `recommendations-grpc`, `proteins-grpc`, `water-grpc`, `calories-grpc`, `imc-grpc`, and `bmr-grpc`, all in the `nutrition` namespace — redeployed fresh via ArgoCD `ApplicationSet`s for this branch, with a continuous stream of requests kept running against it throughout so that the effect of each chaos experiment is visible in real time.

> **Note:** the transcript mentions pasting in a script or tool to keep generating traffic against the Health API while experiments run, but the name is unclear in the recording. Since no such script exists in the repository and the tool's identity can't be confirmed, this README describes it generically as "a continuous traffic-generation loop against the Health API" rather than naming a specific tool.

> **Note:** applying a chaos experiment on this EKS cluster fails against a `ValidatingWebhookConfiguration` installed by the Chaos Mesh Helm chart, and must be deleted (`kubectl delete validatingwebhookconfiguration <name>`) before any experiment can be created. This is a real, necessary workaround demonstrated in the transcript, not an optional step — it's called out explicitly in Lesson 3 rather than silently assumed away.

> **Note:** `helm_chaos_mesh.tf`'s `chaos_virtual_service` resource declares `depends_on = [helm_release.argo_rollouts]`, not `helm_release.chaos_mesh`. This looks like a copy-paste artifact from the Argo Rollouts dashboard's own Gateway/VirtualService pair (which this resource is structurally identical to), rather than a deliberate dependency. It's reproduced here exactly as it exists in the real file rather than silently corrected.

> **Note:** this branch's `helm/prometheus/values.yml` has the `karpenter.sh/nodepool: "prometheus"` `nodeSelector` commented out on the Prometheus, Grafana, and Prometheus Operator sections, and disables Grafana's `persistence` and Prometheus's `storageSpec`. This is consistent with there being no dedicated `prometheus` NodePool defined anywhere in this repository's current Terraform — the change is real and quoted as-is in Lesson 2, not a bug introduced by this README.

## Table of Contents

- [Lesson 1: Introduction to Chaos Engineering with Chaos Mesh](#lesson-1-introduction-to-chaos-engineering-with-chaos-mesh)
  - [1. What Is Chaos Engineering](#1-what-is-chaos-engineering)
  - [2. Chaos Mesh's Experiment Types](#2-chaos-meshs-experiment-types)
- [Lesson 2: Installing Chaos Mesh](#lesson-2-installing-chaos-mesh)
  - [1. Helm Installation and the Bottlerocket Containerd Socket](#1-helm-installation-and-the-bottlerocket-containerd-socket)
  - [2. Redeploying the Health API Lab](#2-redeploying-the-health-api-lab)
- [Lesson 3: Pod-Level Chaos Experiments](#lesson-3-pod-level-chaos-experiments)
  - [1. Creating the assaults Folder](#1-creating-the-assaults-folder)
  - [2. Killing a Single Pod with PodChaos](#2-killing-a-single-pod-with-podchaos)
  - [3. The ValidatingWebhookConfiguration Admission Error](#3-the-validatingwebhookconfiguration-admission-error)
  - [4. Killing All Matching Pods](#4-killing-all-matching-pods)
  - [5. Killing a Fixed or Random Percentage of Pods](#5-killing-a-fixed-or-random-percentage-of-pods)
  - [6. Failing Pods Without Killing Them](#6-failing-pods-without-killing-them)
- [Lesson 4: Network Chaos Experiments](#lesson-4-network-chaos-experiments)
  - [1. Injecting Network Delay](#1-injecting-network-delay)
  - [2. Partitioning Network Traffic](#2-partitioning-network-traffic)
  - [3. Limiting Bandwidth](#3-limiting-bandwidth)
- [Lesson 5: Stress Chaos Experiments](#lesson-5-stress-chaos-experiments)
  - [1. Visualizing Resource Usage with a Kubernetes Dashboard](#1-visualizing-resource-usage-with-a-kubernetes-dashboard)
  - [2. Injecting Memory Stress](#2-injecting-memory-stress)
  - [3. Injecting CPU Stress](#3-injecting-cpu-stress)
- [Lesson 6: DNS Chaos Experiments](#lesson-6-dns-chaos-experiments)
  - [1. Forcing DNS Resolution Errors](#1-forcing-dns-resolution-errors)
  - [2. Returning a Random IP Address](#2-returning-a-random-ip-address)
- [Lesson 7: Exposing and Authenticating to the Chaos Mesh Dashboard](#lesson-7-exposing-and-authenticating-to-the-chaos-mesh-dashboard)
  - [1. Gateway and VirtualService for the Dashboard](#1-gateway-and-virtualservice-for-the-dashboard)
  - [2. Generating an RBAC Token with scope.yml](#2-generating-an-rbac-token-with-scopeyml)
- [Lesson 8: Chaos Mesh Workflows](#lesson-8-chaos-mesh-workflows)
  - [1. Serial Workflows](#1-serial-workflows)
  - [2. Parallel Workflows](#2-parallel-workflows)
- [Lesson 9: Scheduling Recurring Chaos Experiments](#lesson-9-scheduling-recurring-chaos-experiments)
  - [1. The Schedule CRD](#1-the-schedule-crd)
  - [2. A Word of Caution](#2-a-word-of-caution)
- [Key Takeaways](#key-takeaways)

---

# Lesson 1: Introduction to Chaos Engineering with Chaos Mesh

## 1. What Is Chaos Engineering

Chaos Engineering is defined here as the ability to intentionally inject errors or unusual conditions into a workload, on purpose, to observe what actually happens — rather than assuming resilience without ever testing it. Tests can be scoped narrowly, targeting a very specific piece of the system, or broadened to let as many random failures as possible happen across the cluster at once.

## 2. Chaos Mesh's Experiment Types

Chaos Mesh is introduced as a suite of chaos tooling installed into a Kubernetes cluster, exposing chaos scenarios as declarative CRDs applied like any other Kubernetes manifest. Its documentation groups experiments by the dimension they target, and this module works through most of them hands-on:

- **Pod-level**: `PodChaos` — `pod-kill` (abruptly deletes matching pods) and `pod-failure` (fails a pod's liveness/readiness without deleting it, forcing a restart loop); Chaos Mesh also supports killing a single container inside a multi-container pod.
- **Network**: `NetworkChaos` — delay, packet loss, packet duplication, packet corruption, and partition (isolating two groups of pods from each other entirely).
- **Stress**: `StressChaos` — intentionally spiking CPU and/or memory usage inside a pod.
- **DNS**: `DNSChaos` — delaying name resolution, forcing resolution errors, or returning an incorrect/random IP address.
- **Time**: clock-skew experiments, useful for seeing what happens when two communicating applications disagree about the current time.
- **JVM**: JVM-specific faults, such as forcing garbage collection or adding artificial memory pressure inside the JVM.
- **Cloud provider faults**: experiments targeting cloud-provider-specific failure modes — called out in the transcript as a comparatively weak, underdeveloped area of Chaos Mesh.

The rest of this module works hands-on through the Pod, Network, Stress, and DNS dimensions, plus Chaos Mesh's Workflow and Schedule features for orchestrating and automating experiments.

---

# Lesson 2: Installing Chaos Mesh

## 1. Helm Installation and the Bottlerocket Containerd Socket

Chaos Mesh is installed via its own Helm chart, version **2.7.2**, into the `linuxtips-eks-vanilla` cluster used since the start of the course:

```hcl
# helm_chaos_mesh.tf
resource "helm_release" "chaos_mesh" {
  name       = "chaos-mesh"
  namespace  = "chaos-mesh"
  chart      = "chaos-mesh"
  repository = "https://charts.chaos-mesh.org"

  version    = "2.7.2"

  create_namespace = true

  set = [
    {
      name  = "chaosDaemon.runtime"
      value = "containerd"
    },
    {
      name  = "chaosDaemon.socketPath"
      value = "/run/containerd/containerd.sock"
    }
  ]

  // ContainerD no Bottlerocket
  depends_on = [
    aws_eks_cluster.main,
    helm_release.karpenter,
  ]
}
```

The two `set` values are the one customization needed beyond a default install: because this cluster's nodes run **Bottlerocket**, the container runtime socket doesn't sit at the path Chaos Mesh's `chaosDaemon` DaemonSet expects by default. Setting `chaosDaemon.runtime` to `containerd` and `chaosDaemon.socketPath` to `/run/containerd/containerd.sock` points the daemon at Bottlerocket's actual runtime socket, which is what lets Chaos Mesh's daemon reach into each node's container runtime to inject faults. After the chart installs, the cluster gains a dashboard Deployment, a DNS server used for DNS-chaos experiments, a controller manager, and the `chaosDaemon` DaemonSet that does the actual fault injection on every node.

## 2. Redeploying the Health API Lab

The lab used for every experiment in this module is the same multi-service Health API demo built up across earlier modules in this course, redeployed fresh on this branch via ArgoCD `ApplicationSet`s — one per component (`nutrition-proteins`, `nutrition-water`, `nutrition-calories`, `nutrition-imc`, `nutrition-bmr`, `nutrition-recommendations`, `nutrition-health-api`), each sourced from the same ChartMuseum-hosted `linuxtips` Helm chart used throughout the course, deployed into the `nutrition` namespace. With the ApplicationSets applied and every component up, a continuous traffic-generation loop is kept running against the Health API's endpoint so that the effect of each chaos experiment that follows is visible in real time in the request logs.

> **Note:** the transcript flags that applying chaos experiments on EKS can trip up against admission webhooks Chaos Mesh installs — the specific fix is demonstrated in Lesson 3.

---

# Lesson 3: Pod-Level Chaos Experiments

## 1. Creating the assaults Folder

A local folder, `files/assaults`, is created to hold every chaos experiment manifest built across the rest of this module, numbered in the order they're introduced.

## 2. Killing a Single Pod with PodChaos

The first and simplest experiment kills a single pod matching a label selector, using `PodChaos` in `mode: one`:

```yaml
# files/assaults/1-pod-kill.yml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  generateName: kill-recommendations-grpc-
  namespace: nutrition
spec:
  action: pod-kill
  mode: one
  selector:
    labelSelectors:
      app: recommendations-grpc
  duration: "30s"
```

`action: pod-kill` deletes a matching pod outright, and `mode: one` picks exactly one pod out of everything matched by `selector.labelSelectors`. Applied against `recommendations-grpc`, nothing user-visible happens beyond the one killed pod — the application's own retry logic absorbs the deletion, and the pod is replaced by its Deployment/ReplicaSet as usual. Because the manifest uses `generateName` rather than `name`, it can be applied (`kubectl apply -f`) repeatedly to kill additional pods on demand, or `kubectl create -f`'d multiple times in a row to escalate the damage.

## 3. The ValidatingWebhookConfiguration Admission Error

The first attempt to apply a `PodChaos` manifest on this EKS cluster fails with an admission webhook error. The fix is to find and delete the `ValidatingWebhookConfiguration` that Chaos Mesh's Helm chart installs for validating its own CRDs:

```bash
kubectl get validatingwebhookconfigurations
kubectl delete validatingwebhookconfiguration <chaos-mesh-validation-webhook-name>
```

> **Note:** this is a real, necessary step demonstrated in the transcript to get any Chaos Mesh experiment working on this EKS cluster — not an optional cleanup step. The exact webhook name shown in the recording isn't legible enough to transcribe with confidence, so the command above is shown with a placeholder rather than a guessed name; running `kubectl get validatingwebhookconfigurations` first shows the real name to delete.

Once the webhook is deleted, the same `PodChaos` manifest from §2 applies successfully.

## 4. Killing All Matching Pods

Scaling from "kill one" to "kill everything the selector matches" only requires changing `mode`:

```yaml
# files/assaults/2-pod-kill-all.yml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  generateName: kill-recommendations-grpc-
  namespace: nutrition
spec:
  action: pod-kill
  mode: all
  selector:
    labelSelectors:
      app: recommendations-grpc
  duration: "30s"
```

`mode: all` kills every pod matching the label selector at once, rather than a single one. Applied against `recommendations-grpc` (running with 2 replicas at this point), both replicas are deleted simultaneously; the workload recovers once both are rescheduled.

## 5. Killing a Fixed or Random Percentage of Pods

Between "kill one" and "kill all," Chaos Mesh supports killing a fixed percentage of matching pods:

```yaml
# files/assaults/3-pod-kill-percent.yml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  generateName: kill-recommendations-grpc-
  namespace: nutrition
spec:
  action: pod-kill
  mode: fixed-percent
  value: "50"
  selector:
    labelSelectors:
      app: recommendations-grpc
  duration: "30s"
```

To make the percentage meaningful, `recommendations-grpc` is first scaled up to 10 replicas (`kubectl scale` / adjusting the rollout). With `mode: fixed-percent` and `value: "50"`, applying this manifest against 10 replicas kills exactly 5 of them every time it's applied.

A looser variant randomizes the percentage under a ceiling, using `mode: random-max-percent`:

```yaml
# files/assaults/4-pod-kill-percent-random.yml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  generateName: kill-recommendations-grpc-
  namespace: nutrition
spec:
  action: pod-kill
  mode: random-max-percent
  value: "90"
  selector:
    labelSelectors:
      app: recommendations-grpc
  duration: "30s"
```

Here, `value: "90"` sets a ceiling rather than a fixed amount: each time the experiment is applied, Chaos Mesh kills a random percentage of matching pods up to, but never exceeding, 90%. Applying it repeatedly against the same 10 replicas produced different kill counts each time in the demo (60%, then 10%, then 20%) — random, but always bounded by the configured maximum.

## 6. Failing Pods Without Killing Them

`pod-failure` is a different action from `pod-kill`: instead of deleting the pod, it fails the pod's liveness/readiness probe (or injects a non-zero exit code), so the pod restarts continuously without ever being replaced or rescheduled onto a new pod:

```yaml
# files/assaults/5-pod-failure.yml
apiVersion: chaos-mesh.org/v1alpha1
kind: PodChaos
metadata:
  generateName: failure-health-api-
  namespace: nutrition
spec:
  action: pod-failure
  mode: fixed-percent
  value: "50"
  selector:
    labelSelectors:
      app: health-api
  duration: "60s"
```

Applied against `health-api` at `fixed-percent`/`50`, half of the matching pods spend the full 60-second `duration` stuck in a restart loop, visibly slowing down request handling while the experiment runs, then recovering once it ends. `pod-failure` supports the same `mode` values as `pod-kill` (`one`, `all`, `fixed-percent`, `random-max-percent`) — the only difference between the two actions is failing the pod's health checks in place versus deleting it outright.

---

# Lesson 4: Network Chaos Experiments

## 1. Injecting Network Delay

`NetworkChaos` with `action: delay` injects latency into a pod's outbound connections without dropping any requests:

```yaml
# files/assaults/6-network-delay.yml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  generateName: nutrition-network-delay-
  namespace: nutrition
spec:
  action: delay
  mode: all
  selector:
    namespaces:
      - nutrition
    labelSelectors:
      'app': 'health-api'
  delay:
    latency: '100ms'
    correlation: '100'
    jitter: '500ms'
  duration: "30s"
```

`delay.latency` sets the base latency injected into every affected connection; `delay.jitter` adds variability around that base value rather than a fixed, constant delay; `delay.correlation` controls how correlated consecutive injected delays are to each other (100% here means each successive delay stays close to the previous one rather than being fully random). Applied against `health-api`, requests keep succeeding but visibly slow down for the 30-second duration — a way to test whether timeouts elsewhere in the system are configured generously enough to tolerate a slower downstream dependency.

## 2. Partitioning Network Traffic

`action: partition` is a more severe network experiment: it isolates one group of pods from another entirely, as if the network between them had been physically split:

```yaml
# files/assaults/7-network-partition.yml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  generateName: proteins-partition-
  namespace: nutrition
spec:
  action: partition
  mode: fixed-percent
  value: "50"
  selector:
    namespaces:
      - nutrition
    labelSelectors:
      'app': 'recommendations-grpc'
  direction: to
  target:
    mode: all
    selector:
      namespaces:
        - nutrition
      labelSelectors:
        'app': 'proteins-grpc'
  duration: "30s"
```

`selector` picks the source pod group (50% of `recommendations-grpc`), `target` picks the pod group it's being partitioned from (`proteins-grpc`), and `direction: to` scopes the partition to traffic flowing from the source toward the target. During the experiment, calls from the affected `recommendations-grpc` pods to `proteins-grpc` don't time out gradually — they fail outright, since the traffic is blocked rather than merely delayed. The lesson notes this kind of test is especially useful for validating partition-tolerance claims on systems explicitly designed for it, such as a Cassandra cluster split into two groups of nodes.

## 3. Limiting Bandwidth

`action: bandwidth` throttles the rate, burst limit, and buffer size available to matching pods' connections:

```yaml
# files/assaults/8-netwok-bandwidth.yml
apiVersion: chaos-mesh.org/v1alpha1
kind: NetworkChaos
metadata:
  generateName: proteins-bandwidth-
  namespace: nutrition
spec:
  action: bandwidth
  mode: fixed-percent
  value: "50"
  selector:
    namespaces:
      - nutrition
    labelSelectors:
      'app': 'recommendations-grpc'
  bandwidth:
    rate: '1mbps'
    limit: 20971520
    buffer: 10000
  duration: "30s"
```

> **Note:** this filename is `8-netwok-bandwidth.yml` (missing the "r" in "network") in the real repository — reproduced here exactly rather than silently corrected.

`bandwidth.rate` caps throughput to 1 Mbps, with `limit` (bytes) and `buffer` (packets) tuning how much can queue up before packets are dropped. The lesson is candid that this experiment is hard to demonstrate meaningfully in this lab: bandwidth throttling only shows a visible effect against traffic that actually moves large payloads (large file transfers, bulk uploads to S3, and similar), which this gRPC-based health-check lab doesn't generate — the manifest is real and functional, but its effect isn't something this lab can showcase convincingly.

---

# Lesson 5: Stress Chaos Experiments

## 1. Visualizing Resource Usage with a Kubernetes Dashboard

Before running stress tests, a community Kubernetes dashboard is imported into Grafana (via its dashboard ID, against the Prometheus data source) to visualize per-pod CPU and memory usage in real time, filtered down to the `nutrition` namespace's `imc-grpc` service — giving a live view to watch while each `StressChaos` experiment runs.

## 2. Injecting Memory Stress

`StressChaos` intentionally spikes CPU or memory usage inside matching pods:

```yaml
# files/assaults/9-stress-memory.yml
apiVersion: chaos-mesh.org/v1alpha1
kind: StressChaos
metadata:
  generateName: imc-stress-memory-
  namespace: nutrition
spec:
  mode: all
  selector:
    labelSelectors:
      'app': 'imc-grpc'
    namespaces:
      - nutrition
  stressors:
    memory:
      workers: 4
      size: '512MB'
  duration: "1m"
```

`stressors.memory.workers` sets how many parallel stress workers allocate memory, and `size` sets how much each targets. The lesson runs this progressively: with `imc-grpc` pods normally sitting around 12-13 MB of usage, a first pass at `size: '256MB'` produced a much larger real effect than expected — a cascading spike that pushed each pod's usage to around 512 MB, attributed to the 4 parallel workers compounding the requested size rather than dividing it. Re-running with `size: '512MB'` (as quoted above) pushed usage to nearly 1 GB per pod. In both cases the application kept serving requests without a visible performance impact, demonstrating that the workload tolerates a 2-minute memory spike of this size without degrading — the value of the test either way is observing the actual ceiling before something breaks, not assuming one.

## 3. Injecting CPU Stress

The CPU variant follows the identical shape, using `stressors.cpu` instead of `stressors.memory`:

```yaml
# files/assaults/10-stress-cpu.yml
apiVersion: chaos-mesh.org/v1alpha1
kind: StressChaos
metadata:
  generateName: imc-stress-cpu-
  namespace: nutrition
spec:
  mode: all
  selector:
    labelSelectors:
      'app': 'imc-grpc'
  stressors:
    cpu:
      workers: 4
      load: 80
  duration: "60s"
```

`stressors.cpu.load: 80` targets 80% CPU load across `workers: 4` parallel stress processes. With `imc-grpc`'s pods configured with half a CPU as their limit, the experiment pushed usage up to that 512m ceiling for a short window before settling back down — a small, contained spike that didn't impact the application's ability to serve requests during the 60-second test. As with the memory experiments, the takeaway is that this workload tolerates short CPU spikes at this configured limit without visible degradation.

---

# Lesson 6: DNS Chaos Experiments

## 1. Forcing DNS Resolution Errors

`DNSChaos` with `action: error` forces name resolution to fail for any hostname matching a configured pattern:

```yaml
# files/assaults/11-dns-error.yml
apiVersion: chaos-mesh.org/v1alpha1
kind: DNSChaos
metadata:
  generateName: dns-chaos-
  namespace: nutrition
spec:
  action: error
  mode: all
  patterns:
    - water-grpc.nutrition.svc.cluster.local
  selector:
    namespaces:
      - nutrition
  duration: "30s"
```

`patterns` scopes the failure to only the matching hostname — here, `water-grpc`'s internal Service DNS name. With this applied, `recommendations-grpc` (which calls `water-grpc`) starts failing immediately with a DNS lookup error rather than a timeout, since the name simply can't be resolved at all, and its retries surface that error directly in the logs until the experiment ends and the workload recovers.

## 2. Returning a Random IP Address

A softer variant, `action: random`, resolves the hostname successfully but to an incorrect, effectively random IP address, rather than failing resolution outright:

```yaml
# files/assaults/12-dns-random-ip.yml
apiVersion: chaos-mesh.org/v1alpha1
kind: DNSChaos
metadata:
  generateName: dns-chaos-
  namespace: nutrition
spec:
  action: random
  mode: all
  patterns:
    - water-grpc.nutrition.svc.cluster.local
  selector:
    namespaces:
      - nutrition
  duration: "30s"
```

Unlike the DNS-error case, this failure surfaces as a **connection timeout** rather than an immediate resolution error, since the client believes it has a valid address and tries — unsuccessfully — to connect to it, retrying until the experiment ends.

---

# Lesson 7: Exposing and Authenticating to the Chaos Mesh Dashboard

## 1. Gateway and VirtualService for the Dashboard

Chaos Mesh ships its own dashboard, giving a visual, browser-based alternative to applying experiments as raw YAML. It's exposed through the same Istio ingress pattern used for every other tool in this course, via a new Terraform host variable:

```hcl
# variables.tf
// Chaos Mesh
variable "chaos_mesh_host" {
  default = "chaos-mesh.msfidelis.com.br"
}
```

```hcl
# helm_chaos_mesh.tf
resource "kubectl_manifest" "chaos_gateway" {
  yaml_body = <<YAML
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: chaos-mesh-gateway
  namespace: chaos-mesh
spec:
  selector:
    istio: ingressgateway 
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - ${var.chaos_mesh_host}
YAML

  depends_on = [
    helm_release.chaos_mesh
  ]
}

resource "kubectl_manifest" "chaos_virtual_service" {
  yaml_body = <<YAML
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: chaos-mesh
  namespace: chaos-mesh
spec:
  hosts:
  - ${var.chaos_mesh_host}
  gateways:
  - chaos-mesh-gateway
  http:
  - match:
    - uri:
        prefix: /
    route:
    - destination:
        host: chaos-dashboard
        port:
          number: 2333
YAML

  depends_on = [
    helm_release.argo_rollouts
  ]
}
```

The `VirtualService` routes all traffic for `chaos-mesh.msfidelis.com.br` to the `chaos-dashboard` Service on port `2333`, the dashboard's own port. The lesson deliberately doesn't leave this exposed permanently: the dashboard is only stood up while it's actually being used, then torn back down, since giving broad, standing access to a tool that can trigger cluster-wide failures is treated as a real operational risk.

## 2. Generating an RBAC Token with scope.yml

Opening the dashboard for the first time requires an RBAC token, since the dashboard has no access on its own by default. A dedicated `ServiceAccount`/`ClusterRole`/`ClusterRoleBinding` set grants it cluster-wide permission over pods, namespaces, and every Chaos Mesh CRD:

```yaml
# scope.yml
kind: ServiceAccount
apiVersion: v1
metadata:
  namespace: default
  name: account-cluster-manager-hlfcn

---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: role-cluster-manager-hlfcn
rules:
- apiGroups: [""]
  resources: ["pods", "namespaces"]
  verbs: ["get", "watch", "list"]
- apiGroups: ["chaos-mesh.org"]
  resources: [ "*" ]
  verbs: ["get", "list", "watch", "create", "delete", "patch", "update"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: bind-cluster-manager-hlfcn
subjects:
- kind: ServiceAccount
  name: account-cluster-manager-hlfcn
  namespace: default
roleRef:
  kind: ClusterRole
  name: role-cluster-manager-hlfcn
  apiGroup: rbac.authorization.k8s.io
```

The `chaos-mesh.org` API group is granted full CRUD (`get`, `list`, `watch`, `create`, `delete`, `patch`, `update`) over every Chaos Mesh resource type, which is what lets the dashboard both display and create experiments. After applying `scope.yml`, a token is minted for the `account-cluster-manager-hlfcn` ServiceAccount with `kubectl create token account-cluster-manager-hlfcn`, selecting **cluster scoped** access and the `manager` role when prompted in the dashboard's login screen, and pasting in the resulting JWT. Once authenticated, the dashboard shows every experiment that's been run (including ones already completed), their event timelines, and visual summaries of each run.

---

# Lesson 8: Chaos Mesh Workflows

## 1. Serial Workflows

A `Workflow` lets multiple chaos experiments run in a defined order or in parallel, instead of being applied one at a time by hand. A `templateType: Serial` workflow runs its children one after another:

```yaml
# files/assaults/13-workflows.yml
apiVersion: chaos-mesh.org/v1alpha1
kind: Workflow
metadata:
  generateName: health-api-failure-workflow-
  namespace: nutrition
spec:
  entry: serial-failure
  templates:
    - name: serial-failure
      templateType: Serial

      children:
        - failure-proteins
        - failure-water
        - failure-api

    - name: failure-proteins
      templateType: PodChaos
      deadline: 30s
      podChaos:
        action: pod-failure
        mode: fixed-percent
        value: "50"
        selector:
          labelSelectors:
            app: proteins-grpc

    - name: failure-water
      templateType: PodChaos
      deadline: 30s
      podChaos:
        action: pod-failure
        mode: fixed-percent
        value: "50"
        selector:
          labelSelectors:
            app: water-grpc

    - name: failure-api
      templateType: PodChaos
      deadline: 30s
      podChaos:
        action: pod-failure
        mode: fixed-percent
        value: "50"
        selector:
          labelSelectors:
            app: health-api
```

`spec.entry` names which template kicks the workflow off; that template's `children` list — `failure-proteins`, then `failure-water`, then `failure-api` — is what actually gets executed, in that exact order, since its `templateType` is `Serial`. Each child is itself a full `PodChaos` template embedded inline (via `podChaos:`), each with its own `deadline` bounding how long it runs before the workflow moves to the next step. Watching the dashboard while this runs shows each stage's events in turn: `failure-proteins` starts and completes, then `failure-water` starts, and only once that finishes does `failure-api` begin — the whole run reported as a single successful `Workflow` once all three stages complete.

## 2. Parallel Workflows

Changing `templateType` to `Parallel` runs every child at the same time instead of in sequence:

```yaml
# files/assaults/14-workflows-parallel.yml
apiVersion: chaos-mesh.org/v1alpha1
kind: Workflow
metadata:
  generateName: health-api-stress-workflow-
  namespace: nutrition
spec:
  entry: serial-stress
  templates:
    - name: serial-stress
      templateType: Parallel

      children:
        - stress-proteins
        - stress-water
        - stress-api
        - stress-recommendations
        - stress-imc
        - stress-bmr

    - name: stress-proteins
      templateType: StressChaos
      deadline: 30s
      stressChaos:
        mode: all
        stressors:
          cpu:
            workers: 4
            load: 80
          memory:
            workers: 4
            size: '512MB'
        selector:
          labelSelectors:
            app: proteins-grpc

    # ...stress-water, stress-recommendations, stress-imc, stress-bmr, and
    # stress-api repeat the same StressChaos shape, each targeting its own
    # service via `selector.labelSelectors.app`.
```

Despite the entry template being named `serial-stress`, its `templateType` is `Parallel` — the name is a leftover from copying the previous workflow's structure, not a functional label. Every child here is a `StressChaos` template, each injecting the same combined CPU (80% load, 4 workers) and memory (512MB, 4 workers) stress into a different service — `proteins-grpc`, `water-grpc`, `health-api`, `recommendations-grpc`, `imc-grpc`, and `bmr-grpc` — all six starting simultaneously rather than one after another. Applying it visibly slows down the whole Health API demo's response times at once, since every downstream dependency is under stress at the same time instead of just one in isolation.

---

# Lesson 9: Scheduling Recurring Chaos Experiments

## 1. The Schedule CRD

A `Schedule` wraps any existing chaos experiment type in a cron expression, letting Chaos Mesh trigger it automatically and repeatedly instead of being applied by hand each time:

```yaml
# files/assaults/15-schedule.yml
apiVersion: chaos-mesh.org/v1alpha1
kind: Schedule
metadata:
  name: kill-recommendations-grpc-schedule
  namespace: chaos-mesh
spec:
  schedule: '* * * * *'
  historyLimit: 20
  concurrencyPolicy: 'Allow'
  type: PodChaos
  podChaos:
    action: pod-kill
    mode: one
    selector:
      labelSelectors:
        app: recommendations-grpc
      namespaces:
        - nutrition
    duration: "30s"
```

`spec.schedule` is a standard crontab expression — `* * * * *` here means "every minute." `spec.type` names which chaos kind is being scheduled (`PodChaos` in this case), and the matching `podChaos:` block embeds that experiment's spec exactly as it would look applied standalone — here, the same single-pod-kill experiment from Lesson 3 §2, now killing one `recommendations-grpc` pod automatically every minute. `historyLimit` caps how many past runs are kept, and `concurrencyPolicy: 'Allow'` permits overlapping runs rather than skipping or queuing them. This same `Schedule` shape extends to every experiment type covered in this module — `NetworkChaos`, `StressChaos`, `DNSChaos`, and Workflows can all be wrapped in a cron schedule the same way.

## 2. A Word of Caution

The lesson closes on a deliberate caveat: scheduling chaos experiments to run continuously, as demonstrated here, is not something to reach for casually in production. It's presented as a last resort, appropriate only for systems whose resilience has already been validated to the point where continuous chaos testing is a meaningful signal rather than a self-inflicted outage — not a default practice to adopt straight from this lab.

---

## Key Takeaways

- This module returns to `linuxtips-eks-vanilla`, but on a dedicated `lesson/chaos-mesh` branch rather than `main` — this repository's ground truth comes from one commit per lesson topic on its own branch, unlike the flat, single-final-state observability-cluster repositories used in Modules 18-22.
- Chaos Mesh 2.7.2 is installed via Helm with one required customization for this cluster's **Bottlerocket** nodes: pointing `chaosDaemon.runtime`/`chaosDaemon.socketPath` at containerd's actual socket location.
- A `ValidatingWebhookConfiguration` installed by the Chaos Mesh chart must be deleted before any experiment can be applied on this EKS cluster — a real, necessary workaround, not an optional step.
- `PodChaos` offers two distinct failure modes: `pod-kill` (deletes the pod outright) and `pod-failure` (fails its health checks in place, forcing a restart loop without ever deleting it) — both support the same `mode` values (`one`, `all`, `fixed-percent`, `random-max-percent`).
- `NetworkChaos` covers three tested scenarios in this lab: `delay` (variable added latency), `partition` (a hard, one-directional network split between two pod groups), and `bandwidth` (throttled throughput) — the last of which the lesson admits is hard to demonstrate meaningfully without a workload that moves large payloads.
- `StressChaos` intentionally spikes CPU or memory inside matching pods; the lesson's own memory test showed a real, larger-than-expected cascading effect from stacking multiple parallel workers, underscoring the value of testing an actual ceiling rather than assuming one.
- `DNSChaos` supports two distinct failure shapes for the same target hostname: `error` (an immediate resolution failure) and `random` (a successful-looking resolution to a bad IP, which manifests as a connection timeout instead).
- The Chaos Mesh dashboard is exposed only on demand through the same Istio Gateway/VirtualService pattern used elsewhere in this course, gated behind an RBAC token minted from a dedicated `ServiceAccount`/`ClusterRole`/`ClusterRoleBinding` — deliberately not left permanently accessible.
- `Workflow` orchestrates multiple experiments either in sequence (`Serial`) or all at once (`Parallel`); `Schedule` wraps any single experiment type in a cron expression to trigger it automatically and repeatedly — with an explicit warning against treating continuous scheduled chaos as a default production practice.
