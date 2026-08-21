# 📊 CloudForge — Week 3: Full Observability Stack (LGTM)

**What We're Building This Week**

```
Week 3 Deliverables:
├── Prometheus (metrics collection + alerting rules)
├── Grafana (dashboards — cluster, app, business metrics)
├── Loki (log aggregation + LogQL queries)
├── Tempo (distributed tracing)
├── OpenTelemetry Collector (vendor-neutral instrumentation)
├── Alertmanager (alert routing → Slack + PagerDuty)
├── Grafana OnCall (on-call scheduling)
├── SLO monitoring (error budget tracking)
└── Custom dashboards (EKS, ArgoCD, app-level)

By end of week:
- Every metric, log, trace in one Grafana UI
- Correlated: click a trace → see logs → see metrics
- Alerts firing to Slack with runbook links
- SLO dashboard showing error budget burn rate
```

## PART 1 — Prometheus Stack

**kube-prometheus-stack Values**

```
mkdir -p kubernetes/platform/monitoring

cat > kubernetes/platform/monitoring/prometheus-values.yaml << 'EOF'
# ═══════════════════════════════════════════════════════
# kube-prometheus-stack Production Values
# Includes: Prometheus, Grafana, Alertmanager, node-exporter
# ═══════════════════════════════════════════════════════

nameOverride: ""
fullnameOverride: ""

# ── Global settings ───────────────────────────────────────
global:
  rbac:
    create: true
    pspEnabled: false

# ── Prometheus Operator ───────────────────────────────────
prometheusOperator:
  enabled: true
  replicas: 2

  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi

  # Watch all namespaces
  namespaces: {}

  admissionWebhooks:
    enabled: true
    patch:
      enabled: true

# ── Prometheus ────────────────────────────────────────────
prometheus:
  enabled: true

  serviceAccount:
    annotations:
      eks.amazonaws.com/role-arn: PROMETHEUS_ROLE_ARN

  prometheusSpec:
    replicas: 2
    replicaExternalLabelName: prometheus_replica
    prometheusExternalLabelName: prometheus_cluster

    # Retention
    retention: 30d
    retentionSize: "45GB"

    # Storage
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3
          accessModes: [ReadWriteOnce]
          resources:
            requests:
              storage: 50Gi

    # Resources
    resources:
      requests:
        cpu: 500m
        memory: 2Gi
      limits:
        cpu: 2000m
        memory: 8Gi

    # Scrape all ServiceMonitors/PodMonitors in all namespaces
    serviceMonitorSelectorNilUsesHelmValues: false
    podMonitorSelectorNilUsesHelmValues: false
    ruleSelectorNilUsesHelmValues: false
    probeSelectorNilUsesHelmValues: false

    # Global scrape config
    scrapeInterval: 30s
    scrapeTimeout: 10s
    evaluationInterval: 30s

    # Remote write to Grafana Cloud (optional — for long-term storage)
    # remoteWrite:
    #   - url: https://prometheus-prod-01-eu-west-0.grafana.net/api/prom/push
    #     basicAuth:
    #       username:
    #         name: grafana-cloud-secret
    #         key: username
    #       password:
    #         name: grafana-cloud-secret
    #         key: password

    # Additional scrape configs
    additionalScrapeConfigs:
      # Scrape Karpenter metrics
      - job_name: karpenter
        kubernetes_sd_configs:
          - role: pod
            namespaces:
              names: [karpenter]
        relabel_configs:
          - source_labels: [__meta_kubernetes_pod_label_app_kubernetes_io_name]
            action: keep
            regex: karpenter

      # Scrape ArgoCD
      - job_name: argocd
        kubernetes_sd_configs:
          - role: service
            namespaces:
              names: [argocd]
        relabel_configs:
          - source_labels: [__meta_kubernetes_service_label_app_kubernetes_io_name]
            action: keep
            regex: argocd-.*

    # Pod affinity — spread replicas
    affinity:
      podAntiAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app.kubernetes.io/name: prometheus
            topologyKey: kubernetes.io/hostname

  # Ingress for Prometheus UI (internal only)
  ingress:
    enabled: true
    ingressClassName: alb
    annotations:
      alb.ingress.kubernetes.io/scheme: internal
      alb.ingress.kubernetes.io/target-type: ip
    hosts:
      - prometheus.internal.yourdomain.com

# ── Alertmanager ──────────────────────────────────────────
alertmanager:
  enabled: true

  alertmanagerSpec:
    replicas: 2
    retention: 120h

    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: gp3
          resources:
            requests:
              storage: 10Gi

    resources:
      requests:
        cpu: 50m
        memory: 64Mi
      limits:
        cpu: 200m
        memory: 256Mi

    affinity:
      podAntiAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app.kubernetes.io/name: alertmanager
            topologyKey: kubernetes.io/hostname

  # Alert routing configuration
  config:
    global:
      resolve_timeout: 5m
      slack_api_url: SLACK_WEBHOOK_URL
      pagerduty_url: https://events.pagerduty.com/v2/enqueue

    # Alert inhibition — don't fire child alerts if parent is firing
    inhibit_rules:
      - source_matchers:
          - severity="critical"
        target_matchers:
          - severity="warning"
        equal: [alertname, cluster, namespace]
      - source_matchers:
          - alertname="KubernetesNodeNotReady"
        target_matchers:
          - alertname="KubePodNotReady"
        equal: [node]

    # Routing tree
    route:
      group_by: [alertname, cluster, namespace, severity]
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 12h
      receiver: slack-platform

      routes:
        # P1 — Critical → PagerDuty immediately
        - matchers:
            - severity="critical"
          receiver: pagerduty-critical
          group_wait: 0s
          repeat_interval: 1h
          continue: true   # also send to Slack

        # P1 also goes to Slack
        - matchers:
            - severity="critical"
          receiver: slack-critical

        # P2 — Warning → Slack only
        - matchers:
            - severity="warning"
          receiver: slack-warning

        # Watchdog — heartbeat alert (proves alerting works)
        - matchers:
            - alertname="Watchdog"
          receiver: deadmanssnitch
          group_wait: 0s
          repeat_interval: 1m

    receivers:
      - name: slack-platform
        slack_configs:
          - channel: '#platform-alerts'
            send_resolved: true
            title: |-
              [{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] {{ .CommonLabels.alertname }}
            text: |-
              {{ range .Alerts }}
              *Alert:* {{ .Annotations.summary }}
              *Description:* {{ .Annotations.description }}
              *Severity:* {{ .Labels.severity }}
              *Namespace:* {{ .Labels.namespace }}
              *Runbook:* {{ .Annotations.runbook_url }}
              {{ end }}
            actions:
              - type: button
                text: View in Grafana
                url: '{{ .CommonAnnotations.grafana_url }}'
              - type: button
                text: View Runbook
                url: '{{ .CommonAnnotations.runbook_url }}'

      - name: slack-critical
        slack_configs:
          - channel: '#incidents'
            send_resolved: true
            title: '🚨 CRITICAL: {{ .CommonLabels.alertname }}'
            text: |-
              {{ range .Alerts }}
              *{{ .Annotations.summary }}*
              {{ .Annotations.description }}
              *Runbook:* {{ .Annotations.runbook_url }}
              {{ end }}

      - name: slack-warning
        slack_configs:
          - channel: '#platform-warnings'
            send_resolved: true
            title: '⚠️ WARNING: {{ .CommonLabels.alertname }}'

      - name: pagerduty-critical
        pagerduty_configs:
          - routing_key: PAGERDUTY_INTEGRATION_KEY
            severity: critical
            description: '{{ .CommonLabels.alertname }}: {{ .CommonAnnotations.summary }}'
            details:
              namespace: '{{ .CommonLabels.namespace }}'
              cluster: '{{ .CommonLabels.cluster }}'
              runbook: '{{ .CommonAnnotations.runbook_url }}'

      - name: deadmanssnitch
        webhook_configs:
          - url: DEADMANSSNITCH_URL
            send_resolved: false

# ── Grafana ───────────────────────────────────────────────
grafana:
  enabled: true
  replicas: 2

  adminPassword: GRAFANA_ADMIN_PASSWORD

  persistence:
    enabled: true
    storageClassName: gp3
    size: 10Gi

  resources:
    requests:
      cpu: 100m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 1Gi

  serviceAccount:
    annotations:
      eks.amazonaws.com/role-arn: GRAFANA_ROLE_ARN

  # Datasources — auto-configured
  datasources:
    datasources.yaml:
      apiVersion: 1
      datasources:
        - name: Prometheus
          type: prometheus
          url: http://kube-prometheus-stack-prometheus:9090
          isDefault: true
          jsonData:
            timeInterval: 30s
            exemplarTraceIdDestinations:
              - name: traceID
                datasourceUid: tempo

        - name: Loki
          type: loki
          url: http://loki-gateway.logging.svc.cluster.local
          jsonData:
            derivedFields:
              - datasourceUid: tempo
                matcherRegex: '"traceId":"(\w+)"'
                name: TraceID
                url: '$${__value.raw}'

        - name: Tempo
          type: tempo
          url: http://tempo.tracing.svc.cluster.local:3100
          uid: tempo
          jsonData:
            tracesToLogsV2:
              datasourceUid: loki
              spanStartTimeShift: '-1h'
              spanEndTimeShift: '1h'
              filterByTraceID: true
              filterBySpanID: false
            tracesToMetrics:
              datasourceUid: prometheus
            serviceMap:
              datasourceUid: prometheus
            search:
              hide: false
            nodeGraph:
              enabled: true

        - name: CloudWatch
          type: cloudwatch
          jsonData:
            authType: default
            defaultRegion: ap-south-1

  # Dashboard providers
  dashboardProviders:
    dashboardproviders.yaml:
      apiVersion: 1
      providers:
        - name: platform
          orgId: 1
          folder: Platform
          type: file
          disableDeletion: false
          editable: true
          options:
            path: /var/lib/grafana/dashboards/platform

        - name: applications
          orgId: 1
          folder: Applications
          type: file
          disableDeletion: false
          editable: true
          options:
            path: /var/lib/grafana/dashboards/applications

        - name: slos
          orgId: 1
          folder: SLOs
          type: file
          disableDeletion: false
          editable: true
          options:
            path: /var/lib/grafana/dashboards/slos

  # Pre-built dashboards from Grafana.com
  dashboards:
    platform:
      kubernetes-cluster:
        gnetId: 15661
        revision: 1
        datasource: Prometheus
      kubernetes-nodes:
        gnetId: 1860
        revision: 37
        datasource: Prometheus
      karpenter:
        gnetId: 16237
        revision: 1
        datasource: Prometheus
      argocd:
        gnetId: 14584
        revision: 1
        datasource: Prometheus
      cert-manager:
        gnetId: 11001
        revision: 1
        datasource: Prometheus

  # Grafana plugins
  plugins:
    - grafana-piechart-panel
    - grafana-clock-panel
    - grafana-worldmap-panel
    - marcusolsson-json-datasource

  # Ingress
  ingress:
    enabled: true
    ingressClassName: alb
    annotations:
      alb.ingress.kubernetes.io/scheme: internet-facing
      alb.ingress.kubernetes.io/target-type: ip
      alb.ingress.kubernetes.io/listen-ports: '[{"HTTPS":443}]'
      alb.ingress.kubernetes.io/certificate-arn: ACM_CERT_ARN
    hosts:
      - grafana.yourdomain.com
    tls:
      - hosts:
          - grafana.yourdomain.com

  # SMTP for alerts (optional)
  smtp:
    enabled: false

  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        - labelSelector:
            matchLabels:
              app.kubernetes.io/name: grafana
          topologyKey: kubernetes.io/hostname

# ── Node Exporter ─────────────────────────────────────────
nodeExporter:
  enabled: true
  resources:
    requests:
      cpu: 50m
      memory: 64Mi
    limits:
      cpu: 200m
      memory: 128Mi

# ── kube-state-metrics ────────────────────────────────────
kubeStateMetrics:
  enabled: true

kube-state-metrics:
  resources:
    requests:
      cpu: 50m
      memory: 64Mi
    limits:
      cpu: 200m
      memory: 256Mi

# ── Default PrometheusRules (built-in alerts) ─────────────
defaultRules:
  create: true
  rules:
    alertmanager: true
    etcd: false          # managed EKS — no etcd access
    configReloaders: true
    general: true
    k8sContainerMemoryCache: true
    k8sContainerMemoryRss: true
    k8sContainerMemorySwap: true
    k8sContainerResource: true
    k8sContainerMemoryWorkingSetBytes: true
    k8sPodOwner: true
    kubeApiserverAvailability: true
    kubeApiserverBurnrate: true
    kubeApiserverHistogram: true
    kubeApiserverSlos: true
    kubeControllerManager: false  # managed EKS
    kubelet: true
    kubeProxy: false              # managed EKS
    kubePrometheusGeneral: true
    kubePrometheusNodeRecording: true
    kubernetesApps: true
    kubernetesResources: true
    kubernetesStorage: true
    kubernetesSystem: true
    kubeSchedulerAlerting: false  # managed EKS
    kubeSchedulerRecording: false
    kubeStateMetrics: true
    network: true
    node: true
    nodeExporterAlerting: true
    nodeExporterRecording: true
    prometheus: true
    prometheusOperator: true
    windows: false
EOF
```

