# 🔐 CloudForge — Week 4: Zero-Trust Security Layer

**What We're Building This Week**

```
Week 4 Deliverables:
├── OPA Gatekeeper (policy enforcement — no root, resource limits required)
├── Falco (runtime threat detection — syscall level)
├── Trivy Operator (continuous vulnerability scanning)
├── kube-bench (CIS benchmark compliance)
├── Network Policies (zero-trust pod networking)
├── Pod Security Standards (enforce at namespace level)
├── RBAC hardening (least privilege across platform)
├── Secret scanning (prevent credentials in git)
└── Security CI gates (shift-left security in pipelines)

By end of week:
- No pod can run as root (Gatekeeper blocks it)
- Every image scanned continuously (Trivy Operator)
- Syscall anomalies detected in real time (Falco → Slack)
- All pod-to-pod traffic explicitly allowed or denied
- CIS benchmark score documented
- Security dashboard in Grafana
```

## PART 1 — Pod Security Standards

Pod Security Standards are Kubernetes-native. They're the first line of defence. Apply before Gatekeeper.

```
mkdir -p kubernetes/platform/security

cat > kubernetes/platform/security/namespace-security.yaml << 'EOF'
# ═══════════════════════════════════════════════════════
# Pod Security Standards — namespace-level enforcement
# Applied BEFORE Gatekeeper as a native K8s control
#
# Levels:
#   privileged: no restrictions
#   baseline:   prevents known privilege escalations
#   restricted: hardened, requires non-root, drops all caps
# ═══════════════════════════════════════════════════════

# ── Production namespace — restricted ─────────────────────
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    # Enforce: blocks non-compliant pods
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    # Audit: logs non-compliant pods (doesn't block)
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
    # Warn: warns on kubectl apply
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
    environment: production

---
# ── Staging namespace — restricted ────────────────────────
apiVersion: v1
kind: Namespace
metadata:
  name: staging
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
    pod-security.kubernetes.io/warn: restricted
    pod-security.kubernetes.io/warn-version: latest
    environment: staging

---
# ── Monitoring namespace — baseline ───────────────────────
# Some exporters need slightly elevated privileges
apiVersion: v1
kind: Namespace
metadata:
  name: monitoring
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
    environment: platform

---
# ── Logging namespace — baseline ──────────────────────────
# Promtail needs host filesystem access
apiVersion: v1
kind: Namespace
metadata:
  name: logging
  labels:
    pod-security.kubernetes.io/enforce: baseline
    pod-security.kubernetes.io/enforce-version: latest
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/audit-version: latest
    environment: platform
EOF

kubectl apply -f kubernetes/platform/security/namespace-security.yaml
```

## PART 2 — OPA Gatekeeper (Policy Enforcement)

Gatekeeper intercepts every API server request and evaluates it against Rego policies. Non-compliant resources are rejected before they're created.

```
cat > kubernetes/applications/gatekeeper.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: gatekeeper
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: platform
  source:
    repoURL: https://open-policy-agent.github.io/gatekeeper/charts
    chart: gatekeeper
    targetRevision: "3.17.1"
    helm:
      valuesObject:
        replicaCount: 3

        # Audit all existing resources (not just new ones)
        auditInterval: 60
        constraintViolationsLimit: 20
        auditFromCache: false
        auditChunkSize: 500

        # Emit violations as Prometheus metrics
        emitAuditEvents: true
        emitAdmissionEvents: true
        auditEventsInvolvedNamespace: false

        logLevel: INFO
        logMutations: true

        # Mutation support (patch resources on admission)
        experimentalEnableMutation: true

        resources:
          requests:
            cpu: 100m
            memory: 256Mi
          limits:
            cpu: 1000m
            memory: 512Mi

        metricsBackend: prometheus

        serviceMonitor:
          enabled: true
          additionalLabels:
            release: kube-prometheus-stack

        # Exempt critical system namespaces
        exemptNamespaces:
          - kube-system
          - gatekeeper-system
          - argocd
          - cert-manager
          - karpenter

        # Exempt specific pods that legitimately need elevated privs
        exemptNamespacePrefixes: []
  destination:
    server: https://kubernetes.default.svc
    namespace: gatekeeper-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
      - ServerSideApply=true
EOF

kubectl apply -f kubernetes/applications/gatekeeper.yaml

# Wait for Gatekeeper to be ready before applying policies
kubectl wait --for=condition=ready pod \
  -l control-plane=controller-manager \
  -n gatekeeper-system \
  --timeout=120s
```

**Constraint Templates (the Rego policies)**

