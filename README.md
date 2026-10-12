# AstroLumina-ArgoCD

GitOps bootstrap for the AstroLumina platform on RKE2: install prerequisites
for ArgoCD plus the `Application` that syncs
[AstroLumina-Kubernetes](https://github.com/AstroLumina-Rise-n-Shine/AstroLumina-Kubernetes)
(`HEAD` of `main`) into the cluster with **manual** sync.

Theory and day-2 operations live in [ARGOCD.md](./ARGOCD.md) -- read its
section 0 (prerequisites) before the first sync.

## Layout

- `install/` -- namespace + explicit cluster-admin grant for the ArgoCD
  service accounts, applied once before the Helm release, plus the
  admin-password script (run once right after the Helm release).
- `apps/` -- the `astrolumina-kubernetes` Application (whole repo, manual sync).

## Install order

```bash
export K8S_CP_CONN=$(doppler secrets get K8S_CP_CONN --plain --project astrolumina --config prd)

# 1. Namespace + RBAC grant (from this repo root).
kubectl apply -k install/   # via ssh on the CP, same pattern as ARGOCD.md

# 2. ArgoCD itself via the Helm chart (always the latest chart version;
#    rationale in ARGOCD.md section 2).
ssh "$K8S_CP_CONN" -- "helm version >/dev/null 2>&1 || curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash"
ssh "$K8S_CP_CONN" -- "helm repo add argo https://argoproj.github.io/argo-helm 2>/dev/null || true; helm repo update"
cat <<'EOF' | ssh "$K8S_CP_CONN" -- "cat > ~/argocd-values.yaml"
# Traefik IngressRoute replaces the chart's own Ingress. The server listens
# HTTPS on 8080 by default; Traefik terminates TLS at the edge and talks
# plaintext to the backend, so the server must serve plain HTTP here.
server:
  ingress:
    enabled: false
  extraArgs:
    - --insecure
EOF
CHART_VERSION=$(ssh "$K8S_CP_CONN" -- "helm show chart argo/argo-cd | grep '^version:' | awk '{print \$2}'")
echo "Installing argo-cd chart version: ${CHART_VERSION}"
ssh "$K8S_CP_CONN" -- "helm upgrade --install argocd argo/argo-cd -n argocd --create-namespace --version ${CHART_VERSION} -f ~/argocd-values.yaml"
ssh "$K8S_CP_CONN" -- "kubectl rollout status deploy/argocd-server -n argocd --timeout=300s"

# 3. Set the admin password from Doppler (needs the bcrypt hash from
#    `htpasswd -nbB admin` in ARGOCD_ADMIN_PASSWORD_HASH (preferred) or
#    ARGOCD_ADMIN_PASSWORD, Doppler astrolumina/prd; safe to re-run).
./install/set-argocd-admin-password.sh

# 4. Register the app (ArgoCD takes over from here).
kubectl apply -f apps/astrolumina-kubernetes.yaml -n argocd
```

Then open the ArgoCD UI, review the diff, and press **SYNC** per change.
