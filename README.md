# Kubernetes Observability Stack on Amazon EKS

A production-style observability setup deployed on Amazon EKS using Prometheus, Grafana, Loki, Promtail, and Alertmanager.

---

## Stack Overview

| Component | Role |
|---|---|
| **Prometheus** | Metrics collection and storage |
| **Grafana** | Visualization and dashboards |
| **Loki** | Log aggregation |
| **Promtail** | Kubernetes log collector (DaemonSet) |
| **Alertmanager** | Alert routing and notifications |

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    Amazon EKS Cluster                    │
│                                                         │
│  ┌─────────────┐     ┌─────────────┐                   │
│  │  Prometheus │────▶│   Grafana   │◀── Browser        │
│  │  (Metrics)  │     │ (Dashboards)│                   │
│  └─────────────┘     └─────────────┘                   │
│         ▲                   ▲                           │
│         │                   │                           │
│  ┌─────────────┐     ┌─────────────┐                   │
│  │ Node        │     │    Loki     │                   │
│  │ Exporter    │     │    (Logs)   │                   │
│  │ Kube-State  │     └─────────────┘                   │
│  │ Metrics     │            ▲                           │
│  └─────────────┘            │                           │
│                      ┌─────────────┐                   │
│                      │  Promtail   │                   │
│                      │ (DaemonSet) │                   │
│                      └─────────────┘                   │
│                                                         │
│  ┌─────────────┐                                        │
│  │Alertmanager │                                        │
│  │  (Alerts)   │                                        │
│  └─────────────┘                                        │
└─────────────────────────────────────────────────────────┘
         │
         ▼
  Classic ELB (Port 80)
         │
         ▼
  Browser / Grafana UI
```

---

## Prerequisites

- AWS Account with appropriate IAM permissions
- AWS CLI configured
- `kubectl` installed
- `helm` installed
- EKS cluster running (**EKS Auto Mode** with Bottlerocket nodes)

---

## Cluster Setup

### Create EKS Cluster (via AWS Console)

1. Go to **AWS Console → EKS → Create Cluster**
2. Configure:
   - **Name:** `observability-cluster`
   - **Kubernetes version:** `1.35` (or latest)
   - **Cluster endpoint:** Public
3. Add Node Group:
   - **Instance type:** `t3.xlarge` (4 vCPU / 16GB)
   - **Desired capacity:** 3 nodes
4. Enable Add-ons:
   - CoreDNS
   - kube-proxy
   - Amazon VPC CNI
   - Amazon EBS CSI Driver

### Connect via CloudShell

```bash
# Install kubectl
curl -LO "https://dl.k8s.io/release/$(curl -L -s \
  https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
chmod +x kubectl && mkdir -p ~/.local/bin && mv kubectl ~/.local/bin/kubectl

# Install Helm
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Configure kubectl
aws eks update-kubeconfig --region us-east-1 --name observability-cluster

# Verify
kubectl get nodes
```

---

## Deployment

### 1. Create Namespace

```bash
kubectl create namespace observability
```

### 2. Add Helm Repositories

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

### 3. Install Prometheus + Grafana + Alertmanager

```bash
helm install prometheus \
  prometheus-community/kube-prometheus-stack \
  --namespace observability \
  --create-namespace
```

This single command deploys:
- Prometheus
- Grafana (with pre-built dashboards)
- Alertmanager
- Node Exporter (DaemonSet on every node)
- Kube State Metrics

### 4. Install Loki + Promtail

```bash
helm install loki \
  grafana/loki-stack \
  --namespace observability \
  --set grafana.enabled=false
```

> `grafana.enabled=false` because Grafana is already installed above.

---

## Known Issue — Loki Datasource Conflict

When Loki is installed alongside `kube-prometheus-stack`, both datasources are marked as `isDefault: true` which causes Grafana to crash.

### Fix

```bash
# Patch the Loki datasource ConfigMap to set isDefault: false
kubectl patch configmap loki-loki-stack -n observability --type merge \
  -p '{"data":{"loki-stack-datasource.yaml":"apiVersion: 1\ndatasources:\n- name: Loki\n  type: loki\n  access: proxy\n  url: \"http://loki:3100\"\n  version: 1\n  isDefault: false\n  jsonData:\n    {}\n"}}'

# Restart Grafana
kubectl delete pod -n observability -l "app.kubernetes.io/name=grafana"
```

---

## Accessing Grafana

### Expose via LoadBalancer

```bash
kubectl patch svc prometheus-grafana -n observability \
  -p '{"spec": {"type": "LoadBalancer"}}'

# Get the URL
kubectl get svc prometheus-grafana -n observability
```

### EKS Auto Mode — Fix: Register Nodes with ELB

> EKS Auto Mode does not automatically register nodes with Classic ELB.
> You must do this manually.

```bash
# Get node security group
aws ec2 describe-instances \
  --instance-ids <node-instance-ids> \
  --query "Reservations[].Instances[].[InstanceId,SecurityGroups[].GroupId]" \
  --output table