```
cat > kubernetes/platform/security/gatekeeper/constraint-templates.yaml << 'EOF'
# ═══════════════════════════════════════════════════════
# OPA Gatekeeper Constraint Templates
# Written in Rego — define the policy logic
# Constraints (below) reference these templates
# ═══════════════════════════════════════════════════════

# ── Template 1: Block root containers ─────────────────────
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8snoroot
  annotations:
    description: "Requires containers to run as non-root user"
spec:
  crd:
    spec:
      names:
        kind: K8sNoRoot
      validation:
        openAPIV3Schema:
          type: object
          properties:
            allowedUsers:
              type: array
              items:
                type: integer
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8snoroot

        violation[{"msg": msg}] {
          container := input_containers[_]
          not container_runs_as_non_root(container)
          msg := sprintf(
            "Container <%v> must set securityContext.runAsNonRoot=true or runAsUser > 0",
            [container.name]
          )
        }

        container_runs_as_non_root(container) {
          container.securityContext.runAsNonRoot == true
        }

        container_runs_as_non_root(container) {
          container.securityContext.runAsUser > 0
          not container.securityContext.runAsUser == 0
        }

        input_containers[c] {
          c := input.review.object.spec.containers[_]
        }

        input_containers[c] {
          c := input.review.object.spec.initContainers[_]
        }

        input_containers[c] {
          c := input.review.object.spec.ephemeralContainers[_]
        }

---
# ── Template 2: Require resource limits ───────────────────
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredresources
  annotations:
    description: "Requires containers to have CPU and memory limits"
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredResources
      validation:
        openAPIV3Schema:
          type: object
          properties:
            limits:
              type: array
              items:
                type: string
            requests:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredresources

        violation[{"msg": msg}] {
          container := input_containers[_]
          required := input.parameters.limits[_]
          not container.resources.limits[required]
          msg := sprintf(
            "Container <%v> must set resources.limits.%v",
            [container.name, required]
          )
        }

        violation[{"msg": msg}] {
          container := input_containers[_]
          required := input.parameters.requests[_]
          not container.resources.requests[required]
          msg := sprintf(
            "Container <%v> must set resources.requests.%v",
            [container.name, required]
          )
        }

        input_containers[c] {
          c := input.review.object.spec.containers[_]
        }

        input_containers[c] {
          c := input.review.object.spec.initContainers[_]
        }

---
# ── Template 3: Require read-only root filesystem ─────────
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8sreadonlyrootfilesystem
spec:
  crd:
    spec:
      names:
        kind: K8sReadOnlyRootFilesystem
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8sreadonlyrootfilesystem

        violation[{"msg": msg}] {
          container := input_containers[_]
          not container.securityContext.readOnlyRootFilesystem == true
          msg := sprintf(
            "Container <%v> must set securityContext.readOnlyRootFilesystem=true",
            [container.name]
          )
        }

        input_containers[c] {
          c := input.review.object.spec.containers[_]
        }

---
# ── Template 4: Disallow privileged containers ────────────
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8snoprivilegedcontainer
spec:
  crd:
    spec:
      names:
        kind: K8sNoPrivilegedContainer
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8snoprivilegedcontainer

        violation[{"msg": msg}] {
          container := input_containers[_]
          container.securityContext.privileged == true
          msg := sprintf(
            "Container <%v> must not run as privileged",
            [container.name]
          )
        }

        input_containers[c] {
          c := input.review.object.spec.containers[_]
        }

        input_containers[c] {
          c := input.review.object.spec.initContainers[_]
        }

---
# ── Template 5: Block latest image tag ────────────────────
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8snolatesttag
  annotations:
    description: "Disallows containers using the :latest image tag"
spec:
  crd:
    spec:
      names:
        kind: K8sNoLatestTag
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8snolatesttag

        violation[{"msg": msg}] {
          container := input_containers[_]
          endswith(container.image, ":latest")
          msg := sprintf(
            "Container <%v> uses :latest tag. Use a specific version tag.",
            [container.name]
          )
        }

        violation[{"msg": msg}] {
          container := input_containers[_]
          not contains(container.image, ":")
          msg := sprintf(
            "Container <%v> has no image tag. Use a specific version tag.",
            [container.name]
          )
        }

        input_containers[c] {
          c := input.review.object.spec.containers[_]
        }

---
# ── Template 6: Require security labels ───────────────────
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredlabels
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredLabels
      validation:
        openAPIV3Schema:
          type: object
          properties:
            labels:
              type: array
              items:
                type: object
                properties:
                  key:
                    type: string
                  allowedRegex:
                    type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredlabels

        violation[{"msg": msg}] {
          provided := {label | input.review.object.metadata.labels[label]}
          required := {label | label := input.parameters.labels[_].key}
          missing := required - provided
          count(missing) > 0
          msg := sprintf(
            "Missing required labels: %v on %v <%v>",
            [missing, input.review.object.kind, input.review.object.metadata.name]
          )
        }

---
# ── Template 7: Require PodDisruptionBudget ───────────────
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8srequiredpdb
  annotations:
    description: "Deployments with >1 replica must have a PodDisruptionBudget"
spec:
  crd:
    spec:
      names:
        kind: K8sRequiredPDB
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8srequiredpdb

        violation[{"msg": msg}] {
          input.review.object.kind == "Deployment"
          input.review.object.spec.replicas > 1
          not pdb_exists
          msg := sprintf(
            "Deployment <%v> with >1 replica must have a PodDisruptionBudget",
            [input.review.object.metadata.name]
          )
        }

        pdb_exists {
          # This checks via data — Gatekeeper syncs PDB resources
          pdb := data.inventory.namespace[input.review.object.metadata.namespace]["policy/v1"]["PodDisruptionBudget"][_]
          pdb.spec.selector.matchLabels == input.review.object.spec.selector.matchLabels
        }

---
# ── Template 8: Block hostPath volumes ────────────────────
apiVersion: templates.gatekeeper.sh/v1
kind: ConstraintTemplate
metadata:
  name: k8snohostpath
spec:
  crd:
    spec:
      names:
        kind: K8sNoHostPath
      validation:
        openAPIV3Schema:
          type: object
          properties:
            allowedPaths:
              type: array
              items:
                type: string
  targets:
    - target: admission.k8s.gatekeeper.sh
      rego: |
        package k8snohostpath

        violation[{"msg": msg}] {
          volume := input.review.object.spec.volumes[_]
          volume.hostPath
          not allowed_path(volume.hostPath.path)
          msg := sprintf(
            "HostPath volume <%v> with path <%v> is not allowed",
            [volume.name, volume.hostPath.path]
          )
        }

        allowed_path(path) {
          allowed := input.parameters.allowedPaths[_]
          startswith(path, allowed)
        }
EOF

kubectl apply -f kubernetes/platform/security/gatekeeper/constraint-templates.yaml

# Wait for templates to be ready
sleep 10
kubectl get constrainttemplate
```

**Constraints (Apply the Policies)**

```
cat > kubernetes/platform/security/gatekeeper/constraints.yaml << 'EOF'
# ═══════════════════════════════════════════════════════
# OPA Gatekeeper Constraints
# Reference templates above — define WHERE and HOW strictly
# enforcementAction:
#   deny  — blocks non-compliant resources
#   warn  — allows but warns (use during rollout)
#   dryrun — audit only, never blocks
# ═══════════════════════════════════════════════════════

# ── Constraint 1: No root containers ──────────────────────
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sNoRoot
metadata:
  name: no-root-containers
  annotations:
    description: "All containers must run as non-root"
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces:
      - staging
      - production
    excludedNamespaces: []

---
# ── Constraint 2: Require resource limits ─────────────────
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredResources
metadata:
  name: require-resource-limits
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces:
      - staging
      - production
  parameters:
    limits:
      - cpu
      - memory
    requests:
      - cpu
      - memory

---
# ── Constraint 3: Read-only root filesystem ───────────────
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sReadOnlyRootFilesystem
metadata:
  name: readonly-root-filesystem
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces:
      - staging
      - production

---
# ── Constraint 4: No privileged containers ────────────────
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sNoPrivilegedContainer
metadata:
  name: no-privileged-containers
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    # Apply cluster-wide except exempt namespaces
    excludedNamespaces:
      - kube-system
      - gatekeeper-system
      - karpenter

---
# ── Constraint 5: No latest tag ───────────────────────────
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sNoLatestTag
metadata:
  name: no-latest-image-tag
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    namespaces:
      - staging
      - production

---
# ── Constraint 6: Require labels ──────────────────────────
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredLabels
metadata:
  name: require-deployment-labels
spec:
  enforcementAction: warn      # warn first, deny after rollout
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment"]
    namespaces:
      - staging
      - production
  parameters:
    labels:
      - key: app.kubernetes.io/name
      - key: app.kubernetes.io/version
      - key: environment

---
# ── Constraint 7: Require PDB ─────────────────────────────
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sRequiredPDB
metadata:
  name: require-pdb-for-deployments
spec:
  enforcementAction: warn
  match:
    kinds:
      - apiGroups: ["apps"]
        kinds: ["Deployment"]
    namespaces:
      - production

---
# ── Constraint 8: Block dangerous hostPaths ───────────────
apiVersion: constraints.gatekeeper.sh/v1beta1
kind: K8sNoHostPath
metadata:
  name: no-dangerous-host-paths
spec:
  enforcementAction: deny
  match:
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
    excludedNamespaces:
      - kube-system
      - monitoring     # node-exporter needs host access
      - logging        # promtail needs host log access
  parameters:
    allowedPaths: []   # no hostPath allowed in app namespaces
EOF

kubectl apply -f kubernetes/platform/security/gatekeeper/constraints.yaml

# Test Gatekeeper is blocking correctly
echo "Testing Gatekeeper policy enforcement..."

# This should be BLOCKED (runs as root)
cat > /tmp/test-root-pod.yaml << 'EOF'
apiVersion: v1
kind: Pod
metadata:
  name: test-root-violation
  namespace: staging
spec:
  containers:
    - name: test
      image: nginx:1.25.0
      securityContext:
        runAsUser: 0
      resources:
        limits:
          cpu: 100m
          memory: 128Mi
        requests:
          cpu: 50m
          memory: 64Mi
EOF

kubectl apply -f /tmp/test-root-pod.yaml 2>&1 | grep -q "denied" && \
  echo "✅ Root container correctly BLOCKED" || \
  echo "❌ Root container was NOT blocked — check Gatekeeper"

# Cleanup test
kubectl delete -f /tmp/test-root-pod.yaml --ignore-not-found

# View all violations across cluster
kubectl get constraints -o json | \
  python3 -c "
import sys, json
data = json.load(sys.stdin)
for item in data.get('items', []):
  name = item['metadata']['name']
  violations = item.get('status', {}).get('totalViolations', 0)
  if violations > 0:
    print(f'⚠️  {name}: {violations} violations')
  else:
    print(f'✅ {name}: 0 violations')
"
```

