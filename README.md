# Kubernetes Monitoring with Prometheus

- Concept https://www.youtube.com/watch?v=6xmWr7p5TE0
- Concept https://www.youtube.com/watch?v=O4pKFUE8FDg

- Helm Prometheus Installation

- https://github.com/prometheus-community/helm-charts/blob/main/charts/kube-prometheus-stack/values.yaml
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts 
helm repo update 
helm repo ls
helm search repo prometheus-community
helm search repo prometheus-community/prometheus
helm search repo prometheus-community/prometheus --versions
helm search repo prometheus-community/kube-prometheus-stack
helm search repo prometheus-community/kube-prometheus-stack --versions
helm show values prometheus-community/kube-prometheus-stack --version 58.5.3

helm upgrade --install prometheus-community prometheus-community/kube-prometheus-stack \
--version 58.5.3 \
--namespace monitoring \
--create-namespace \
--timeout 30m \
--wait \
--set grafana.adminPassword="secret" \
--set kubeStateMetrics.enabled="true" \
-f grafana/persistent.yaml  \
-f alerts/alertmanager-dev.yaml  \
-f prometheus/persistent.yaml \
-f prometheus/disablings.yaml \
-f prometheus/additionalScrapeConfigsSecret.yaml
```

- blackbox-exporter installation

```bash
helm -n monitoring upgrade --install blackbox blackbox/blackbox-exporter -f blackbox/blackbox-exporter/dev-values.yaml --wait
```

- Apply Alert Rules
```bash
kubectl -n monitoring apply -f alerts/rules/
```

- Note: Enable persistance for prometheus and grafana, as I am using sandbox environment, so I disabled it.

- Slack APP Creation https://api.slack.com/apps
```bash
https://api.slack.com/apps
```

- CRD & ServiceMonitors
```bash
k get crd
k get crd | grep -i servicemonitor
k get servicemonitor -A
k get prometheusrule -A
```

- BlackBox DashBoard https://grafana.com/grafana/dashboards/7587-prometheus-blackbox-exporter
- Import 7587
