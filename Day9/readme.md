# 🔄 CloudForge — Week 2: GitOps with ArgoCD

**What We're Building This Week**

```
Week 2 Deliverables:
├── ArgoCD (production-grade install, HA, SSO-ready)
├── ArgoCD Image Updater (auto-update image tags in git)
├── Argo Rollouts (canary + blue/green progressive delivery)
├── ApplicationSets (multi-env app deployment pattern)
├── App of Apps pattern (self-managing ArgoCD applications)
├── First real application deployed via GitOps
├── Sync policies (auto-sync, self-heal, prune)
└── GitHub Actions → ECR → ArgoCD full flow working

By end of week:
git push → GitHub Actions builds → ECR push →
ArgoCD Image Updater detects → updates git →
ArgoCD syncs → pods rolling update → done
Zero manual kubectl apply ever again.
```

## PART 1 — ArgoCD Production Install

**Why ArgoCD Over FluxCD**

```
ArgoCD:
├── UI (visual diff, sync status, app health)
├── Multi-cluster management from one control plane
├── RBAC built-in (project-level access control)
├── ApplicationSets (template-driven multi-env)
├── Notifications (Slack, PagerDuty, email)
└── Industry adoption: Spotify, Intuit, Red Hat use it

FluxCD:
├── CLI-first (no built-in UI)
├── More Kubernetes-native (CRD-heavy)
├── Better Helm + Kustomize integration
└── Lighter weight

Decision: ArgoCD — better for a portfolio (visual, demonstrable)
```

**ArgoCD Helm Values — Production Grade**

```
mkdir -p kubernetes/platform/argocd

cat > kubernetes/platform/argocd/values.yaml << 'EOF'
# ═══════════════════════════════════════════════════════
# ArgoCD Production Values
# HA mode, metrics, notifications, RBAC, SSO-ready
# ═══════════════════════════════════════════════════════

global:
  domain: argocd.yourdomain.com
  image:
    tag: "v2.11.3"

configs:
  cm:
    # Enable status badge on GitHub README
    statusbadge.enabled: "true"

    # Resource health checks for custom resources
    resource.customizations.health.argoproj.io_Application: |
      hs = {}
      hs.status = "Progressing"
      hs.message = ""
      if obj.status ~= nil then
        if obj.status.health ~= nil then
          hs.status = obj.status.health.status
          if obj.status.health.message ~= nil then
            hs.message = obj.status.health.message
          end
        end
      end
      return hs

    # Ignore differences in these fields (avoid drift noise)
    resource.customizations.ignoreDifferences.admissionregistration.k8s.io_MutatingWebhookConfiguration: |
      jqPathExpressions:
        - '.webhooks[]?.clientConfig.caBundle'
    resource.customizations.ignoreDifferences.admissionregistration.k8s.io_ValidatingWebhookConfiguration: |
      jqPathExpressions:
        - '.webhooks[]?.clientConfig.caBundle'

    # Helm release name annotation
    application.instanceLabelKey: argocd.argoproj.io/app-name

    # Enable exec into pods from UI
    exec.enabled: "true"

    # Timeout for app reconciliation
    timeout.reconciliation: 180s

    # OIDC / SSO config (GitHub OAuth)
    oidc.config: |
      name: GitHub
      issuer: https://token.actions.githubusercontent.com
      clientID: $argocd-github-sso:client-id
      clientSecret: $argocd-github-sso:client-secret
      requestedScopes:
        - openid
        - profile
        - email
        - read:org

  # RBAC config
  rbac:
    policy.default: role:readonly
    policy.csv: |
      # Platform team — full admin
      p, role:platform-admin, applications, *, */*, allow
      p, role:platform-admin, clusters, *, *, allow
      p, role:platform-admin, repositories, *, *, allow
      p, role:platform-admin, projects, *, *, allow
      p, role:platform-admin, accounts, *, *, allow
      g, platform-team, role:platform-admin

      # Dev team — sync own apps only
      p, role:developer, applications, get, */*, allow
      p, role:developer, applications, sync, */*, allow
      p, role:developer, applications, override, */*, allow
      g, developers, role:developer

  params:
    # Enable insecure mode (TLS terminated by ALB/ingress)
    server.insecure: true
    # Application controller sharding
    controller.sharding.algorithm: round-robin

  # Known hosts for Git providers
  ssh:
    knownHosts: |
      github.com ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBEmKSENjQEezOmxkZMy7opKgwFB9nkt5YRrYMjNuG5N87uRgg6CLrbo5wAdT/y6v0mKV0U2w0WZ2YB/++Tpockg=
      github.com ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIOMqqnkVzrm0SdG6UOoqKLsabgH5C9okWi0dh2l9GKJl
      github.com ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCj7ndNxQowgcQnjshcLrqPEiiphnt+VTTvDP6mHBL9j1aNUkY4Ue1gvwnGLVlOhGeYrnZaMgRK6+PKCUXaDbC7qtbW8gIkhL7aGCsOr/C56SJMy/BCZfxd1nWzAOxSDPgVsmerOBYfNqltV9/hWCqBywINIR+5dIg6JTJ72pcEpEjcYgXkE2YEFR5u8O9SHK1EfYqRB+vf2NOWTW7m9QvUFHrpFHUo3cNgBqMsKjppA+NXvFCbzOB1BsSFqWPuWlUFJkMoR3xh5n3E3DHFHDgnMQcS+5GTvgBF7vhL8JsBkrBBxPk/H7gEtH+hMNL0bLvEDPd/J6wZt+8cz/E3TdIMmM3vhpbZ+pMV+Z4uT+j9qJQa/0NE9JRx2XnDVdHUHWFuJT2JCnHXnMBXGaEFzMjW8/Gv4r5wBMHivezC9GKj5N7nJkXsm8RXhzHVhB7URqsBFouvEfGxJgJMnW8aKQPHOZsN0EY4RuKSBNenB8oBGnUGm8FKJDGf8l8eFSISmCEfOv/gBNuMgVRaP+1mMKNXqFGk2VovC1+00=

# ArgoCD Server
server:
  replicas: 2

  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 5
    targetCPUUtilizationPercentage: 60

  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi

  metrics:
    enabled: true
    serviceMonitor:
      enabled: true      # picks up by Prometheus

  ingress:
    enabled: true
    ingressClassName: alb
    annotations:
      alb.ingress.kubernetes.io/scheme: internet-facing
      alb.ingress.kubernetes.io/target-type: ip
      alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80},{"HTTPS":443}]'
      alb.ingress.kubernetes.io/ssl-redirect: "443"
      alb.ingress.kubernetes.io/certificate-arn: "ACM_CERT_ARN"
      alb.ingress.kubernetes.io/healthcheck-path: /healthz
    hosts:
      - argocd.yourdomain.com
    tls:
      - hosts:
          - argocd.yourdomain.com

# Application Controller
controller:
  replicas: 1

  autoscaling:
    enabled: true
    minReplicas: 1
    maxReplicas: 3

  resources:
    requests:
      cpu: 250m
      memory: 256Mi
    limits:
      cpu: 2000m
      memory: 2Gi

  metrics:
    enabled: true
    serviceMonitor:
      enabled: true

# Repo Server
repoServer:
  replicas: 2

  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 5

  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 1000m
      memory: 1Gi

  metrics:
    enabled: true
    serviceMonitor:
      enabled: true

  # Enable Helm secrets plugin
  env:
    - name: HELM_PLUGINS
      value: /gitops-tools/helm-secrets/plugins/helm-secrets
    - name: HELM_SECRETS_CURL_PATH
      value: /gitops-tools/curl
    - name: HELM_SECRETS_BACKEND
      value: awssecrets

# Redis HA for production
redis-ha:
  enabled: true
  redis:
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 500m
        memory: 512Mi

# ApplicationSet controller
applicationSet:
  replicas: 2
  metrics:
    enabled: true
    serviceMonitor:
      enabled: true
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi

# Notifications controller
notifications:
  enabled: true
  metrics:
    enabled: true
    serviceMonitor:
      enabled: true
  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 500m
      memory: 512Mi

  # Notification templates
  notifiers:
    service.slack: |
      token: $slack-token

  templates:
    template.app-deployed: |
      email:
        subject: "✅ {{.app.metadata.name}} deployed to {{.app.spec.destination.namespace}}"
      message: |
        ✅ *{{.app.metadata.name}}* synced successfully
        Revision: {{.app.status.sync.revision}}
        Environment: {{.app.spec.destination.namespace}}
    template.app-health-degraded: |
      email:
        subject: "🚨 {{.app.metadata.name}} health degraded"
      message: |
        🚨 *{{.app.metadata.name}}* health is *{{.app.status.health.status}}*
        Message: {{.app.status.health.message}}
    template.app-sync-failed: |
      email:
        subject: "❌ {{.app.metadata.name}} sync failed"
      message: |
        ❌ *{{.app.metadata.name}}* sync failed
        Error: {{.app.status.operationState.message}}

  triggers:
    trigger.on-deployed: |
      - description: Application synced successfully
        oncePer: app.status.operationState.syncResult.revision
        send: [app-deployed]
        when: app.status.operationState.phase in ['Succeeded']
          and app.status.health.status == 'Healthy'
    trigger.on-health-degraded: |
      - description: Application health degraded
        send: [app-health-degraded]
        when: app.status.health.status == 'Degraded'
    trigger.on-sync-failed: |
      - description: Application sync failed
        send: [app-sync-failed]
        when: app.status.operationState.phase in ['Error', 'Failed']

  subscriptions:
    - recipients:
        - slack:platform-alerts
      triggers:
        - on-sync-failed
        - on-health-degraded
    - recipients:
        - slack:deployments
      triggers:
        - on-deployed
EOF
```

