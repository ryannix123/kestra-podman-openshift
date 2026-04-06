<p align="center">
  <img src="https://kestra.io/logo.svg" alt="Kestra Logo" width="300"/>
</p>

<p align="center">
  <img alt="Kestra version" src="https://img.shields.io/badge/kestra-1.3.7+-blueviolet"/>
  <img alt="Helm chart" src="https://img.shields.io/badge/helm-v1.0+-blue?logo=helm&logoColor=white"/>
  <img alt="PostgreSQL" src="https://img.shields.io/badge/PostgreSQL-16-336791?logo=postgresql&logoColor=white"/>
  <img alt="OpenShift" src="https://img.shields.io/badge/OpenShift-4.x-EE0000?logo=redhatopenshift&logoColor=white"/>
  <img alt="License" src="https://img.shields.io/badge/license-Apache%202.0-blue"/>
</p>

# 🚀 Deploying Kestra on OpenShift (Developer Sandbox)

This guide documents a battle-tested deployment of [Kestra](https://kestra.io) on the Red Hat OpenShift Developer Sandbox using Helm. It captures every real-world issue encountered — SCC restrictions, PostgreSQL version compatibility, Helm v1.0 schema changes, and probe port mismatches — so you don't have to debug them yourself.

> 🐳 **Running locally with Podman?** See the [Podman deployment guide](../README.md).

---

## ✅ Prerequisites

- OpenShift 4.x cluster (Developer Sandbox or dedicated)
- `oc` CLI authenticated to your cluster
- `helm` CLI installed
- For automated deployment: Ansible with `kubernetes.core` collection

---

## ⚠️ Key Lessons Learned (Read First)

Before diving in, these are the non-obvious gotchas this guide resolves:

| Issue | Root Cause | Fix |
|---|---|---|
| `configuration is not authorized anymore` | Kestra Helm chart v1.0 broke the old `configuration:` key | Use `configurations.application:` instead |
| Pods never created (SCC error) | Hardcoded `runAsUser`/`fsGroup` outside the namespace UID range | Remove all hardcoded UIDs — let OpenShift assign them |
| Flyway migration fails (`syntax error at RETURN`) | PostgreSQL 10 (from `oc new-app postgresql-persistent`) is too old | Deploy PostgreSQL **16** explicitly |
| Pod restarts every 60–90s | Startup/liveness/readiness probes hitting port 8080 — health is on **8081** | Set all probes to port `8081` |
| `AccessDeniedException: /app/storage` | Container runs as random UID with no write access to `/app/storage` | Set storage `base-path` to `/tmp/kestra-storage` |
| `nc` not found in init container | `ubi8/ubi-minimal` does not include netcat | Use bash's built-in `/dev/tcp` test instead |
| Rolling update deadlocked | Old pod never becomes Ready so Kubernetes won't terminate it | Force-delete the old pod to unblock the rollout |

---

## 🤖 Automated Deployment (Ansible)

The included `deploy-kestra.yml` playbook automates the entire deployment. See [deploy-kestra.yml](deploy-kestra.yml) for details.

```bash
# Install dependencies (one-time)
ansible-galaxy collection install kubernetes.core
pip install kubernetes openshift

# Deploy
ansible-playbook deploy-kestra.yml \
  -e namespace=ryan-nix-dev \
  -e kestra_db_password=k3str4
```

---

## 🔧 Manual Step-by-Step Deployment

### 1. Add the Kestra Helm Repository

```bash
helm repo add kestra https://helm.kestra.io/
helm repo update
```

### 2. Deploy PostgreSQL 16

The Kestra Helm chart's Flyway migrations require **PostgreSQL 13 or newer**. The `oc new-app postgresql-persistent` template deploys PostgreSQL 10, which will fail. The newest Red Hat-supported container image is PostgreSQL 16:

```bash
oc new-app registry.redhat.io/rhel9/postgresql-16:latest \
  -e POSTGRESQL_USER=kestra \
  -e POSTGRESQL_PASSWORD=k3str4 \
  -e POSTGRESQL_DATABASE=kestra \
  --name=kestra-postgresql \
  -n <your-namespace>

# Wait for it to be ready
oc rollout status deployment/kestra-postgresql -n <your-namespace>
```

### 3. Create `values-openshift.yaml`

The Kestra Helm chart underwent **breaking changes in v1.0.0**. The old top-level keys (`configuration`, `securityContext`, `resources`, `livenessProbe`, etc.) no longer exist. Everything now lives under `configurations.application` and `common`.

```yaml
# ─── Application Configuration ─────────────────────────────────────────────────
configurations:
  application:
    datasources:
      postgres:
        url: jdbc:postgresql://kestra-postgresql:5432/kestra
        username: kestra
        password: k3str4
        driverClassName: org.postgresql.Driver
    kestra:
      repository:
        type: postgres
      queue:
        type: postgres
      storage:
        type: local
        local:
          # /tmp is always writable by any UID assigned by OpenShift
          base-path: "/tmp/kestra-storage"

# ─── Common (applies to all Kestra components) ─────────────────────────────────
common:
  # Do NOT set runAsUser or fsGroup — OpenShift assigns these per namespace
  podSecurityContext: {}
  securityContext:
    allowPrivilegeEscalation: false

  resources:
    requests:
      memory: "512Mi"
      cpu: "500m"
    limits:
      memory: "2Gi"
      cpu: "1000m"

  # Health endpoint is on the MANAGEMENT port (8081), not the app port (8080)
  startupProbe:
    httpGet:
      path: /health
      port: 8081
    initialDelaySeconds: 10
    periodSeconds: 5
    failureThreshold: 60      # 5 min window — Kestra loads 1000+ plugins on boot

  livenessProbe:
    httpGet:
      path: /health
      port: 8081
    initialDelaySeconds: 120
    periodSeconds: 30
    timeoutSeconds: 10
    failureThreshold: 5

  readinessProbe:
    httpGet:
      path: /health
      port: 8081
    initialDelaySeconds: 60
    periodSeconds: 10
    timeoutSeconds: 5
    failureThreshold: 5

  # Reuse the Kestra image already on the node — no extra registry dependency
  # Uses bash /dev/tcp since ubi-minimal does not include netcat
  initContainers:
    - name: wait-for-postgres
      image: kestra/kestra:latest
      command:
        - /bin/bash
        - -c
        - |
          until bash -c 'echo > /dev/tcp/kestra-postgresql/5432' 2>/dev/null; do
            echo "Waiting for PostgreSQL..."
            sleep 2
          done
          echo "PostgreSQL is ready!"

# ─── Disable built-in dependencies ────────────────────────────────────────────
postgresql:
  enabled: false

dind:
  enabled: false

# ─── Expose via oc expose after deploy (see Step 5) ───────────────────────────
ingress:
  enabled: false
```

### 4. Install Kestra

```bash
helm install kestra kestra/kestra \
  --namespace <your-namespace> \
  -f values-openshift.yaml
```

Watch the pods:

```bash
oc get pods -n <your-namespace> -w
```

Expected progression:

```
Init:0/1  →  PodInitializing  →  Running (0/1)  →  Running (1/1)
```

The JVM startup + 1013 plugin scan takes approximately 90 seconds. The `0/1 Running` state is normal during this window.

### 5. Expose via OpenShift Route

```bash
# Find the service name (typically just 'kestra')
oc get svc -n <your-namespace> | grep kestra

# Expose it
oc expose svc/kestra --port=8080 --name=kestra-route -n <your-namespace>

# Add TLS edge termination
oc patch route kestra-route -n <your-namespace> \
  --type=merge \
  -p '{"spec":{"tls":{"termination":"edge","insecureEdgeTerminationPolicy":"Redirect"}}}'

# Get your URL
oc get route kestra-route -n <your-namespace> -o jsonpath='{.spec.host}'
```

### 6. Access the UI

Open the route URL in your browser. On first launch, Kestra presents a setup wizard to create an admin user.

---

## 🔄 Upgrading Kestra

```bash
helm repo update

helm upgrade kestra kestra/kestra \
  --namespace <your-namespace> \
  -f values-openshift.yaml

oc rollout status deployment/kestra-standalone -n <your-namespace>
```

If the rolling update deadlocks (new pod initializing, old pod stuck `0/1`), force it:

```bash
oc delete pod $(oc get pod -l app.kubernetes.io/component=standalone \
  -n <your-namespace> --sort-by=.metadata.creationTimestamp \
  -o name | head -1) -n <your-namespace>
```

---

## 🧹 Uninstalling

```bash
helm uninstall kestra -n <your-namespace>

oc delete deployment/kestra-postgresql \
   svc/kestra-postgresql \
   -n <your-namespace>

oc delete pvc --all -n <your-namespace>
oc delete route kestra-route -n <your-namespace>
```

> ⚠️ Deleting PVCs removes all persisted flow and execution data.

---

## 🗺️ Architecture Overview

```
OpenShift Route (HTTPS/edge TLS)
        │
        ▼
  Service: kestra
  port 8080 (UI/API)  ──► kestra-standalone pod
  port 8081 (health)       │
                           ├── init: wait-for-postgres (/dev/tcp)
                           ├── storage: /tmp/kestra-storage
                           └── DB: jdbc:postgresql://kestra-postgresql:5432/kestra
                                        │
                                        ▼
                           Deployment: kestra-postgresql
                           (registry.redhat.io/rhel9/postgresql-16)
```

---

## 📋 Helm v1.0 Key Migration Reference

If you have an old `values-openshift.yaml` using the pre-1.0 schema:

| Old Key (v0.x) | New Key (v1.0+) |
|---|---|
| `configuration:` | `configurations.application:` |
| `securityContext:` | `common.podSecurityContext:` + `common.securityContext:` |
| `resources:` | `common.resources:` |
| `livenessProbe:` | `common.livenessProbe:` |
| `readinessProbe:` | `common.readinessProbe:` |
| `initContainers:` | `common.initContainers:` |

The chart's `checks.yaml` will hard-block installation with `configuration is not authorized anymore` if any old top-level keys are present.

---

## 📎 References

- [Kestra Helm Chart Migration Guide (v1.0)](https://kestra.io/docs/migration-guide/v1.0.0/helm-charts)
- [Kestra Helm Charts GitHub](https://github.com/kestra-io/helm-charts)
- [Red Hat OpenShift Developer Sandbox](https://developers.redhat.com/developer-sandbox)
- [OpenShift SCC Documentation](https://docs.openshift.com/container-platform/latest/authentication/managing-security-context-constraints.html)
- [Red Hat PostgreSQL 16 Container](https://catalog.redhat.com/software/containers/rhel9/postgresql-16)