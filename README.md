# Kubernetes Monitoring with Prometheus

- Concept https://www.youtube.com/watch?v=6xmWr7p5TE0
- Concept https://www.youtube.com/watch?v=O4pKFUE8FDg
- Concept https://www.youtube.com/watch?v=HOmdYtsB950
- Concept https://www.youtube.com/watch?v=dMca4jHaft8

- Helm Prometheus Installation
- https://artifacthub.io/packages/helm/prometheus-community/kube-prometheus-stack

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

- ArgoCD Installation with Helm
```bash
helm upgrade --install argocd argo/argo-cd -n argocd --create-namespace \
  --version 5.46.8 \
  --set server.serviceMonitor.enabled=true \
  --set controller.metrics.enabled=true \
  --set controller.metrics.serviceMonitor.enabled=true \
  --set repoServer.metrics.enabled=true \
  --set repoServer.metrics.serviceMonitor.enabled=true \
  --set applicationController.metrics.enabled=true \
  --set applicationController.metrics.serviceMonitor.enabled=true \
  --set notifications.enabled=true \
  --set notifications.secret.create=false \
  --set notifications.cm.create=false \
  --wait
```

- ArgoCD Grafana Dashboards
- https://grafana.com/grafana/dashboards/14584-argocd/
- https://grafana.com/grafana/dashboards/19974-argocd-application-overview/

- PodMonitor for ArgoCD
```bash
k get crds
k -n monitoring get prometheus.monitoring.coreos.com -o yaml | grep -iC5 servicemonitor
```
- PodMonitor
```bash
k -n argocd apply -f PodMonitor.yaml
```

- PodMonitor.yaml
- https://argo-cd.readthedocs.io/en/stable/operator-manual/metrics/
  
```yaml
---
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: argocd-server
  labels:
    release: prometheus-community # This is the helm chart NAME helm ls -A
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-server
  namespaceSelector:
    any: true
  podMetricsEndpoints:
  - port: metrics
---
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: argocd-repo-server
  labels:
    release: prometheus-community
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-repo-server
  namespaceSelector:
    any: true
  podMetricsEndpoints:
  - port: metrics
---
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: argocd-applicationset-controller
  labels:
    release: prometheus-community
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-applicationset-controller
  namespaceSelector:
    any: true
  podMetricsEndpoints:
  - port: metrics
---
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: argocd-notifications-controller
  labels:
    release: prometheus-community
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-notifications-controller
  namespaceSelector:
    any: true
  podMetricsEndpoints:
  - port: metrics
```

```bash
k get podmonitor -n argocd
```

- ArgoCD ServiceMonitor

```bash
k -n argocd apply -f ServiceMonitor.yaml
```
- ServiceMonitor.yaml
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: argocd-application-controller-metrics
  labels:
    release: prometheus-community
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-metrics
  endpoints:
  - port: http-metrics
---
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: argocd-repo-server-metrics
  labels:
    release: prometheus-community
spec:
  selector:
    matchLabels:
      app.kubernetes.io/name: argocd-repo-server-metrics
  endpoints:
  - port: http-metrics
```

```bash
k get servicemonitor -n argocd
```

- ArgoCD Notifications
- Bitnami Helm Chart Repo https://charts.bitnami.com/bitnami

- Slack1
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: argocd-notifications-secret # Dont change the name, notification service by default look for this secret. 
  namespace: argocd
stringData:
  slack-token: "SLACK TOKEN"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  service.slack: |
    token: $slack-token 
  defaultTriggers: |   # https://argo-cd.readthedocs.io/en/stable/operator-manual/notifications/triggers/#default-triggers 
    - on-deployed
  trigger.on-deployed: |  # https://argo-cd.readthedocs.io/en/stable/operator-manual/notifications/triggers/ 
    - description: Application is synced and healthy. Triggered once per commit.
      oncePer: app.status.sync.revision
      send:
      - app-deployed
      when: app.status.sync.status in ['Synced'] and app.status.health.status == 'Healthy'
  template.app-deployed: | # https://argo-cd.readthedocs.io/en/stable/operator-manual/notifications/templates/
    message: |
      {{if eq .serviceType "slack"}}:white_check_mark:{{end}} Application {{.app.metadata.name}} is now running new version of deployments manifests.
    slack:
      attachments: |
        [{
          "title": "{{ .app.metadata.name}}",
          "title_link":"{{.context.argocdUrl}}/applications/{{.app.metadata.name}}",
          "color": "#18be52",
          "fields": [
          {
            "title": "Sync Status",
            "value": "{{.app.status.sync.status}}",
            "short": true
          },
          {
            "title": "Repository",
            "value": "{{.app.spec.source.repoURL}}",
            "short": true
          },
          {
            "title": "Revision",
            "value": "{{.app.status.sync.revision}}",
            "short": true
          }
          {{range $index, $c := .app.status.conditions}}
          {{if not $index}},{{end}}
          {{if $index}},{{end}}
          {
            "title": "{{$c.type}}",
            "value": "{{$c.message}}",
            "short": true
          }
          {{end}}
          ]
        }]
```

- Slack2
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: argocd-notifications-secret # Dont change the name, notification service by default look for this secret. 
  namespace: argocd
stringData:
  slack-token: "YOUR SLACK TOKEN"
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-notifications-cm
  namespace: argocd