# Allow NodePort traffic through security group
aws ec2 authorize-security-group-ingress \
  --group-id <sg-id> \
  --protocol tcp \
  --port <nodeport> \
  --cidr 0.0.0.0/0

# Register nodes with ELB
aws elb register-instances-with-load-balancer \
  --load-balancer-name <elb-name> \
  --instances <instance-id-1> <instance-id-2> <instance-id-3>

# Verify instances are healthy
aws elb describe-instance-health \
  --load-balancer-name <elb-name>
```

### Get Grafana Admin Password

```bash
kubectl --namespace observability get secrets prometheus-grafana \
  -o jsonpath="{.data.admin-password}" | base64 -d ; echo
```

### Login

```
URL:      http://<elb-dns-name>
Username: admin
Password: <from above command>
```

---

## Verify Deployment

```bash
# All pods should be Running
kubectl get pods -n observability

# Expected output:
# alertmanager-prometheus-kube-prometheus-alertmanager-0   2/2   Running
# loki-0                                                   1/1   Running
# loki-promtail-xxxxx                                      1/1   Running  (one per node)
# prometheus-grafana-xxxxx                                 3/3   Running
# prometheus-kube-prometheus-operator-xxxxx                1/1   Running
# prometheus-kube-state-metrics-xxxxx                      1/1   Running
# prometheus-prometheus-kube-prometheus-prometheus-0       2/2   Running
# prometheus-prometheus-node-exporter-xxxxx                1/1   Running  (one per node)
```

```bash
# Check all services
kubectl get svc -n observability

# Check Helm releases
helm list -n observability
```

---

## Grafana Dashboards

The following dashboards come pre-installed:

| Dashboard | What it shows |
|---|---|
| **Kubernetes / Compute Resources / Cluster** | Cluster-wide CPU & memory |
| **Kubernetes / Compute Resources / Node** | Per-node resource usage |
| **Kubernetes / Compute Resources / Pod** | Per-pod resource usage |
| **Kubernetes / Networking** | Network traffic |
| **Node Exporter / Nodes** | Detailed node metrics |
| **Alertmanager / Overview** | Alert status |
| **Prometheus / Overview** | Prometheus internals |

---

## Datasources in Grafana

| Datasource | URL | Default |
|---|---|---|
| **Prometheus** | `http://prometheus-kube-prometheus-prometheus:9090` | ✅ Yes |
| **Alertmanager** | `http://prometheus-kube-prometheus-alertmanager:9093` | No |
| **Loki** | `http://loki:3100` | No |

---

## Helm Releases

```bash
helm list -n observability

# NAME        NAMESPACE      CHART                          APP VERSION
# prometheus  observability  kube-prometheus-stack-86.3.2   v0.91.0
# loki        observability  loki-stack-2.10.3              ...
```

---

## Useful Commands

```bash
# Watch all pods
kubectl get pods -n observability -w

# View Grafana logs
kubectl logs -n observability -l "app.kubernetes.io/name=grafana" --tail=50

# Port-forward Grafana locally
kubectl port-forward svc/prometheus-grafana 3000:80 -n observability

# Port-forward Prometheus locally
kubectl port-forward svc/prometheus-operated 9090:9090 -n observability

# Port-forward Alertmanager locally
kubectl port-forward svc/prometheus-kube-prometheus-alertmanager 9093:9093 -n observability

# Rollback Helm release
helm rollback prometheus -n observability

# Export current Helm values
helm get values prometheus -n observability > my-prometheus-values.yaml
helm get values loki -n observability > my-loki-values.yaml
```

---

## Cleanup

```bash
# Uninstall Helm releases
helm uninstall prometheus -n observability
helm uninstall loki -n observability

# Delete namespace
kubectl delete namespace observability

# Delete EKS cluster
eksctl delete cluster --name observability-cluster --region us-east-1
```

---

## Troubleshooting

| Issue | Cause | Fix |
|---|---|---|
| Grafana `CrashLoopBackOff` | Duplicate `isDefault` datasource | Patch `loki-loki-stack` ConfigMap |
| ELB `Empty reply from server` | Nodes not registered with ELB | Manually register instances |
| ELB `DNS_PROBE_FINISHED_NXDOMAIN` | DNS propagation delay | Wait 2-5 minutes |
| NodePort unreachable | Security group missing rule | Add inbound rule for NodePort |
| `curl: (52) Empty reply` | Grafana still initializing | Wait 1-2 minutes and retry |

---

## Tech Stack Versions

| Component | Version |
|---|---|
| Kubernetes | 1.35 (EKS Auto) |
| kube-prometheus-stack | 86.3.2 |
| Grafana | 13.0.2 |
| loki-stack | 2.10.3 |
| Node OS | Bottlerocket (EKS Auto Standard) |

---

## Creating a simple Dashboard
<img width="1392" height="778" alt="image" src="https://github.com/user-attachments/assets/a9b49275-109f-4d48-9d7f-8ea0d03db7ca" />
