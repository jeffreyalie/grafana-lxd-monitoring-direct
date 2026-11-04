### Grafana monitoring lxd resources direct
1. Reusing homelab/data/lxd

It works. An unrestricted trusted client cert can read /1.0/metrics, so no new cert, no lxc config trust add --type=metrics, and no policy change. The runner policy already allows homelab/data/lxd, and the keys are the same client_cert and client_key your Go app reads.

The trade-off is that this cert has full control of LXD (create, delete, exec), not just metrics. Copying it into a Secret in monitoring means anyone who can read secrets in that namespace, or who compromises Prometheus, gets full LXD access. The dedicated metrics-only cert avoids that. For a homelab it’s an acceptable choice, as long as you know that’s what you’re accepting.

2. Helm version

The workflow creates the Secret first, then Helm deploys only the ScrapeConfig and dashboard. Helm never sees the cert, so it stays out of release history.

helm-chart/
├── Chart.yaml
├── values.yaml
├── dashboards/lxd-resources.json     # same JSON as in my first reply
└── templates/
    ├── scrapeconfig.yaml
    └── grafana-dashboard.yaml

Chart.yaml

yaml
apiVersion: v2
name: lxd-monitoring-direct
description: Scrapes LXD native metrics into kube-prometheus-stack
type: application
version: 0.1.0
appVersion: "1.0.0"

values.yaml

yaml
lxd:
  host: "10.0.0.250:8443"
  hostLabel: lxd01

scrape:
  interval: 30s

tls:
  secretName: lxd-metrics-tls   # created by the workflow, not by Helm

dashboard:
  enabled: true

templates/scrapeconfig.yaml

yaml
apiVersion: monitoring.coreos.com/v1alpha1
kind: ScrapeConfig
metadata:
  name: {{ .Release.Name }}
  namespace: {{ .Release.Namespace }}
  labels:
    release: kube-prometheus-stack
spec:
  scheme: HTTPS
  metricsPath: /1.0/metrics
  scrapeInterval: {{ .Values.scrape.interval }}
  staticConfigs:
    - targets: [{{ .Values.lxd.host | quote }}]
      labels:
        lxd_host: {{ .Values.lxd.hostLabel }}
  tlsConfig:
    insecureSkipVerify: true
    cert:
      secret:
        name: {{ .Values.tls.secretName }}
        key: tls.crt
    keySecret:
      name: {{ .Values.tls.secretName }}
      key: tls.key

templates/grafana-dashboard.yaml

yaml
{{- if .Values.dashboard.enabled }}
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ .Release.Name }}-dashboard
  namespace: {{ .Release.Namespace }}
  labels:
    grafana_dashboard: "1"
data:
  lxd-resources.json: |-
{{ .Files.Get "dashboards/lxd-resources.json" | indent 4 }}
{{- end }}

.gitea/workflows/deploy.yml

yaml
name: Deploy LXD monitoring (direct)
on:
  push:
    branches: [main]
    paths: ['helm-chart/**', '.gitea/workflows/deploy.yml']
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest
    env:
      VAULT_ADDR:      ${{ secrets.VAULT_ADDR }}
      VAULT_ROLE_ID:   ${{ secrets.VAULT_ROLE_ID }}
      VAULT_SECRET_ID: ${{ secrets.VAULT_SECRET_ID }}
    steps:
      - uses: actions/checkout@v4
      - uses: azure/setup-helm@v4
        with: { version: 'v3.16.4' }
      - uses: azure/setup-kubectl@v4

      - name: Login to OpenBao
        run: |
          T=$(curl -sk -X POST "${VAULT_ADDR}/v1/auth/approle/login" \
            -d "{\"role_id\":\"${VAULT_ROLE_ID}\",\"secret_id\":\"${VAULT_SECRET_ID}\"}" \
            | jq -r '.auth.client_token')
          echo "::add-mask::$T"
          echo "VAULT_TOKEN=$T" >> $GITHUB_ENV

      - name: Fetch kubeconfig and LXD cert
        run: |
          mkdir -p ~/.kube /tmp/lxd
          curl -sk -H "X-Vault-Token: ${VAULT_TOKEN}" \
            "${VAULT_ADDR}/v1/homelab/data/microk8s" \
            | jq -r '.data.data.kubeconfig' > ~/.kube/config
          chmod 600 ~/.kube/config
          curl -sk -H "X-Vault-Token: ${VAULT_TOKEN}" \
            "${VAULT_ADDR}/v1/homelab/data/lxd" > /tmp/lxd/raw.json
          jq -r '.data.data.client_cert' /tmp/lxd/raw.json > /tmp/lxd/tls.crt
          jq -r '.data.data.client_key'  /tmp/lxd/raw.json > /tmp/lxd/tls.key
          for f in ~/.kube/config /tmp/lxd/tls.crt /tmp/lxd/tls.key; do
            [ -s "$f" ] || { echo "empty: $f"; exit 1; }
          done

      - name: Create/update TLS secret
        run: |
          kubectl create secret generic lxd-metrics-tls -n monitoring \
            --from-file=tls.crt=/tmp/lxd/tls.crt \
            --from-file=tls.key=/tmp/lxd/tls.key \
            --dry-run=client -o yaml | kubectl apply -f -

      - name: Helm deploy
        run: |
          helm upgrade --install grafan-lxd-monitoring-direct ./helm-chart \
            --namespace monitoring --wait --timeout 3m

      - name: Show result
        if: always()
        run: kubectl get secret lxd-metrics-tls,scrapeconfig -n monitoring

      - name: Cleanup
        if: always()
        run: rm -rf /tmp/lxd ~/.kube/config
Notes
No openbao-creds in monitoring. Only the runner talks to OpenBao here, using the Gitea org secrets.
Rotation: the Secret refreshes only when the workflow runs. After rotating the cert in OpenBao, trigger it manually.
Prometheus selectors: you still need scrapeConfigSelector: {}, scrapeConfigNamespaceSelector: {} and scrapeConfigSelectorNilUsesHelmValues: false in the monitoring stack’s values.yaml.
Uninstall: helm uninstall leaves lxd-metrics-tls behind, since Helm didn’t create it. Delete it manually if needed.
```
In your monitoring repo’s values.yaml, add this under prometheusSpec so Prometheus picks up ScrapeConfigs (you only have serviceMonitorSelector: {} today):

yaml
    scrapeConfigSelector: {}
    scrapeConfigNamespaceSelector: {}

    scrapeConfigSelectorNilUsesHelmValues: false
```

Metric names: verify them first with curl -k --cert ... --key ... https://10.0.0.250:8443/1.0/metrics | grep '^# TYPE', because the dashboard queries depend on them.