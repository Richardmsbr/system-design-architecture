# Kubernetes Platform (EKS)

Production-grade Kubernetes platform on AWS EKS with GitOps, observability, and security.

---

## Architecture Overview

```
                              DEVELOPERS
                                   │
                    ┌──────────────┴──────────────┐
                    │                             │
                    ▼                             ▼
            ┌─────────────┐               ┌─────────────┐
            │   GitHub    │               │   ArgoCD    │
            │  (Source)   │──────────────>│  (GitOps)   │
            └─────────────┘               └──────┬──────┘
                    │                            │
                    │ CI                         │ CD
                    ▼                            ▼
            ┌─────────────┐               ┌─────────────┐
            │   GitHub    │               │     EKS     │
            │   Actions   │──────────────>│   Cluster   │
            └─────────────┘               └──────┬──────┘
                    │                            │
                    │ Push                       │
                    ▼                            │
            ┌─────────────┐                      │
            │     ECR     │◄─────────────────────┘
            │  (Images)   │
            └─────────────┘


    ┌─────────────────────────────────────────────────────────────────────────┐
    │                              EKS CLUSTER                                │
    ├─────────────────────────────────────────────────────────────────────────┤
    │                                                                         │
    │   CONTROL PLANE (AWS Managed)                                          │
    │   ┌─────────────────────────────────────────────────────────────────┐  │
    │   │  API Server │ etcd │ Controller Manager │ Scheduler             │  │
    │   └─────────────────────────────────────────────────────────────────┘  │
    │                                                                         │
    │   DATA PLANE                                                            │
    │   ┌─────────────────────────────────────────────────────────────────┐  │
    │   │                                                                  │  │
    │   │   SYSTEM NAMESPACE (kube-system)                                │  │
    │   │   ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐          │  │
    │   │   │CoreDNS   │ │AWS LB    │ │Karpenter │ │Metrics   │          │  │
    │   │   │          │ │Controller│ │          │ │Server    │          │  │
    │   │   └──────────┘ └──────────┘ └──────────┘ └──────────┘          │  │
    │   │                                                                  │  │
    │   │   PLATFORM NAMESPACE                                             │  │
    │   │   ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐          │  │
    │   │   │ArgoCD    │ │External  │ │Cert      │ │Secrets   │          │  │
    │   │   │          │ │Secrets   │ │Manager   │ │Store CSI │          │  │
    │   │   └──────────┘ └──────────┘ └──────────┘ └──────────┘          │  │
    │   │                                                                  │  │
    │   │   MONITORING NAMESPACE                                           │  │
    │   │   ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐          │  │
    │   │   │Prometheus│ │Grafana   │ │Loki      │ │Tempo     │          │  │
    │   │   │          │ │          │ │          │ │          │          │  │
    │   │   └──────────┘ └──────────┘ └──────────┘ └──────────┘          │  │
    │   │                                                                  │  │
    │   │   APPLICATION NAMESPACES                                         │  │
    │   │   ┌──────────────────────────────────────────────────────────┐  │  │
    │   │   │  production  │  staging  │  development                  │  │  │
    │   │   │                                                          │  │  │
    │   │   │  ┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐            │  │  │
    │   │   │  │ App A  │ │ App B  │ │ App C  │ │  ...   │            │  │  │
    │   │   │  └────────┘ └────────┘ └────────┘ └────────┘            │  │  │
    │   │   └──────────────────────────────────────────────────────────┘  │  │
    │   │                                                                  │  │
    │   └─────────────────────────────────────────────────────────────────┘  │
    │                                                                         │
    │   NODE GROUPS                                                           │
    │   ┌─────────────┐ ┌─────────────┐ ┌─────────────┐                     │
    │   │  System     │ │ Application │ │  Spot       │                     │
    │   │  (m6i.large)│ │ (c6i.xlarge)│ │ (mixed)     │                     │
    │   │  Min: 2     │ │  Min: 2     │ │  Min: 0     │                     │
    │   │  Max: 4     │ │  Max: 20    │ │  Max: 50    │                     │
    │   └─────────────┘ └─────────────┘ └─────────────┘                     │
    │                                                                         │
    └─────────────────────────────────────────────────────────────────────────┘
```

