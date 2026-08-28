# Module 7: AWS Load Balancer Controller

## Overview

In this module we install the **AWS Load Balancer Controller**, the component responsible for turning Kubernetes objects into real AWS load balancers.

It gives us two ways to expose a service:

```txt
Service type LoadBalancer -> Network Load Balancer (Layer 4, TCP/UDP)
Ingress                   -> Application Load Balancer (Layer 7, HTTP/HTTPS)
```

Both are configured almost entirely through annotations: which subnets to use, how to health-check the targets, what scheme to expose (internet-facing or internal), and so on. We will not adopt the controller's own Ingress support as this course's main way of routing HTTP traffic — that role goes to a dedicated Ingress Controller (NGINX) in the next module — but one of its CRDs, **`TargetGroupBinding`**, stays with us for the rest of the course. It lets Kubernetes attach a Service to an AWS-provisioned target group without Kubernetes ever owning the load balancer itself, which is the pattern the next module builds on to pair an NLB (Layer 4, Terraform-managed) with the NGINX Ingress Controller (Layer 7, Kubernetes-managed).

This module reuses the groupless cluster built in [Module 6](../06%20-%20Karpenter%20Groupless%20Architecture/README.md) as its starting point, but everything here applies equally on top of any earlier version of the cluster (vanilla, Fargate, or Karpenter with node groups).

## Table of Contents

