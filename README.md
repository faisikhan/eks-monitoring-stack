# Production Kubernetes Observability Stack on Amazon EKS

A production-style Kubernetes observability setup deployed on Amazon EKS
using:

-   Prometheus → Metrics collection
-   Grafana → Visualization and dashboards
-   Loki → Log aggregation
-   Promtail → Kubernetes log collector
-   Alertmanager → Alerting

## Architecture

    Kubernetes Workloads
            |
            |
       Prometheus
            |
            |
         Grafana


    Container Logs
            |
            |
        Promtail
            |
            |
          Loki
            |
            |
         Grafana

## Project Goals

This project demonstrates:

-   Helm deployments
-   Kubernetes monitoring
-   Service discovery on EKS
-   Prometheus metrics collection
-   Grafana dashboards
-   Kubernetes log aggregation
-   Loki queries

## Prerequisites

-   AWS EKS Cluster
-   kubectl configured
-   Helm installed

Verify:

``` bash
kubectl get nodes
```

## Step 1: Create Monitoring Namespace

``` bash
kubectl create namespace monitoring
```

## Step 2: Add Helm Repositories

``` bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts

helm repo add grafana https://grafana.github.io/helm-charts

helm repo update
```

## Step 3: Install Prometheus + Grafana

``` bash
helm install prometheus \
prometheus-community/kube-prometheus-stack \
--namespace monitoring
```

Verify:

``` bash
kubectl get pods -n monitoring
```

Installed components:

-   Prometheus
-   Grafana
-   Alertmanager
-   Node Exporter
-   kube-state-metrics

## Step 4: Expose Grafana

``` bash
kubectl patch svc prometheus-grafana \
-n monitoring \
-p '{"spec":{"type":"LoadBalancer"}}'
```

Get endpoint:

``` bash
kubectl get svc -n monitoring
```

Grafana login:

    Username: admin
    Password: prom-operator

## Step 5: Install Loki

``` bash
helm install loki grafana/loki-stack \
--namespace monitoring \
--set grafana.enabled=false \
--set promtail.enabled=false
```

Verify:

``` bash
kubectl get pods -n monitoring
```

## Step 6: Install Promtail

``` bash
helm install promtail grafana/promtail \
--namespace monitoring \
--set loki.serviceName=loki
```

Promtail collects logs from:

    /var/log/containers/

## Step 7: Connect Loki with Grafana

Grafana → Data Sources → Add Data Source → Loki

URL:

    http://loki.monitoring.svc.cluster.local:3100

Save and test.

## View Logs

Grafana → Explore → Loki

Examples:

All default namespace logs:

``` logql
{namespace="default"}
```

Search errors:

``` logql
{namespace="default"} |= "error"
```

## Observability Flow

Metrics:

    Kubernetes
         |
    Prometheus
         |
    Grafana

Logs:

    Pods
     |
    Promtail
     |
    Loki
     |
    Grafana

## Future Improvements

-   AWS Managed Prometheus integration
-   Remote write configuration
-   Alert rules
-   Slack/email notifications
-   Custom application metrics

