# Session 20: Monitoring, Observability & GitOps

Verification of the three pillars of observability, Prometheus metrics and PromQL, a Prometheus + Grafana stack, and GitOps with Argo CD.

**Environment:** Docker Desktop Kubernetes (`docker-desktop`, v1.36) with metrics-server, Docker Compose, PowerShell. Images: `prom/prometheus:v3.5.0`, `grafana/grafana:12.1.1`, Argo CD `stable`.

---

## Table of Contents

- [Part 1 — Metrics & Logs from a Kubernetes Workload](#part-1--metrics--logs-from-a-kubernetes-workload)
- [Part 2 — Prometheus & PromQL Queries](#part-2--prometheus--promql-queries)
- [Part 3 — Prometheus + Grafana Stack](#part-3--prometheus--grafana-stack)
- [Part 4 — Argo CD Installation](#part-4--argo-cd-installation)
- [Part 5 — GitOps Flow (to complete)](#part-5--gitops-flow-to-complete)

---

## Part 1 — Metrics & Logs from a Kubernetes Workload

```bash
cd session20-monitoring-observability-gitops\02-metrics-logs-traces\k8s-demo
kubectl apply -f deployment.yaml -f service.yaml
kubectl rollout status deployment/session20-demo
kubectl get pods -l app=session20-demo
kubectl logs deployment/session20-demo --tail=8      # LOGS: what happened
kubectl top pods -l app=session20-demo               # METRICS: how much (CPU / memory)
```

**Result:** The Pod runs and emits structured log lines (`Request received`, `Health check OK`). `kubectl top` reports live resource metrics from metrics-server. **Traces**, the third pillar, need request-level instrumentation such as OpenTelemetry, which this demo app doesn't have.

![Part 1 — Metrics & Logs](./SS/01-metrics-logs.png)

---

## Part 2 — Prometheus & PromQL Queries

```bash
cd session20-monitoring-observability-gitops\03-prometheus
docker compose up -d
docker compose ps
curl.exe -s http://localhost:9090/-/ready

# Scrape targets
(Invoke-RestMethod 'http://localhost:9090/api/v1/targets').data.activeTargets | Select-Object scrapeUrl, health, lastScrapeDuration | Format-Table -AutoSize

# PromQL via the HTTP API
up
prometheus_tsdb_head_series
sum by (handler) (rate(prometheus_http_requests_total[1m])) > 0

docker compose down
```

**Result:**
- Target `http://prometheus:9090/metrics` is `up` (Prometheus scrapes itself every 5s, per `prometheus.yml`).
- `up{job="prometheus"} = 1`
- `prometheus_tsdb_head_series = 573` active time series.
- `rate(...)` shows the `/metrics` handler being hit ~0.13 req/s (once every ~5s by the scraper), which matches the scrape interval.

![Part 2 — Prometheus Queries](./SS/02-prometheus-queries.png)

---

## Part 3 — Prometheus + Grafana Stack

```bash
cd session20-monitoring-observability-gitops\04-grafana
docker compose up -d
docker compose ps
curl.exe -s http://localhost:3000/api/health
curl.exe -s http://localhost:9090/-/ready
```

**Result:** `session20-prometheus` (9090) and `session20-grafana` (3000) are up; Grafana health shows `"database": "ok"`, `version 12.1.1`.

![Part 3 — Grafana Stack](./SS/03-grafana-stack.png)

### Grafana data source & dashboard (provisioned)

The Prometheus data source and a dashboard were added through **Grafana provisioning**: YAML/JSON files copied into the running container's `/etc/grafana/provisioning/`, which Grafana loads at startup.

```yaml
# /etc/grafana/provisioning/datasources/prometheus.yaml
apiVersion: 1
datasources:
  - name: Prometheus
    uid: session20-prom
    type: prometheus
    access: proxy
    url: http://prometheus:9090   # Compose service name, not localhost
    isDefault: true
```

Dashboard **Session 20 — Prometheus Monitoring** (folder *Session 20*, refreshes every 5s):

| Panel                        | PromQL                                                            |
|:-----------------------------|:------------------------------------------------------------------|
| Target Up                    | `up{job="prometheus"}`                                            |
| Active Time Series           | `prometheus_tsdb_head_series`                                     |
| Uptime                       | `time() - process_start_time_seconds`                             |
| Memory (RSS)                 | `process_resident_memory_bytes`                                   |
| HTTP Request Rate by Handler | `sum by (handler) (rate(prometheus_http_requests_total[1m])) > 0` |
| Scrape Duration              | `scrape_duration_seconds`                                         |
| Samples Ingested / s         | `rate(prometheus_tsdb_head_samples_appended_total[1m])`           |
| CPU Usage                    | `rate(process_cpu_seconds_total[1m])`                             |

**Result:** Live dashboard (last 15 min, 5s refresh). Target **Up = 1**, **758** active series, uptime ~11 min, ~108 MiB RSS. The request rate is dominated by the `/metrics` scrape (~0.2 req/s), with scrape duration mostly under 5 ms and ~150 samples/s ingested.

![Part 3 — Grafana Dashboard](./SS/05-grafana-dashboard.png)

### Prometheus UI

The same PromQL from Part 2, run in the Prometheus web UI (`/query`): `up` returns `up{instance="prometheus:9090", job="prometheus"} = 1`.

![Part 3 — Prometheus UI](./SS/06-prometheus-ui.png)

> If `localhost` shows a different app in the browser, use `http://127.0.0.1:3000` / `:9090`. A WSL app is relayed on `[::1]` for the same ports.

---

## Part 4 — Argo CD Installation

```bash
cd session20-monitoring-observability-gitops\07-argocd
kubectl create namespace argocd
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=Ready pods --all -n argocd --timeout=420s
kubectl get pods -n argocd
kubectl get svc -n argocd argocd-server
```

> `--server-side` is needed because the Argo CD CRDs are too large for the `last-applied-configuration` annotation used by client-side apply.

**Result:** All 7 Argo CD components are `1/1 Running`: application-controller, applicationset-controller, dex-server, notifications-controller, redis, repo-server, and server.

![Part 4 — Argo CD Install](./SS/04-argocd-install.png)

---

## Part 5 — GitOps Flow (to complete)

Argo CD is installed and ready. The rest needs **your own Git repository**, because Argo CD watches the repo and you'll push changes to it.

```bash
# 1. Open the Argo CD UI
kubectl port-forward svc/argocd-server -n argocd 8443:443      # → https://localhost:8443 (8080 is used by a local Apache service)
# admin password:
[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String((kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}")))

# 2. Create a GitHub repo with an app/ folder (copy 07-argocd/app/deployment.yaml + service.yaml),
#    then set repoURL in 07-argocd/app/argocd-application.yaml to YOUR repo
#    (currently: https://github.com/Nency-Ravaliya/gitops-demo.git)

# 3. Register the application
kubectl apply -f app/argocd-application.yaml
kubectl get applications -n argocd
kubectl get pods -n session20                       # → SS/07-gitops-2-replicas.png

# 4. GitOps change: edit replicas 2 → 5 in YOUR repo's app/deployment.yaml
git add app/deployment.yaml
git commit -m "Scale application from 2 to 5 replicas"
git push
kubectl get pods -n session20 -w                    # → SS/08-gitops-5-replicas.png

# 5. Self-healing (selfHeal: true): manual drift is reverted
kubectl scale deployment <name> -n session20 --replicas=1
kubectl get deployment -n session20                 # back to the Git-declared count
```

Screenshot the Argo CD dashboard (*Synced / Healthy*) as `SS/09-argocd-dashboard.png`. The same flow applies to `08-mini-project/` (`session20-mini`, scaled to 3 replicas).

---

## Cleanup

```bash
cd session20-monitoring-observability-gitops\04-grafana; docker compose down
kubectl delete -f ..\07-argocd\app\argocd-application.yaml
kubectl delete namespace argocd session20
```