- [Lesson 1: Installing the AWS Load Balancer Controller](#lesson-1-installing-the-aws-load-balancer-controller)
  - [1. What the Controller Provides](#1-what-the-controller-provides)
  - [2. Creating the IAM Role and Policy](#2-creating-the-iam-role-and-policy)
  - [3. Installing the Controller with Helm](#3-installing-the-controller-with-helm)
  - [4. Verifying the Installation](#4-verifying-the-installation)
  - [5. What We Will Build Next](#5-what-we-will-build-next)
  - [Key Takeaways](#key-takeaways)
- [Lesson 2: Exposing a Service with a Network Load Balancer](#lesson-2-exposing-a-service-with-a-network-load-balancer)
  - [1. Annotation-Driven Load Balancing](#1-annotation-driven-load-balancing)
  - [2. The NLB Annotation Set](#2-the-nlb-annotation-set)
  - [3. Choosing Explicit Subnets Over Subnet Tagging](#3-choosing-explicit-subnets-over-subnet-tagging)
  - [4. Deploying the Service](#4-deploying-the-service)
  - [5. Enabling Cross-Zone Load Balancing](#5-enabling-cross-zone-load-balancing)
  - [Key Takeaways](#key-takeaways-1)
- [Lesson 3: Routing with an Application Load Balancer and Ingress](#lesson-3-routing-with-an-application-load-balancer-and-ingress)
  - [1. From Service to Ingress](#1-from-service-to-ingress)
  - [2. The ALB Ingress Annotation Set](#2-the-alb-ingress-annotation-set)
  - [3. Host-Based Routing](#3-host-based-routing)
  - [4. Testing with the Host Header](#4-testing-with-the-host-header)
  - [Key Takeaways](#key-takeaways-2)
- [Lesson 4: Securing Ingress Traffic with ACM and HTTPS](#lesson-4-securing-ingress-traffic-with-acm-and-https)
  - [1. Provisioning a DNS-Validated ACM Certificate](#1-provisioning-a-dns-validated-acm-certificate)
  - [2. Adding HTTPS to the Ingress](#2-adding-https-to-the-ingress)
  - [3. Testing the Redirect and the TLS Handshake](#3-testing-the-redirect-and-the-tls-handshake)
  - [4. Pointing DNS at the Load Balancer](#4-pointing-dns-at-the-load-balancer)
  - [Key Takeaways](#key-takeaways-3)
- [Lesson 5: Decoupling Load Balancer Lifecycle with Target Group Binding](#lesson-5-decoupling-load-balancer-lifecycle-with-target-group-binding)
  - [1. The Problem With Kubernetes-Managed Load Balancers](#1-the-problem-with-kubernetes-managed-load-balancers)
  - [2. The Target Group Binding Model](#2-the-target-group-binding-model)
  - [3. Provisioning the NLB, Target Group and Listener with Terraform](#3-provisioning-the-nlb-target-group-and-listener-with-terraform)
  - [4. Exposing the Service as NodePort and Binding It](#4-exposing-the-service-as-nodeport-and-binding-it)
  - [5. Choosing Between targetType ip and instance](#5-choosing-between-targettype-ip-and-instance)
  - [6. What Comes Next: The NGINX Ingress Controller](#6-what-comes-next-the-nginx-ingress-controller)
  - [Key Takeaways](#key-takeaways-4)

# Lesson 1: Installing the AWS Load Balancer Controller

## 1. What the Controller Provides

The AWS Load Balancer Controller watches Kubernetes objects and provisions real AWS load balancers on their behalf:

```txt
Service (type: LoadBalancer) -> Network Load Balancer
Ingress (ingressClassName: alb) -> Application Load Balancer
```

Both are driven almost entirely by annotations — the controller doesn't need a CRD to know what kind of health check to run or which subnets to use, it reads that straight off the object it's watching.

## 2. Creating the IAM Role and Policy

Just like Karpenter, the controller runs under its own IRSA role: an IAM role that only the `aws-load-balancer-controller` service account in `kube-system` is allowed to assume, via the cluster's OIDC provider.

```hcl
data "aws_iam_policy_document" "aws_lb_role" {
  statement {
    actions = ["sts:AssumeRoleWithWebIdentity"]
    effect  = "Allow"

    condition {
      test     = "StringEquals"
      variable = "${replace(aws_iam_openid_connect_provider.eks.url, "https://", "")}:sub"
      values   = ["system:serviceaccount:kube-system:aws-load-balancer-controller"]
    }

    principals {
      identifiers = [aws_iam_openid_connect_provider.eks.arn]
      type        = "Federated"
    }
  }
}

resource "aws_iam_role" "aws_lb_controller" {
  assume_role_policy = data.aws_iam_policy_document.aws_lb_role.json
  name               = format("%s-aws-load-balancer", var.project_name)
}
```

The policy attached to that role is the official one published by the project — it covers everything the controller needs to create and manage ELBv2 resources, target groups, security groups, ACM certificates, and WAF associations:

```hcl
data "aws_iam_policy_document" "aws_lb_policy" {
  version = "2012-10-17"

  statement {
    effect = "Allow"
    actions = [
      "acm:DescribeCertificate",
      "acm:ListCertificates",
      "acm:GetCertificate",
      "ec2:AuthorizeSecurityGroupIngress",
      "ec2:CreateSecurityGroup",
      "ec2:CreateTags",
      "ec2:DeleteSecurityGroup",
      "ec2:DeleteTags",
      "ec2:DescribeAccountAttributes",
      "ec2:DescribeAddresses",
      "ec2:DescribeAvailabilityZones",
      "ec2:DescribeInstances",
      "ec2:DescribeInstanceStatus",
      "ec2:DescribeInternetGateways",
      "ec2:DescribeNetworkInterfaces",
      "ec2:DescribeSecurityGroups",
      "ec2:DescribeSubnets",
      "ec2:DescribeTags",
      "ec2:DescribeVpcs",
      "ec2:ModifyInstanceAttribute",
      "ec2:ModifyNetworkInterfaceAttribute",
      "ec2:RevokeSecurityGroupIngress",
      "elasticloadbalancing:AddListenerCertificates",
      "elasticloadbalancing:AddTags",
      "elasticloadbalancing:CreateListener",
      "elasticloadbalancing:CreateLoadBalancer",
      "elasticloadbalancing:CreateRule",
      "elasticloadbalancing:CreateTargetGroup",
      "elasticloadbalancing:DeleteListener",
      "elasticloadbalancing:DeleteLoadBalancer",
      "elasticloadbalancing:DeleteRule",
      "elasticloadbalancing:DeleteTargetGroup",
      "elasticloadbalancing:DeregisterTargets",
      "elasticloadbalancing:DescribeListeners",
      "elasticloadbalancing:DescribeLoadBalancers",
      "elasticloadbalancing:DescribeLoadBalancerAttributes",
      "elasticloadbalancing:DescribeRules",
      "elasticloadbalancing:DescribeSSLPolicies",
      "elasticloadbalancing:DescribeTags",
      "elasticloadbalancing:DescribeTargetGroups",
      "elasticloadbalancing:DescribeTargetGroupAttributes",
      "elasticloadbalancing:DescribeTargetHealth",
      "elasticloadbalancing:ModifyListener",
      "elasticloadbalancing:ModifyLoadBalancerAttributes",
      "elasticloadbalancing:ModifyRule",
      "elasticloadbalancing:ModifyTargetGroup",
      "elasticloadbalancing:ModifyTargetGroupAttributes",
      "elasticloadbalancing:RegisterTargets",
      "elasticloadbalancing:RemoveListenerCertificates",
      "elasticloadbalancing:RemoveTags",
      "elasticloadbalancing:SetIpAddressType",
      "elasticloadbalancing:SetSecurityGroups",
      "elasticloadbalancing:SetSubnets",
      "elasticloadbalancing:SetWebAcl",
      "elasticloadbalancing:DescribeListenerAttributes",
      "iam:CreateServiceLinkedRole",
      "iam:GetServerCertificate",
      "iam:ListServerCertificates",
      "waf:GetWebACL",
      "waf:AssociateWebACL",
      "waf:DisassociateWebACL",
      "wafv2:GetWebACL",
      "wafv2:GetWebACLForResource",
      "wafv2:AssociateWebACL",
      "wafv2:DisassociateWebACL",
      "shield:DescribeProtection",
      "shield:GetSubscriptionState",
      "shield:DeleteProtection",
      "shield:CreateProtection",
      "shield:DescribeSubscription",
      "shield:ListProtections",
      "cloudwatch:DescribeAlarms",
      "cloudwatch:GetMetricData",
      "cloudwatch:GetMetricStatistics",
      "cloudwatch:ListMetrics",
      "tag:GetResources",
      "tag:TagResources",
      "tag:UntagResources"
    ]

    resources = ["*"]
  }
}

resource "aws_iam_policy" "aws_lb_policy" {
  name        = format("%s-aws-load-balancer", var.project_name)
  path        = "/"
  description = var.project_name

  policy = data.aws_iam_policy_document.aws_lb_policy.json
}

resource "aws_iam_role_policy_attachment" "aws_lb_policy" {
  role       = aws_iam_role.aws_lb_controller.name
  policy_arn = aws_iam_policy.aws_lb_policy.arn
}
```

## 3. Installing the Controller with Helm

The controller is installed from the official chart repository, into `kube-system`, with the IRSA role wired in through the service account annotation:

```hcl
resource "helm_release" "alb_ingress_controller" {
  name             = "aws-load-balancer-controller"
  repository       = "https://aws.github.io/eks-charts"
  chart            = "aws-load-balancer-controller"
  namespace        = "kube-system"
  create_namespace = true

  set = [
    {
      name  = "clusterName"
      value = var.project_name
    },
    {
      name  = "serviceAccount.create"
      value = "true"
    },
    {
      name  = "serviceAccount.name"
      value = "aws-load-balancer-controller"
    },
    {
      name  = "serviceAccount.annotations.eks\\.amazonaws\\.com/role-arn"
      value = aws_iam_role.aws_lb_controller.arn
    },
    {
      name  = "region"
      value = var.region
    },
    {
      name  = "vpcId"
      value = data.aws_ssm_parameter.vpc.value
    }
  ]

  depends_on = [
    aws_eks_cluster.main,
    helm_release.karpenter,
    aws_eks_fargate_profile.karpenter,
    aws_eks_fargate_profile.kube_system,
  ]
}
```

Since this cluster is running the groupless architecture from [Module 6](../06%20-%20Karpenter%20Groupless%20Architecture/README.md), `kube-system` is a Fargate namespace — the controller's own pods land on Fargate. On a cluster that still has node groups, the same Helm release installs onto regular EC2 nodes without any changes. Either way, the `depends_on` makes sure the controller only gets deployed once Karpenter is running and `kube-system` has somewhere to schedule pods.

## 4. Verifying the Installation

```bash
terraform apply

kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller

kubectl logs -n kube-system -l app.kubernetes.io/name=aws-load-balancer-controller
```

The logs should show the controller's leader election succeeding and its informers starting up, with no permission errors — confirming the IRSA role and policy are wired correctly.

## 5. What We Will Build Next

With the controller installed, the rest of this module walks through its main use cases in increasing order of complexity: a plain NLB behind a `LoadBalancer` Service, an ALB driven by an `Ingress`, HTTPS on that ALB via ACM, and finally the `TargetGroupBinding` CRD that decouples the load balancer's lifecycle from Kubernetes entirely.

## Key Takeaways

- The AWS Load Balancer Controller turns `Service type LoadBalancer` into NLBs and `Ingress` objects into ALBs, both configured through annotations.
- It runs under its own IRSA role, scoped to the `aws-load-balancer-controller` service account in `kube-system`, following the same OIDC-federation pattern used for Karpenter.
- The IAM policy attached to that role is the project's official policy document — it must use a non-exclusive attachment (`aws_iam_role_policy_attachment`), not `aws_iam_policy_attachment`, so it doesn't strip other policies from a shared role.
- The Helm release depends on Karpenter and on the Fargate profiles that back `kube-system`, since on a groupless cluster the controller's own pods need somewhere stable to run.

# Lesson 2: Exposing a Service with a Network Load Balancer

## 1. Annotation-Driven Load Balancing

The simplest way to use the controller is to change a Service's `type` to `LoadBalancer` and let annotations describe how that load balancer should be built. No Ingress, no extra CRD — the Service itself becomes the load balancer's specification.

## 2. The NLB Annotation Set

Applied to the `chip` Service, this set of annotations provisions an internet-facing Network Load Balancer, targeting pods directly by IP, with an explicit list of subnets and a TCP health check:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: chip
  namespace: chip
  labels:
    app.kubernetes.io/name: chip
    app.kubernetes.io/instance: chip
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-scheme: "internet-facing"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-protocol: "TCP"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-port: "8080"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-interval: "20"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-timeout: "5"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-healthy-threshold: "3"
    service.beta.kubernetes.io/aws-load-balancer-healthcheck-unhealthy-threshold: "3"
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: "ip"
    service.beta.kubernetes.io/aws-load-balancer-subnets: "<SUBNET_ID_1>,<SUBNET_ID_2>,<SUBNET_ID_3>"
spec:
  ports:
  - name: web
    port: 8080
    protocol: TCP
    targetPort: 8080
  selector:
    app: chip
  type: LoadBalancer
```

`<SUBNET_ID_1>,<SUBNET_ID_2>,<SUBNET_ID_3>` stands in for the real, comma-separated list of public subnet IDs from this cluster's own VPC — grab those from the console or from `data.aws_ssm_parameter.public_subnets` and substitute your own.

## 3. Choosing Explicit Subnets Over Subnet Tagging

The controller can also discover subnets automatically through tags on the subnet resources themselves, instead of the `aws-load-balancer-subnets` annotation. This module deliberately uses the explicit, comma-separated form instead: keeping the subnet list directly in the manifest means everything the load balancer depends on is visible in one place, rather than split between the manifest and tags on unrelated VPC resources.

## 4. Deploying the Service

```bash
kubectl apply -f chip-nlb.yml

kubectl get pods -n chip --watch

kubectl get svc -n chip
```

Since Karpenter hasn't provisioned any nodes for `chip` yet, the pods stay pending briefly while new capacity comes up. Once they're running, the Service shows an `EXTERNAL-IP` — the DNS name of the NLB the controller just created. The AWS console shows the same load balancer as `network` type, with a target group registering the pods by IP and health checks turning healthy as they come up.

```bash
curl http://<nlb-dns-name>:8080/
```

## 5. Enabling Cross-Zone Load Balancing

An NLB does **not** balance traffic across Availability Zones by default — each AZ's listener only forwards to targets in that same AZ. If every `chip` pod happens to land in a single AZ, only that AZ's listener has anything to send traffic to.

The `aws-load-balancer-attributes` annotation exposes NLB-level attributes, including cross-zone load balancing:

```yaml
    service.beta.kubernetes.io/aws-load-balancer-attributes: "load_balancing.cross_zone.enabled=true"
```

Applying it and inspecting the load balancer's attributes in the console confirms cross-zone load balancing switches from `Off` to `On` — from that point on, traffic reaching any AZ's listener can be routed to a healthy target in any other AZ, not just its own.

## Key Takeaways

- A Service with `type: LoadBalancer` and `service.beta.kubernetes.io/aws-load-balancer-*` annotations is enough to provision a fully configured NLB — no Ingress required.
- `nlb-target-type: "ip"` registers pods directly as NLB targets; explicit `aws-load-balancer-subnets` keeps the subnet list in the manifest instead of relying on subnet tags.
- Cross-zone load balancing is **off** by default on an NLB — without `load_balancing.cross_zone.enabled=true` in `aws-load-balancer-attributes`, traffic can pile up on whichever AZ happens to have pods.

# Lesson 3: Routing with an Application Load Balancer and Ingress

## 1. From Service to Ingress

An NLB only understands Layer 4 (TCP/UDP) — it has no concept of HTTP hosts or paths. For Layer 7 routing, the controller provisions an **Application Load Balancer** instead, driven by an `Ingress` object rather than a `Service`.

Before switching, delete the NLB from Lesson 2 and wait for it to finish deprovisioning:

```bash
kubectl delete -f chip-nlb.yml
```

## 2. The ALB Ingress Annotation Set

The `chip` Service goes back to being a plain `ClusterIP`, and a new `Ingress` object carries the ALB-specific annotations instead:

```yaml
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
    targetPort: 8080
  selector:
    app: chip
  type: LoadBalancer
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: chip-ingress
  namespace: chip
  annotations:
    alb.ingress.kubernetes.io/scheme: "internet-facing"
    alb.ingress.kubernetes.io/subnets: "<SUBNET_ID_1>,<SUBNET_ID_2>,<SUBNET_ID_3>"
    alb.ingress.kubernetes.io/target-type: "ip"
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}]'
    alb.ingress.kubernetes.io/healthcheck-path: "/liveness"
    alb.ingress.kubernetes.io/healthcheck-port: "traffic-port"
    alb.ingress.kubernetes.io/healthcheck-protocol: "HTTP"
  labels:
    app.kubernetes.io/name: chip
spec:
  ingressClassName: alb
  rules:
    - host: chip.luisgustavo.com.br
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: chip
                port:
                  number: 8080
```

`ingressClassName: alb` is what tells the controller to handle this specific Ingress — that `IngressClass` was installed automatically alongside the controller's Helm chart in Lesson 1.

## 3. Host-Based Routing

The `rules[].host` field is what makes this Layer 7: the ALB only forwards to `chip`'s target group when the request's `Host` header matches `chip.luisgustavo.com.br`. Any other host falls through to the ALB's default action.

## 4. Testing with the Host Header

```bash
kubectl apply -f chip-alb.yml

kubectl get ingress -n chip
```

Once the ALB finishes provisioning, a plain request with no `Host` header returns a `404` — the fixed default action for anything that doesn't match a rule:

```bash
curl http://<alb-dns-name>/
# HTTP/1.1 404 Not Found
```

Passing the expected host, even without a real DNS record pointing at it yet, routes correctly to `chip`:

```bash
curl -H "Host: chip.luisgustavo.com.br" http://<alb-dns-name>/
```

Any path under that host — `/readiness`, `/liveness`, and so on — resolves through the same rule, since the Ingress path is `/` with `pathType: Prefix`.

## Key Takeaways

- `Ingress` with `ingressClassName: alb` and `alb.ingress.kubernetes.io/*` annotations provisions an Application Load Balancer for Layer 7 routing, the same way Service annotations provisioned an NLB in Lesson 2.
- Routing rules are host-based (`rules[].host`); anything that doesn't match a rule falls through to the ALB's default `404` action.
- A request can be tested against host-based routing before any real DNS record exists, by overriding the `Host` header directly with `curl -H`.

# Lesson 4: Securing Ingress Traffic with ACM and HTTPS

## 1. Provisioning a DNS-Validated ACM Certificate

Before adding HTTPS to the ALB, delete the HTTP-only Ingress from Lesson 3 and provision a certificate. The pattern is the standard DNS-validated ACM certificate: create the certificate, create the DNS validation records automatically from its `domain_validation_options`, then wait for validation to complete.

```hcl
resource "aws_acm_certificate" "main" {
  domain_name       = var.dns_name
  validation_method = "DNS"

  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_route53_record" "main" {
  for_each = {
    for dvo in aws_acm_certificate.main.domain_validation_options : dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }

  allow_overwrite = true
  name            = each.value.name
  records         = [each.value.record]
  ttl             = 60
  type            = each.value.type
  zone_id         = var.route53_hosted_zone
}

resource "aws_acm_certificate_validation" "main" {
  certificate_arn         = aws_acm_certificate.main.arn
  validation_record_fqdns = [for record in aws_route53_record.main : record.fqdn]
}
```

`var.dns_name` is a wildcard for the domain this course uses (`*.luisgustavo.com.br`), and `var.route53_hosted_zone` is that domain's hosted zone ID in Route 53 — substitute your own domain and hosted zone ID here (`<HOSTED_ZONE_ID>`) if you're following along with a different one. `lifecycle { create_before_destroy = true }` matters because the certificate is referenced by the Ingress annotation below — Terraform needs the replacement certificate to exist before it can safely destroy the old one.

## 2. Adding HTTPS to the Ingress

The Ingress from Lesson 3 gains a second listener, an HTTP-to-HTTPS redirect, and the certificate's ARN:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: chip-ingress
  namespace: chip
  annotations:
    alb.ingress.kubernetes.io/scheme: "internet-facing"
    alb.ingress.kubernetes.io/subnets: "<SUBNET_ID_1>,<SUBNET_ID_2>,<SUBNET_ID_3>"
    alb.ingress.kubernetes.io/target-type: "ip"
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP": 80}, {"HTTPS": 443}]'
    alb.ingress.kubernetes.io/ssl-redirect: '443'
    alb.ingress.kubernetes.io/certificate-arn: "arn:aws:acm:us-east-1:<ACCOUNT_ID>:certificate/<CERTIFICATE_ID>"
    alb.ingress.kubernetes.io/ssl-policy: "ELBSecurityPolicy-2016-08"
    alb.ingress.kubernetes.io/healthcheck-path: "/liveness"
    alb.ingress.kubernetes.io/healthcheck-port: "traffic-port"
    alb.ingress.kubernetes.io/healthcheck-protocol: "HTTP"
  labels:
    app.kubernetes.io/name: chip
spec:
  ingressClassName: alb
  rules:
    - host: chip.luisgustavo.com.br
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: chip
                port:
                  number: 8080
```

`listen-ports` now lists both `80` and `443` — a single Ingress can carry more than one listener. `ssl-redirect: '443'` turns port `80`'s listener into a fixed `301` redirect to the same host on `443`, instead of routing HTTP traffic directly. `certificate-arn` attaches the ACM certificate from step 1 to the `443` listener, and `ssl-policy` pins the TLS policy the listener negotiates with.

## 3. Testing the Redirect and the TLS Handshake

```bash
kubectl apply -f acm.tf   # or terraform apply, if managed alongside the cluster
kubectl apply -f chip-acm-https.yml
```

Once the ALB finishes provisioning, port `80` responds with a `301` redirect to the same path on `443`:

```bash
curl -I http://<alb-dns-name>/
# HTTP/1.1 301 Moved Permanently
# Location: https://chip.luisgustavo.com.br:443/
```

Hitting the ALB's own DNS name directly on `443` fails the certificate's hostname check, since the certificate only covers `*.luisgustavo.com.br` — that's expected, and `curl -k` skips the check for a quick local test:

```bash
curl -k https://<alb-dns-name>/
```

## 4. Pointing DNS at the Load Balancer

To test the real hostname end-to-end without waiting on a permanent DNS change, create a short-lived CNAME record pointing `chip.luisgustavo.com.br` at the ALB's DNS name, wait for it to propagate, then test again:

```bash
dig chip.luisgustavo.com.br

curl https://chip.luisgustavo.com.br/
```

With the real hostname resolving, the TLS handshake completes cleanly against the certificate and the response comes back from `chip` over HTTPS.

## Key Takeaways

- ACM certificates are provisioned with `validation_method = "DNS"`, validated by a set of Route 53 records generated from `domain_validation_options`, and finalized with `aws_acm_certificate_validation`.
- `lifecycle { create_before_destroy = true }` on the certificate avoids a window where the Ingress annotation points at a certificate ARN that no longer exists.
- A single Ingress can carry multiple `listen-ports`; `ssl-redirect` turns the HTTP listener into a fixed redirect to HTTPS instead of routing traffic directly.
- Hitting the ALB's own DNS name on HTTPS fails the certificate's hostname check by design — a real domain (or a temporary CNAME) pointing at the ALB is what makes the handshake succeed.

# Lesson 5: Decoupling Load Balancer Lifecycle with Target Group Binding

## 1. The Problem With Kubernetes-Managed Load Balancers

Every load balancer built so far in this module is owned by Kubernetes: delete the Service or Ingress, and the controller deletes the load balancer with it. That's convenient, but it has a cost — the load balancer's DNS name changes every time it's recreated, and anything else that depends on that specific load balancer (a VPC Link, for example) gets destroyed along with it.

## 2. The Target Group Binding Model

`TargetGroupBinding` splits the ownership in two: the load balancer, listener, and target group are provisioned and destroyed by Terraform, entirely outside Kubernetes' control, while Kubernetes only manages the association between a Service and that existing target group. Kubernetes never creates or deletes the load balancer itself — it just keeps the target group's membership in sync with the Service's endpoints.

This is useful for migrating traffic between clusters (repoint a `TargetGroupBinding` at a different target group and traffic shifts almost instantly, without touching DNS) and for attaching resources a Service alone can't reach, like a VPC Link.

## 3. Provisioning the NLB, Target Group and Listener with Terraform

Delete the ACM/Ingress setup from Lesson 4, then provision the load balancer directly with Terraform, the same way [Module 5](../05%20-%20Autoscaling%20with%20Karpenter/README.md) and [Module 6](../06%20-%20Karpenter%20Groupless%20Architecture/README.md) provisioned other infrastructure:

```hcl
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

`enable_deletion_protection = false` here is only so the lab can be torn down easily — in production, that should be `true`, precisely to avoid an accidental `terraform destroy` taking out the load balancer this whole pattern is meant to protect.

## 4. Exposing the Service as NodePort and Binding It

The `chip` Service switches to `NodePort` — the same exposure model the next module's NGINX Ingress Controller will use — and a `TargetGroupBinding` ties it to the target group created above:

```yaml
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
  type: NodePort
---
apiVersion: elbv2.k8s.aws/v1beta1
kind: TargetGroupBinding
metadata:
  name: chip
  namespace: chip
spec:
  serviceRef:
    name: chip
    port: web
  targetGroupARN: "arn:aws:elasticloadbalancing:us-east-1:<ACCOUNT_ID>:targetgroup/<TARGET_GROUP_NAME>/<TARGET_GROUP_ID>"
  targetType: instance
```

```bash
kubectl apply -f nlb.tf   # or terraform apply
kubectl apply -f chip-tgb.yml

kubectl describe targetgroupbinding chip -n chip
```

`kubectl describe` reports `Successfully reconciled` once the controller has registered the Service's endpoints in the target group — at that point the pods show up as healthy targets in the AWS console, and the NLB's DNS name serves traffic exactly as it did with the Kubernetes-managed NLB in Lesson 2, without Kubernetes owning the load balancer resource itself.

## 5. Choosing Between targetType ip and instance

`TargetGroupBinding` supports the same two target types as any ELBv2 target group:

```txt
targetType: ip       -> registers pods directly by IP (works with a ClusterIP Service)
targetType: instance -> registers the EC2 instance's NodePort (requires a NodePort Service)
```

Since `chip` was exposed as `NodePort` in this lesson, `instance` is the correct choice — it registers every node's NodePort with the target group, which is also the pattern used to expose a DaemonSet-based Ingress Controller like NGINX in the next module.

## 6. What Comes Next: The NGINX Ingress Controller

The next module deploys the NGINX Ingress Controller and binds it to a Terraform-managed NLB using the exact same `TargetGroupBinding` pattern from this lesson: the NLB continues to handle Layer 4 traffic distribution, while NGINX takes over Layer 7 routing rules inside the cluster.

## Key Takeaways

- `TargetGroupBinding` (`elbv2.k8s.aws/v1beta1`) lets Kubernetes manage only the association between a Service and an AWS-provisioned target group, without Kubernetes ever creating or destroying the load balancer itself.
- The load balancer, target group, and listener are provisioned directly with Terraform (`aws_lb`, `aws_lb_target_group`, `aws_lb_listener`) — the same tool used for every other piece of this cluster's infrastructure.
- `targetType: instance` pairs with a `NodePort` Service; `targetType: ip` pairs with a `ClusterIP` Service and registers pods directly.
- This decoupled pattern is what makes near-instant traffic migration between clusters possible, and it's the same pattern the next module uses to attach the NGINX Ingress Controller to a Terraform-managed NLB.