**Deploy Prometheus Stack via ArgoCD**

```
cat > kubernetes/applications/monitoring.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: kube-prometheus-stack
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: platform
  source:
    repoURL: https://prometheus-community.github.io/helm-charts
    chart: kube-prometheus-stack
    targetRevision: "61.7.2"
    helm:
      valueFiles:
        - $values/kubernetes/platform/monitoring/prometheus-values.yaml
  sources:
    - repoURL: https://prometheus-community.github.io/helm-charts
      chart: kube-prometheus-stack
      targetRevision: "61.7.2"
    - repoURL: https://github.com/YOUR_USERNAME/cloudforge
      targetRevision: main
      ref: values
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true   # needed for large CRDs
      - Replace=true
EOF

kubectl apply -f kubernetes/applications/monitoring.yaml

# Watch it come up
kubectl get pods -n monitoring -w
```

## PART 2 — Custom Alert Rules

```
cat > kubernetes/platform/monitoring/alert-rules.yaml << 'EOF'
# ═══════════════════════════════════════════════════════
# CloudForge Custom Alert Rules
# Business + Platform + Application alerts
# ═══════════════════════════════════════════════════════
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: cloudforge-alerts
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  groups:

    # ── Platform SLO Alerts ────────────────────────────
    - name: platform.slos
      interval: 30s
      rules:
        # API availability SLO — 99.9% target
        - alert: APIAvailabilitySLOBreach
          expr: |
            (
              sum(rate(http_requests_total{
                job="api-service",
                status!~"5.."
              }[5m]))
              /
              sum(rate(http_requests_total{
                job="api-service"
              }[5m]))
            ) < 0.999
          for: 5m
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "API availability SLO breached"
            description: >-
              API success rate is {{ $value | humanizePercentage }},
              below 99.9% SLO target.
            runbook_url: "https://github.com/YOUR_USERNAME/cloudforge/blob/main/docs/runbooks/api-availability.md"
            grafana_url: "https://grafana.yourdomain.com/d/api-slo"

        # Latency SLO — p99 < 500ms
        - alert: APILatencySLOBreach
          expr: |
            histogram_quantile(0.99,
              sum(rate(http_request_duration_seconds_bucket{
                job="api-service"
              }[5m])) by (le)
            ) > 0.5
          for: 5m
          labels:
            severity: warning
            team: platform
          annotations:
            summary: "API p99 latency SLO breached"
            description: >-
              API p99 latency is {{ $value | humanizeDuration }},
              above 500ms SLO target.
            runbook_url: "https://github.com/YOUR_USERNAME/cloudforge/blob/main/docs/runbooks/api-latency.md"

        # Error budget burn rate — fast burn (1h window)
        - alert: ErrorBudgetFastBurn
          expr: |
            (
              sum(rate(http_requests_total{
                job="api-service",
                status=~"5.."
              }[1h]))
              /
              sum(rate(http_requests_total{
                job="api-service"
              }[1h]))
            ) > (14.4 * 0.001)
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "Error budget burning too fast (1h window)"
            description: >-
              Error rate {{ $value | humanizePercentage }} is burning
              the monthly error budget 14.4x faster than sustainable.
            runbook_url: "https://github.com/YOUR_USERNAME/cloudforge/blob/main/docs/runbooks/error-budget.md"

        # Error budget burn rate — slow burn (6h window)
        - alert: ErrorBudgetSlowBurn
          expr: |
            (
              sum(rate(http_requests_total{
                job="api-service",
                status=~"5.."
              }[6h]))
              /
              sum(rate(http_requests_total{
                job="api-service"
              }[6h]))
            ) > (6 * 0.001)
          labels:
            severity: warning
            team: platform
          annotations:
            summary: "Error budget burning too fast (6h window)"
            description: >-
              Sustained error rate {{ $value | humanizePercentage }}
              will exhaust monthly error budget within 5 days.

    # ── Kubernetes Platform Alerts ─────────────────────
    - name: platform.kubernetes
      rules:
        # Pod crash loop
        - alert: PodCrashLooping
          expr: |
            increase(kube_pod_container_status_restarts_total[15m]) > 3
          labels:
            severity: warning
            team: platform
          annotations:
            summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} is crash looping"
            description: >-
              Container {{ $labels.container }} has restarted
              {{ $value }} times in the last 15 minutes.
            runbook_url: "https://github.com/YOUR_USERNAME/cloudforge/blob/main/docs/runbooks/pod-crashloop.md"

        # Pod OOMKilled
        - alert: PodOOMKilled
          expr: |
            kube_pod_container_status_last_terminated_reason{reason="OOMKilled"} == 1
          labels:
            severity: warning
            team: platform
          annotations:
            summary: "Pod {{ $labels.namespace }}/{{ $labels.pod }} OOMKilled"
            description: >-
              Container {{ $labels.container }} was killed due to
              out-of-memory. Consider increasing memory limits.
            runbook_url: "https://github.com/YOUR_USERNAME/cloudforge/blob/main/docs/runbooks/oom-killed.md"

        # Persistent volume nearly full
        - alert: PersistentVolumeFillingUp
          expr: |
            (
              kubelet_volume_stats_available_bytes
              /
              kubelet_volume_stats_capacity_bytes
            ) < 0.15
          for: 5m
          labels:
            severity: warning
            team: platform
          annotations:
            summary: "PV {{ $labels.namespace }}/{{ $labels.persistentvolumeclaim }} is {{ $value | humanizePercentage }} full"
            description: "Less than 15% disk space remaining."
            runbook_url: "https://github.com/YOUR_USERNAME/cloudforge/blob/main/docs/runbooks/pv-full.md"

        # Deployment not progressing
        - alert: DeploymentNotProgressing
          expr: |
            kube_deployment_status_condition{
              condition="Progressing",
              status="false"
            } == 1
          for: 15m
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "Deployment {{ $labels.namespace }}/{{ $labels.deployment }} not progressing"
            description: "Deployment has been stuck for 15 minutes."
            runbook_url: "https://github.com/YOUR_USERNAME/cloudforge/blob/main/docs/runbooks/deployment-stuck.md"

        # HPA at max replicas
        - alert: HPAAtMaxReplicas
          expr: |
            kube_horizontalpodautoscaler_status_current_replicas
            ==
            kube_horizontalpodautoscaler_spec_max_replicas
          for: 15m
          labels:
            severity: warning
            team: platform
          annotations:
            summary: "HPA {{ $labels.namespace }}/{{ $labels.horizontalpodautoscaler }} at max replicas"
            description: >-
              HPA has been at maximum replicas for 15 minutes.
              Service may be under-provisioned.

    # ── Karpenter Alerts ───────────────────────────────
    - name: platform.karpenter
      rules:
        - alert: KarpenterUnschedulablePods
          expr: |
            karpenter_unschedulable_pods_count > 0
          for: 10m
          labels:
            severity: warning
            team: platform
          annotations:
            summary: "{{ $value }} pods unschedulable for 10 minutes"
            description: "Karpenter cannot provision nodes for pending pods."

        - alert: KarpenterNodeTerminationFailure
          expr: |
            increase(karpenter_nodes_termination_time_seconds_count[5m]) > 0
            and
            karpenter_nodes_termination_time_seconds_sum
              / karpenter_nodes_termination_time_seconds_count > 300
          labels:
            severity: warning
            team: platform
          annotations:
            summary: "Karpenter node termination taking > 5 minutes"

    # ── Application Business Alerts ────────────────────
    - name: application.business
      rules:
        - alert: HighURLCreationFailureRate
          expr: |
            sum(rate(url_creation_total{status="error"}[5m]))
            /
            sum(rate(url_creation_total[5m])) > 0.05
          for: 5m
          labels:
            severity: warning
            team: application
          annotations:
            summary: "URL creation failure rate above 5%"
            description: >-
              {{ $value | humanizePercentage }} of URL creation
              requests are failing.

        - alert: RedirectLatencyHigh
          expr: |
            histogram_quantile(0.99,
              sum(rate(redirect_duration_seconds_bucket[5m])) by (le)
            ) > 0.2
          for: 5m
          labels:
            severity: warning
            team: application
          annotations:
            summary: "Redirect p99 latency above 200ms"
            description: >-
              p99 redirect latency is {{ $value | humanizeDuration }}.
              Target is < 200ms.

    # ── Infrastructure Alerts ──────────────────────────
    - name: platform.infrastructure
      rules:
        # Node disk pressure
        - alert: NodeDiskPressure
          expr: |
            kube_node_status_condition{
              condition="DiskPressure",
              status="true"
            } == 1
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "Node {{ $labels.node }} has disk pressure"
            runbook_url: "https://github.com/YOUR_USERNAME/cloudforge/blob/main/docs/runbooks/node-disk-pressure.md"

        # Node memory pressure
        - alert: NodeMemoryPressure
          expr: |
            kube_node_status_condition{
              condition="MemoryPressure",
              status="true"
            } == 1
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "Node {{ $labels.node }} has memory pressure"

        # EBS volume IOPS approaching limit
        - alert: EBSIOPSHigh
          expr: |
            rate(container_fs_reads_total[5m])
            + rate(container_fs_writes_total[5m]) > 2500
          for: 10m
          labels:
            severity: warning
            team: platform
          annotations:
            summary: "EBS IOPS approaching gp3 baseline limit (3000)"
EOF

kubectl apply -f kubernetes/platform/monitoring/alert-rules.yaml
```