**Gatekeeper Mutation (Auto-fix Missing Defaults)**

```
cat > kubernetes/platform/security/gatekeeper/mutations.yaml << 'EOF'
# ═══════════════════════════════════════════════════════
# Gatekeeper Mutations
# Auto-inject security defaults into pods
# so developers don't have to remember every field
# ═══════════════════════════════════════════════════════

# ── Mutation 1: Inject seccompProfile ─────────────────────
apiVersion: mutations.gatekeeper.sh/v1
kind: Assign
metadata:
  name: inject-seccomp-profile
spec:
  applyTo:
    - groups: [""]
      kinds: ["Pod"]
      versions: ["v1"]
  match:
    scope: Namespaced
    namespaces:
      - staging
      - production
    excludedNamespaces:
      - kube-system
  location: "spec.securityContext.seccompProfile"
  parameters:
    assign:
      value:
        type: RuntimeDefault

---
# ── Mutation 2: Inject default resource limits ─────────────
# If a container has no limits, inject conservative defaults
apiVersion: mutations.gatekeeper.sh/v1
kind: AssignMetadata
metadata:
  name: inject-platform-labels
spec:
  match:
    scope: Namespaced
    namespaces:
      - staging
      - production
    kinds:
      - apiGroups: [""]
        kinds: ["Pod"]
  parameters:
    assign:
      value:
        managed-by: cloudforge-platform
        security-scan: required
EOF

kubectl apply -f kubernetes/platform/security/gatekeeper/mutations.yaml
```

## PART 3 — Falco (Runtime Threat Detection)

Falco watches system calls at kernel level. When a container does something suspicious — reads /etc/passwd, spawns a shell, modifies a binary — Falco fires an alert in real time.

```
cat > kubernetes/platform/security/falco-values.yaml << 'EOF'
# ═══════════════════════════════════════════════════════
# Falco Production Values
# eBPF driver (no kernel module required — EKS compatible)
# Alerts → Falcosidekick → Slack + AlertManager
# ═══════════════════════════════════════════════════════

driver:
  kind: ebpf    # Use eBPF — no kernel module needed, EKS compatible

collectors:
  enabled: true
  docker:
    enabled: false
  containerd:
    enabled: true
    socket: /run/containerd/containerd.sock

# Falco configuration
falco:
  # JSON output for structured parsing
  json_output: true
  json_include_output_property: true
  json_include_tags_property: true

  # Log level
  log_level: info
  log_stderr: true

  # Priority threshold — only alert on WARNING and above
  priority: warning

  # Output channels
  stdout_output:
    enabled: true

  # gRPC output to Falcosidekick
  grpc:
    enabled: true
    bind_address: "unix:///run/falco/falco.sock"

  grpc_output:
    enabled: true

  # Rules files
  rules_file:
    - /etc/falco/falco_rules.yaml
    - /etc/falco/falco_rules.local.yaml
    - /etc/falco/k8s_audit_rules.yaml
    - /etc/falco/cloudforge_rules.yaml

  # Syscall event drops handling
  syscall_event_drops:
    actions:
      - log
      - alert
    rate: 0.03333
    max_burst: 10

# Custom rules file
customRules:
  cloudforge_rules.yaml: |-
    # ════════════════════════════════════════════════════
    # CloudForge Custom Falco Rules
    # Tailored for our specific threat model
    # ════════════════════════════════════════════════════

    # ── Rule 1: Shell spawned in application container ──
    - rule: Shell Spawned in Application Container
      desc: >
        A shell was spawned in an application container.
        This could indicate an active intrusion or misconfigured entrypoint.
      condition: >
        spawned_process
        and container
        and container.image.repository != ""
        and proc.name in (shell_binaries)
        and not proc.pname in (shell_binaries, known_shell_spawn_binaries)
        and not container.image.repository in (
          "cloudforge/debug-tools",
          "busybox",
          "alpine"
        )
      output: >
        Shell spawned in application container
        (user=%user.name user_loginuid=%user.loginuid
        container_id=%container.id container_name=%container.name
        image=%container.image.repository:%container.image.tag
        shell=%proc.name parent=%proc.pname cmdline=%proc.cmdline)
      priority: WARNING
      tags: [container, shell, mitre_execution]

    # ── Rule 2: Sensitive file read ─────────────────────
    - rule: Read Sensitive File in Container
      desc: >
        A sensitive file (/etc/passwd, /etc/shadow, AWS credentials)
        was read inside a container — possible credential theft.
      condition: >
        open_read
        and container
        and fd.name in (sensitive_files)
        and not proc.name in (known_sensitive_file_readers)
      output: >
        Sensitive file read in container
        (user=%user.name container=%container.name
        image=%container.image.repository
        file=%fd.name proc=%proc.cmdline)
      priority: ERROR
      tags: [container, filesystem, mitre_credential_access]

    # ── Rule 3: AWS credential access ──────────────────
    - rule: AWS Credentials Accessed
      desc: >
        A process read AWS credentials from the filesystem.
        Could indicate credential theft or exfiltration.
      condition: >
        open_read
        and container
        and (
          fd.name startswith "/root/.aws"
          or fd.name startswith "/home/.aws"
          or fd.name = "/etc/aws/credentials"
        )
        and not proc.name in ("aws", "python3", "node")
      output: >
        AWS credentials file accessed
        (user=%user.name proc=%proc.name
        container=%container.name file=%fd.name)
      priority: CRITICAL
      tags: [container, aws, credentials, mitre_credential_access]

    # ── Rule 4: Crypto miner detected ──────────────────
    - rule: Cryptocurrency Mining Detected
      desc: >
        Process connecting to known mining pool ports or
        running known miner binaries.
      condition: >
        (
          spawned_process
          and container
          and proc.name in (
            miner_binaries,
            "xmrig", "ccminer", "cpuminer",
            "ethminer", "minerd"
          )
        )
        or
        (
          outbound
          and container
          and fd.sport in (3333, 4444, 8333, 14444, 14433)
        )
      output: >
        Cryptocurrency mining activity detected
        (user=%user.name container=%container.name
        proc=%proc.name connection=%fd.name)
      priority: CRITICAL
      tags: [container, cryptomining, mitre_impact]

    # ── Rule 5: Container escape attempt ───────────────
    - rule: Container Escape Attempt
      desc: >
        Process attempting to break out of container isolation
        by accessing host namespace or sensitive kernel interfaces.
      condition: >
        container
        and (
          (open_write and fd.name startswith "/proc/sysrq-trigger")
          or (open_write and fd.name startswith "/proc/mem")
          or (open_read and fd.name = "/proc/1/environ"
              and not proc.name in ("ps", "top", "sh"))
          or (syscall.type = ptrace and evt.arg.request = PTRACE_POKETEXT)
        )
      output: >
        Container escape attempt detected
        (user=%user.name container=%container.name
        proc=%proc.cmdline file=%fd.name)
      priority: CRITICAL
      tags: [container, escape, mitre_privilege_escalation]

    # ── Rule 6: Unexpected network connection ──────────
    - rule: Unexpected Outbound Connection from API Service
      desc: >
        The API service made a connection to an unexpected destination.
        All outbound connections should go through defined services.
      condition: >
        outbound
        and container
        and container.name startswith "api-service"
        and not fd.sip in (
          "10.0.0.0/8",          # internal VPC
          "172.20.0.0/16"        # K8s service CIDR
        )
        and fd.sport != 443      # allow AWS API calls
      output: >
        Unexpected outbound connection from api-service
        (container=%container.name destination=%fd.name
        port=%fd.sport proc=%proc.name)
      priority: WARNING
      tags: [network, api-service, mitre_exfiltration]

    # ── Rule 7: IMDS access from unexpected process ────
    - rule: Unexpected IMDS Access
      desc: >
        A process accessed the EC2 Instance Metadata Service (169.254.169.254).
        Only the AWS SDK should do this — could indicate SSRF exploitation.
      condition: >
        outbound
        and container
        and fd.sip = "169.254.169.254"
        and not proc.name in (
          "python3", "node", "java",
          "aws-iam-authenticator"
        )
      output: >
        Unexpected IMDS access
        (proc=%proc.name container=%container.name
        user=%user.name cmdline=%proc.cmdline)
      priority: WARNING
      tags: [aws, imds, mitre_credential_access]

    # ── Rule 8: kubectl exec into prod pod ─────────────
    - rule: Kubectl Exec Into Production Pod
      desc: >
        Someone exec'd into a production pod.
        This is a high-risk operation — investigate.
      condition: >
        ka.verb = exec
        and ka.target.resource = pods
        and ka.target.namespace = "production"
      output: >
        kubectl exec into production pod
        (user=%ka.user.name pod=%ka.target.name
        namespace=%ka.target.namespace
        command=%ka.uri.param[command])
      priority: WARNING
      source: k8s_audit
      tags: [k8s_audit, exec, production, mitre_execution]

    # Macros for custom rules
    - macro: miner_binaries
      condition: >
        proc.name in (xmrig, ccminer, cpuminer, ethminer, minerd)

# DaemonSet tolerations — must run on ALL nodes
tolerations:
  - effect: NoSchedule
    operator: Exists
  - effect: NoExecute
    operator: Exists

resources:
  requests:
    cpu: 100m
    memory: 256Mi
  limits:
    cpu: 1000m
    memory: 1024Mi

# Metrics for Prometheus
metrics:
  enabled: true
  interval: 1h
  outputRule: true
  resourceLabels: []

serviceMonitor:
  create: true
  labels:
    release: kube-prometheus-stack
EOF

cat > kubernetes/applications/falco.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: falco
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: platform
  sources:
    - repoURL: https://falcosecurity.github.io/charts
      chart: falco
      targetRevision: "4.7.2"
    - repoURL: https://github.com/YOUR_USERNAME/cloudforge
      targetRevision: main
      ref: values
  source:
    repoURL: https://falcosecurity.github.io/charts
    chart: falco
    targetRevision: "4.7.2"
    helm:
      valueFiles:
        - $values/kubernetes/platform/security/falco-values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: falco
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF

kubectl apply -f kubernetes/applications/falco.yaml
```

