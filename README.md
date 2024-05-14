# Kubernetes Monitoring with Prometheus

- Helm Prometheus Installation

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts 
helm repo update 
helm repo ls
helm search repo prometheus-community
helm search repo prometheus-community/prometheus
helm search repo prometheus-community/prometheus --versions
helm search repo prometheus-community/kube-prometheus-stack
helm search repo prometheus-community/kube-prometheus-stack --versions
helm show values prometheus-community/kube-prometheus-stack --version 58.5.1

helm upgrade --install prometheus-community prometheus-community/kube-prometheus-stack \
--version 58.5.1 \
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