## PART 3 — Loki (Log Aggregation)

```
mkdir -p kubernetes/platform/logging

cat > kubernetes/platform/logging/loki-values.yaml << 'EOF'
# ═══════════════════════════════════════════════════════
# Loki Production Values
# Simple Scalable deployment mode
# Chunks → S3, Index → BoltDB/Shipper
# ═══════════════════════════════════════════════════════

loki:
  auth_enabled: false

  commonConfig:
    replication_factor: 2

  storage:
    type: s3
    s3:
      region: ap-south-1
      bucketnames: cloudforge-loki-chunks-ACCOUNT_ID
      s3ForcePathStyle: false

  schemaConfig:
    configs:
      - from: "2024-01-01"
        store: tsdb
        object_store: s3
        schema: v13
        index:
          prefix: loki_index_
          period: 24h

  limits_config:
    retention_period: 744h    # 31 days
    ingestion_rate_mb: 20
    ingestion_burst_size_mb: 40
    max_streams_per_user: 10000
    max_chunks_per_query: 2000000
    max_query_series: 5000
    max_query_parallelism: 32
    split_queries_by_interval: 24h
    # Enforce structured logging labels
    max_label_names_per_series: 20

  query_range:
    results_cache:
      cache:
        embedded_cache:
          enabled: true
          max_size_mb: 100

  ruler:
    alertmanager_url: http://kube-prometheus-stack-alertmanager.monitoring.svc.cluster.local:9093
    ring:
      kvstore:
        store: inmemory
    rule_path: /tmp/rules
    storage:
      type: local
      local:
        directory: /rules

# Deployment mode
deploymentMode: SimpleScalable

# Backend (query scheduler, ruler, compactor)
backend:
  replicas: 2
  persistence:
    storageClass: gp3
    size: 10Gi

# Read path (query frontend + querier)
read:
  replicas: 2

# Write path (distributor + ingester)
write:
  replicas: 3
  persistence:
    storageClass: gp3
    size: 10Gi

# Gateway (nginx proxy)
gateway:
  enabled: true
  replicas: 2
  ingress:
    enabled: false   # internal only

# Monitoring
monitoring:
  serviceMonitor:
    enabled: true
    additionalLabels:
      release: kube-prometheus-stack
  rules:
    enabled: true
    additionalLabels:
      release: kube-prometheus-stack

# Test pod (quick validation)
test:
  enabled: false

# Single binary (for local dev — disable in prod)
singleBinary:
  replicas: 0
EOF

cat > kubernetes/applications/loki.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: loki
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: platform
  sources:
    - repoURL: https://grafana.github.io/helm-charts
      chart: loki
      targetRevision: "6.10.0"
    - repoURL: https://github.com/YOUR_USERNAME/cloudforge
      targetRevision: main
      ref: values
  source:
    repoURL: https://grafana.github.io/helm-charts
    chart: loki
    targetRevision: "6.10.0"
    helm:
      valueFiles:
        - $values/kubernetes/platform/logging/loki-values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: logging
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF

kubectl apply -f kubernetes/applications/loki.yaml
```