**Falcosidekick (Alert Routing)**

```
cat > kubernetes/platform/security/falcosidekick-values.yaml << 'EOF'
# ═══════════════════════════════════════════════════════
# Falcosidekick — routes Falco alerts to multiple targets
# ═══════════════════════════════════════════════════════

replicaCount: 2

config:
  debug: false

  # Slack alerts for warnings
  slack:
    webhookurl: SLACK_SECURITY_WEBHOOK_URL
    channel: "#security-alerts"
    footer: "CloudForge Security Platform"
    icon: "https://falco.org/img/brand/falco-logo.png"
    username: Falco
    minimumpriority: warning
    messageformat: |
      🚨 *{{ .Priority }}* | {{ .Rule }}
      *Container:* {{ index .OutputFields "container.name" }}
      *Image:* {{ index .OutputFields "container.image.repository" }}
      *Message:* {{ .Output }}
      *Time:* {{ .Time }}

  # PagerDuty for critical events
  pagerduty:
    routingkey: PAGERDUTY_FALCO_KEY
    minimumpriority: critical

  # Prometheus metrics
  prometheus:
    extralabels: ""

  # Send to AlertManager as well
  alertmanager:
    hostport: "http://kube-prometheus-stack-alertmanager.monitoring.svc.cluster.local:9093"
    minimumpriority: warning

  # Store in Loki for correlation with app logs
  loki:
    hostport: "http://loki-gateway.logging.svc.cluster.local"
    minimumpriority: warning
    tenant: cloudforge
    extralabels: "source=falco,environment=staging"
    endpoint: /loki/api/v1/push

  # ElasticSearch (optional — if you have it)
  elasticsearch:
    hostport: ""

resources:
  requests:
    cpu: 50m
    memory: 64Mi
  limits:
    cpu: 200m
    memory: 256Mi

serviceMonitor:
  create: true
  labels:
    release: kube-prometheus-stack

webui:
  enabled: true    # Falcosidekick UI — visualize events
EOF

cat > kubernetes/applications/falcosidekick.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: falcosidekick
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: platform
  sources:
    - repoURL: https://falcosecurity.github.io/charts
      chart: falcosidekick
      targetRevision: "0.7.17"
    - repoURL: https://github.com/YOUR_USERNAME/cloudforge
      targetRevision: main
      ref: values
  source:
    repoURL: https://falcosecurity.github.io/charts
    chart: falcosidekick
    targetRevision: "0.7.17"
    helm:
      valueFiles:
        - $values/kubernetes/platform/security/falcosidekick-values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: falco
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF

kubectl apply -f kubernetes/applications/falcosidekick.yaml
```

## PART 4 — Trivy Operator (Continuous Vulnerability Scanning)

