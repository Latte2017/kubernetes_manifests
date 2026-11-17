# GitOps & Kubernetes Operations Guide

This guide covers the complete architecture, configuration management, ArgoCD App-of-Apps setup, troubleshooting, and day-to-day operational procedures for your `nginx-server` and `auth-service` microservices.

---

## 1. Repository & Helm Chart Structure

All deployment resources are packaged inside a modular Helm chart (`charts/nginx-app/`) to prevent configuration drift and duplicate resource conflicts. Standalone raw YAML manifests in the root directory should remain removed.

### Directory Layout
```text
├── argocd/
│   ├── root.yaml             # Root App-of-Apps controller
│   └── nginx-app.yaml        # Child application definition
└── charts/
    └── nginx-app/
        ├── Chart.yaml
        ├── values.yaml
        ├── files/
        │   └── nginx.conf    # Custom Nginx configuration
        └── templates/
            ├── _helpers.tpl  # Helm template helper definitions
            ├── configmap.yaml
            ├── deployment-auth.yaml
            ├── deployment-nginx.yaml
            └── service.yaml
```

### Dynamic Configuration Reloads (`checksum/config`)
Because Kubernetes does not automatically restart pods when a mounted ConfigMap updates, `deployment-nginx.yaml` uses a SHA-256 checksum annotation to trigger automatic rolling restarts whenever `nginx.conf` changes in Git:

```yaml
spec:
  template:
    metadata:
      labels:
        app: nginx
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/configmap.yaml") . | sha256sum }}
```

### Packaging `nginx.conf`
Keep `nginx.conf` inside `charts/nginx-app/files/` so Helm's `.Files.Get` helper can bundle it cleanly into `templates/configmap.yaml`:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: {{ include "nginx-app.fullname" . }}-config
  namespace: production
data:
  nginx.conf: |
{{ .Files.Get "files/nginx.conf" | indent 4 }}
```

---

## 2. ArgoCD App-of-Apps Pattern

To prevent your ArgoCD `Application` resource from vanishing across cluster resets or reconciliations, manage it declaratively using an **App-of-Apps** pattern.

### Child Application (`argocd/nginx-app.yaml`)
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: nginx-gitops
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: '<YOUR_GITHUB_REPO_URL>'
    targetRevision: HEAD
    path: 'charts/nginx-app'
  destination:
    server: https://kubernetes.default.svc
    namespace: production
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### Root Application (`argocd/root.yaml`)
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: root-app
  namespace: argocd
  finalizers:
    - resources-finalizer.argocd.argoproj.io
spec:
  project: default
  source:
    repoURL: '<YOUR_GITHUB_REPO_URL>'
    targetRevision: HEAD
    path: 'argocd'
  destination:
    server: https://kubernetes.default.svc
    namespace: argocd
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### Bootstrapping Command (Run Once)
```bash
kubectl apply -f argocd/root.yaml
```

---

## 3. Post-Reboot Startup Checklist

After a full desktop system reboot, follow these steps in order to bring your local cluster and GitOps pipeline back online:

1. **Start Container Runtime & Minikube:**
   ```bash
   minikube start
   kubectl cluster-info
   ```

2. **Verify ArgoCD & Sync Status:**
   ```bash
   kubectl get pods -n argocd
   argocd app get nginx-gitops
   ```
   *(Optional)* Force a manual sync if needed:
   ```bash
   argocd app sync nginx-gitops
   ```

3. **Verify Production Workloads:**
   ```bash
   kubectl get pods -n production
   ```
   Ensure both `nginx-server` and `auth-service` show `Running` (`1/1` ready).

4. **Restore Local Access Tunnel:**
   ```bash
   kubectl port-forward svc/nginx-service 9090:80 -n production
   ```
   Access your application at `http://localhost:9090`.

---

## 4. Persisting Local Port-Forwarding

`kubectl port-forward` is a client-side tunnel that terminates when your terminal closes. Use either method below to keep it running persistently in the background:

### Option A: Detached Background Process
```bash
kubectl port-forward svc/nginx-service 9090:80 -n production > /dev/null 2>&1 &
```
* **Check status:** `ps aux | grep port-forward`
* **Stop process:** `kill $(pgrep -f "port-forward svc/nginx-service")`

### Option B: Systemd User Service (Auto-Restart on Boot)
1. Create `~/.config/systemd/user/nginx-port-forward.service`:
   ```ini
   [Unit]
   Description=Kubernetes Port Forward for Nginx Service
   After=network.target

   [Service]
   ExecStart=/usr/local/bin/kubectl port-forward svc/nginx-service 9090:80 -n production
   Restart=always
   RestartSec=5

   [Install]
   WantedBy=default.target
   ```
2. Enable and start the service:
   ```bash
   systemctl --user daemon-reload
   systemctl --user enable --now nginx-port-forward.service
   ```
3. Verify status:
   ```bash
   systemctl --user status nginx-port-forward.service
   ```

---

## 5. Quick Troubleshooting Reference

| Issue | Diagnostic / Fix Command |
| :--- | :--- |
| **Verify live `nginx.conf` inside pod** | `kubectl exec -it deployment/nginx-server -n production -c nginx -- cat /etc/nginx/nginx.conf` |
| **Force Nginx pod recreation** | `kubectl rollout restart deployment nginx-server -n production` |
| **Check ArgoCD sync drift** | `argocd app diff nginx-gitops` |
| **Inspect ArgoCD cluster events** | `kubectl get events -n argocd --sort-by='.metadata.creationTimestamp' \| grep -i "nginx-gitops"` |
| **Port already in use (`8080`)** | Find PID via `lsof -i :8080` and terminate with `kill -9 <PID>`, or bind to an alternate port (`9090:80`). |