**Promtail (Log Shipper)**

```
cat > kubernetes/platform/logging/promtail-values.yaml << 'EOF'
# ═══════════════════════════════════════════════════════
# Promtail — ships logs from every node to Loki
# DaemonSet: one pod per node
# ═══════════════════════════════════════════════════════

config:
  logLevel: info
  serverPort: 3101

  clients:
    - url: http://loki-gateway.logging.svc.cluster.local/loki/api/v1/push
      tenant_id: cloudforge

  snippets:
    # Pipeline stages enrich logs with metadata
    pipelineStages:
      # Parse JSON logs (our FastAPI service emits JSON)
      - json:
          expressions:
            level: level
            message: message
            trace_id: trace_id
            span_id: span_id
            duration_ms: duration_ms
            status_code: status_code

      # Add labels from JSON fields
      - labels:
          level:
          trace_id:

      # Drop DEBUG logs in production (reduce volume)
      - drop:
          expression: '.*level="DEBUG".*'
          drop_counter_reason: debug_logs

      # Add timestamp from log if present
      - timestamp:
          source: timestamp
          format: RFC3339Nano
          fallback_formats:
            - RFC3339
            - "2006-01-02T15:04:05"

      # Limit log line length (prevent huge logs)
      - limit:
          rate: 100
          burst: 200

    # Additional scrape configs beyond K8s default
    extraScrapeConfigs: |
      # Scrape system journal
      - job_name: journal
        journal:
          max_age: 12h
          labels:
            job: systemd-journal
            node: ${NODE_NAME}
        relabel_configs:
          - source_labels: [__journal__systemd_unit]
            target_label: unit
          - source_labels: [__journal_priority_keyword]
            target_label: level

tolerations:
  - effect: NoSchedule
    operator: Exists
  - effect: NoExecute
    operator: Exists

resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 200m
    memory: 256Mi

serviceMonitor:
  enabled: true
  additionalLabels:
    release: kube-prometheus-stack

# Extra environment variables
extraEnv:
  - name: NODE_NAME
    valueFrom:
      fieldRef:
        fieldPath: spec.nodeName
EOF

cat > kubernetes/applications/promtail.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: promtail
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: platform
  sources:
    - repoURL: https://grafana.github.io/helm-charts
      chart: promtail
      targetRevision: "6.16.4"
    - repoURL: https://github.com/YOUR_USERNAME/cloudforge
      targetRevision: main
      ref: values
  source:
    repoURL: https://grafana.github.io/helm-charts
    chart: promtail
    targetRevision: "6.16.4"
    helm:
      valueFiles:
        - $values/kubernetes/platform/logging/promtail-values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: logging
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF

kubectl apply -f kubernetes/applications/promtail.yaml
```

**PART 4 — Tempo (Distributed Tracing)**

```
mkdir -p kubernetes/platform/tracing

cat > kubernetes/platform/tracing/tempo-values.yaml << 'EOF'
# ═══════════════════════════════════════════════════════
# Tempo Production Values
# Stores traces in S3
# Receives: OTLP (gRPC + HTTP), Jaeger, Zipkin
# ═══════════════════════════════════════════════════════

tempo:
  reportingEnabled: false

  storage:
    trace:
      backend: s3
      s3:
        bucket: cloudforge-tempo-traces-ACCOUNT_ID
        region: ap-south-1
        forcepathstyle: false
      wal:
        path: /var/tempo/wal
      block:
        bloom_filter_false_positive: 0.05
        v2_encoding: snappy

  metricsGenerator:
    enabled: true
    remoteWriteUrl: "http://kube-prometheus-stack-prometheus.monitoring.svc.cluster.local:9090/api/v1/write"
    # Generate RED metrics (Rate, Errors, Duration) from traces
    processors:
      - service-graphs
      - span-metrics

  limits:
    global:
      ingestion:
        rate_strategy: global
        rate_limit_bytes: 20000000    # 20MB/s
        burst_size_bytes: 40000000
        max_traces_per_user: 10000
      max_bytes_per_trace: 50000
      max_search_bytes_per_trace: 0

  server:
    http_listen_port: 3100
    grpc_listen_port: 9095
    log_level: info

  compactor:
    ring:
      kvstore:
        store: memberlist

  distributor:
    receivers:
      otlp:
        protocols:
          grpc:
            endpoint: 0.0.0.0:4317
          http:
            endpoint: 0.0.0.0:4318
      jaeger:
        protocols:
          thrift_http:
            endpoint: 0.0.0.0:14268
          grpc:
            endpoint: 0.0.0.0:14250
      zipkin:
        endpoint: 0.0.0.0:9411

  querier:
    max_concurrent_queries: 10
    frontend_worker:
      frontend_address: tempo-query-frontend:9095

  queryFrontend:
    search:
      default_result_limit: 20
      max_result_limit: 0
      max_duration: 0s

# Deployment
replicas: 2

persistence:
  enabled: true
  storageClassName: gp3
  size: 10Gi

serviceAccount:
  annotations:
    eks.amazonaws.com/role-arn: TEMPO_ROLE_ARN

resources:
  requests:
    cpu: 500m
    memory: 1Gi
  limits:
    cpu: 2000m
    memory: 4Gi

serviceMonitor:
  enabled: true
  additionalLabels:
    release: kube-prometheus-stack
EOF

cat > kubernetes/applications/tempo.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: tempo
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: platform
  sources:
    - repoURL: https://grafana.github.io/helm-charts
      chart: tempo-distributed
      targetRevision: "1.14.0"
    - repoURL: https://github.com/YOUR_USERNAME/cloudforge
      targetRevision: main
      ref: values
  source:
    repoURL: https://grafana.github.io/helm-charts
    chart: tempo-distributed
    targetRevision: "1.14.0"
    helm:
      valueFiles:
        - $values/kubernetes/platform/tracing/tempo-values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: tracing
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF

kubectl apply -f kubernetes/applications/tempo.yaml
```