**Install ArgoCD**

```
# Add ArgoCD Helm repo
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# Create namespace
kubectl create namespace argocd

# Create ArgoCD admin secret
kubectl create secret generic argocd-github-sso \
  --namespace argocd \
  --from-literal=client-id="your-github-oauth-app-client-id" \
  --from-literal=client-secret="your-github-oauth-app-client-secret"

# Install ArgoCD
helm upgrade --install argocd argo/argo-cd \
  --namespace argocd \
  --version 7.3.11 \
  --values kubernetes/platform/argocd/values.yaml \
  --wait \
  --timeout 10m

# Get initial admin password
kubectl get secret argocd-initial-admin-secret \
  -n argocd \
  -o jsonpath='{.data.password}' | base64 -d
echo

# Port-forward to access UI locally
kubectl port-forward svc/argocd-server -n argocd 8080:443 &

# Login via CLI
argocd login localhost:8080 \
  --username admin \
  --password $(kubectl get secret argocd-initial-admin-secret \
    -n argocd \
    -o jsonpath='{.data.password}' | base64 -d) \
  --insecure

# Change admin password
argocd account update-password \
  --current-password $(kubectl get secret argocd-initial-admin-secret \
    -n argocd \
    -o jsonpath='{.data.password}' | base64 -d) \
  --new-password "YourSecurePassword123!"

# Add GitHub repo to ArgoCD
argocd repo add https://github.com/YOUR_USERNAME/cloudforge \
  --username YOUR_USERNAME \
  --password YOUR_GITHUB_PAT \
  --name cloudforge

# Verify
argocd repo list
```

## PART 2 — App of Apps Pattern

The App of Apps pattern is how production platforms manage ArgoCD at scale. One root Application manages all other Applications. Add a new service = add one YAML file to git. ArgoCD picks it up automatically.

```
App of Apps Pattern:
                    ┌─────────────────┐
                    │  root-app       │  ← ONE Application
                    │  (App of Apps)  │    pointing to /kubernetes/applications/
                    └────────┬────────┘
                             │ manages
          ┌──────────────────┼──────────────────┐
          ▼                  ▼                  ▼
  ┌───────────────┐  ┌───────────────┐  ┌───────────────┐
  │ platform-apps │  │ api-service   │  │ worker-service│
  │ (ArgoCD App)  │  │ (ArgoCD App)  │  │ (ArgoCD App)  │
  └───────┬───────┘  └───────┬───────┘  └───────┬───────┘
          │ manages           │ deploys           │ deploys
          ▼                  ▼                  ▼
  ArgoCD, Prometheus,   api-service pods   worker pods
  Grafana, Loki...      in staging/prod    in staging/prod
```

**Root Application**

```
cat > kubernetes/applications/root-app.yaml << 'EOF'
# ═══════════════════════════════════════════════════════
# Root Application — App of Apps Pattern
# This single Application manages all other Applications.
# Adding a new app = add a YAML file to /kubernetes/applications/
# ═══════════════════════════════════════════════════════
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
  # Cascade delete — removing root-app removes all child apps
  finalizers:
    - resources-finalizer.argocd.argoproj.io
  annotations:
    notifications.argoproj.io/subscribe.on-sync-failed.slack: platform-alerts
    notifications.argoproj.io/subscribe.on-health-degraded.slack: platform-alerts
spec:
  project: default
  source:
    repoURL: https://github.com/YOUR_USERNAME/cloudforge
    targetRevision: main
    path: kubernetes/applications
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true       # remove resources deleted from git
      selfHeal: true    # revert manual kubectl changes
      allowEmpty: false
    syncOptions:
      - CreateNamespace=true
      - PrunePropagationPolicy=foreground
      - PruneLast=true
    retry:
      limit: 5
      backoff:
        duration: 5s
        factor: 2
        maxDuration: 3m
EOF

# Apply the root app — this bootstraps everything
kubectl apply -f kubernetes/applications/root-app.yaml

# Watch it sync
argocd app get root-app
argocd app sync root-app
```

**Platform Applications (managed by root-app)**