---

## EKS Cluster Configuration

### Terraform

```hcl
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 19.0"

  cluster_name    = "production-cluster"
  cluster_version = "1.28"

  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  cluster_endpoint_public_access  = true
  cluster_endpoint_private_access = true

  # Encryption
  cluster_encryption_config = {
    provider_key_arn = aws_kms_key.eks.arn
    resources        = ["secrets"]
  }

  # Logging
  cluster_enabled_log_types = [
    "api", "audit", "authenticator", "controllerManager", "scheduler"
  ]

  # Add-ons
  cluster_addons = {
    coredns = {
      most_recent = true
    }
    kube-proxy = {
      most_recent = true
    }
    vpc-cni = {
      most_recent              = true
      service_account_role_arn = module.vpc_cni_irsa.iam_role_arn
    }
    aws-ebs-csi-driver = {
      most_recent              = true
      service_account_role_arn = module.ebs_csi_irsa.iam_role_arn
    }
  }

  # Managed Node Groups
  eks_managed_node_groups = {
    system = {
      name           = "system"
      instance_types = ["m6i.large"]
      min_size       = 2
      max_size       = 4
      desired_size   = 2

      labels = {
        role = "system"
      }

      taints = [{
        key    = "CriticalAddonsOnly"
        value  = "true"
        effect = "NO_SCHEDULE"
      }]
    }

    application = {
      name           = "application"
      instance_types = ["c6i.xlarge", "c6i.2xlarge"]
      min_size       = 2
      max_size       = 20
      desired_size   = 3

      labels = {
        role = "application"
      }
    }
  }
}
```

---

## GitOps with ArgoCD

### Installation

```yaml
# argocd/install.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argocd
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://argoproj.github.io/argo-helm
    targetRevision: 5.46.0
    chart: argo-cd
    helm:
      values: |
        server:
          ingress:
            enabled: true
            ingressClassName: alb
            annotations:
              alb.ingress.kubernetes.io/scheme: internet-facing
              alb.ingress.kubernetes.io/certificate-arn: ${ACM_CERT_ARN}
        configs:
          params:
            server.insecure: true
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### App of Apps Pattern

```yaml
# applications/root.yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: applications
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/company/gitops-repo
    targetRevision: main
    path: applications
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true

---
# applications/platform/kustomization.yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
resources:
  - cert-manager.yaml
  - external-secrets.yaml
  - karpenter.yaml
  - prometheus-stack.yaml
  - loki.yaml
```

---

## Auto Scaling

### Karpenter Configuration

```yaml
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: default
spec:
  requirements:
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["on-demand", "spot"]
    - key: kubernetes.io/arch
      operator: In
      values: ["amd64"]
    - key: node.kubernetes.io/instance-type
      operator: In
      values:
        - c6i.large
        - c6i.xlarge
        - c6i.2xlarge
        - m6i.large
        - m6i.xlarge

  limits:
    resources:
      cpu: 1000
      memory: 2000Gi

  providerRef:
    name: default

  ttlSecondsAfterEmpty: 60
  ttlSecondsUntilExpired: 604800  # 7 days

---
apiVersion: karpenter.k8s.aws/v1alpha1
kind: AWSNodeTemplate
metadata:
  name: default
spec:
  subnetSelector:
    karpenter.sh/discovery: "production-cluster"
  securityGroupSelector:
    karpenter.sh/discovery: "production-cluster"

  blockDeviceMappings:
    - deviceName: /dev/xvda
      ebs:
        volumeSize: 100Gi
        volumeType: gp3
        encrypted: true

  tags:
    Environment: production
    ManagedBy: karpenter
```

### Horizontal Pod Autoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api-server
  namespace: production
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api-server
  minReplicas: 3
  maxReplicas: 50
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
    - type: Pods
      pods:
        metric:
          name: http_requests_per_second
        target:
          type: AverageValue
          averageValue: "1000"
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 10
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15
```

---

## Observability Stack

### Prometheus Stack