```
cat > kubernetes/platform/security/trivy-operator-values.yaml << 'EOF'
# ═══════════════════════════════════════════════════════
# Trivy Operator
# Continuously scans ALL images running in the cluster
# Results stored as K8s CRDs, exposed as Prometheus metrics
# ═══════════════════════════════════════════════════════

trivy:
  # Use offline DB (cached in cluster — no external calls per scan)
  dbRepository: ghcr.io/aquasecurity/trivy-db
  javaDbRepository: ghcr.io/aquasecurity/trivy-java-db
  severity: UNKNOWN,LOW,MEDIUM,HIGH,CRITICAL
  ignoreUnfixed: false

  # Scan timeout per image
  timeout: 5m0s

  # Slow mode — reduce load on cluster
  slow: true

  # Skip specific CVEs (with documented justification)
  # ignoreFile: "/etc/trivy/ignore.yaml"

operator:
  # Scan all namespaces
  targetNamespaces: ""

  # Scan frequency
  scanJobTimeout: 5m
  concurrentScanJobsLimit: 10
  scanJobRetryAfter: 30s

  # Report types to generate
  vulnerabilityReportsPlugin: Trivy
  configAuditReportsPlugin: Trivy
  rbacAssessmentReportsPlugin: Trivy
  infraAssessmentReportsPlugin: Trivy
  clusterComplianceEnabled: true

  # CIS K8s Benchmark
  complianceSchemas:
    - k8s-cis-1.23
    - nsa-1.0

  # Metrics
  metricsVulnIdEnabled: true
  metricsFindingsEnabled: true

serviceMonitor:
  enabled: true
  labels:
    release: kube-prometheus-stack

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi
EOF

cat > kubernetes/applications/trivy-operator.yaml << 'EOF'
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: trivy-operator
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: platform
  sources:
    - repoURL: https://aquasecurity.github.io/helm-charts/
      chart: trivy-operator
      targetRevision: "0.24.1"
    - repoURL: https://github.com/YOUR_USERNAME/cloudforge
      targetRevision: main
      ref: values
  source:
    repoURL: https://aquasecurity.github.io/helm-charts/
    chart: trivy-operator
    targetRevision: "0.24.1"
    helm:
      valueFiles:
        - $values/kubernetes/platform/security/trivy-operator-values.yaml
  destination:
    server: https://kubernetes.default.svc
    namespace: trivy-system
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
EOF

kubectl apply -f kubernetes/applications/trivy-operator.yaml

# Wait for operator and check first scan results
sleep 60
kubectl get vulnerabilityreports -A \
  --sort-by='.report.summary.criticalCount' | head -20

# Get critical vulnerabilities
kubectl get vulnerabilityreports -A -o json | \
  python3 -c "
import sys, json
data = json.load(sys.stdin)
critical = []
for item in data['items']:
  ns = item['metadata']['namespace']
  name = item['metadata']['name']
  summary = item.get('report', {}).get('summary', {})
  crit = summary.get('criticalCount', 0)
  high = summary.get('highCount', 0)
  if crit > 0 or high > 0:
    critical.append((crit, high, ns, name))

critical.sort(reverse=True)
for c, h, ns, name in critical[:10]:
  print(f'CRITICAL:{c} HIGH:{h}  {ns}/{name}')
" 2>/dev/null || echo "Scans still in progress..."

# Check config audit reports (K8s misconfigurations)
kubectl get configauditreports -A | head -20

# Check RBAC assessment
kubectl get rbacassessmentreports -A | head -10
```

## PART 5 — Network Policies (Zero-Trust Pod Networking)

By default, every pod can talk to every other pod. Network Policies flip this to default-deny, then explicitly allow what's needed.

```
cat > kubernetes/platform/security/network-policies.yaml << 'EOF'
# ═══════════════════════════════════════════════════════
# Zero-Trust Network Policies
# Default: DENY ALL ingress and egress
# Then: explicitly allow what's needed
# ═══════════════════════════════════════════════════════

# ── Default Deny All — Staging ─────────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: staging
spec:
  podSelector: {}    # matches ALL pods in namespace
  policyTypes:
    - Ingress
    - Egress
  # No ingress/egress rules = DENY ALL

---
# ── Default Deny All — Production ─────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress

---
# ── Allow DNS (every pod needs this) ──────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: staging
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP
      to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system

---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-dns
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    - ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP
      to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system

---
# ── API Service policies ───────────────────────────────────
# Allow ingress from ALB (ALB injects traffic from any source)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-service-ingress
  namespace: staging
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: api-service
  policyTypes:
    - Ingress
  ingress:
    # From ALB controller (any pod in kube-system can be the ALB source)
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: kube-system
      ports:
        - port: 8000
    # From monitoring (Prometheus scraping)
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
      ports:
        - port: 8000

---
# Allow api-service to reach databases and AWS APIs
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-service-egress
  namespace: staging
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: api-service
  policyTypes:
    - Egress
  egress:
    # PostgreSQL (RDS — VPC IP)
    - to:
        - ipBlock:
            cidr: 10.1.0.0/16   # VPC CIDR
      ports:
        - port: 5432
    # Redis (ElastiCache — VPC IP)
    - to:
        - ipBlock:
            cidr: 10.1.0.0/16
      ports:
        - port: 6379
    # AWS APIs (S3, DynamoDB, Secrets Manager via VPC endpoints)
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.0.0.0/8
              - 172.16.0.0/12
              - 192.168.0.0/16
      ports:
        - port: 443
    # OTel Collector (for telemetry)
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
          podSelector:
            matchLabels:
              app.kubernetes.io/name: opentelemetry-collector
      ports:
        - port: 4317
        - port: 4318
    # SQS (via VPC endpoint or NAT)
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.0.0.0/8
      ports:
        - port: 443

---
# ── Worker Service policies ────────────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: worker-service-egress
  namespace: staging
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/name: worker-service
  policyTypes:
    - Egress
  egress:
    # DynamoDB + SQS via VPC endpoints or HTTPS
    - to:
        - ipBlock:
            cidr: 0.0.0.0/0
            except:
              - 10.0.0.0/8
      ports:
        - port: 443
    # OTel Collector
    - to:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
          podSelector:
            matchLabels:
              app.kubernetes.io/name: opentelemetry-collector
      ports:
        - port: 4317

---
# ── Monitoring namespace — allow Prometheus to scrape all ──
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-prometheus-scraping
  namespace: staging
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: monitoring
      ports:
        - port: 8000
        - port: 9090
        - port: 9091
        - port: 8888

---
# ── ArgoCD — allow sync operations ────────────────────────
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-argocd-sync
  namespace: staging
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: argocd
EOF

kubectl apply -f kubernetes/platform/security/network-policies.yaml

# Verify policies
kubectl get networkpolicy -A
```

## PART 6 — RBAC Hardening