```
cat > kubernetes/applications/platform-apps.yaml << 'EOF'
# ═══════════════════════════════════════════════════════
# Platform Applications
# All platform tooling managed as ArgoCD Applications
# ═══════════════════════════════════════════════════════

# ── AWS Load Balancer Controller ──────────────────────────
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: aws-load-balancer-controller
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: platform
  source:
    repoURL: https://aws.github.io/eks-charts
    chart: aws-load-balancer-controller
    targetRevision: "1.8.1"
    helm:
      valuesObject:
        clusterName: cloudforge-staging
        serviceAccount:
          create: false
          name: aws-load-balancer-controller
        region: ap-south-1
        vpcId: VPC_ID_FROM_TERRAFORM_OUTPUT
        replicaCount: 2
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        podDisruptionBudget:
          maxUnavailable: 1
        affinity:
          podAntiAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:
              - labelSelector:
                  matchLabels:
                    app.kubernetes.io/name: aws-load-balancer-controller
                topologyKey: kubernetes.io/hostname
  destination:
    server: https://kubernetes.default.svc
    namespace: kube-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true

---
# ── External DNS ──────────────────────────────────────────
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: external-dns
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: platform
  source:
    repoURL: https://kubernetes-sigs.github.io/external-dns/
    chart: external-dns
    targetRevision: "1.14.5"
    helm:
      valuesObject:
        provider: aws
        aws:
          region: ap-south-1
        serviceAccount:
          create: false
          name: external-dns
        policy: upsert-only
        registry: txt
        txtOwnerId: cloudforge-staging
        domainFilters:
          - yourdomain.com
        sources:
          - service
          - ingress
        interval: 1m
  destination:
    server: https://kubernetes.default.svc
    namespace: external-dns
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true

---
# ── Cert Manager ─────────────────────────────────────────
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: cert-manager
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: platform
  source:
    repoURL: https://charts.jetstack.io
    chart: cert-manager
    targetRevision: "v1.15.1"
    helm:
      valuesObject:
        installCRDs: true
        replicaCount: 2
        serviceAccount:
          annotations:
            eks.amazonaws.com/role-arn: CERT_MANAGER_ROLE_ARN
        resources:
          requests:
            cpu: 10m
            memory: 32Mi
  destination:
    server: https://kubernetes.default.svc
    namespace: cert-manager
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true

---
# ── External Secrets Operator ─────────────────────────────
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: external-secrets
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: platform
  source:
    repoURL: https://charts.external-secrets.io
    chart: external-secrets
    targetRevision: "0.10.0"
    helm:
      valuesObject:
        installCRDs: true
        replicaCount: 2
        serviceAccount:
          annotations:
            eks.amazonaws.com/role-arn: ESO_ROLE_ARN
        resources:
          requests:
            cpu: 10m
            memory: 64Mi
  destination:
    server: https://kubernetes.default.svc
    namespace: external-secrets
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true

---
# ── Metrics Server ────────────────────────────────────────
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: metrics-server
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: platform
  source:
    repoURL: https://kubernetes-sigs.github.io/metrics-server/
    chart: metrics-server
    targetRevision: "3.12.1"
    helm:
      valuesObject:
        replicas: 2
        args:
          - --cert-dir=/tmp
          - --kubelet-preferred-address-types=InternalIP
          - --kubelet-use-node-status-port
          - --metric-resolution=15s
  destination:
    server: https://kubernetes.default.svc
    namespace: kube-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF
```

## PART 3 — ArgoCD Projects (RBAC Isolation)

```
cat > kubernetes/platform/argocd/projects.yaml << 'EOF'
# ═══════════════════════════════════════════════════════
# ArgoCD Projects — Isolate platform vs application teams
# ═══════════════════════════════════════════════════════

# Platform project — only platform team can deploy here
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: platform
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  description: "CloudForge platform infrastructure"
  sourceRepos:
    - 'https://github.com/YOUR_USERNAME/cloudforge'
    - 'https://aws.github.io/eks-charts'
    - 'https://charts.jetstack.io'
    - 'https://charts.external-secrets.io'
    - 'https://kubernetes-sigs.github.io/metrics-server/'
    - 'https://kubernetes-sigs.github.io/external-dns/'
    - 'https://argoproj.github.io/argo-helm'
    - 'https://prometheus-community.github.io/helm-charts'
    - 'https://grafana.github.io/helm-charts'
  destinations:
    - namespace: 'kube-system'
      server: 'https://kubernetes.default.svc'
    - namespace: 'argocd'
      server: 'https://kubernetes.default.svc'
    - namespace: 'monitoring'
      server: 'https://kubernetes.default.svc'
    - namespace: 'logging'
      server: 'https://kubernetes.default.svc'
    - namespace: 'cert-manager'
      server: 'https://kubernetes.default.svc'
    - namespace: 'external-dns'
      server: 'https://kubernetes.default.svc'
    - namespace: 'external-secrets'
      server: 'https://kubernetes.default.svc'
    - namespace: 'karpenter'
      server: 'https://kubernetes.default.svc'
  clusterResourceWhitelist:
    - group: '*'
      kind: '*'
  roles:
    - name: platform-admin
      policies:
        - p, proj:platform:platform-admin, applications, *, platform/*, allow
      groups:
        - platform-team

---
# Applications project — dev team deploys here
apiVersion: argoproj.io/v1alpha1
kind: AppProject
metadata:
  name: apps
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  description: "CloudForge application workloads"
  sourceRepos:
    - 'https://github.com/YOUR_USERNAME/cloudforge'
  destinations:
    - namespace: 'staging'
      server: 'https://kubernetes.default.svc'
    - namespace: 'production'
      server: 'https://kubernetes.default.svc'
  namespaceResourceWhitelist:
    - group: 'apps'
      kind: 'Deployment'
    - group: 'apps'
      kind: 'StatefulSet'
    - group: ''
      kind: 'Service'
    - group: ''
      kind: 'ConfigMap'
    - group: 'networking.k8s.io'
      kind: 'Ingress'
    - group: 'autoscaling'
      kind: 'HorizontalPodAutoscaler'
    - group: 'policy'
      kind: 'PodDisruptionBudget'
    - group: 'argoproj.io'
      kind: 'Rollout'
  clusterResourceWhitelist:
    - group: ''
      kind: 'Namespace'
  roles:
    - name: developer
      policies:
        - p, proj:apps:developer, applications, get, apps/*, allow
        - p, proj:apps:developer, applications, sync, apps/*, allow
      groups:
        - developers
EOF

kubectl apply -f kubernetes/platform/argocd/projects.yaml
```

## PART 4 — ApplicationSets (Multi-Environment Pattern)

ApplicationSets are the most powerful ArgoCD feature. One template → generates Applications for every environment automatically.