## PART 5 — OpenTelemetry Collector

The OTel Collector is the backbone. All services send telemetry here. Collector fans it out to Prometheus, Loki, and Tempo. Vendor-neutral.

```
mkdir -p kubernetes/platform/otel

cat > kubernetes/platform/otel/collector-values.yaml << 'EOF'
# ═══════════════════════════════════════════════════════
# OpenTelemetry Collector
# Mode: DaemonSet (one per node — collects from all pods)
# + Deployment (gateway — handles batching/routing)
# ═══════════════════════════════════════════════════════

mode: daemonset

image:
  repository: otel/opentelemetry-collector-contrib
  tag: 0.105.0

# Collector configuration
config:
  # ── Receivers (how telemetry comes IN) ──────────────
  receivers:
    otlp:
      protocols:
        grpc:
          endpoint: 0.0.0.0:4317
        http:
          endpoint: 0.0.0.0:4318
          cors:
            allowed_origins:
              - "https://*.yourdomain.com"

    # Collect host metrics
    hostmetrics:
      root_path: /hostfs
      collection_interval: 30s
      scrapers:
        cpu:
          metrics:
            system.cpu.logical.count:
              enabled: true
        disk: {}
        filesystem:
          exclude_mount_points:
            mount_points: [/dev/*, /proc/*, /sys/*, /run/k3s/**, /run/containerd/**, /var/lib/docker/**, /var/lib/kubelet/**, /snap/**]
            match_type: regexp
          exclude_fs_types:
            fs_types: [autofs, binfmt_misc, bpf, cgroup2, configfs, debugfs, devpts, devtmpfs, fusectl, hugetlbfs, iso9660, mqueue, nsfs, overlay, proc, procfs, pstore, rpc_pipefs, securityfs, selinuxfs, squashfs, sysfs, tracefs]
            match_type: strict
        load: {}
        memory:
          metrics:
            system.memory.utilization:
              enabled: true
        network: {}
        paging:
          metrics:
            system.paging.utilization:
              enabled: true
        processes: {}

    # Collect Kubernetes events
    k8s_events:
      namespaces: []   # all namespaces

    # Collect from Prometheus endpoints
    prometheus:
      config:
        scrape_configs:
          - job_name: otel-collector
            static_configs:
              - targets: ['0.0.0.0:8888']

  # ── Processors (enrich/transform telemetry) ─────────
  processors:
    # Add K8s metadata to all telemetry
    k8sattributes:
      auth_type: serviceAccount
      passthrough: false
      filter:
        node_from_env_var: KUBE_NODE_NAME
      extract:
        metadata:
          - k8s.pod.name
          - k8s.pod.uid
          - k8s.deployment.name
          - k8s.namespace.name
          - k8s.node.name
          - k8s.pod.start_time
          - k8s.replicaset.name
          - k8s.replicaset.uid
          - k8s.daemonset.name
          - k8s.daemonset.uid
          - k8s.statefulset.name
          - k8s.statefulset.uid
          - k8s.cronjob.name
          - k8s.job.name
          - k8s.job.uid
        labels:
          - tag_name: app.kubernetes.io/name
            key: app.kubernetes.io/name
            from: pod
          - tag_name: app.kubernetes.io/version
            key: app.kubernetes.io/version
            from: pod
        annotations:
          - tag_name: prometheus.io/scrape
            key: prometheus.io/scrape
            from: pod

    # Add resource attributes
    resourcedetection:
      detectors: [eks, ec2, env]
      timeout: 15s
      override: false
      ec2:
        tags:
          - ^kubernetes.io/cluster/.*$
          - ^k8s.io/cluster-autoscaler/.*$

    # Memory limiter — prevent OOM
    memory_limiter:
      check_interval: 5s
      limit_percentage: 80
      spike_limit_percentage: 25

    # Batch for efficiency
    batch:
      timeout: 10s
      send_batch_size: 10000
      send_batch_max_size: 11000

    # Tail sampling — keep interesting traces, drop high-volume healthy ones
    tail_sampling:
      decision_wait: 10s
      expected_new_traces_per_sec: 100
      policies:
        # Always keep error traces
        - name: errors-policy
          type: status_code
          status_code:
            status_codes: [ERROR]

        # Always keep slow traces (> 1s)
        - name: slow-traces-policy
          type: latency
          latency:
            threshold_ms: 1000

        # Sample 10% of healthy fast traces
        - name: probabilistic-policy
          type: probabilistic
          probabilistic:
            sampling_percentage: 10

        # Always keep traces with user IDs
        - name: user-traces-policy
          type: string_attribute
          string_attribute:
            key: user.id
            values: ['.*']
            enabled_regex_matching: true

    # Transform to add environment label
    transform:
      metric_statements:
        - context: resource
          statements:
            - set(attributes["deployment.environment"], "${env:ENVIRONMENT}")
      log_statements:
        - context: resource
          statements:
            - set(attributes["deployment.environment"], "${env:ENVIRONMENT}")
      trace_statements:
        - context: resource
          statements:
            - set(attributes["deployment.environment"], "${env:ENVIRONMENT}")

  # ── Exporters (where telemetry goes OUT) ────────────
  exporters:
    # Metrics → Prometheus (via remote write)
    prometheusremotewrite:
      endpoint: "http://kube-prometheus-stack-prometheus.monitoring.svc.cluster.local:9090/api/v1/write"
      tls:
        insecure: true
      resource_to_telemetry_conversion:
        enabled: true

    # Traces → Tempo
    otlp/tempo:
      endpoint: "tempo.tracing.svc.cluster.local:4317"
      tls:
        insecure: true

    # Logs → Loki
    loki:
      endpoint: "http://loki-gateway.logging.svc.cluster.local/loki/api/v1/push"
      default_labels_enabled:
        exporter: false
        job: true
        instance: true
        level: true
      labels:
        resource:
          k8s.namespace.name: "namespace"
          k8s.pod.name: "pod"
          k8s.deployment.name: "deployment"
          app.kubernetes.io/name: "app"
          deployment.environment: "environment"
        attributes:
          level: "level"

    # Debug (development only — disable in prod)
    # debug:
    #   verbosity: detailed

  # ── Extensions ───────────────────────────────────────
  extensions:
    health_check:
      endpoint: 0.0.0.0:13133
    pprof:
      endpoint: 0.0.0.0:1777
    zpages:
      endpoint: 0.0.0.0:55679

  # ── Service (wire everything together) ───────────────
  service:
    extensions: [health_check, pprof, zpages]
    pipelines:
      traces:
        receivers: [otlp]
        processors: [memory_limiter, k8sattributes, resourcedetection, transform, tail_sampling, batch]
        exporters: [otlp/tempo]

      metrics:
        receivers: [otlp, hostmetrics, prometheus]
        processors: [memory_limiter, k8sattributes, resourcedetection, transform, batch]
        exporters: [prometheusremotewrite]

      logs:
        receivers: [otlp, k8s_events]
        processors: [memory_limiter, k8sattributes, resourcedetection, transform, batch]
        exporters: [loki]

    telemetry:
      logs:
        level: info
      metrics:
        address: 0.0.0.0:8888

# DaemonSet tolerations
tolerations:
  - effect: NoSchedule
    operator: Exists
  - effect: NoExecute
    operator: Exists

# Resource limits
resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

# Service account for K8s API access
serviceAccount:
  annotations:
    eks.amazonaws.com/role-arn: OTEL_COLLECTOR_ROLE_ARN

# Ports
ports:
  otlp:
    enabled: true
    containerPort: 4317
    servicePort: 4317
    protocol: TCP
  otlp-http:
    enabled: true
    containerPort: 4318
    servicePort: 4318
    protocol: TCP
  metrics:
    enabled: true
    containerPort: 8888
    servicePort: 8888
    protocol: TCP

serviceMonitor:
  enabled: true
  additionalLabels:
    release: kube-prometheus-stack

# Pod monitor for self-monitoring
podMonitor:
  enabled: true
  additionalLabels:
    release: kube-prometheus-stack

# Host filesystem access for hostmetrics
extraVolumes:
  - name: hostfs
    hostPath:
      path: /

extraVolumeMounts:
  - name: hostfs
    mountPath: /hostfs
    readOnly: true
    mountPropagation: HostToContainer

extraEnv:
  - name: KUBE_NODE_NAME
    valueFrom:
      fieldRef:
        apiVersion: v1
        fieldPath: spec.nodeName
  - name: ENVIRONMENT
    value: staging
EOF

cat > kubernetes/applications/otel-collector.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: otel-collector
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: platform
  sources:
    - repoURL: https://open-telemetry.github.io/opentelemetry-helm-charts
      chart: opentelemetry-collector
      targetRevision: "0.100.0"
    - repoURL: https://github.com/YOUR_USERNAME/cloudforge
      targetRevision: main
      ref: values
  source:
    repoURL: https://open-telemetry.github.io/opentelemetry-helm-charts
    chart: opentelemetry-collector
    targetRevision: "0.100.0"
    helm:
      valueFiles:
        - $values/kubernetes/platform/otel/collector-values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: monitoring
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF

kubectl apply -f kubernetes/applications/otel-collector.yaml
```