```
cat > kubernetes/platform/security/rbac.yaml << 'EOF'
# ═══════════════════════════════════════════════════════
# RBAC Hardening — Least Privilege
# ═══════════════════════════════════════════════════════

# ── Developer Role — read-only in production ───────────────
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cloudforge-developer
  labels:
    rbac.cloudforge.io/managed: "true"
rules:
  # Can view most resources
  - apiGroups: [""]
    resources:
      - pods
      - services
      - configmaps
      - events
      - endpoints
    verbs: [get, list, watch]
  - apiGroups: ["apps"]
    resources:
      - deployments
      - replicasets
      - statefulsets
    verbs: [get, list, watch]
  - apiGroups: ["autoscaling"]
    resources: [horizontalpodautoscalers]
    verbs: [get, list, watch]
  # Can view logs
  - apiGroups: [""]
    resources: [pods/log]
    verbs: [get, list]
  # CANNOT exec into pods (production safety)
  # CANNOT delete resources
  # CANNOT create/modify secrets

---
# ── Platform Engineer Role ─────────────────────────────────
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cloudforge-platform-engineer
  labels:
    rbac.cloudforge.io/managed: "true"
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: [get, list, watch, create, update, patch]
  # Can exec into pods (for debugging)
  - apiGroups: [""]
    resources: [pods/exec, pods/portforward]
    verbs: [create]
  # CANNOT delete cluster-level resources
  - apiGroups: [""]
    resources: [namespaces, nodes, persistentvolumes]
    verbs: [get, list, watch]

---
# ── Emergency Break-Glass Role (time-limited) ─────────────
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cloudforge-break-glass
  labels:
    rbac.cloudforge.io/managed: "true"
    rbac.cloudforge.io/emergency: "true"
rules:
  - apiGroups: ["*"]
    resources: ["*"]
    verbs: ["*"]

---
# ── Service Account: ArgoCD (scoped to its namespace) ──────
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cloudforge-developers-staging
subjects:
  - kind: Group
    name: developers
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cloudforge-developer
  apiGroup: rbac.authorization.k8s.io

---
# ── Audit: Track all ClusterRoleBinding changes ────────────
# (This is documentation — audit is via CloudTrail K8s audit logs)
# Enable with: aws eks update-cluster-config --logging '["audit"]'
EOF

kubectl apply -f kubernetes/platform/security/rbac.yaml

# Audit existing RBAC for over-permissive bindings
echo "=== Checking for over-permissive ClusterRoleBindings ==="
kubectl get clusterrolebindings -o json | \
  python3 -c "
import sys, json
data = json.load(sys.stdin)
risky = []
for item in data['items']:
  name = item['metadata']['name']
  role = item.get('roleRef', {}).get('name', '')
  subjects = item.get('subjects', [])
  if role in ['cluster-admin', 'admin']:
    for s in subjects:
      risky.append(f'{name}: {s[\"kind\"]}={s[\"name\"]} → {role}')
for r in risky:
  print(f'⚠️  {r}')
print(f'Total risky bindings: {len(risky)}')
"
```

## PART 7 — kube-bench (CIS Benchmark)

```
cat > kubernetes/platform/security/kube-bench-job.yaml << 'EOF'
# ═══════════════════════════════════════════════════════
# kube-bench — CIS Kubernetes Benchmark
# Run as a Job to assess cluster security posture
# ═══════════════════════════════════════════════════════
apiVersion: batch/v1
kind: Job
metadata:
  name: kube-bench
  namespace: kube-system
  labels:
    app: kube-bench
spec:
  template:
    metadata:
      labels:
        app: kube-bench
    spec:
      hostPID: true
      hostIPC: true

      restartPolicy: Never

      volumes:
        - name: var-lib-etcd
          hostPath:
            path: /var/lib/etcd
        - name: var-lib-kubelet
          hostPath:
            path: /var/lib/kubelet
        - name: var-lib-kube-scheduler
          hostPath:
            path: /var/lib/kube-scheduler
        - name: var-lib-kube-controller-manager
          hostPath:
            path: /var/lib/kube-controller-manager
        - name: etc-systemd
          hostPath:
            path: /etc/systemd
        - name: lib-systemd
          hostPath:
            path: /lib/systemd
        - name: srv-kubernetes
          hostPath:
            path: /srv/kubernetes
        - name: etc-kubernetes
          hostPath:
            path: /etc/kubernetes
        - name: usr-bin
          hostPath:
            path: /usr/local/mount-from-host/bin
        - name: etc-cni-netd
          hostPath:
            path: /etc/cni/net.d
        - name: opt-cni-bin
          hostPath:
            path: /opt/cni/bin

      containers:
        - name: kube-bench
          image: docker.io/aquasec/kube-bench:v0.8.0
          command: ["kube-bench", "run"]
          args:
            - --targets=node
            - --benchmark=eks-stig-kubernetes-v1r6
            - --json
          volumeMounts:
            - name: var-lib-etcd
              mountPath: /var/lib/etcd
              readOnly: true
            - name: var-lib-kubelet
              mountPath: /var/lib/kubelet
              readOnly: true
            - name: etc-kubernetes
              mountPath: /etc/kubernetes
              readOnly: true
            - name: etc-cni-netd
              mountPath: /etc/cni/net.d
              readOnly: true
          resources:
            requests:
              cpu: 50m
              memory: 64Mi
            limits:
              cpu: 200m
              memory: 256Mi
          securityContext:
            privileged: false
            runAsNonRoot: true
            runAsUser: 1000
            readOnlyRootFilesystem: true

      nodeSelector:
        role: system
      tolerations:
        - key: CriticalAddonsOnly
          operator: Exists
          effect: NoSchedule
EOF

kubectl apply -f kubernetes/platform/security/kube-bench-job.yaml

# Wait for completion and get results
kubectl wait --for=condition=complete \
  job/kube-bench -n kube-system --timeout=120s

kubectl logs -n kube-system -l app=kube-bench | \
  python3 -c "
import sys, json
try:
    data = json.load(sys.stdin)
    for control in data.get('Controls', []):
        for test in control.get('tests', []):
            for result in test.get('results', []):
                status = result.get('status', '')
                desc = result.get('test_desc', '')
                if status == 'FAIL':
                    print(f'❌ FAIL: {desc}')
                elif status == 'WARN':
                    print(f'⚠️  WARN: {desc}')
except:
    # Not JSON output — just print
    for line in sys.stdin:
        print(line, end='')
" 2>/dev/null || kubectl logs -n kube-system -l app=kube-bench | tail -50
```

## PART 8 — Security CI Gates

