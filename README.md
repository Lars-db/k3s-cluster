# k3s Cluster — Ticket Shop

A single-node k3s Kubernetes cluster running a ticket shop application with a PostgreSQL database and a Grafana monitoring stack.

## Cluster overview

| Component | Description |
|---|---|
| **k3s** | Lightweight Kubernetes distribution |
| **Node** | `the-mothership` (control-plane) |
| **Ingress controller** | Traefik (bundled with k3s) |
| **Namespaces** | `default` (app), `monitoring` (observability) |

---

## Applications

### Ticket Shop API (`default` namespace)

A REST API for managing events and ticket purchases.

- **Image:** `larstechchamps/ticket-shop:1.0.0`
- **Replicas:** 2 (for availability)
- **Port:** 8080
- **Accessible at:** `ticket-api.com` (via Traefik ingress)

The API connects to the database using the environment variable `DATABASE_URL`. Health checks are configured on `/healthz` so Kubernetes can detect when a pod is ready to receive traffic and restart it if it becomes unhealthy.

### PostgreSQL Database (`default` namespace)

Stores events and tickets.

- **Image:** `postgres:15`
- **Type:** StatefulSet (1 replica) — StatefulSet is used instead of Deployment because databases need a stable identity and persistent storage that survives restarts
- **Persistent storage:** 1Gi via `local-path` storage class
- **Internal DNS:** `ticket-db:5432` (only reachable inside the cluster)

The database schema is automatically applied on first startup via a ConfigMap (`ticket-db-init`) mounted into `/docker-entrypoint-initdb.d`. It creates two tables:

- `events` — event name, location, date, and ticket counts
- `tickets` — individual ticket purchases linked to an event

---

## Monitoring (`monitoring` namespace)

### Architecture

```
k3s pods & nodes
      │
      ├── node-exporter        # hardware/OS metrics per node
      └── kube-state-metrics   # pod/deployment/container metrics
              │
              ▼
         Prometheus             # scrapes & stores all metrics
              │
              ▼
           Grafana              # queries Prometheus and renders dashboards
```

### Prometheus

Installed via `prometheus-community/prometheus` Helm chart (release name: `prom`).

Automatically discovers and scrapes metrics from all pods and nodes in the cluster every 15 seconds. Two exporters run alongside it:

- **node-exporter** — exposes CPU, memory, disk, and network stats at the node level
- **kube-state-metrics** — talks to the Kubernetes API and exposes pod restarts, resource requests/limits, deployment status, etc.

Prometheus is only accessible internally (`ClusterIP`) — it does not need to be exposed externally since Grafana queries it from within the cluster.

### Grafana

Installed via `grafana/grafana` Helm chart (release name: `prometheus`).

- **Accessible at:** `grafana.ticket-api.com` and `http://<node-ip>:32000` (NodePort)
- **Default login:** `admin` / `admin`
- **Persistent storage:** 1Gi (dashboards and settings survive pod restarts)

#### Datasource

Grafana is pre-configured (via provisioning in `grafana-values.yaml`) to connect to Prometheus at:

```
http://prom-prometheus-server.monitoring.svc.cluster.local
```

This uses Kubernetes internal DNS — no external exposure of Prometheus needed.

#### Dashboards

Two community dashboards are automatically loaded at startup under the **Kubernetes** folder:

| Dashboard | Grafana ID | Shows |
|---|---|---|
| Kubernetes Pod & Namespace | 6417 | CPU/memory per pod, restarts, network |
| Kubernetes Cluster Monitoring | 315 | Cluster-wide resource usage overview |

---

## File structure

```
yaml-files/
├── deployments/
│   └── ticket-shop-deployment.yaml     # Ticket API deployment (2 replicas)
├── services/
│   ├── ticket-shop-service.yaml        # ClusterIP service for the API
│   └── ticket-db-service.yaml          # ClusterIP service for PostgreSQL
├── statefulSets/
│   └── ticket-db-statefulset.yaml      # PostgreSQL with persistent volume
├── configmaps/
│   └── ticket-db-init-configmap.yaml   # SQL schema applied on DB first boot
├── ingress/
│   ├── ingress-ticket-shop.yaml        # Routes ticket-api.com → API service
│   └── ingress-grafana.yaml            # Routes grafana.ticket-api.com → Grafana
├── grafana-values.yaml                 # Grafana Helm values (datasource, dashboards, NodePort)
└── helm/
    └── monitoring/
        ├── values.yaml                 # Full Grafana Helm chart values
        └── prometheus-values.yaml      # Prometheus Helm values (storage, exporters)
```

---

## Deploying from scratch

### Prerequisites

- k3s installed and running
- `kubectl` configured
- `helm` installed

### 1. Add Helm repositories

```bash
helm repo add grafana https://grafana.github.io/helm-charts
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

### 2. Deploy the monitoring stack

```bash
# Create the monitoring namespace
kubectl create namespace monitoring

# Install Prometheus (includes node-exporter and kube-state-metrics)
helm install prom prometheus-community/prometheus \
  -n monitoring \
  -f helm/monitoring/prometheus-values.yaml

# Install Grafana
helm install prometheus grafana/grafana \
  -n monitoring \
  -f grafana-values.yaml
```

### 3. Deploy the application

```bash
kubectl apply -f configmaps/
kubectl apply -f statefulSets/
kubectl apply -f deployments/
kubectl apply -f services/
kubectl apply -f ingress/
```

### 4. Access the cluster

Point the following entries to your node's IP in your `/etc/hosts` or DNS:

```
<node-ip>  ticket-api.com
<node-ip>  grafana.ticket-api.com
```

- **Ticket API:** http://ticket-api.com
- **Grafana:** http://grafana.ticket-api.com (or http://\<node-ip\>:32000)