data:
  service.slack: |
    token: $slack-token
  defaultTriggers: |   
    - on-deployed
    - on-sync-failed
    - on-sync-succeeded
    - on-health-healthy
    - on-health-degraded
  trigger.on-deployed: |  
    - description: Application is synced and healthy. Triggered once per commit.
      oncePer: app.status.sync.revision
      send:
      - app-deployed
      when: app.status.sync.status in ['Synced'] and app.status.health.status == 'Healthy'
  trigger.on-sync-failed: |
    - description: Application sync operation failed.
      send:
      - app-sync-failed
      when: app.status.operationState.phase in ['Failed']
  trigger.on-sync-succeeded: |
    - description: Application sync operation succeeded.
      send:
      - app-sync-succeeded
      when: app.status.operationState.phase in ['Succeeded']
  trigger.on-health-healthy: |
    - description: Application has become healthy.
      send:
      - app-health-healthy
      when: app.status.health.status == 'Healthy'
  trigger.on-health-degraded: |
    - description: Application health has degraded.
      send:
      - app-health-degraded
      when: app.status.health.status == 'Degraded'
  template.app-deployed: |
    message: |
      {{if eq .serviceType "slack"}}:white_check_mark:{{end}} Application {{.app.metadata.name}} is now running new version of deployments manifests.
    slack:
      attachments: |
        [{
          "title": "{{ .app.metadata.name}}",
          "title_link":"{{.context.argocdUrl}}/applications/{{.app.metadata.name}}",
          "color": "#18be52",
          "fields": [
          {
            "title": "Sync Status",
            "value": "{{.app.status.sync.status}}",
            "short": true
          },
          {
            "title": "Repository",
            "value": "{{.app.spec.source.repoURL}}",
            "short": true
          },
          {
            "title": "Revision",
            "value": "{{.app.status.sync.revision}}",
            "short": true
          }
          {{range $index, $c := .app.status.conditions}}
          {{if not $index}},{{end}}
          {{if $index}},{{end}}
          {
            "title": "{{$c.type}}",
            "value": "{{$c.message}}",
            "short": true
          }
          {{end}}
          ]
        }]
  template.app-sync-failed: |
    message: |
      Application {{.app.metadata.name}} sync operation failed.
    slack:
      attachments: |
        [{
          "title": "{{ .app.metadata.name}}",
          "title_link":"{{.context.argocdUrl}}/applications/{{.app.metadata.name}}",
          "color": "#ff0000",
          "fields": [
          {
            "title": "Sync Status",
            "value": "{{.app.status.sync.status}}",
            "short": true
          },
          {
            "title": "Repository",
            "value": "{{.app.spec.source.repoURL}}",
            "short": true
          },
          {
            "title": "Revision",
            "value": "{{.app.status.sync.revision}}",
            "short": true
          }
          ]
        }]
  template.app-sync-succeeded: |
    message: |
      Application {{.app.metadata.name}} sync operation succeeded.
    slack:
      attachments: |
        [{
          "title": "{{ .app.metadata.name}}",
          "title_link":"{{.context.argocdUrl}}/applications/{{.app.metadata.name}}",
          "color": "#18be52",
          "fields": [
          {
            "title": "Sync Status",
            "value": "{{.app.status.sync.status}}",
            "short": true
          },
          {
            "title": "Repository",
            "value": "{{.app.spec.source.repoURL}}",
            "short": true
          },
          {
            "title": "Revision",
            "value": "{{.app.status.sync.revision}}",
            "short": true
          }
          ]
        }]
  template.app-health-healthy: |
    message: |
      Application {{.app.metadata.name}} is healthy.
    slack:
      attachments: |
        [{
          "title": "{{ .app.metadata.name}}",
          "title_link":"{{.context.argocdUrl}}/applications/{{.app.metadata.name}}",
          "color": "#18be52",
          "fields": [
          {
            "title": "Health Status",
            "value": "{{.app.status.health.status}}",
            "short": true
          },
          {
            "title": "Repository",
            "value": "{{.app.spec.source.repoURL}}",
            "short": true
          },
          {
            "title": "Revision",
            "value": "{{.app.status.sync.revision}}",
            "short": true
          }
          ]
        }]
  template.app-health-degraded: |
    message: |
      Application {{.app.metadata.name}} health status is degraded.
    slack:
      attachments: |
        [{
          "title": "{{ .app.metadata.name}}",
          "title_link":"{{.context.argocdUrl}}/applications/{{.app.metadata.name}}",
          "color": "#ff0000",
          "fields": [
          {
            "title": "Health Status",
            "value": "{{.app.status.health.status}}",
            "short": true
          },
          {
            "title": "Repository",
            "value": "{{.app.spec.source.repoURL}}",
            "short": true
          },
          {
            "title": "Revision",
            "value": "{{.app.status.sync.revision}}",
            "short": true
          }
          ]
        }]
```

- ArgoCD Sample App Bitnami nginx Yaml
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-app
  namespace: argocd
  annotations:
    notifications.argoproj.io/subscribe.on-deployed.slack: notifications
    notifications.argoproj.io/subscribe.on-sync-failed.slack: notifications
    notifications.argoproj.io/subscribe.on-sync-succeeded.slack: notifications
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  destination:
    name: 'in-cluster'
    namespace: 'default'
  source:
    repoURL: 'https://charts.bitnami.com/bitnami'
    chart: 'nginx'
    targetRevision: '13.2.31'  # Specify the desired version of the chart
    helm:
      values: |  # Inline values, you can customize these as per your requirements
        service:
          type: ClusterIP
        replicaCount: 2
  project: 'default'
  syncPolicy:
    #automated:
    #  prune: false
    #  selfHeal: false
    syncOptions:
      - CreateNamespace=false
```

- Note: Always apply like below
```bash
k -n argocd apply -f notification.yaml
```

- Argocd debug notification logs
```bash
kubectl logs -n argocd deployment/argocd-notifications-controller -f
```