```yaml
# monitoring/prometheus-values.yaml
prometheus:
  prometheusSpec:
    retention: 15d
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3
          resources:
            requests:
              storage: 100Gi

    additionalScrapeConfigs:
      - job_name: 'kubernetes-pods'
        kubernetes_sd_configs:
          - role: pod
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_annotation_prometheus_io_scrape]
            action: keep
            regex: true

grafana:
  adminPassword: ${GRAFANA_ADMIN_PASSWORD}
  persistence:
    enabled: true
    size: 10Gi

  datasources:
    datasources.yaml:
      apiVersion: 1
      datasources:
        - name: Prometheus
          type: prometheus
          url: http://prometheus-server
          isDefault: true
        - name: Loki
          type: loki
          url: http://loki:3100
        - name: Tempo
          type: tempo
          url: http://tempo:3100

alertmanager:
  config:
    global:
      slack_api_url: ${SLACK_WEBHOOK_URL}
    route:
      receiver: 'slack'
      group_by: ['alertname', 'namespace']
    receivers:
      - name: 'slack'
        slack_configs:
          - channel: '#alerts'
            send_resolved: true
```

### Loki for Logs

```yaml
# monitoring/loki-values.yaml
loki:
  auth_enabled: false

  storage:
    type: s3
    s3:
      region: us-east-1
      bucketnames: company-loki-logs
      s3ForcePathStyle: false

  schemaConfig:
    configs:
      - from: 2024-01-01
        store: boltdb-shipper
        object_store: s3
        schema: v11
        index:
          prefix: loki_index_
          period: 24h

promtail:
  config:
    clients:
      - url: http://loki:3100/loki/api/v1/push

    scrape_configs:
      - job_name: kubernetes-pods
        kubernetes_sd_configs:
          - role: pod
        pipeline_stages:
          - docker: {}
          - json:
              expressions:
                level: level
                msg: msg
          - labels:
              level:
```

---

## Security

### Pod Security Standards

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### Network Policies

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-api-ingress
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api-server
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx
      ports:
        - protocol: TCP
          port: 8080
```

### External Secrets

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: aws-secrets-manager
  namespace: production
spec:
  provider:
    aws:
      service: SecretsManager
      region: us-east-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets-sa

---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: database-credentials
  namespace: production
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: aws-secrets-manager
    kind: SecretStore
  target:
    name: database-credentials
  data:
    - secretKey: username
      remoteRef:
        key: production/database
        property: username
    - secretKey: password
      remoteRef:
        key: production/database
        property: password
```

---

## Cost Optimization

### Spot Instances with Karpenter

```yaml
# 70% spot, 30% on-demand
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: spot
spec:
  requirements:
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["spot"]
  weight: 70

---
apiVersion: karpenter.sh/v1alpha5
kind: Provisioner
metadata:
  name: on-demand
spec:
  requirements:
    - key: karpenter.sh/capacity-type
      operator: In
      values: ["on-demand"]
  weight: 30
```

### Resource Quotas

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: team-quota
  namespace: team-a
spec:
  hard:
    requests.cpu: "20"
    requests.memory: 40Gi
    limits.cpu: "40"
    limits.memory: 80Gi
    persistentvolumeclaims: "10"
```

---

## Deployment Strategy

### Canary with Argo Rollouts

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: api-server
  namespace: production
spec:
  replicas: 10
  strategy:
    canary:
      steps:
        - setWeight: 5
        - pause: {duration: 5m}
        - setWeight: 20
        - pause: {duration: 10m}
        - setWeight: 50
        - pause: {duration: 10m}
        - setWeight: 80
        - pause: {duration: 10m}

      analysis:
        templates:
          - templateName: success-rate
        startingStep: 1
        args:
          - name: service-name
            value: api-server

  selector:
    matchLabels:
      app: api-server
  template:
    metadata:
      labels:
        app: api-server
    spec:
      containers:
        - name: api
          image: company/api:v2.0.0
          ports:
            - containerPort: 8080
```

---

## Terraform

See [terraform/aws/eks/](../../../terraform/aws/eks/) for complete implementation.