```
cat > kubernetes/applications/api-service-appset.yaml << 'EOF'
# ═══════════════════════════════════════════════════════
# ApplicationSet — api-service
# One template → deploys to staging AND production
# Add a new environment = add one line to the list
# ═══════════════════════════════════════════════════════
apiVersion: argoproj.io/v1alpha1
kind: ApplicationSet
metadata:
  name: api-service
  namespace: argocd
spec:
  # Generate one Application per environment
  generators:
    - list:
        elements:
          - environment: staging
            namespace: staging
            cluster: https://kubernetes.default.svc
            replicaCount: "2"
            minReplicas: "2"
            maxReplicas: "10"
            resources_cpu_request: "100m"
            resources_mem_request: "128Mi"
            resources_cpu_limit: "500m"
            resources_mem_limit: "512Mi"
            canaryWeight: "20"
            autosync: "true"

          - environment: production
            namespace: production
            cluster: https://kubernetes.default.svc
            replicaCount: "5"
            minReplicas: "5"
            maxReplicas: "50"
            resources_cpu_request: "250m"
            resources_mem_request: "512Mi"
            resources_cpu_limit: "1000m"
            resources_mem_limit: "1Gi"
            canaryWeight: "10"
            autosync: "false"   # production requires manual sync approval

  template:
    metadata:
      name: 'api-service-{{environment}}'
      namespace: argocd
      finalizers:
        - resources-finalizer.argocd.argoproj.io
      annotations:
        notifications.argoproj.io/subscribe.on-deployed.slack: deployments
        notifications.argoproj.io/subscribe.on-sync-failed.slack: platform-alerts
        notifications.argoproj.io/subscribe.on-health-degraded.slack: platform-alerts
    spec:
      project: apps
      source:
        repoURL: https://github.com/YOUR_USERNAME/cloudforge
        targetRevision: main
        path: helm/cloudforge-app
        helm:
          releaseName: 'api-service-{{environment}}'
          valueFiles:
            - values.yaml
            - 'values-{{environment}}.yaml'
          parameters:
            - name: environment
              value: '{{environment}}'
            - name: replicaCount
              value: '{{replicaCount}}'
            - name: autoscaling.minReplicas
              value: '{{minReplicas}}'
            - name: autoscaling.maxReplicas
              value: '{{maxReplicas}}'
            - name: resources.requests.cpu
              value: '{{resources_cpu_request}}'
            - name: resources.requests.memory
              value: '{{resources_mem_request}}'
            - name: rollout.canaryWeight
              value: '{{canaryWeight}}'
      destination:
        server: '{{cluster}}'
        namespace: '{{namespace}}'
      syncPolicy:
        automated:
          prune: true
          selfHeal: true
          allowEmpty: false
        syncOptions:
          - CreateNamespace=true
          - PrunePropagationPolicy=foreground
          - RespectIgnoreDifferences=true
        retry:
          limit: 3
          backoff:
            duration: 5s
            factor: 2
            maxDuration: 2m
      ignoreDifferences:
        - group: apps
          kind: Deployment
          jsonPointers:
            - /spec/replicas   # HPA manages replicas — ignore drift
EOF
```

## PART 5 — Argo Rollouts (Progressive Delivery)

Argo Rollouts replaces Kubernetes Deployments with advanced deployment strategies.

**Install Argo Rollouts**

```
cat > kubernetes/applications/argo-rollouts.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argo-rollouts
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: platform
  source:
    repoURL: https://argoproj.github.io/argo-helm
    chart: argo-rollouts
    targetRevision: "2.37.5"
    helm:
      valuesObject:
        replicas: 2
        serviceAccount:
          create: true
        dashboard:
          enabled: true
          service:
            type: ClusterIP
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
        metrics:
          enabled: true
          serviceMonitor:
            enabled: true
  destination:
    server: https://kubernetes.default.svc
    namespace: argo-rollouts
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF

kubectl apply -f kubernetes/applications/argo-rollouts.yaml
```

**Canary Rollout for API Service**

```
mkdir -p kubernetes/platform/rollouts

cat > kubernetes/platform/rollouts/api-service-rollout.yaml << 'EOF'
# ═══════════════════════════════════════════════════════
# Argo Rollout — Canary Deployment Strategy
# Replaces standard Deployment for the api-service
# ═══════════════════════════════════════════════════════
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: api-service
  namespace: staging
  labels:
    app: api-service
    version: v1.0.0
spec:
  replicas: 5
  revisionHistoryLimit: 3
  selector:
    matchLabels:
      app: api-service

  template:
    metadata:
      labels:
        app: api-service
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8000"
        prometheus.io/path: "/metrics"
    spec:
      serviceAccountName: api-service-sa
      terminationGracePeriodSeconds: 60

      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: api-service

      containers:
        - name: api
          image: ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com/cloudforge/api-service:latest
          imagePullPolicy: Always
          ports:
            - containerPort: 8000
              name: http

          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 500m
              memory: 512Mi

          livenessProbe:
            httpGet:
              path: /health/live
              port: 8000
            initialDelaySeconds: 30
            periodSeconds: 10
            failureThreshold: 3

          readinessProbe:
            httpGet:
              path: /health/ready
              port: 8000
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 3

          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            runAsNonRoot: true
            runAsUser: 1001
            capabilities:
              drop: ["ALL"]

          volumeMounts:
            - name: tmp
              mountPath: /tmp

          lifecycle:
            preStop:
              exec:
                command: ["/bin/sh", "-c", "sleep 10"]

      volumes:
        - name: tmp
          emptyDir: {}

      securityContext:
        fsGroup: 1001

  strategy:
    canary:
      # Canary service receives canary traffic
      canaryService: api-service-canary
      # Stable service receives stable traffic
      stableService: api-service-stable

      trafficRouting:
        alb:
          ingress: api-service-ingress
          servicePort: 80

      # Analysis runs automatically at each step
      analysis:
        templates:
          - templateName: api-service-success-rate
        startingStep: 2
        args:
          - name: service-name
            value: api-service-canary

      steps:
        # Step 1: send 10% traffic to canary
        - setWeight: 10
        # Step 2: run analysis for 5 minutes
        - pause:
            duration: 5m
        # Step 3: if analysis passed, go to 25%
        - setWeight: 25
        - pause:
            duration: 5m
        # Step 4: half and half
        - setWeight: 50
        - pause:
            duration: 5m
        # Step 5: promote to 100% (old pods terminated)
        - setWeight: 100

      # Anti-affinity: spread canary pods across different nodes than stable
      antiAffinity:
        requiredDuringSchedulingIgnoredDuringExecution: {}

---
# Stable service (receives established traffic)
apiVersion: v1
kind: Service
metadata:
  name: api-service-stable
  namespace: staging
spec:
  selector:
    app: api-service
  ports:
    - port: 80
      targetPort: 8000
  type: ClusterIP

---
# Canary service (receives canary traffic)
apiVersion: v1
kind: Service
metadata:
  name: api-service-canary
  namespace: staging
spec:
  selector:
    app: api-service
  ports:
    - port: 80
      targetPort: 8000
  type: ClusterIP

---
# AnalysisTemplate — defines what "success" means
apiVersion: argoproj.io/v1alpha1
kind: AnalysisTemplate
metadata:
  name: api-service-success-rate
  namespace: staging
spec:
  args:
    - name: service-name
  metrics:
    # Metric 1: error rate must be < 5%
    - name: success-rate
      interval: 1m
      successCondition: result[0] >= 0.95
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus-server.monitoring.svc.cluster.local
          query: |
            sum(rate(
              http_requests_total{
                service="{{args.service-name}}",
                status!~"5.."
              }[5m]
            )) /
            sum(rate(
              http_requests_total{
                service="{{args.service-name}}"
              }[5m]
            ))

    # Metric 2: p99 latency must be < 500ms
    - name: latency-p99
      interval: 1m
      successCondition: result[0] <= 0.5
      failureLimit: 3
      provider:
        prometheus:
          address: http://prometheus-server.monitoring.svc.cluster.local
          query: |
            histogram_quantile(0.99,
              sum(rate(
                http_request_duration_seconds_bucket{
                  service="{{args.service-name}}"
                }[5m]
              )) by (le)
            )

    # Metric 3: CloudWatch ALB 5xx rate
    - name: alb-5xx-rate
      interval: 1m
      successCondition: result[0] <= 0.01
      failureLimit: 2
      provider:
        cloudWatch:
          dimensionsByName:
            LoadBalancer: "app/cloudforge-alb/*"
            TargetGroup: "targetgroup/api-service-canary/*"
          metricName: HTTPCode_Target_5XX_Count
          namespace: AWS/ApplicationELB
          periodSeconds: 60
          statistic: Sum
EOF

kubectl apply -f kubernetes/platform/rollouts/api-service-rollout.yaml
```