## PART 6 — Instrument the API Service

```
# services/api-service/app/telemetry.py
# ═══════════════════════════════════════════════════════
# OpenTelemetry instrumentation — zero vendor lock-in
# Sends to OTel Collector which routes to Tempo/Prometheus/Loki
# ═══════════════════════════════════════════════════════

import os
import logging
from typing import Optional

from opentelemetry import trace, metrics
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.sdk.metrics import MeterProvider
from opentelemetry.sdk.metrics.export import PeriodicExportingMetricReader
from opentelemetry.sdk.resources import Resource, SERVICE_NAME, SERVICE_VERSION
from opentelemetry.exporter.otlp.proto.grpc.trace_exporter import OTLPSpanExporter
from opentelemetry.exporter.otlp.proto.grpc.metric_exporter import OTLPMetricExporter
from opentelemetry.instrumentation.fastapi import FastAPIInstrumentor
from opentelemetry.instrumentation.boto3 import Boto3Instrumentor
from opentelemetry.instrumentation.redis import RedisInstrumentor
from opentelemetry.instrumentation.httpx import HTTPXClientInstrumentor
from opentelemetry.propagate import set_global_textmap
from opentelemetry.propagators.b3 import B3MultiFormat
from opentelemetry.propagators.composite import CompositePropagator
from opentelemetry.trace.propagation.tracecontext import TraceContextTextMapPropagator
import structlog

logger = structlog.get_logger()


def setup_telemetry(app=None) -> None:
    """
    Initialize OpenTelemetry SDK.
    Call once at application startup.
    """
    environment = os.getenv("ENVIRONMENT", "local")
    service_name = os.getenv("OTEL_SERVICE_NAME", "api-service")
    service_version = os.getenv("SERVICE_VERSION", "unknown")
    otel_endpoint = os.getenv(
        "OTEL_EXPORTER_OTLP_ENDPOINT",
        "http://otel-collector.monitoring.svc.cluster.local:4317"
    )

    # Resource — metadata attached to ALL telemetry from this service
    resource = Resource.create({
        SERVICE_NAME: service_name,
        SERVICE_VERSION: service_version,
        "deployment.environment": environment,
        "k8s.pod.name": os.getenv("POD_NAME", "unknown"),
        "k8s.namespace.name": os.getenv("POD_NAMESPACE", "unknown"),
        "k8s.node.name": os.getenv("NODE_NAME", "unknown"),
    })

    # ── Tracing setup ────────────────────────────────────
    trace_exporter = OTLPSpanExporter(
        endpoint=otel_endpoint,
        insecure=True,
    )

    tracer_provider = TracerProvider(resource=resource)
    tracer_provider.add_span_processor(
        BatchSpanProcessor(
            trace_exporter,
            max_queue_size=2048,
            max_export_batch_size=512,
            export_timeout_millis=30000,
        )
    )
    trace.set_tracer_provider(tracer_provider)

    # ── Metrics setup ─────────────────────────────────────
    metric_exporter = OTLPMetricExporter(
        endpoint=otel_endpoint,
        insecure=True,
    )

    metric_reader = PeriodicExportingMetricReader(
        exporter=metric_exporter,
        export_interval_millis=30000,
    )

    meter_provider = MeterProvider(
        resource=resource,
        metric_readers=[metric_reader]
    )
    metrics.set_meter_provider(meter_provider)

    # ── Propagation (trace context across service boundaries) ──
    set_global_textmap(CompositePropagator([
        TraceContextTextMapPropagator(),
        B3MultiFormat(),
    ]))

    # ── Auto-instrumentation ──────────────────────────────
    if app is not None:
        FastAPIInstrumentor.instrument_app(
            app,
            tracer_provider=tracer_provider,
            meter_provider=meter_provider,
            excluded_urls="/health.*,/metrics",
        )

    Boto3Instrumentor().instrument()
    RedisInstrumentor().instrument()
    HTTPXClientInstrumentor().instrument()

    logger.info(
        "OpenTelemetry initialized",
        service=service_name,
        environment=environment,
        endpoint=otel_endpoint,
    )


def get_tracer(name: str):
    return trace.get_tracer(name)


def get_meter(name: str):
    return metrics.get_meter(name)
```

```
# services/api-service/app/metrics.py
# Custom business metrics
from opentelemetry import metrics

meter = metrics.get_meter("api-service")

# Counters
url_creation_counter = meter.create_counter(
    name="url_creation_total",
    description="Total URL creation attempts",
    unit="1",
)

url_redirect_counter = meter.create_counter(
    name="url_redirect_total",
    description="Total URL redirect attempts",
    unit="1",
)

# Histograms
redirect_duration = meter.create_histogram(
    name="redirect_duration_seconds",
    description="URL redirect duration in seconds",
    unit="s",
)

# Gauges
active_urls_gauge = meter.create_observable_gauge(
    name="active_urls_total",
    description="Total active (non-expired) short URLs",
    unit="1",
)

# Usage in handlers:
# url_creation_counter.add(1, {"status": "success", "owner": owner})
# redirect_duration.record(elapsed, {"cached": str(cache_hit)})
```

```
# services/api-service/app/logging_config.py
# Structured logging — every log has trace context
import logging
import sys
import structlog
from opentelemetry import trace


def add_trace_context(logger, method, event_dict):
    """Inject trace ID and span ID into every log record."""
    span = trace.get_current_span()
    if span and span.is_recording():
        ctx = span.get_span_context()
        event_dict["trace_id"] = format(ctx.trace_id, "032x")
        event_dict["span_id"] = format(ctx.span_id, "016x")
        event_dict["trace_flags"] = ctx.trace_flags
    return event_dict


def setup_logging(log_level: str = "INFO") -> None:
    """Configure structlog for JSON output with trace correlation."""

    structlog.configure(
        processors=[
            structlog.contextvars.merge_contextvars,
            structlog.processors.add_log_level,
            structlog.processors.TimeStamper(fmt="iso"),
            add_trace_context,                    # inject trace IDs
            structlog.processors.StackInfoRenderer(),
            structlog.processors.ExceptionRenderer(),
            structlog.processors.JSONRenderer(),  # JSON output for Loki
        ],
        wrapper_class=structlog.make_filtering_bound_logger(
            logging.getLevelName(log_level)
        ),
        context_class=dict,
        logger_factory=structlog.PrintLoggerFactory(sys.stdout),
        cache_logger_on_first_use=True,
    )
```

## PART 7 — Grafana Dashboards