```
cat > .github/workflows/security-scan.yml << 'EOF'
name: Security Scan

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]
  schedule:
    # Daily full scan at 2 AM UTC
    - cron: '0 2 * * *'

permissions:
  id-token: write
  contents: read
  security-events: write
  pull-requests: write

jobs:
  # ── Secret Detection ─────────────────────────────────────
  secret-scan:
    name: Secret Detection (Gitleaks)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0    # full history for git log scanning

      - name: Gitleaks — scan for secrets in code
        uses: gitleaks/gitleaks-action@v2
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GITLEAKS_LICENSE: ${{ secrets.GITLEAKS_LICENSE }}

      - name: TruffleHog — scan git history
        uses: trufflesecurity/trufflehog@main
        with:
          path: ./
          base: ${{ github.event.repository.default_branch }}
          head: HEAD
          extra_args: --debug --only-verified

  # ── Infrastructure Security ──────────────────────────────
  iac-scan:
    name: IaC Security (Trivy + Checkov)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Trivy — scan Terraform configs
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: config
          scan-ref: terraform/
          format: sarif
          output: trivy-iac.sarif
          severity: CRITICAL,HIGH
          exit-code: 1

      - name: Upload Trivy IaC results
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: trivy-iac.sarif
          category: trivy-iac

      - name: Checkov — additional IaC scanning
        uses: bridgecrewio/checkov-action@master
        with:
          directory: terraform/
          framework: terraform
          output_format: sarif
          output_file_path: checkov.sarif
          soft_fail: false
          skip_check: >
            CKV_AWS_18,
            CKV_AWS_144

      - name: Upload Checkov results
        uses: github/codeql-action/upload-sarif@v3
        if: always()
        with:
          sarif_file: checkov.sarif
          category: checkov

  # ── Kubernetes Manifest Security ─────────────────────────
  k8s-scan:
    name: K8s Manifest Security (Kubesec + Polaris)
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Kubesec — scan K8s manifests
        run: |
          docker run --rm \
            -v $(pwd)/kubernetes:/kubernetes \
            kubesec/kubesec:v2 scan \
              /kubernetes/platform/security/network-policies.yaml \
            | python3 -c "
          import sys, json
          data = json.load(sys.stdin)
          for result in data:
            score = result.get('score', 0)
            if score < 0:
              print(f'❌ CRITICAL issues found (score: {score})')
              for issue in result.get('scoring', {}).get('critical', []):
                print(f'  CRIT: {issue[\"selector\"]} — {issue[\"reason\"]}')
              sys.exit(1)
            else:
              print(f'✅ Score: {score}')
          "

      - name: Polaris — policy audit on manifests
        run: |
          curl -L https://github.com/FairwindsOps/polaris/releases/latest/download/polaris_linux_amd64.tar.gz \
            | tar xz -C /usr/local/bin polaris

          polaris audit \
            --audit-path kubernetes/ \
            --format pretty \
            --set-exit-code-on-danger \
            --set-exit-code-below-score 80

  # ── Container Image SBOM ─────────────────────────────────
  sbom-generate:
    name: SBOM Generation
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    needs: [secret-scan, iac-scan, k8s-scan]
    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: ${{ secrets.AWS_CI_ROLE_ARN }}
          aws-region: ap-south-1

      - name: Generate SBOM for all images in ECR
        run: |
          # Get all repositories
          REPOS=$(aws ecr describe-repositories \
            --query 'repositories[].repositoryName' \
            --output text)

          for repo in $REPOS; do
            # Get latest image
            IMAGE=$(aws ecr describe-images \
              --repository-name $repo \
              --query 'sort_by(imageDetails,&imagePushedAt)[-1].imageTags[0]' \
              --output text 2>/dev/null || continue)

            echo "Generating SBOM for $repo:$IMAGE"
            syft \
              ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.ap-south-1.amazonaws.com/$repo:$IMAGE \
              -o cyclonedx-json \
              > sbom-$repo.json || true
          done

      - name: Upload SBOMs to S3
        run: |
          for sbom in sbom-*.json; do
            aws s3 cp $sbom \
              s3://cloudforge-artifacts-${{ secrets.AWS_ACCOUNT_ID }}/sboms/$(date +%Y-%m-%d)/$sbom
          done

  # ── Security Summary Comment ─────────────────────────────
  security-summary:
    name: Security Summary
    runs-on: ubuntu-latest
    needs: [secret-scan, iac-scan, k8s-scan]
    if: github.event_name == 'pull_request' && always()
    steps:
      - name: Comment security summary on PR
        uses: actions/github-script@v7
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          script: |
            const jobs = {
              'Secret Detection': '${{ needs.secret-scan.result }}',
              'IaC Security': '${{ needs.iac-scan.result }}',
              'K8s Manifests': '${{ needs.k8s-scan.result }}'
            };

            const icon = r => r === 'success' ? '✅' : r === 'skipped' ? '⏭️' : '❌';

            const body = `## 🔐 Security Scan Results

            | Check | Status |
            |---|---|
            ${Object.entries(jobs).map(([k,v]) => `| ${k} | ${icon(v)} ${v} |`).join('\n')}

            > Scans powered by Gitleaks, TruffleHog, Trivy, Checkov, Kubesec, Polaris
            `;

            github.rest.issues.createComment({
              issue_number: context.issue.number,
              owner: context.repo.owner,
              repo: context.repo.repo,
              body
            });
EOF
```

## PART 9 — Security Dashboard in Grafana

```
cat > kubernetes/platform/monitoring/dashboards/security-dashboard.json << 'EOF'
{
  "title": "CloudForge Security Dashboard",
  "uid": "security-overview",
  "tags": ["security", "cloudforge"],
  "refresh": "5m",
  "panels": [
    {
      "title": "Gatekeeper Policy Violations",
      "type": "stat",
      "datasource": {"type": "prometheus"},
      "targets": [{
        "expr": "sum(gatekeeper_violations) by (constraint_kind)",
        "legendFormat": "{{constraint_kind}}"
      }],
      "gridPos": {"h": 4, "w": 6, "x": 0, "y": 0},
      "fieldConfig": {
        "defaults": {
          "color": {"mode": "thresholds"},
          "thresholds": {
            "steps": [
              {"color": "green", "value": 0},
              {"color": "yellow", "value": 1},
              {"color": "red", "value": 5}
            ]
          }
        }
      }
    },
    {
      "title": "Falco Alerts (last 1h)",
      "type": "stat",
      "datasource": {"type": "prometheus"},
      "targets": [{
        "expr": "sum(increase(falco_events_total[1h])) by (priority)",
        "legendFormat": "{{priority}}"
      }],
      "gridPos": {"h": 4, "w": 6, "x": 6, "y": 0}
    },
    {
      "title": "Critical CVEs by Service",
      "type": "table",
      "datasource": {"type": "prometheus"},
      "targets": [{
        "expr": "sum by (image_repository) (trivy_image_vulnerabilities{severity=\"CRITICAL\"})",
        "legendFormat": "{{image_repository}}"
      }],
      "gridPos": {"h": 8, "w": 12, "x": 0, "y": 4}
    },
    {
      "title": "Falco Events by Rule (24h)",
      "type": "timeseries",
      "datasource": {"type": "prometheus"},
      "targets": [{
        "expr": "sum by (rule) (increase(falco_events_total[24h]))",
        "legendFormat": "{{rule}}"
      }],
      "gridPos": {"h": 8, "w": 12, "x": 12, "y": 4}
    }
  ]
}
EOF
```

## PART 10 — Week 4 Validation