## PART 6 — ArgoCD Image Updater

Image Updater watches ECR, detects new image tags, commits the update to git, and ArgoCD deploys automatically.

```
cat > kubernetes/applications/argocd-image-updater.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: argocd-image-updater
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: platform
  source:
    repoURL: https://argoproj.github.io/argo-helm
    chart: argocd-image-updater
    targetRevision: "0.10.1"
    helm:
      valuesObject:
        replicaCount: 2

        config:
          # Check for updates every 2 minutes
          interval: 2m
          logLevel: info

          # AWS ECR authentication
          registries:
            - name: ECR
              api_url: https://ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com
              prefix: ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com
              ping: true
              insecure: false
              credentials: ext:/scripts/ecr-auth.sh
              credsexpire: 10h

        authScripts:
          enabled: true
          scripts:
            ecr-auth.sh: |
              #!/bin/sh
              aws ecr get-login-password --region ap-south-1 | \
                awk '{print "AWS:" $0}'

        serviceAccount:
          annotations:
            eks.amazonaws.com/role-arn: IMAGE_UPDATER_ROLE_ARN

        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi

        metrics:
          enabled: true
          serviceMonitor:
            enabled: true

        sshConfig:
          config: |
            Host github.com
              User git
              IdentityFile ~/.ssh/id_rsa
              IdentitiesOnly yes
              StrictHostKeyChecking no
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
EOF

kubectl apply -f kubernetes/applications/argocd-image-updater.yaml
```

**Configure Image Updater on ApplicationSet**

Add these annotations to your ApplicationSet to enable auto-updates:

```
# These annotations on the Application tell Image Updater
# what images to watch and how to update git

cat >> kubernetes/applications/api-service-appset.yaml << 'EOF'

# Add to the ApplicationSet template metadata annotations:
# argocd-image-updater.argoproj.io/image-list: |
#   api=ACCOUNT_ID.dkr.ecr.ap-south-1.amazonaws.com/cloudforge/api-service
# argocd-image-updater.argoproj.io/api.update-strategy: semver
# argocd-image-updater.argoproj.io/api.allow-tags: regexp:^v[0-9]+\.[0-9]+\.[0-9]+$
# argocd-image-updater.argoproj.io/write-back-method: git
# argocd-image-updater.argoproj.io/git-branch: main
EOF
```

## PART 7 — Custom Helm Chart (cloudforge-app)

Every service uses this chart. One chart, all services. Change the chart = all services get the update.

```
helm create helm/cloudforge-app
# Remove the default templates — replace with our own

rm -rf helm/cloudforge-app/templates/*
rm helm/cloudforge-app/values.yaml

cat > helm/cloudforge-app/Chart.yaml << 'EOF'
apiVersion: v2
name: cloudforge-app
description: Generic application chart for CloudForge platform
type: application
version: 0.1.0
appVersion: "1.0.0"
keywords:
  - cloudforge
  - devops
  - platform
maintainers:
  - name: Chetan
    email: chetan@company.com
EOF

cat > helm/cloudforge-app/values.yaml << 'EOF'
# ═══════════════════════════════════════════════════════
# CloudForge App Chart — Default Values
# Override per environment in values-staging.yaml etc.
# ═══════════════════════════════════════════════════════

# Required — override per service
image:
  repository: ""
  tag: "latest"
  pullPolicy: Always

nameOverride: ""
fullnameOverride: ""

environment: staging
replicaCount: 2

# Service account with IRSA annotation
serviceAccount:
  create: true
  annotations: {}
  name: ""

# Pod-level security
podSecurityContext:
  fsGroup: 1001
  seccompProfile:
    type: RuntimeDefault

# Container-level security
securityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  runAsNonRoot: true
  runAsUser: 1001
  runAsGroup: 1001
  capabilities:
    drop: ["ALL"]

# Resources — override per environment
resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

# Service configuration
service:
  type: ClusterIP
  port: 80
  targetPort: 8000
  annotations: {}

# Ingress
ingress:
  enabled: true
  className: alb
  annotations:
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/listen-ports: '[{"HTTP":80},{"HTTPS":443}]'
    alb.ingress.kubernetes.io/ssl-redirect: "443"
  hosts: []
  tls: []

# HPA
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 20
  targetCPUUtilizationPercentage: 60
  targetMemoryUtilizationPercentage: 75
  behavior:
    scaleOut:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 4
          periodSeconds: 60
    scaleIn:
      stabilizationWindowSeconds: 300

# PodDisruptionBudget
podDisruptionBudget:
  enabled: true
  minAvailable: 1

# Topology spread
topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector: {}

# Health probes
livenessProbe:
  httpGet:
    path: /health/live
    port: 8000
  initialDelaySeconds: 30
  periodSeconds: 10
  failureThreshold: 3
  timeoutSeconds: 5

readinessProbe:
  httpGet:
    path: /health/ready
    port: 8000
  initialDelaySeconds: 10
  periodSeconds: 5
  failureThreshold: 3
  timeoutSeconds: 3

startupProbe:
  httpGet:
    path: /health/live
    port: 8000
  failureThreshold: 30
  periodSeconds: 5

# ConfigMap values injected as env vars
config: {}

# Secrets injected from External Secrets Operator
externalSecrets:
  enabled: false
  secretStoreRef:
    name: aws-secrets-store
    kind: ClusterSecretStore
  secrets: []

# Argo Rollout strategy
rollout:
  enabled: false
  strategy: canary
  canaryWeight: 20

# Prometheus ServiceMonitor
monitoring:
  enabled: true
  path: /metrics
  port: 8000

# Additional volumes
extraVolumes: []
extraVolumeMounts: []

# Node selection
nodeSelector: {}
tolerations: []
affinity: {}

# Lifecycle hooks
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 10"]

terminationGracePeriodSeconds: 60
EOF
```