```
mkdir -p kubernetes/platform/monitoring/dashboards

cat > kubernetes/platform/monitoring/dashboards/slo-dashboard.json << 'EOF'
{
  "__inputs": [],
  "__requires": [],
  "annotations": {"list": []},
  "description": "CloudForge API Service SLO Dashboard",
  "editable": true,
  "fiscalYearStartMonth": 0,
  "graphTooltip": 1,
  "id": null,
  "links": [],
  "panels": [
    {
      "datasource": {"type": "prometheus", "uid": "prometheus"},
      "fieldConfig": {
        "defaults": {
          "color": {"mode": "thresholds"},
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {"color": "red", "value": null},
              {"color": "yellow", "value": 99.5},
              {"color": "green", "value": 99.9}
            ]
          },
          "unit": "percent",
          "min": 99,
          "max": 100
        }
      },
      "gridPos": {"h": 8, "w": 6, "x": 0, "y": 0},
      "id": 1,
      "options": {
        "orientation": "auto",
        "reduceOptions": {"calcs": ["lastNotNull"]},
        "showThresholdLabels": true,
        "showThresholdMarkers": true
      },
      "title": "Availability SLO (99.9% target)",
      "type": "gauge",
      "targets": [
        {
          "expr": "sum(rate(http_requests_total{job=\"api-service\",status!~\"5..\"}[30d])) / sum(rate(http_requests_total{job=\"api-service\"}[30d])) * 100",
          "legendFormat": "30d availability"
        }
      ]
    },
    {
      "datasource": {"type": "prometheus", "uid": "prometheus"},
      "fieldConfig": {
        "defaults": {
          "color": {"mode": "thresholds"},
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {"color": "green", "value": null},
              {"color": "yellow", "value": 50},
              {"color": "red", "value": 80}
            ]
          },
          "unit": "percent",
          "min": 0,
          "max": 100
        }
      },
      "gridPos": {"h": 8, "w": 6, "x": 6, "y": 0},
      "id": 2,
      "options": {
        "orientation": "auto",
        "reduceOptions": {"calcs": ["lastNotNull"]},
        "showThresholdLabels": true,
        "showThresholdMarkers": true
      },
      "title": "Error Budget Remaining",
      "type": "gauge",
      "targets": [
        {
          "expr": "(1 - (sum(rate(http_requests_total{job=\"api-service\",status=~\"5..\"}[30d])) / sum(rate(http_requests_total{job=\"api-service\"}[30d])) / 0.001)) * 100",
          "legendFormat": "Error budget %"
        }
      ]
    },
    {
      "datasource": {"type": "prometheus", "uid": "prometheus"},
      "fieldConfig": {
        "defaults": {
          "color": {"mode": "palette-classic"},
          "unit": "s"
        }
      },
      "gridPos": {"h": 8, "w": 12, "x": 12, "y": 0},
      "id": 3,
      "options": {
        "legend": {"calcs": ["p50", "p95", "p99"], "displayMode": "table"},
        "tooltip": {"mode": "multi"}
      },
      "title": "Request Latency Percentiles",
      "type": "timeseries",
      "targets": [
        {
          "expr": "histogram_quantile(0.50, sum(rate(http_request_duration_seconds_bucket{job=\"api-service\"}[5m])) by (le))",
          "legendFormat": "p50"
        },
        {
          "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket{job=\"api-service\"}[5m])) by (le))",
          "legendFormat": "p95"
        },
        {
          "expr": "histogram_quantile(0.99, sum(rate(http_request_duration_seconds_bucket{job=\"api-service\"}[5m])) by (le))",
          "legendFormat": "p99"
        }
      ]
    }
  ],
  "refresh": "30s",
  "schemaVersion": 39,
  "tags": ["slo", "api-service", "cloudforge"],
  "time": {"from": "now-24h", "to": "now"},
  "timepicker": {},
  "timezone": "browser",
  "title": "API Service SLO",
  "uid": "api-service-slo",
  "version": 1
}
EOF

# ConfigMap to mount dashboards into Grafana
cat > kubernetes/platform/monitoring/dashboards/dashboards-configmap.yaml << 'EOF'
apiVersion: v1
kind: ConfigMap
metadata:
  name: cloudforge-dashboards
  namespace: monitoring
  labels:
    grafana_dashboard: "1"    # Grafana sidecar picks this up automatically
data:
  slo-dashboard.json: |-
    # contents of slo-dashboard.json go here (reference the file)
EOF
```

## PART 8 — Loki Alert Rules

```
cat > kubernetes/platform/logging/loki-rules.yaml << 'EOF'
# ═══════════════════════════════════════════════════════
# Loki Alerting Rules
# Log-based alerts (complement Prometheus metric alerts)
# ═══════════════════════════════════════════════════════
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: loki-alerts
  namespace: monitoring
  labels:
    release: kube-prometheus-stack
spec:
  groups:
    - name: loki.application
      rules:
        # High error log rate from any service
        - alert: HighErrorLogRate
          expr: |
            sum by (namespace, app) (
              rate({namespace=~".+"}
                |= "level=error"
                [5m])
            ) > 10
          for: 5m
          labels:
            severity: warning
            team: platform
          annotations:
            summary: "High error log rate in {{ $labels.namespace }}/{{ $labels.app }}"
            description: "{{ $value }} error log lines per second."

        # Panic or fatal logs detected
        - alert: PanicOrFatalLogDetected
          expr: |
            sum by (namespace, pod) (
              count_over_time({namespace=~".+"}
                |~ "(?i)(panic|fatal|CRITICAL)"
                [5m])
            ) > 0
          labels:
            severity: critical
            team: platform
          annotations:
            summary: "Panic/Fatal log detected in {{ $labels.namespace }}/{{ $labels.pod }}"
            description: "Immediate investigation required."

        # OOM detected in logs
        - alert: OOMDetectedInLogs
          expr: |
            sum by (namespace, pod) (
              count_over_time({namespace=~".+"}
                |~ "(?i)(out of memory|oom|killed)"
                [5m])
            ) > 0
          labels:
            severity: warning
            team: platform
          annotations:
            summary: "OOM detected in logs for {{ $labels.namespace }}/{{ $labels.pod }}"
EOF

kubectl apply -f kubernetes/platform/logging/loki-rules.yaml
```

## PART 9 — Week 3 Validation Script

