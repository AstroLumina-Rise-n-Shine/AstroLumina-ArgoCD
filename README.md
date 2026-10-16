# AstroLumina-ArgoCD

GitOps bootstrap for the AstroLumina platform on RKE2: a complete ArgoCD
install plus the wiring that syncs
[AstroLumina-Kubernetes](https://github.com/AstroLumina-Rise-n-Shine/AstroLumina-Kubernetes)
and the Helm releases into the cluster, all with **manual** sync.

Theory and day-2 operations live in [ARGOCD.md](./ARGOCD.md) -- read its
section 0 (prerequisites) before the first sync.

## Layout

- `install/` -- namespace + explicit cluster-admin grant for the ArgoCD
  service accounts (applied once before the Helm release), plus the
  admin-password script (run once right after the Helm release).
- `apps/` -- two ApplicationSets (one per source repo), each stamping three
  per-environment Applications, all with manual sync:
  `astrolumina-kubernetes.yaml` (Kustomize env directories) and
  `astrolumina-helm.yaml` (Helm releases from the Helm repo).

## Install order

1. **ArgoCD itself -- follow [ARGOCD.md §2](./ARGOCD.md#2-how-to-install-it)
   end to end.** That section is the complete install process: RKE2 kubectl
   bootstrap → namespace + RBAC → Helm chart → admin-password script →
   Traefik route → first login. Nothing below works until the UI answers
   with your admin login.
2. **Doppler** (operator + tokens) -- section below.
3. **Apps** -- apply both ApplicationSets, then SYNC per app from the UI.

```bash
export K8S_CP_CONN=$(doppler secrets get K8S_CP_CONN --plain --project astrolumina --config prd)
```

## Doppler (operator + token Secrets)

Every Application below syncs `DopplerSecret` CRs, and every Deployment
mounts its env from the Doppler-managed Secrets via `secretKeyRef` -- 100%
of variables come from Doppler, there are no ConfigMaps. Without the
operator, the synced `DopplerSecret` objects resolve to nothing and every
pod fails at startup (zod validation, exit 1). So the operator goes in
first, once per cluster rebuild, before the first SYNC.

Two-machine model, same as everywhere: `doppler` runs on your LAPTOP,
`kubectl` runs on the CONTROL PLANE (`K8S_CP_CONN` itself comes from
Doppler: `doppler secrets get K8S_CP_CONN --plain --project astrolumina
--config <dev|stg|prd>`).

```bash
# The operator itself (once per cluster rebuild).
ssh "$K8S_CP_CONN" -- "kubectl apply -f https://github.com/DopplerHQ/kubernetes-operator/releases/latest/download/recommended.yaml"
```

The operator authenticates per environment with a token Secret that is
created imperatively (NOT in git), one per Doppler config:

| Doppler config | Token Secret (imperative, NOT in git) | App namespace | Managed Secrets |
|---|---|---|---|
| `dev` | `doppler-token-dev` | `astrolumina-development` | `env-frontend-secrets`, `env-astrology-api-secrets`, `env-booking-api-secrets`, `env-payment-api-secrets` |
| `stg` | `doppler-token-stg` | `astrolumina-staging` | same 4 names |
| `prd` | `doppler-token-prd` | `astrolumina-production` | same 4 names |

The token Secrets themselves all live in `doppler-operator-system` (the
operator's namespace); the "App namespace" column is where the synced
`env-*` Secrets land.

Create each token from the `DOPPLER_SERVICE_TOKEN` Doppler secret (fetch it
like any other key, no dashboard copy-paste), e.g. for development:

```bash
export DOPPLER_SERVICE_TOKEN=$(doppler secrets get DOPPLER_SERVICE_TOKEN --plain --project astrolumina --config dev)
ssh "$K8S_CP_CONN" -- 'kubectl create secret generic doppler-token-dev -n doppler-operator-system --from-literal=serviceToken='"$DOPPLER_SERVICE_TOKEN"' --dry-run=client -o yaml | kubectl apply -f -'
```

Repeat with `--config stg` / `--config prd` for the other two tokens.
Then verify the sync before touching ArgoCD:

```bash
ssh "$K8S_CP_CONN" -- "kubectl get secrets -n astrolumina-development | grep '^env-'"
```

You should see the 4 `env-*` Secrets populated. Two things worth knowing:

- Re-applying is a no-op on secrets, so once the tokens exist ArgoCD can
  sync the `DopplerSecret` CRs freely without ever overwriting values.
- The Deployments carry the `secrets.doppler.com/reload` annotation, so
  pods restart automatically whenever a synced value changes in Doppler.
- If a pod still fails at startup after SYNC, the fix is always "add the
  missing key in Doppler" -- the manifests plus the sync lists define the
  exact schema.

Full command sequences with expected outputs per environment live in the
Kubernetes repo (`README.md` -> "Secrets and Doppler", plus each
environment's own `README.md`).

## Apps (the two ApplicationSets)

- `apps/astrolumina-kubernetes.yaml` -- stamps
  `astrolumina-kubernetes-development/staging/production`. ArgoCD runs `kustomize
  build` on the env directory and syncs the result. Staging/production
  carry the blue + green Deployments plus the `53-live-services.yaml`
  selectors -- a live flip is a git commit changing those selectors, then
  SYNC. Production image upgrades are commits changing tags in git (a
  rollback is a revert).
- `apps/astrolumina-helm.yaml` -- stamps
  `astrolumina-helm-development/staging/production`. ArgoCD runs `helm template`
  with `values.yaml` plus the env file (later files win).
  Staging/production do blue/green via the per-service
  `blueGreen.liveColors` entries plus `imageTags.blue/green` in their
  values file -- a flip is a git commit changing one service line, then
  SYNC.

Both files carry `namespace: argocd` in their own metadata, so apply them
plainly (manual sync everywhere -- nothing auto-applies). `kubectl` lives
on the control plane only, and this repo is not mounted there, so stream
the files to it over stdin -- run from the repo root on your laptop:

```bash
ssh "$K8S_CP_CONN" -- "kubectl apply -f -" < apps/astrolumina-kubernetes.yaml
ssh "$K8S_CP_CONN" -- "kubectl apply -f -" < apps/astrolumina-helm.yaml
```

Before the first SYNC: ARGOCD.md §0 completed (placeholders out of the
live kustomizations, production images pinned, everything pushed), the
`ghcr-secret` image-pull secret created in all three namespaces (see
"GHCR image-pull secret" at the end of this file), and the Traefik CRDs
present on staging/production (`IngressRoute`, `Middleware` -- shipped
with RKE2).

Then open the ArgoCD UI, review the diff per app, and press **SYNC** --
staging first, production last, development whenever (ARGOCD.md §5).

## GHCR image-pull secret (required before the first SYNC)

Every app image lives in the **private** GitHub Container Registry
(`ghcr.io/AstroLumina-Rise-n-Shine/...`), and every Deployment in the
Kubernetes repo declares `imagePullSecrets: [{name: ghcr-secret}]`. Without
that Secret the kubelet cannot authenticate to GHCR, so **every pod stays
`ImagePullBackOff` / `ErrImagePull` forever** -- ArgoCD will show the apps
as synced while nothing actually runs. This is the single most common
reason a fresh cluster looks "deployed but dead".

Three things to know before creating it:

- The Secret holds a GitHub token, so it is **never committed to git** --
  it is created imperatively with `kubectl create secret docker-registry`
  (same commands as in the Kubernetes env READMEs). The
  `01-ghcr-secret.yaml` placeholder must stay out of the live
  kustomizations (ARGOCD.md §0) or the first sync overwrites the real
  Secret with a placeholder.
- Kubernetes Secrets are **per-namespace**: you need `ghcr-secret` in all
  three app namespaces -- `astrolumina-development`, `astrolumina-staging`,
  `astrolumina-production` (one command per namespace below).
- Each block first creates its namespace with `create ... --dry-run=client
  | apply` -- fully idempotent, a no-op if ArgoCD or a previous run already
  created it. This matters because the Secret must exist **before** the
  first SYNC, and before that sync the namespace itself may not exist yet
  (it normally comes from `00-namespace.yaml` in each env).
- The GitHub token needs only the `read:packages` scope. A token with
  `$`, `!` or quotes still works: the quote splice `'...'"$VAR"'...'`
  expands values locally, so special characters survive the SSH trip
  intact, while the `|` pipe executes remotely on the control plane.

Run one block per environment (each uses its own Doppler config):

```bash
# --- development (namespace astrolumina-development) ---
export K8S_CP_CONN=$(doppler secrets get K8S_CP_CONN --plain --project astrolumina --config dev)
export GITHUB_USER=$(doppler secrets get GITHUB_USER --plain --project astrolumina --config dev)
export GITHUB_TOKEN=$(doppler secrets get GITHUB_TOKEN --plain --project astrolumina --config dev)
export GITHUB_EMAIL=$(doppler secrets get GITHUB_EMAIL --plain --project astrolumina --config dev)
ssh "$K8S_CP_CONN" -- 'kubectl create namespace astrolumina-development --dry-run=client -o yaml | kubectl apply -f -'
ssh "$K8S_CP_CONN" -- 'kubectl create secret docker-registry ghcr-secret -n astrolumina-development --docker-server=ghcr.io --docker-username='"$GITHUB_USER"' --docker-password='"$GITHUB_TOKEN"' --docker-email='"$GITHUB_EMAIL"' --dry-run=client -o yaml | kubectl apply -f -'
unset GITHUB_TOKEN GITHUB_USER GITHUB_EMAIL
```

```bash
# --- staging (namespace astrolumina-staging) ---
export K8S_CP_CONN=$(doppler secrets get K8S_CP_CONN --plain --project astrolumina --config stg)
export GITHUB_USER=$(doppler secrets get GITHUB_USER --plain --project astrolumina --config stg)
export GITHUB_TOKEN=$(doppler secrets get GITHUB_TOKEN --plain --project astrolumina --config stg)
export GITHUB_EMAIL=$(doppler secrets get GITHUB_EMAIL --plain --project astrolumina --config stg)
ssh "$K8S_CP_CONN" -- 'kubectl create namespace astrolumina-staging --dry-run=client -o yaml | kubectl apply -f -'
ssh "$K8S_CP_CONN" -- 'kubectl create secret docker-registry ghcr-secret -n astrolumina-staging --docker-server=ghcr.io --docker-username='"$GITHUB_USER"' --docker-password='"$GITHUB_TOKEN"' --docker-email='"$GITHUB_EMAIL"' --dry-run=client -o yaml | kubectl apply -f -'
unset GITHUB_TOKEN GITHUB_USER GITHUB_EMAIL
```

```bash
# --- production (namespace astrolumina-production) ---
export K8S_CP_CONN=$(doppler secrets get K8S_CP_CONN --plain --project astrolumina --config prd)
export GITHUB_USER=$(doppler secrets get GITHUB_USER --plain --project astrolumina --config prd)
export GITHUB_TOKEN=$(doppler secrets get GITHUB_TOKEN --plain --project astrolumina --config prd)
export GITHUB_EMAIL=$(doppler secrets get GITHUB_EMAIL --plain --project astrolumina --config prd)
ssh "$K8S_CP_CONN" -- 'kubectl create namespace astrolumina-production --dry-run=client -o yaml | kubectl apply -f -'
ssh "$K8S_CP_CONN" -- 'kubectl create secret docker-registry ghcr-secret -n astrolumina-production --docker-server=ghcr.io --docker-username='"$GITHUB_USER"' --docker-password='"$GITHUB_TOKEN"' --docker-email='"$GITHUB_EMAIL"' --dry-run=client -o yaml | kubectl apply -f -'
unset GITHUB_TOKEN GITHUB_USER GITHUB_EMAIL
```

Verify all three exist, then SYNC:

```bash
ssh "$K8S_CP_CONN" -- "kubectl get secret ghcr-secret -n astrolumina-development"
ssh "$K8S_CP_CONN" -- "kubectl get secret ghcr-secret -n astrolumina-staging"
ssh "$K8S_CP_CONN" -- "kubectl get secret ghcr-secret -n astrolumina-production"
```

If pods still report `ErrImagePull` after this, the token is expired or
lacks `read:packages` -- re-run the block for that namespace with a fresh
token from GitHub Settings → Developer settings → Personal access tokens
(classic).