**Helm Chart Templates**

```
cat > helm/cloudforge-app/templates/deployment.yaml << 'EOF'
{{- if not .Values.rollout.enabled }}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "cloudforge-app.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "cloudforge-app.labels" . | nindent 4 }}
  annotations:
    deployment.kubernetes.io/revision: "{{ .Release.Revision }}"
spec:
  replicas: {{ .Values.replicaCount }}
  revisionHistoryLimit: 3
  selector:
    matchLabels:
      {{- include "cloudforge-app.selectorLabels" . | nindent 6 }}
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        {{- include "cloudforge-app.selectorLabels" . | nindent 8 }}
        version: {{ .Values.image.tag | quote }}
      annotations:
        prometheus.io/scrape: {{ .Values.monitoring.enabled | quote }}
        prometheus.io/port: {{ .Values.monitoring.port | quote }}
        prometheus.io/path: {{ .Values.monitoring.path | quote }}
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
    spec:
      serviceAccountName: {{ include "cloudforge-app.serviceAccountName" . }}
      terminationGracePeriodSeconds: {{ .Values.terminationGracePeriodSeconds }}
      securityContext:
        {{- toYaml .Values.podSecurityContext | nindent 8 }}

      topologySpreadConstraints:
        {{- range .Values.topologySpreadConstraints }}
        - maxSkew: {{ .maxSkew }}
          topologyKey: {{ .topologyKey }}
          whenUnsatisfiable: {{ .whenUnsatisfiable }}
          labelSelector:
            matchLabels:
              {{- include "cloudforge-app.selectorLabels" $ | nindent 14 }}
        {{- end }}

      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          imagePullPolicy: {{ .Values.image.pullPolicy }}
          ports:
            - containerPort: {{ .Values.service.targetPort }}
              name: http
              protocol: TCP

          {{- if .Values.config }}
          envFrom:
            - configMapRef:
                name: {{ include "cloudforge-app.fullname" . }}
          {{- end }}

          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
            - name: NODE_NAME
              valueFrom:
                fieldRef:
                  fieldPath: spec.nodeName
            - name: ENVIRONMENT
              value: {{ .Values.environment | quote }}

          resources:
            {{- toYaml .Values.resources | nindent 12 }}

          livenessProbe:
            {{- toYaml .Values.livenessProbe | nindent 12 }}

          readinessProbe:
            {{- toYaml .Values.readinessProbe | nindent 12 }}

          startupProbe:
            {{- toYaml .Values.startupProbe | nindent 12 }}

          securityContext:
            {{- toYaml .Values.securityContext | nindent 12 }}

          lifecycle:
            {{- toYaml .Values.lifecycle | nindent 12 }}

          volumeMounts:
            - name: tmp
              mountPath: /tmp
            {{- with .Values.extraVolumeMounts }}
            {{- toYaml . | nindent 12 }}
            {{- end }}

      volumes:
        - name: tmp
          emptyDir: {}
        {{- with .Values.extraVolumes }}
        {{- toYaml . | nindent 8 }}
        {{- end }}

      {{- with .Values.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
{{- end }}
EOF

cat > helm/cloudforge-app/templates/_helpers.tpl << 'EOF'
{{/*
Expand the name of the chart.
*/}}
{{- define "cloudforge-app.name" -}}
{{- default .Chart.Name .Values.nameOverride | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Create a default fully qualified app name.
*/}}
{{- define "cloudforge-app.fullname" -}}
{{- if .Values.fullnameOverride }}
{{- .Values.fullnameOverride | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- $name := default .Chart.Name .Values.nameOverride }}
{{- if contains $name .Release.Name }}
{{- .Release.Name | trunc 63 | trimSuffix "-" }}
{{- else }}
{{- printf "%s-%s" .Release.Name $name | trunc 63 | trimSuffix "-" }}
{{- end }}
{{- end }}
{{- end }}

{{/*
Common labels
*/}}
{{- define "cloudforge-app.labels" -}}
helm.sh/chart: {{ include "cloudforge-app.chart" . }}
{{ include "cloudforge-app.selectorLabels" . }}
app.kubernetes.io/version: {{ .Values.image.tag | quote }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
app.kubernetes.io/component: application
environment: {{ .Values.environment }}
{{- end }}

{{/*
Selector labels
*/}}
{{- define "cloudforge-app.selectorLabels" -}}
app.kubernetes.io/name: {{ include "cloudforge-app.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
{{- end }}

{{/*
Chart label
*/}}
{{- define "cloudforge-app.chart" -}}
{{- printf "%s-%s" .Chart.Name .Chart.Version | replace "+" "_" | trunc 63 | trimSuffix "-" }}
{{- end }}

{{/*
Service account name
*/}}
{{- define "cloudforge-app.serviceAccountName" -}}
{{- if .Values.serviceAccount.create }}
{{- default (include "cloudforge-app.fullname" .) .Values.serviceAccount.name }}
{{- else }}
{{- default "default" .Values.serviceAccount.name }}
{{- end }}
{{- end }}
EOF

# Generate remaining templates
for template in hpa.yaml pdb.yaml service.yaml serviceaccount.yaml configmap.yaml ingress.yaml servicemonitor.yaml; do
  touch helm/cloudforge-app/templates/$template
done

cat > helm/cloudforge-app/templates/hpa.yaml << 'EOF'
{{- if .Values.autoscaling.enabled }}
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: {{ include "cloudforge-app.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "cloudforge-app.labels" . | nindent 4 }}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: {{ include "cloudforge-app.fullname" . }}
  minReplicas: {{ .Values.autoscaling.minReplicas }}
  maxReplicas: {{ .Values.autoscaling.maxReplicas }}
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetCPUUtilizationPercentage }}
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: {{ .Values.autoscaling.targetMemoryUtilizationPercentage }}
  behavior:
    {{- toYaml .Values.autoscaling.behavior | nindent 4 }}
{{- end }}
EOF

cat > helm/cloudforge-app/templates/pdb.yaml << 'EOF'
{{- if .Values.podDisruptionBudget.enabled }}
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: {{ include "cloudforge-app.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "cloudforge-app.labels" . | nindent 4 }}
spec:
  minAvailable: {{ .Values.podDisruptionBudget.minAvailable }}
  selector:
    matchLabels:
      {{- include "cloudforge-app.selectorLabels" . | nindent 6 }}
{{- end }}
EOF

cat > helm/cloudforge-app/templates/servicemonitor.yaml << 'EOF'
{{- if .Values.monitoring.enabled }}
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: {{ include "cloudforge-app.fullname" . }}
  namespace: {{ .Release.Namespace }}
  labels:
    {{- include "cloudforge-app.labels" . | nindent 4 }}
    release: kube-prometheus-stack
spec:
  selector:
    matchLabels:
      {{- include "cloudforge-app.selectorLabels" . | nindent 6 }}
  endpoints:
    - port: http
      path: {{ .Values.monitoring.path }}
      interval: 30s
      scrapeTimeout: 10s
{{- end }}
EOF
```

