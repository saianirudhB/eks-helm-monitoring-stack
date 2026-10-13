# EKS Helm Monitoring Stack

![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat&logo=kubernetes&logoColor=white)
![Helm](https://img.shields.io/badge/Helm-0F1689?style=flat&logo=helm&logoColor=white)
![AWS EKS](https://img.shields.io/badge/AWS%20EKS-FF9900?style=flat&logo=amazon-eks&logoColor=white)
![Prometheus](https://img.shields.io/badge/Prometheus-E6522C?style=flat&logo=prometheus&logoColor=white)
![Grafana](https://img.shields.io/badge/Grafana-F46800?style=flat&logo=grafana&logoColor=white)
![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)

Helm chart and monitoring configuration for deploying a containerized Django
REST API to AWS EKS, with Prometheus/Grafana dashboards and Alertmanager
alerting. Built as a portfolio project to demonstrate the Kubernetes/Helm/
monitoring design work described by this CV bullet:

> **Kubernetes Deployment on AWS EKS with Helm and Monitoring** — Deployed the
> containerized API to EKS using Deployments, Services, Ingress, ConfigMaps,
> Secrets and a Horizontal Pod Autoscaler, packaged as a Helm chart. Set up
> Prometheus and Grafana dashboards for pod, node and application metrics,
> with alerts on failed pods and high CPU.

> **Scope note:** this repository is a hand-written, portfolio-grade
> implementation of that Kubernetes/Helm/monitoring design — the chart,
> manifests, dashboards and alert rules here are real, schema-valid
> configuration that would deploy as described. They have been validated with
> `helm lint`, `helm template`, and `kubeconform` schema validation (see
> [Validation](#validation) below), but **no EKS cluster or AWS account was
> provisioned to run this** — there is nothing "live" behind it. Treat it as
> the IaC/config layer you'd point `helm install` at once a cluster exists,
> not as evidence of a running deployment.

The `api` chart is written to deploy the Django API from a sibling portfolio
project ([`credit-strategy-analytics`](../credit-strategy-analytics)), but is
kept generic (image repo/tag, env vars, health-check path are all templated)
so it would work for any containerized HTTP API.

## Architecture

```mermaid
flowchart TB
    client["Client"]

    subgraph aws["AWS"]
        subgraph eks["EKS Cluster"]
            ingress["Ingress / ALB\n(alb.ingress.kubernetes.io)"]
            svc["Service (ClusterIP)\napi"]

            subgraph deploy["Deployment\n(managed by HPA)"]
                pod1["Pod: api"]
                pod2["Pod: api"]
                pod3["Pod: api"]
            end

            hpa["HorizontalPodAutoscaler\nCPU / memory target %"]
            cm["ConfigMap\nnon-sensitive env"]
            secret["Secret\nDJANGO_SECRET_KEY, DATABASE_URL"]

            prom["Prometheus\n(kube-prometheus-stack)"]
            grafana["Grafana\ndashboards"]
            am["Alertmanager"]
        end
    end

    slack["Slack #alerts-critical"]
    email["Email (platform-team)"]

    client -->|HTTPS| ingress --> svc --> pod1 & pod2 & pod3
    hpa -.->|scales| deploy
    cm -.->|envFrom| pod1
    secret -.->|envFrom| pod1
    pod1 & pod2 & pod3 -->|/metrics scraped via ServiceMonitor| prom
    prom --> grafana
    prom -->|PrometheusRule alerts fire| am
    am -->|severity: critical| slack
    am -->|severity: warning| email
```

- **Ingress/ALB → Service → Pods**: the AWS Load Balancer Controller watches
  the `Ingress` (class `alb`) and provisions an internet-facing ALB that
  forwards to the `ClusterIP` Service, which load-balances across the API
  pods.
- **Deployment managed by HPA**: `HorizontalPodAutoscaler` watches CPU/memory
  utilization against the requests set on the Deployment's pods and scales
  replicas between `autoscaling.minReplicas` and `autoscaling.maxReplicas`.
- **ConfigMap + Secret**: non-sensitive config (`DJANGO_SETTINGS_MODULE`,
  `ALLOWED_HOSTS`, etc.) comes from a `ConfigMap`; sensitive values
  (`DJANGO_SECRET_KEY`, `DATABASE_URL`) come from a `Secret`, both wired in via
  `envFrom`.
- **Prometheus → Grafana + Alertmanager**: a `ServiceMonitor` tells the
  Prometheus Operator to scrape `/metrics` on each pod. Dashboards in
  `monitoring/dashboards/` visualize that data in Grafana; `PrometheusRule`s in
  `monitoring/alerts/` evaluate it continuously and hand firing alerts to
  Alertmanager, which routes criticals to Slack and warnings to email.

## Repository layout

```
.
├── charts/api/                       # Helm chart for the Django API
│   ├── Chart.yaml
│   ├── values.yaml                   # defaults
│   ├── values-staging.yaml           # staging overrides
│   ├── values-production.yaml        # production overrides
│   └── templates/
│       ├── _helpers.tpl
│       ├── deployment.yaml
│       ├── service.yaml
│       ├── ingress.yaml
│       ├── configmap.yaml
│       ├── secret.yaml
│       ├── hpa.yaml
│       ├── poddisruptionbudget.yaml
│       ├── serviceaccount.yaml
│       ├── servicemonitor.yaml
│       ├── NOTES.txt
│       └── tests/test-connection.yaml
├── monitoring/
│   ├── kube-prometheus-stack-values.yaml
│   ├── dashboards/
│   │   ├── pod-node-metrics.json
│   │   └── application-metrics.json
│   └── alerts/
│       ├── prometheus-rules.yaml
│       └── alertmanager-config.yaml
└── .github/workflows/helm-ci.yml     # lint + template + kubeconform on every PR
```

## Install

These commands assume an EKS cluster already exists and `kubectl`/`helm` are
configured against it. **They provision billable AWS resources** — see
[Cost & teardown](#cost--teardown) before running them for real.

### 1. Create the cluster (skip if one already exists)

```bash
eksctl create cluster \
  --name credit-strategy-api \
  --region eu-west-2 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 3 \
  --nodes-min 2 \
  --nodes-max 5 \
  --managed
```

### 2. Install the AWS Load Balancer Controller (needed for `ingress.className: alb`)

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=credit-strategy-api \
  --set serviceAccount.create=true
```

### 3. Install the monitoring stack (Prometheus Operator, Prometheus, Grafana, Alertmanager)

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace \
  -f monitoring/kube-prometheus-stack-values.yaml
```

### 4. Apply the alert rules

```bash
kubectl apply -f monitoring/alerts/prometheus-rules.yaml -n monitoring

# Load the Alertmanager routing config (Slack for critical, email for warning)
kubectl create secret generic alertmanager-kube-prometheus-stack-alertmanager \
  --from-file=alertmanager.yaml=monitoring/alerts/alertmanager-config.yaml \
  -n monitoring --dry-run=client -o yaml | kubectl apply -f -
```

### 5. Import the Grafana dashboards

The sidecar enabled in `monitoring/kube-prometheus-stack-values.yaml`
auto-imports any ConfigMap labelled `grafana_dashboard: "1"`:

```bash
kubectl create configmap pod-node-metrics-dashboard \
  --from-file=monitoring/dashboards/pod-node-metrics.json -n monitoring
kubectl label configmap pod-node-metrics-dashboard grafana_dashboard=1 -n monitoring

kubectl create configmap application-metrics-dashboard \
  --from-file=monitoring/dashboards/application-metrics.json -n monitoring
kubectl label configmap application-metrics-dashboard grafana_dashboard=1 -n monitoring
```

Or import them manually in the Grafana UI: **Dashboards → New → Import**,
paste the JSON from `monitoring/dashboards/*.json`.

### 6. Install the API chart

```bash
helm install api charts/api \
  -f charts/api/values-production.yaml \
  --set secrets.DJANGO_SECRET_KEY="$(openssl rand -hex 32)" \
  --set secrets.DATABASE_URL="postgres://user:pass@host:5432/db"
```

For staging, swap in `values-staging.yaml`. See the comment at the top of
`charts/api/templates/secret.yaml` for why secrets are passed via `--set`
rather than committed — never put real credentials in a values file.

### 7. Run the Helm test

```bash
helm test api
```

This runs `templates/tests/test-connection.yaml`, a short-lived pod that
curls the Service's `/healthz` endpoint from inside the cluster.

## Validation

No live cluster was used to build or verify this project. Instead, every
manifest was validated locally:

```bash
helm lint charts/api
helm lint charts/api -f charts/api/values-staging.yaml
helm lint charts/api -f charts/api/values-production.yaml

helm template api charts/api | kubeconform -strict -summary \
  -skip ServiceMonitor,PrometheusRule -kubernetes-version 1.29.0
```

The same checks run automatically on every pull request that touches
`charts/**` via `.github/workflows/helm-ci.yml` (`helm lint`, `helm template`,
then `kubeconform` schema-validates the rendered output, including the
`PrometheusRule`/`ServiceMonitor` CRDs against the Prometheus Operator
schema).

## Chart configuration (`charts/api`)

| Key | Description | Default |
|---|---|---|
| `replicaCount` | Pod replicas (ignored when autoscaling is enabled) | `2` |
| `image.repository` | ECR image repository | `123456789012.dkr.ecr.eu-west-2.amazonaws.com/credit-strategy-api` |
| `image.tag` | Image tag | `latest` |
| `image.pullPolicy` | Image pull policy | `IfNotPresent` |
| `serviceAccount.create` | Create a ServiceAccount (annotate for IRSA) | `true` |
| `podSecurityContext` | Pod-level security context (non-root, fsGroup, seccomp) | see `values.yaml` |
| `securityContext` | Container-level security context (`readOnlyRootFilesystem`, drop `ALL` caps) | see `values.yaml` |
| `service.type` / `service.port` / `service.targetPort` | Service type and ports | `ClusterIP` / `80` / `8000` |
| `ingress.enabled` | Create an Ingress | `true` |
| `ingress.className` | `alb` (AWS Load Balancer Controller) or `nginx` | `alb` |
| `ingress.host` | Hostname routed to the API | `api.example.com` |
| `ingress.tls.enabled` | Attach a TLS secret/block to the Ingress (set `false` when terminating TLS at the ALB via ACM) | `true` |
| `resources.requests` / `resources.limits` | CPU/memory requests and limits | `100m`/`128Mi` → `500m`/`512Mi` |
| `autoscaling.enabled` | Create a HorizontalPodAutoscaler | `true` |
| `autoscaling.minReplicas` / `maxReplicas` | HPA replica bounds | `2` / `10` |
| `autoscaling.targetCPUUtilizationPercentage` | HPA CPU target | `70` |
| `autoscaling.targetMemoryUtilizationPercentage` | HPA memory target | `80` |
| `podDisruptionBudget.enabled` / `minAvailable` | PDB to protect availability during node drains | `true` / `1` |
| `probes.liveness` / `probes.readiness` | Health probe paths & timing (`/healthz`) | see `values.yaml` |
| `config` | Non-sensitive env vars → ConfigMap | see `values.yaml` |
| `secrets.create` | Render a Secret from `secrets.*` (`false` to bring your own via `secrets.externalSecretName`) | `true` |
| `secrets.DJANGO_SECRET_KEY` / `DATABASE_URL` | **Placeholders** — always override with `--set` or an external secret manager | `CHANGEME-...` |
| `serviceMonitor.enabled` | Register a `ServiceMonitor` for Prometheus Operator scraping | `true` |

Environment overrides live in `values-staging.yaml` (1 replica, smaller
limits, autoscale 1-3) and `values-production.yaml` (3 replicas, larger
limits, autoscale 3-15, pod anti-affinity, a 2-pod PDB).

## Alerting

`monitoring/alerts/prometheus-rules.yaml` defines:

| Alert | Trigger | Severity |
|---|---|---|
| `PodCrashLooping` | `kube_pod_container_status_waiting_reason{reason="CrashLoopBackOff"}` for 5m | critical |
| `PodNotReady` | Pod stuck `Pending`/`Unknown`/`Failed` for 15m | critical |
| `HighCPUUsage` | Container CPU usage > 85% of its limit for 10m | warning |
| `HighNodeCPUUsage` | Node CPU utilization > 85% for 10m | warning |
| `HighMemoryUsage` | Container memory usage > 90% of its limit for 10m | warning |
| `HPAMaxedOut` | HPA sitting at `maxReplicas` for 15m | warning |

`monitoring/alerts/alertmanager-config.yaml` routes `severity: critical` to a
Slack webhook (`#alerts-critical`) and `severity: warning` to email, with an
inhibition rule so a crash-looping pod doesn't also page separately for high
CPU/memory on the same pod.

## Cost & teardown

An EKS control plane costs **$0.10/hour (~$73/month)** on its own, plus
worker node EC2 cost and any ALBs/EBS volumes provisioned. **Nothing in this
repository was deployed to a real cluster** — it exists purely as validated
IaC. If you do stand this up for real, tear it down when you're done:

```bash
helm uninstall api
helm uninstall kube-prometheus-stack -n monitoring
eksctl delete cluster --name credit-strategy-api --region eu-west-2
```

Double-check the AWS console afterwards for any ALBs, EBS volumes, or ENIs
that weren't cleaned up automatically.

## Tech stack

Kubernetes · Helm · AWS EKS · AWS Load Balancer Controller · Prometheus
Operator (`kube-prometheus-stack`) · Grafana · Alertmanager · GitHub Actions ·
`kubeconform`