```
cat > scripts/validate-week3.sh << 'EOF'
#!/bin/bash
set -euo pipefail

echo "═══════════════════════════════════════"
echo "  CloudForge Week 3 Validation"
echo "═══════════════════════════════════════"

PASS=0; FAIL=0

check() {
  if eval "$2" &>/dev/null; then
    echo "  ✅ $1"; ((PASS++))
  else
    echo "  ❌ $1"; ((FAIL++))
  fi
}

echo ""
echo "── Prometheus Stack ────────────────────"
check "Prometheus pods running (2)" \
  "[ $(kubectl get pods -n monitoring -l app.kubernetes.io/name=prometheus --no-headers | grep -c Running) -ge 2 ]"
check "Alertmanager running" \
  "kubectl get pods -n monitoring -l app.kubernetes.io/name=alertmanager --no-headers | grep -q Running"
check "Grafana running" \
  "kubectl get pods -n monitoring -l app.kubernetes.io/name=grafana --no-headers | grep -q Running"
check "Node exporter DaemonSet ready" \
  "kubectl get ds -n monitoring -l app.kubernetes.io/name=prometheus-node-exporter --no-headers | awk '{print \$4==\$3}' | grep -q 1"
check "kube-state-metrics running" \
  "kubectl get pods -n monitoring -l app.kubernetes.io/name=kube-state-metrics --no-headers | grep -q Running"

echo ""
echo "── Prometheus Rules ─────────────────────"
check "CloudForge alert rules loaded" \
  "kubectl get prometheusrule cloudforge-alerts -n monitoring &>/dev/null"
check "Default K8s rules active" \
  "[ $(kubectl get prometheusrule -n monitoring --no-headers | wc -l) -ge 5 ]"

echo ""
echo "── Loki Stack ───────────────────────────"
check "Loki write pods running" \
  "kubectl get pods -n logging -l app.kubernetes.io/name=loki,app.kubernetes.io/component=write --no-headers | grep -q Running"
check "Loki read pods running" \
  "kubectl get pods -n logging -l app.kubernetes.io/name=loki,app.kubernetes.io/component=read --no-headers | grep -q Running"
check "Promtail DaemonSet ready" \
  "kubectl get ds -n logging -l app.kubernetes.io/name=promtail --no-headers | awk '{print \$4==\$3}' | grep -q 1"

echo ""
echo "── Tempo Stack ──────────────────────────"
check "Tempo pods running" \
  "kubectl get pods -n tracing -l app.kubernetes.io/name=tempo --no-headers | grep -q Running"

echo ""
echo "── OpenTelemetry Collector ──────────────"
check "OTel Collector DaemonSet ready" \
  "kubectl get ds -n monitoring -l app.kubernetes.io/name=opentelemetry-collector --no-headers | awk '{print \$4==\$3}' | grep -q 1"

echo ""
echo "── Grafana Datasources ──────────────────"
GRAFANA_POD=$(kubectl get pods -n monitoring -l app.kubernetes.io/name=grafana -o jsonpath='{.items[0].metadata.name}')
check "Prometheus datasource healthy" \
  "kubectl exec -n monitoring $GRAFANA_POD -- curl -sf http://localhost:3000/api/datasources/proxy/1/api/v1/query?query=up | python3 -c 'import sys,json; d=json.load(sys.stdin); exit(0 if d[\"status\"]==\"success\" else 1)'"
check "Loki datasource reachable" \
  "kubectl exec -n monitoring $GRAFANA_POD -- curl -sf http://loki-gateway.logging.svc.cluster.local/ready"
check "Tempo datasource reachable" \
  "kubectl exec -n monitoring $GRAFANA_POD -- curl -sf http://tempo.tracing.svc.cluster.local:3100/ready"

echo ""
echo "── Alertmanager ─────────────────────────"
check "Alertmanager config valid" \
  "kubectl exec -n monitoring -l app.kubernetes.io/name=alertmanager -- amtool check-config /etc/alertmanager/config_out/alertmanager.env.yaml"
check "Watchdog alert firing (proves alerting works)" \
  "kubectl exec -n monitoring -l app.kubernetes.io/name=alertmanager -- amtool alert query alertname=Watchdog | grep -q Watchdog"

echo ""
echo "── Log Ingestion Test ───────────────────"
check "Logs flowing to Loki" \
  "kubectl exec -n logging -l app.kubernetes.io/name=promtail -- curl -sf 'http://loki-gateway.logging.svc.cluster.local/loki/api/v1/query?query={namespace=\"kube-system\"}&limit=1' | python3 -c 'import sys,json; d=json.load(sys.stdin); exit(0 if len(d[\"data\"][\"result\"]) > 0 else 1)'"

echo ""
echo "── Trace Ingestion Test ─────────────────"
check "Tempo accepting traces" \
  "kubectl exec -n tracing -l app.kubernetes.io/name=tempo -- curl -sf http://localhost:3100/ready | grep -q ready"

echo ""
echo "═══════════════════════════════════════"
echo "  Results: ✅ $PASS passed | ❌ $FAIL failed"
echo "═══════════════════════════════════════"

[ $FAIL -eq 0 ] && echo "  🎉 Week 3 Complete!" \
  || echo "  ⚠️  Fix failures before Week 4"
EOF

chmod +x scripts/validate-week3.sh
./scripts/validate-week3.sh
```

## PART 10 — Runbooks

```
cat > docs/runbooks/api-availability.md << 'EOF'
# Runbook: API Availability SLO Breach

**Alert:** APIAvailabilitySLOBreach
**Severity:** Critical
**SLO Target:** 99.9% availability

## Impact
API service is returning 5xx errors above threshold.
Users cannot create or resolve short URLs.

## Diagnosis Steps

### Step 1 — Check pod status
bash
kubectl get pods -n staging -l app=api-service
kubectl describe pod -n staging -l app=api-service | tail -30


### Step 2 — Check recent logs
bash
# In Grafana → Explore → Loki
{namespace="staging", app="api-service"} |= "error" | last 15m


### Step 3 — Check error rate by endpoint
promql
sum by (route) (
  rate(http_requests_total{job="api-service",status=~"5.."}[5m])
)


### Step 4 — Check database connectivity
bash
kubectl exec -n staging -l app=api-service -- \
  python3 -c "import psycopg2; psycopg2.connect('$DATABASE_URL')"


### Step 5 — Check downstream dependencies
bash
kubectl get pods -n staging   # Redis, worker services
kubectl top pods -n staging   # Resource pressure


## Remediation

### If pods are crash-looping
bash
kubectl rollout undo deployment/api-service -n staging


### If database is unreachable
Check RDS console → check security groups → check parameter group

### If resource pressure
bash
kubectl patch hpa api-service -n staging \
  -p '{"spec":{"minReplicas":5}}'


## Escalation
- P1 on-call: PagerDuty → Platform Engineering
- Incident channel: #incidents
- Stakeholder update every 15 minutes
EOF
```

## Week 3 Commit

```
git add .
git commit -m "feat: Week 3 — Full LGTM Observability Stack

Metrics:
- kube-prometheus-stack (Prometheus HA x2, Alertmanager HA x2, Grafana HA x2)
- Custom PrometheusRules (SLO, error budget burn, K8s, Karpenter, business)
- Pre-built dashboards (K8s cluster, nodes, Karpenter, ArgoCD)
- SLO dashboard (availability gauge, error budget, latency percentiles)

Logging:
- Loki Simple Scalable (write x3, read x2, backend x2)
- S3 backend (chunks + index), 31-day retention
- Promtail DaemonSet (JSON parsing, trace ID extraction, structured labels)
- Loki alert rules (error rate, panic detection, OOM)

Tracing:
- Tempo Distributed (S3 backend, metrics generator for RED metrics)
- Receives OTLP, Jaeger, Zipkin
- Correlates traces ↔ logs ↔ metrics in Grafana

OpenTelemetry:
- OTel Collector DaemonSet (enriches with K8s metadata, tail sampling)
- Auto-instrumentation: FastAPI, boto3, Redis, httpx
- Structured logging with trace ID injection
- Custom business metrics (url_creation_total, redirect_duration)

Alerting:
- Alertmanager with inhibition rules
- Slack routing (critical → #incidents, warning → #platform-warnings)
- PagerDuty for P1 (critical severity)
- Dead Man's Snitch (proves alerting pipeline is alive)

Runbooks:
- API availability, latency, OOM, crash loop, PV full

Validated:
- All LGTM components healthy ✅
- Logs flowing to Loki ✅
- Watchdog alert firing ✅
- Grafana datasources healthy ✅"

git push origin feat/week1-foundation
```

## Week 3 Summary

```
What you built:
├── ✅ Prometheus HA (metrics, 30d retention, SLO rules)
├── ✅ Alertmanager (routing, inhibition, PagerDuty, Slack)
├── ✅ Grafana (dashboards, datasources, SLO panels)
├── ✅ Loki (log aggregation, S3 backend, structured parsing)
├── ✅ Promtail (DaemonSet, JSON pipeline, trace correlation)
├── ✅ Tempo (distributed tracing, S3, RED metrics)
├── ✅ OTel Collector (enrichment, tail sampling, fan-out)
├── ✅ Service instrumentation (metrics + traces + logs correlated)
├── ✅ Alert rules (SLO, error budget burn, K8s, Karpenter)
└── ✅ Runbooks (linked from every alert)

The Grafana Labs signal:
When you apply to Grafana Labs and they see:
├── Loki (not ELK — their product)
├── Tempo (not Jaeger — their product)
├── OTel Collector (not proprietary agent)
├── Error budget burn rate alerts (Site Reliability Engineering)
├── Trace → log → metric correlation
└── Tail sampling with cost-aware policies
They know you understand their stack deeply.
```
