## PART 8 — External Secrets Operator Setup

No hardcoded secrets. Ever. ESO pulls from AWS Secrets Manager at runtime.

```
cat > kubernetes/platform/external-secrets/cluster-secret-store.yaml << 'EOF'
# ═══════════════════════════════════════════════════════
# ClusterSecretStore — connects ESO to AWS Secrets Manager
# One store for the whole cluster
# ═══════════════════════════════════════════════════════
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: aws-secrets-store
spec:
  provider:
    aws:
      service: SecretsManager
      region: ap-south-1
      auth:
        jwt:
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets

---
# ExternalSecret for API service — pulls from AWS Secrets Manager
# and creates a Kubernetes Secret automatically
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: api-service-secrets
  namespace: staging
spec:
  refreshInterval: 1h    # re-sync from Secrets Manager every hour
  secretStoreRef:
    name: aws-secrets-store
    kind: ClusterSecretStore
  target:
    name: api-service-secrets   # resulting Kubernetes Secret name
    creationPolicy: Owner
    deletionPolicy: Retain
    template:
      type: Opaque
      data:
        # Transform the Secrets Manager JSON into individual keys
        DATABASE_URL: |
          postgresql://{{ .username }}:{{ .password }}@{{ .host }}:{{ .port }}/{{ .dbname }}
        REDIS_URL: "{{ .redis_url }}"
  data:
    - secretKey: username
      remoteRef:
        key: /cloudforge/staging/rds-credentials
        property: username
    - secretKey: password
      remoteRef:
        key: /cloudforge/staging/rds-credentials
        property: password
    - secretKey: host
      remoteRef:
        key: /cloudforge/staging/rds-credentials
        property: host
    - secretKey: port
      remoteRef:
        key: /cloudforge/staging/rds-credentials
        property: port
    - secretKey: dbname
      remoteRef:
        key: /cloudforge/staging/rds-credentials
        property: dbname
    - secretKey: redis_url
      remoteRef:
        key: /cloudforge/staging/redis-url

EOF

kubectl apply -f kubernetes/platform/external-secrets/cluster-secret-store.yaml
```

## PART 9 — GitHub Actions CI Pipeline

**CI Workflow — Build, Test, Scan, Push**

```
cat > .github/workflows/ci-api-service.yml << 'EOF'
name: CI — api-service

on:
  push:
    branches: [main, 'feat/**']
    paths:
      - 'services/api-service/**'
      - '.github/workflows/ci-api-service.yml'
  pull_request:
    paths:
      - 'services/api-service/**'

permissions:
  id-token: write
  contents: write       # Image Updater writes back to git
  pull-requests: write
  security-events: write

env:
  AWS_REGION: ap-south-1
  ECR_REPOSITORY: cloudforge/api-service
  SERVICE_PATH: services/api-service

jobs:
  test:
    name: Test
    runs-on: ubuntu-latest
    defaults:
      run:
        working-directory: ${{ env.SERVICE_PATH }}

    steps:
      - uses: actions/checkout@v4

      - name: Set up Python
        uses: actions/setup-python@v5
        with:
          python-version: '3.12'
          cache: 'pip'

      - name: Install dependencies
        run: |
          pip install --upgrade pip
          pip install -r requirements.txt
          pip install -r requirements-dev.txt

      - name: Lint (ruff)
        run: ruff check . --output-format=github

      - name: Type check (mypy)
        run: mypy app/ --ignore-missing-imports

      - name: Security scan (bandit)
        run: |
          bandit -r app/ -ll -f json -o bandit-report.json
          bandit -r app/ -ll

      - name: Dependency vulnerability scan (safety)
        run: safety check --json > safety-report.json || true

      - name: Unit tests
        run: |
          pytest tests/unit \
            -v \
            --cov=app \
            --cov-report=xml \
            --cov-report=term-missing \
            --junitxml=test-results.xml

      - name: Coverage gate (>= 80%)
        run: |
          COVERAGE=$(python -c "
          import xml.etree.ElementTree as ET
          tree = ET.parse('coverage.xml')
          rate = float(tree.getroot().get('line-rate')) * 100
          print(int(rate))
          ")
          echo "Coverage: ${COVERAGE}%"
          [ "$COVERAGE" -ge 80 ] || \
            (echo "❌ Coverage ${COVERAGE}% is below 80% threshold" && exit 1)

      - name: Publish test results
        uses: EnricoMi/publish-unit-test-result-action@v2
        if: always()
        with:
          junit_files: "${{ env.SERVICE_PATH }}/test-results.xml"

  build:
    name: Build & Push
    runs-on: ubuntu-latest
    needs: test
    if: github.ref == 'refs/heads/main'
    outputs:
      image-tag: ${{ steps.meta.outputs.version }}
      image-uri: ${{ steps.build.outputs.image-uri }}

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_CI_ROLE_ARN }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Generate image metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}
          tags: |
            type=semver,pattern={{version}}
            type=sha,prefix=sha-,format=short
            type=raw,value=latest,enable={{is_default_branch}}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Build and push
        id: build
        uses: docker/build-push-action@v6
        with:
          context: ${{ env.SERVICE_PATH }}
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          build-args: |
            BUILD_DATE=${{ fromJSON(steps.meta.outputs.json).labels['org.opencontainers.image.created'] }}
            GIT_SHA=${{ github.sha }}
            VERSION=${{ steps.meta.outputs.version }}
          outputs: |
            image-uri=${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ steps.meta.outputs.version }}

      - name: Container vulnerability scan (Trivy)
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: >-
            ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ steps.meta.outputs.version }}
          format: sarif
          output: trivy-results.sarif
          severity: CRITICAL,HIGH
          exit-code: 1   # fail on CRITICAL

      - name: Upload Trivy results to GitHub Security
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: trivy-results.sarif

      - name: Generate SBOM
        uses: anchore/sbom-action@v0
        with:
          image: >-
            ${{ steps.login-ecr.outputs.registry }}/${{ env.ECR_REPOSITORY }}:${{ steps.meta.outputs.version }}
          format: cyclonedx-json
          output-file: sbom.cyclonedx.json

      - name: Upload SBOM to S3
        run: |
          aws s3 cp sbom.cyclonedx.json \
            s3://cloudforge-artifacts-${{ secrets.AWS_ACCOUNT_ID }}/sbom/api-service/${{ steps.meta.outputs.version }}.json

  notify:
    name: Notify
    runs-on: ubuntu-latest
    needs: [test, build]
    if: always()

    steps:
      - name: Slack notification
        uses: slackapi/slack-github-action@v1
        with:
          payload: |
            {
              "text": "${{ needs.build.result == 'success' && '✅' || '❌' }} *api-service* build ${{ needs.build.result }}",
              "blocks": [{
                "type": "section",
                "text": {
                  "type": "mrkdwn",
                  "text": "${{ needs.build.result == 'success' && '✅' || '❌' }} *api-service* — `${{ needs.build.outputs.image-tag }}`\nBranch: `${{ github.ref_name }}`\nBy: ${{ github.actor }}\n<${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}|View Run>"
                }
              }]
            }
        env:
          SLACK_WEBHOOK_URL: ${{ secrets.SLACK_WEBHOOK_URL }}
EOF
```