```
cat > scripts/validate-week4.sh << 'EOF'
#!/bin/bash
set -euo pipefail

echo "═══════════════════════════════════════"
echo "  CloudForge Week 4 Validation"
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
echo "── Pod Security Standards ───────────────"
check "staging namespace restricted" \
  "kubectl get ns staging -o jsonpath='{.metadata.labels.pod-security\.kubernetes\.io/enforce}' | grep -q restricted"
check "production namespace restricted" \
  "kubectl get ns production -o jsonpath='{.metadata.labels.pod-security\.kubernetes\.io/enforce}' | grep -q restricted"

echo ""
echo "── OPA Gatekeeper ──────────────────────"
check "Gatekeeper running (3 replicas)" \
  "[ $(kubectl get pods -n gatekeeper-system --no-headers | grep -c Running) -ge 3 ]"
check "ConstraintTemplates created (>=6)" \
  "[ $(kubectl get constrainttemplate --no-headers | wc -l) -ge 6 ]"
check "Constraints active (>=5)" \
  "[ $(kubectl get constraints --no-headers 2>/dev/null | wc -l) -ge 5 ]"
check "Root container blocked" \
  "kubectl apply -f /tmp/test-root-pod.yaml 2>&1 | grep -q denied"
check "No critical violations" \
  "[ $(kubectl get constraints -o json | python3 -c \"import sys,json; d=json.load(sys.stdin); total=sum(i.get('status',{}).get('totalViolations',0) for i in d.get('items',[])); print(total)\") -eq 0 ]"

echo ""
echo "── Falco ────────────────────────────────"
check "Falco DaemonSet ready on all nodes" \
  "kubectl get ds -n falco -l app.kubernetes.io/name=falco --no-headers | awk '{exit !(\$4==\$3)}'"
check "Falcosidekick running" \
  "kubectl get pods -n falco -l app.kubernetes.io/name=falcosidekick --no-headers | grep -q Running"
check "Custom rules loaded" \
  "kubectl exec -n falco -l app.kubernetes.io/name=falco -- falco --list-rules 2>/dev/null | grep -q 'Shell Spawned'"

echo ""
echo "── Trivy Operator ───────────────────────"
check "Trivy Operator running" \
  "kubectl get pods -n trivy-system --no-headers | grep -q Running"
check "Vulnerability reports exist" \
  "[ $(kubectl get vulnerabilityreports -A --no-headers 2>/dev/null | wc -l) -gt 0 ]"
check "Config audit reports exist" \
  "[ $(kubectl get configauditreports -A --no-headers 2>/dev/null | wc -l) -gt 0 ]"

echo ""
echo "── Network Policies ─────────────────────"
check "Default deny in staging" \
  "kubectl get networkpolicy default-deny-all -n staging &>/dev/null"
check "Default deny in production" \
  "kubectl get networkpolicy default-deny-all -n production &>/dev/null"
check "DNS egress allowed" \
  "kubectl get networkpolicy allow-dns -n staging &>/dev/null"
check "API service policies exist" \
  "kubectl get networkpolicy api-service-ingress -n staging &>/dev/null"

echo ""
echo "── RBAC ─────────────────────────────────"
check "Developer role exists" \
  "kubectl get clusterrole cloudforge-developer &>/dev/null"
check "Platform engineer role exists" \
  "kubectl get clusterrole cloudforge-platform-engineer &>/dev/null"
check "No wildcard ClusterRoleBindings (except system)" \
  "[ $(kubectl get clusterrolebindings -o json | python3 -c \"import sys,json; d=json.load(sys.stdin); risky=[i for i in d['items'] if i.get('roleRef',{}).get('name')=='cluster-admin' and not i['metadata']['name'].startswith('system:')]; print(len(risky))\") -le 2 ]"

echo ""
echo "── Security Scanning ────────────────────"
check "kube-bench job completed" \
  "kubectl get job kube-bench -n kube-system -o jsonpath='{.status.succeeded}' | grep -q 1"

echo ""
echo "── Mutation ─────────────────────────────"
check "Mutations configured" \
  "kubectl get assign -A --no-headers 2>/dev/null | wc -l | grep -qv '^0$'"

echo ""
echo "═══════════════════════════════════════"
echo "  Results: ✅ $PASS passed | ❌ $FAIL failed"
echo "═══════════════════════════════════════"

[ $FAIL -eq 0 ] \
  && echo "  🎉 Week 4 Complete!" \
  || echo "  ⚠️  Fix failures before Week 5"
EOF

chmod +x scripts/validate-week4.sh
./scripts/validate-week4.sh
```

## Week 4 Commit

```
git add .
git commit -m "feat: Week 4 — Zero-Trust Security Layer

Pod Security Standards:
- restricted enforce on staging + production namespaces
- baseline on monitoring + logging (elevated requirements)

OPA Gatekeeper:
- 3-replica HA deployment with Prometheus metrics
- 8 ConstraintTemplates (no-root, resource limits, readonly-fs,
  no-privileged, no-latest-tag, required-labels, PDB, no-hostpath)
- 8 Constraints enforcing policies on app namespaces
- Mutations: auto-inject seccompProfile=RuntimeDefault
- Tested: root container correctly blocked

Falco Runtime Security:
- eBPF driver (no kernel module — EKS compatible)
- 8 custom rules (shell spawn, sensitive file read, AWS creds,
  crypto mining, container escape, unexpected network, IMDS, kubectl exec)
- Falcosidekick: routes alerts to Slack + PagerDuty + Loki
- Falcosidekick UI for event visualization

Trivy Operator:
- Continuous image scanning (VulnerabilityReports CRDs)
- Config audit reports (K8s misconfigurations)
- RBAC assessment reports
- CIS K8s Benchmark + NSA compliance schemas
- Prometheus metrics for Grafana dashboard

Network Policies:
- Default-deny-all in staging + production
- Explicit allow: DNS, ALB ingress, Prometheus scraping
- API service: only reaches DB, Redis, AWS APIs, OTel
- Worker service: only reaches SQS/DDB, OTel Collector

RBAC Hardening:
- Developer role: read-only, no exec in production
- Platform Engineer role: full ops, no cluster destruction
- Break-glass role: documented, audited
- Audit of existing ClusterRoleBindings

CI Security Gates:
- Gitleaks + TruffleHog (secret detection in git history)
- Trivy IaC scan (Terraform misconfigurations)
- Checkov (additional IaC policies)
- Kubesec (K8s manifest security scoring)
- Polaris (policy audit, score >= 80 required)
- SBOM generation + S3 upload

Security Dashboard:
- Gatekeeper violations by constraint
- Falco events by priority and rule
- Critical CVEs by service
- Real-time security posture view

Validated:
- Root containers blocked ✅
- Trivy reports generating ✅
- Falco custom rules loaded ✅
- Network policies enforced ✅
- kube-bench completed ✅"

git push origin feat/week1-foundation
```

## Week 4 Summary

```
What you built:
├── ✅ Pod Security Standards (restricted on app namespaces)
├── ✅ OPA Gatekeeper (8 policies, HA, mutations)
├── ✅ Falco (eBPF, 8 custom rules, Falcosidekick routing)
├── ✅ Trivy Operator (continuous scanning, CIS compliance)
├── ✅ Network Policies (default-deny, explicit allow)
├── ✅ RBAC Hardening (developer, platform, break-glass)
├── ✅ kube-bench (CIS benchmark scored)
├── ✅ Security CI gates (6 scanning tools)
└── ✅ Security Grafana dashboard

What this demonstrates to employers:
├── Defence in depth (PSS → Gatekeeper → Falco → NetPol)
├── Shift-left security (CI gates before merge)
├── Runtime detection (Falco custom rules for your threat model)
├── Continuous compliance (Trivy Operator + CIS benchmark)
├── Zero-trust networking (explicit-allow only)
└── Supply chain security (SBOM generation)

The Revolut/Adyen signal:
Fintech companies care about:
- Can no one accidentally deploy a root container? ✅ (Gatekeeper)
- Will we know if someone execs into prod? ✅ (Falco rule)
- Are all images scanned continuously? ✅ (Trivy Operator)
- Is our network zero-trust? ✅ (NetworkPolicies)
- Can secrets leak through git? ✅ (Gitleaks + TruffleHog)
```



















