# LXD Monitoring (direct via LXD API → Prometheus → Grafana)

Scrapes LXD's native `/1.0/metrics` endpoint with the existing `kube-prometheus-stack` and ships a Grafana dashboard. No custom exporter and no Go app involved.

Deployed to MicroK8s with Helm by a Gitea Actions workflow. Secrets come from OpenBao.

---

## Table of Contents

- [Summary](#summary)
- [Architecture](#architecture)
- [Repository Layout](#repository-layout)
- [Prerequisites](#prerequisites)
- [Deploy](#deploy)
- [Destroy](#destroy)
- [Verify](#verify)
- [Secrets Reference](#secrets-reference)
- [Security Notes](#security-notes)
- [Troubleshooting](#troubleshooting)
- [Roadmap](#roadmap)

---

## Summary

| Item | Value |
|---|---|
| Helm release name | `grafan-lxd-monitoring-direct` |
| Namespace | `monitoring` (same as kube-prometheus-stack) |
| LXD endpoint | `https://10.0.0.250:8443/1.0/metrics` |
| Scrape interval | 30s |
| LXD credentials | `homelab/data/lxd` in OpenBao (`client_cert`, `client_key`) |
| Kubernetes Secret | `lxd-metrics-tls` (keys `tls.crt`, `tls.key`) |
| Dashboard | Grafana, auto-imported via the `grafana_dashboard=1` sidecar label |

---

## Architecture

```
Gitea org secrets (VAULT_ADDR, VAULT_ROLE_ID, VAULT_SECRET_ID)
        │  AppRole login
        ▼
OpenBao
  ├── homelab/data/microk8s   → kubeconfig
  └── homelab/data/lxd        → client_cert / client_key
        │
        ▼
Gitea runner (deploy.yml)
  1. kubectl create secret lxd-metrics-tls -n monitoring   (cert from OpenBao)
  2. helm upgrade --install grafan-lxd-monitoring-direct
        │
        ▼
Namespace: monitoring
  ├── Secret        lxd-metrics-tls                  (created by workflow, not Helm)
  ├── ScrapeConfig  grafan-lxd-monitoring-direct     (created by Helm)
  └── ConfigMap     grafan-lxd-monitoring-direct-dashboard  (created by Helm)

Prometheus ──reads Secret──► mTLS ──► LXD /1.0/metrics
Grafana    ──sidecar picks up ConfigMap──► dashboard "LXD Resources (native API)"
```

Key points:

- **Prometheus never talks to OpenBao.** It only reads the Kubernetes Secret `lxd-metrics-tls`.
- **Only the runner talks to OpenBao**, using the Gitea org secrets. No `openbao-creds` secret is needed in `monitoring`.
- **Helm never sees the cert.** The workflow creates the Secret before Helm runs, so the cert stays out of Helm release history.

---

## Repository Layout

```
lxd-monitoring/
├── .gitea/
│   └── workflows/
│       ├── deploy.yml            # OpenBao → Secret → helm upgrade --install
│       └── destroy.yaml          # helm uninstall (+ optional Secret delete)
├── helm-chart/
│   ├── Chart.yaml
│   ├── values.yaml
│   ├── dashboards/
│   │   └── lxd-resources.json    # Grafana dashboard
│   └── templates/
│       ├── scrapeconfig.yaml
│       └── grafana-dashboard.yaml
└── README.md
```

### Helm values

| Value | Default | Purpose |
|---|---|---|
| `lxd.host` | `10.0.0.250:8443` | LXD address scraped by Prometheus |
| `lxd.hostLabel` | `lxd01` | Adds an `lxd_host` label to all series |
| `scrape.interval` | `30s` | Scrape interval |
| `tls.secretName` | `lxd-metrics-tls` | Secret holding the client cert/key (created by the workflow) |
| `dashboard.enabled` | `true` | Deploy the Grafana dashboard ConfigMap |

---

## Prerequisites

### 1. kube-prometheus-stack running in `monitoring`

Deployed by the `monitoring` repo. Its `values.yaml` must let Prometheus select ScrapeConfigs, under `prometheus.prometheusSpec`:

```yaml
scrapeConfigSelector: {}
scrapeConfigNamespaceSelector: {}
scrapeConfigSelectorNilUsesHelmValues: false
```

Check that the CRD exists:

```bash
microk8s kubectl get crd scrapeconfigs.monitoring.coreos.com
```

### 2. LXD client cert in OpenBao

The existing `homelab/data/lxd` secret (`client_cert`, `client_key`) is reused. It is already readable by the `gitea-runner` policy, so no policy change is needed.

```bash
bao kv get homelab/lxd
```

The cert must be trusted by LXD:

```bash
lxc config trust list
```

### 3. Gitea org secrets

```
VAULT_ADDR / VAULT_ROLE_ID / VAULT_SECRET_ID
```

### 4. Network

The Prometheus pod must reach `10.0.0.250:8443`.

### 5. Verify LXD metric names (once)

The dashboard queries depend on the metric names your LXD version exposes:

```bash
curl -k --cert client.crt --key client.key \
  https://10.0.0.250:8443/1.0/metrics | grep '^# TYPE'
```

Memory, disk and network figures for VMs need the `lxd-agent` running inside the guest.

---

## Deploy

**Automatic:** any push to `main` that touches `helm-chart/**` or `.gitea/workflows/deploy.yml`.

**Manual:** Gitea → Actions → "Deploy LXD monitoring (direct)" → Run workflow.

The workflow:

1. Logs in to OpenBao (AppRole).
2. Fetches the kubeconfig (`homelab/data/microk8s`) and the LXD cert (`homelab/data/lxd`).
3. Creates or updates `lxd-metrics-tls` in `monitoring` (`kubectl create ... --dry-run=client | kubectl apply`).
4. Runs `helm upgrade --install grafan-lxd-monitoring-direct ./helm-chart --namespace monitoring`.

Manual equivalent from a machine with cluster access:

```bash
microk8s kubectl create secret generic lxd-metrics-tls -n monitoring \
  --from-file=tls.crt=client.crt \
  --from-file=tls.key=client.key \
  --dry-run=client -o yaml | microk8s kubectl apply -f -

helm upgrade --install grafan-lxd-monitoring-direct ./helm-chart \
  --namespace monitoring --wait --timeout 3m
```

---

## Destroy

**Gitea → Actions → "Destroy LXD monitoring (direct)" → Run workflow**

| Input | Meaning |
|---|---|
| `confirm` | Must be the word `destroy`, otherwise the job is skipped |
| `delete_secret` | `true` (default) also deletes `lxd-metrics-tls`. Set `false` to keep it for a quick redeploy |

What it removes:

- The Helm release (ScrapeConfig and dashboard ConfigMap)
- Any leftover `grafan-lxd-monitoring-direct-dashboard` ConfigMap
- Optionally the `lxd-metrics-tls` Secret

It never touches kube-prometheus-stack, the `monitoring` namespace, or OpenBao.

Manual equivalent:

```bash
helm uninstall grafan-lxd-monitoring-direct -n monitoring --wait
microk8s kubectl delete secret lxd-metrics-tls -n monitoring --ignore-not-found
```

`helm uninstall` leaves the Secret behind because Helm did not create it. That is why the destroy workflow deletes it separately.

---

## Verify

```bash
# Objects exist
microk8s kubectl get secret lxd-metrics-tls -n monitoring
microk8s kubectl get scrapeconfig -n monitoring
microk8s kubectl get configmap -n monitoring -l grafana_dashboard=1

# Helm release
helm list -n monitoring
```

Prometheus: `http://<microk8s-node-ip>:30090/targets` → the `scrapeConfig/monitoring/grafan-lxd-monitoring-direct` target should be **UP**.

Grafana: `http://<microk8s-node-ip>:30300` → Dashboards → **LXD Resources (native API)**.

Quick PromQL checks:

```
up{lxd_host="lxd01"}
lxd_procs_total
```

---

## Secrets Reference

| Secret | Location | Created by | Used by |
|---|---|---|---|
| `VAULT_ADDR`, `VAULT_ROLE_ID`, `VAULT_SECRET_ID` | Gitea org secrets | You, manually | Deploy and destroy workflows |
| kubeconfig | OpenBao `homelab/data/microk8s` | You, manually | Workflows |
| LXD `client_cert` / `client_key` | OpenBao `homelab/data/lxd` | You, manually | Deploy workflow (copied into the Secret below) |
| `lxd-metrics-tls` | K8s Secret, `monitoring` | Deploy workflow | Prometheus (ScrapeConfig) |

**Rotation:** the Secret is refreshed only when the deploy workflow runs. After rotating the cert in OpenBao, trigger the deploy workflow manually.

---

## Security Notes

- This setup reuses the **full-access LXD client cert**. Anyone who can read Secrets in `monitoring`, or who compromises Prometheus, gets full control of LXD (create, delete, exec).
- If you want to narrow this, create a metrics-only cert and point the workflow at it:
  ```bash
  lxc config trust add metrics.crt --type=metrics --name prometheus
  bao kv put homelab/lxd-metrics client_cert=@metrics.crt client_key=@metrics.key
  ```
  Then add `path "homelab/data/lxd-metrics" { capabilities = ["read"] }` to the `gitea-runner` policy (keep all existing paths, since `policy write` overwrites) and change the fetch path in `deploy.yml`.
- The ScrapeConfig uses `insecureSkipVerify: true` because LXD's server cert is self-signed. The client cert is still verified by LXD.
- Never commit certs, keys, role IDs or secret IDs. Use placeholders in docs.

---

## Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| Target missing in Prometheus | ScrapeConfig not selected | Add the three `scrapeConfig*` selector values to the monitoring stack and redeploy it |
| Target DOWN: `tls: bad certificate` | Cert not trusted by LXD or wrong cert in Secret | `lxc config trust list`; compare with `homelab/data/lxd` |
| Target DOWN: `i/o timeout` | Network or firewall | Test from the Prometheus pod to `10.0.0.250:8443` |
| Target DOWN: secret not found | Secret missing or in another namespace | `kubectl get secret lxd-metrics-tls -n monitoring`; re-run deploy |
| Workflow: `OpenBao login failed` | Wrong or expired AppRole credentials | Check the Gitea org secrets |
| Workflow: `empty: tls.crt` | `client_cert` key missing in `homelab/data/lxd` | `bao kv get homelab/lxd` |
| Dashboard panels empty | Metric names differ in your LXD version | Compare with `/1.0/metrics` output and edit `dashboards/lxd-resources.json` |
| Dashboard not in Grafana | Sidecar not watching the namespace | Confirm label `grafana_dashboard=1` and `sidecar.dashboards.searchNamespace: ALL` |

Useful commands:

```bash
microk8s kubectl describe scrapeconfig grafan-lxd-monitoring-direct -n monitoring
microk8s kubectl logs -n monitoring -l app.kubernetes.io/name=grafana -c grafana-sc-dashboard
helm get values grafan-lxd-monitoring-direct -n monitoring
```

---

## Roadmap

- [ ] Switch to a metrics-only LXD cert
- [ ] `PrometheusRule` alerts: instance stopped, filesystem over 85%, scrape target down
- [ ] Multiple LXD hosts (extra `staticConfigs` targets)
- [ ] Automatic cert refresh (CronJob or External Secrets Operator)