**PART 10 — Week 2 Validation**

```
cat > scripts/validate-week2.sh << 'EOF'
#!/bin/bash
set -euo pipefail

echo "═══════════════════════════════════════"
echo "  CloudForge Week 2 Validation"
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
echo "── ArgoCD ──────────────────────────────"
check "ArgoCD server running" \
  "kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server --no-headers | grep -q Running"
check "ArgoCD app controller running" \
  "kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-application-controller --no-headers | grep -q Running"
check "ArgoCD repo server running" \
  "kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-repo-server --no-headers | grep -q Running"
check "ArgoCD ApplicationSet controller" \
  "kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-applicationset-controller --no-headers | grep -q Running"
check "ArgoCD notifications controller" \
  "kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-notifications-controller --no-headers | grep -q Running"

echo ""
echo "── GitOps Applications ──────────────────"
check "root-app exists" \
  "argocd app get root-app &>/dev/null"
check "root-app synced" \
  "argocd app get root-app -o json | python3 -c \"import sys,json; a=json.load(sys.stdin); exit(0 if a['status']['sync']['status']=='Synced' else 1)\""
check "aws-load-balancer-controller synced" \
  "argocd app get aws-load-balancer-controller -o json | python3 -c \"import sys,json; a=json.load(sys.stdin); exit(0 if a['status']['sync']['status']=='Synced' else 1)\""
check "cert-manager synced" \
  "argocd app get cert-manager -o json | python3 -c \"import sys,json; a=json.load(sys.stdin); exit(0 if a['status']['sync']['status']=='Synced' else 1)\""
check "external-secrets synced" \
  "argocd app get external-secrets -o json | python3 -c \"import sys,json; a=json.load(sys.stdin); exit(0 if a['status']['sync']['status']=='Synced' else 1)\""

echo ""
echo "── Argo Rollouts ────────────────────────"
check "Rollouts controller running" \
  "kubectl get pods -n argo-rollouts -l app.kubernetes.io/name=argo-rollouts --no-headers | grep -q Running"
check "Rollout CRD exists" \
  "kubectl get crd rollouts.argoproj.io &>/dev/null"
check "AnalysisTemplate CRD exists" \
  "kubectl get crd analysistemplates.argoproj.io &>/dev/null"

echo ""
echo "── Image Updater ────────────────────────"
check "Image Updater running" \
  "kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-image-updater --no-headers | grep -q Running"

echo ""
echo "── External Secrets ─────────────────────"
check "ESO running" \
  "kubectl get pods -n external-secrets -l app.kubernetes.io/name=external-secrets --no-headers | grep -q Running"
check "ClusterSecretStore ready" \
  "kubectl get clustersecretstore aws-secrets-store -o jsonpath='{.status.conditions[0].status}' | grep -q True"

echo ""
echo "── Helm Chart ───────────────────────────"
check "cloudforge-app chart valid" \
  "helm lint helm/cloudforge-app"

echo ""
echo "── ArgoCD Projects ──────────────────────"
check "platform project exists" \
  "argocd proj get platform &>/dev/null"
check "apps project exists" \
  "argocd proj get apps &>/dev/null"

echo ""
echo "═══════════════════════════════════════"
echo "  Results: ✅ $PASS passed | ❌ $FAIL failed"
echo "═══════════════════════════════════════"

[ $FAIL -eq 0 ] && echo "  🎉 Week 2 Complete!" \
  || echo "  ⚠️  Fix failures before Week 3"
EOF

chmod +x scripts/validate-week2.sh
./scripts/validate-week2.sh
```

## Week 2 Commit

```
git add .
git commit -m "feat: Week 2 — ArgoCD GitOps + Argo Rollouts

GitOps:
- ArgoCD production install (HA, notifications, RBAC, SSO-ready)
- App of Apps pattern (root-app manages all applications)
- ArgoCD Projects (platform vs apps isolation)
- ApplicationSet for multi-environment api-service deployment
- ArgoCD Image Updater (auto ECR → git → deploy loop)

Progressive Delivery:
- Argo Rollouts canary strategy (10% → 25% → 50% → 100%)
- AnalysisTemplate (Prometheus + CloudWatch success metrics)
- Automatic rollback on analysis failure

Platform Add-ons (managed via ArgoCD):
- AWS Load Balancer Controller
- External DNS (Route53 sync)
- Cert Manager (TLS automation)
- External Secrets Operator (Secrets Manager → K8s secrets)
- Metrics Server

Helm:
- cloudforge-app generic chart (used by all services)
- HPA, PDB, ServiceMonitor, RBAC included by default
- Per-environment value overrides

CI:
- Full GitHub Actions pipeline (test → lint → scan → push → SBOM)
- Trivy SARIF upload to GitHub Security tab
- Coverage gate (80% minimum)
- Slack notifications

Validated:
- ArgoCD HA running ✅
- root-app synced ✅
- All platform apps synced ✅
- Argo Rollouts ready ✅
- ESO ClusterSecretStore ready ✅"

git push origin feat/week1-foundation
gh pr create \
  --title "feat: Week 2 — ArgoCD GitOps Layer" \
  --body "Adds complete GitOps platform with ArgoCD, Argo Rollouts, and Image Updater"
```

## Week 2 Summary

```
What you built:
├── ✅ ArgoCD (HA, RBAC, notifications, SSO-ready)
├── ✅ App of Apps (self-managing application catalog)
├── ✅ ArgoCD Projects (team isolation)
├── ✅ ApplicationSets (one template → all environments)
├── ✅ Argo Rollouts (canary with Prometheus analysis)
├── ✅ ArgoCD Image Updater (ECR → git → deploy loop)
├── ✅ External Secrets Operator (zero hardcoded secrets)
├── ✅ cloudforge-app Helm chart (generic, reusable)
├── ✅ Full CI pipeline (test, lint, scan, SBOM, push)
└── ✅ Platform add-ons (ALB controller, cert-manager, external-dns)

What the full GitOps loop looks like now:
git push → GitHub Actions → ECR → Image Updater →
git commit (tag update) → ArgoCD detects → canary rollout →
Prometheus analysis → promote or rollback → Slack notification

What recruiters see:
├── ArgoCD App of Apps (production pattern)
├── Argo Rollouts canary (progressive delivery)
├── AnalysisTemplate (automated quality gates)
├── Image Updater (full GitOps automation)
├── ESO (zero hardcoded secrets)
└── SBOM generation (supply chain security)
```












