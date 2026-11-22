# TP4 — Step-by-step guide (with screenshot checklist)

All commands are for **PowerShell**, run from the `tp4-gitops` folder.
Every 📸 is a screenshot to save in `docs/screenshots/` with the given name — they become the figures of your report.
Every 📝 is what to explain in the report next to that screenshot.

Budget: ~2 h for setup, ~1 h for the scenarios.

---

## Phase 0 — Prerequisites

### 0.1 Start a Kubernetes cluster (Docker Desktop)
1. Start **Docker Desktop** and wait until it says *Engine running*.
2. Settings → **Kubernetes** → tick **Enable Kubernetes** (any provisioning mode, 1 node is enough) → *Apply & restart*. Wait for the green "Kubernetes running".
3. Check:
   ```powershell
   kubectl config use-context docker-desktop
   kubectl get nodes
   ```
   The node must be `Ready`. (Your PC has ~24 GB RAM; Docker gives WSL2 half by default — the whole TP needs ~5 GB, fine.)

📸 `01-cluster-ready.png` — output of `kubectl get nodes`.

### 0.2 Create the GitHub repository
1. On github.com → **New repository** → name `tp4-gitops`, **Public**, *do not* add a README → Create.
2. 📝 Why public: ArgoCD can read it without credentials. In production you'd keep it private and give ArgoCD a read-only deploy key.

---

## Phase 1 — Push the GitOps repository (Partie A, Étape 1)

The ArgoCD manifests already point to `https://github.com/Yosr-attia/tp4-gitops.git`, and the local repo is initialised with a first commit on `main` and the remote `origin` set. Just push:
```powershell
git push -u origin main
```

📸 `02-github-repo.png` — the repo tree on GitHub.
📸 `03-kustomize-structure.png` — VS Code explorer showing `apps/podinfo/base` + `overlays/dev|prod`, or the output of `kubectl kustomize apps/podinfo/overlays/dev`.
📝 Explain **SSOT** and **Kustomize base/overlays**: `base` = what's common, `overlays/dev` and `overlays/prod` only change namespace, replicas, image tag and resources. Nothing is applied by hand, everything comes from this repo.

---

## Phase 2 — Install ArgoCD (Partie A, Étape 2)

```powershell
kubectl create namespace argocd
```
```powershell
kubectl apply -n argocd --server-side --force-conflicts -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.6.2/manifests/install.yaml
```
```powershell
kubectl -n argocd get pods -w
```
Wait until all pods are `Running` (Ctrl+C to stop watching).

Make ArgoCD poll Git every 30 s instead of 3 min (faster demo):
```powershell
kubectl -n argocd patch configmap argocd-cm --type merge --patch-file argocd/bootstrap/argocd-cm-patch.yaml
```
```powershell
kubectl -n argocd rollout restart statefulset argocd-application-controller
```
```powershell
kubectl -n argocd rollout restart deployment argocd-repo-server
```

Open the UI — **keep this terminal open** (one terminal per port-forward):
```powershell
kubectl -n argocd port-forward svc/argocd-server 8080:443
```
Get the admin password (in another terminal):
```powershell
[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String((kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}")))
```
Browse to **https://localhost:8080** (accept the certificate warning), log in as `admin`.

📸 `04-argocd-pods.png` — `kubectl -n argocd get pods`.
📸 `05-argocd-login.png` — the ArgoCD UI after login (empty).
📝 ArgoCD components: *application-controller* (compares Git vs cluster and applies), *repo-server* (clones Git, renders Kustomize/Helm), *server* (API + UI), *redis* (cache). Mention the 30 s polling and that a GitHub webhook is the production way.

---

## Phase 3 — Bootstrap everything with App-of-Apps

```powershell
kubectl apply -f argocd/root-app.yaml
```
```powershell
kubectl -n argocd get applications -w
```
After 3–5 minutes you should see `root`, `kube-prometheus-stack`, `monitoring-config`, `podinfo-dev`, `podinfo-prod` all **Synced / Healthy**.

> If `podinfo-*` or `monitoring-config` show an error like *"no matches for kind ServiceMonitor"*: it synced before the Prometheus CRDs existed. The `retry` policy fixes it automatically within a few minutes; or click **Sync** once.

Check the app itself:
```powershell
kubectl -n podinfo-dev get all
```
```powershell
kubectl -n podinfo-dev port-forward svc/podinfo 9898:9898
```
Open http://localhost:9898 (podinfo UI showing "TP4 GitOps - DEV").

📸 `06-argocd-applications.png` — the Applications list (all green).
📸 `07-root-app-tree.png` — click `root`: the tree showing the child Applications (App-of-Apps).
📸 `08-podinfo-dev-tree.png` — click `podinfo-dev`: Deployment → ReplicaSet → Pods, Service, ServiceMonitor.
📸 `09-podinfo-dev-details.png` — *App Details* → the Sync Policy section showing **Automated, Prune, Self Heal** enabled. (Or `kubectl -n argocd get application podinfo-dev -o yaml` showing `selfHeal: true`.)
📸 `10-podinfo-ui.png` — the podinfo web page.
📝 Explain: the only manual `kubectl apply` is the root app; sync-waves order (projects → monitoring stack → monitoring config → apps); the two AppProjects (`platform` vs `workloads`, least privilege); the difference between **Sync Status** (Git == cluster?) and **Health Status** (do the resources work?).

---

## Phase 4 — Verify observability (Partie B, Étapes 1–4)

### 4.1 Prometheus
```powershell
kubectl -n monitoring port-forward svc/prometheus-kube-prometheus-prometheus 9090:9090
```
Open http://localhost:9090:
- **Status → Targets**: find `serviceMonitor/monitoring/argocd-metrics`, `argocd-server-metrics`, `argocd-repo-server-metrics`, `serviceMonitor/podinfo-dev/podinfo` → all **UP**.
- **Graph**: run `argocd_app_info` (table view), then `argocd_app_sync_total`, then `argocd_git_request_total`.
- **Alerts**: find the group `argocd.rules` and `workload.rules` (all *Inactive* = green).

📸 `11-prometheus-targets.png` — targets UP (ArgoCD x3 + podinfo).
📸 `12-promql-argocd-app-info.png` — `argocd_app_info` result with `sync_status` / `health_status` labels.
📸 `13-promql-sync-total.png` — `argocd_app_sync_total`.
📸 `14-prometheus-alert-rules.png` — the Alerts page with your rules.
📝 Explain ServiceMonitor = "how to find the endpoint" (label selector + port), the Prometheus Operator turns it into scrape config. Explain the key metrics: `argocd_app_info` (one series per app, value 1, state in labels), `argocd_app_sync_total` (counter of syncs by phase), `argocd_git_request_total` (ls-remote / fetch calls to Git).

### 4.2 Grafana
```powershell
kubectl -n monitoring port-forward svc/prometheus-grafana 3000:80
```
Password (user `admin`):
```powershell
[Text.Encoding]::UTF8.GetString([Convert]::FromBase64String((kubectl -n monitoring get secret prometheus-grafana -o jsonpath="{.data.admin-password}")))
```
Open http://localhost:3000:
- **Connections → Data sources** → *Prometheus* is already there (provisioned declaratively by the Helm chart). Click it → **Save & test** → green.
- **Dashboards** → **TP4 - GitOps & Workloads Overview** (loaded from the ConfigMap in Git). Top variables: `app = podinfo-dev`, `namespace = podinfo-dev`. Set refresh to 10s, time range "Last 30 minutes".
- **Alerting → Alert rules** → section *Data source-managed* → your 5 rules, state *Normal*.

📸 `15-grafana-datasource.png` — Prometheus data source "Successfully queried".
📸 `16-grafana-dashboard-overview.png` — the whole dashboard, everything green.
📸 `17-grafana-alert-rules-normal.png` — alert rules in *Normal*.
📝 Explain "dashboards as code": the JSON lives in `monitoring/dashboards/`, Kustomize turns it into a ConfigMap labelled `grafana_dashboard: "1"`, the Grafana sidecar loads it. Go through the panels (the PromQL table is in the README).

---

## Phase 5 — The 5 demo scenarios

Keep 3 windows visible: **ArgoCD UI** (podinfo-dev), **Grafana dashboard**, and a terminal with:
```powershell
kubectl -n podinfo-dev get pods -w
```

### Scenario 1 — Automated continuous deployment
1. Edit `apps/podinfo/overlays/dev/kustomization.yaml`: `count: 2` → `count: 4`
   (optionally also `newTag: 6.16.0` → `6.17.0` to show a rolling update).
2. Push:
   ```powershell
   git commit -am "scale podinfo-dev to 4 replicas"
   ```
   ```powershell
   git push
   ```
3. **Do nothing else.** Within ~30 s ArgoCD detects the new commit and syncs; 2 new pods appear.

📸 `18-s1-git-commit.png` — the commit (terminal or GitHub).
📸 `19-s1-argocd-synced.png` — podinfo-dev showing the new commit SHA in *Sync status* / *Last sync* "auto-sync", 4 pods in the tree.
📸 `20-s1-kubectl-pods.png` — 4 pods Running.
📝 Pull model: nobody ran `kubectl`; the controller *pulls* Git, sees the diff, applies it. Compare with a push CI pipeline that needs cluster credentials.

### Scenario 2 — Drift detection & self-healing
1. Manual change on the cluster:
   ```powershell
   kubectl -n podinfo-dev scale deployment podinfo --replicas=1
   ```
   Watch the pods: they go to 1 and come back to 4 within seconds.
2. Even stronger:
   ```powershell
   kubectl -n podinfo-dev delete deployment podinfo
   ```
   ```powershell
   kubectl -n podinfo-dev get deployment podinfo
   ```
   The deployment is back (AGE a few seconds).

📸 `21-s2-kubectl-drift.png` — terminal: the scale/delete command then the restored deployment (young AGE).
📸 `22-s2-argocd-selfheal.png` — ArgoCD: the app event/sync showing it was re-synced automatically (*App Details → Events* tab, or the *Last Sync* "Sync initiated by automated sync policy"). If you're fast you can catch the orange **OutOfSync** state.
📝 Drift = live state ≠ Git. `selfHeal: true` makes the controller re-apply Git as soon as it sees the difference (it watches the cluster). Any manual "hotfix" is overwritten → the only way to change prod is a commit (auditable, reviewable).

### Scenario 3 — Sync Status vs Health Status
1. Edit dev `kustomization.yaml`: `newTag: 0.0.0-doesnotexist`. Commit + push:
   ```powershell
   git commit -am "deploy broken image tag (demo)"
   ```
   ```powershell
   git push
   ```
2. In ArgoCD: **Synced** (green) but **Progressing** then, after 60 s, **Degraded** (red heart). New pod: `ErrImagePull` / `ImagePullBackOff`. The old pods stay Running.

📸 `23-s3-argocd-synced-degraded.png` — app header showing *Synced* + *Degraded* together.
📸 `24-s3-pod-errimagepull.png` — click the failing pod → *Events*: "Failed to pull image ... not found".
📸 `25-s3-kubectl-pods.png` — old pods Running + new pod ImagePullBackOff.
📝 **Sync** compares manifests (Git said tag X, cluster has tag X → Synced). **Health** checks the real state (Deployment exceeded `progressDeadlineSeconds: 60` → Degraded). ArgoCD did its job perfectly; the *content* of Git is wrong. Thanks to `maxUnavailable: 0`, the old pods keep serving traffic: no outage.

### Scenario 4 — History & Rollback
1. podinfo-dev → **App Details** → *Sync Policy* → **Disable Auto-Sync** (confirm).
   📝 ArgoCD *refuses* a rollback while auto-sync is on, because auto-sync would immediately re-apply Git HEAD (the broken commit).
2. Click **History and Rollback** → pick the last good revision (the 4-replica commit) → **⋮ → Rollback** → OK.
3. Result: **Healthy** again, but **OutOfSync** (cluster = old revision, Git HEAD = broken commit). Auto-sync stays **disabled**.
4. Do the proper GitOps fix and re-enable automation:
   ```powershell
   git revert --no-edit HEAD
   ```
   ```powershell
   git push
   ```
   Then *App Details* → **Enable Auto-Sync**. → **Synced + Healthy**.

📸 `26-s4-history.png` — the History list with the revisions/commits.
📸 `27-s4-after-rollback.png` — app *Healthy* + *OutOfSync*, auto-sync disabled.
📸 `28-s4-git-revert.png` — `git log --oneline` showing the revert commit.
📸 `29-s4-back-to-synced.png` — Synced + Healthy, auto-sync enabled.
📝 Rollback in ArgoCD = quick emergency fix (seconds), but it breaks the SSOT temporarily (cluster ≠ Git). The durable fix is always in Git (`git revert`), then auto-sync is re-enabled. Git history = full audit trail.

### Scenario 5 — GitOps ↔ metrics ↔ alerting correlation (Partie B)
Grafana dashboard open, refresh 10s, "Last 15 minutes".

**5a — Scale up via Git**: `count: 4` → `count: 6`, commit, push. Watch *Pods Desired vs Available* step up, a bar in *Sync history*, a purple **annotation** line "ArgoCD sync".

📸 `30-s5-grafana-scale.png`

**5b — Inject the broken image again**: `newTag: 0.0.0-doesnotexist`, commit, push. Timeline you should observe:
| Time | What you see |
|---|---|
| ~30 s | sync annotation, *Containers waiting*: `ErrImagePull`/`ImagePullBackOff`, *unavailable* = 1 |
| ~1 min 30 | Health stat → **Degraded**; alert `PodImagePullBackOff` **Pending → Firing** |
| ~3 min | `ArgoCDAppDegraded` **Firing**; "Firing alerts" stat turns red |

📸 `31-s5-grafana-degraded.png` — dashboard with Degraded + waiting reasons + alerts panel.
📸 `32-s5-alerts-firing.png` — Grafana *Alerting → Alert rules* (or Prometheus *Alerts*) showing **Firing**.

**5c — Resolve via GitOps**: `git revert --no-edit HEAD` + `git push`. Within ~1–2 min: Healthy, waiting reasons gone, alerts back to **Normal**.

📸 `33-s5-grafana-recovered.png`
📸 `34-s5-alerts-normal.png`
📝 Correlation: the sync annotation (GitOps event) lines up exactly with the change of the pod curves and the alert. `for: 2m` avoids alerting on a normal short Progressing phase (no false positives during a rollout).

**Optional — application traffic** (fills the HTTP panels; podinfo answers 500 on `/status/500`):
```powershell
kubectl -n podinfo-dev run loadgen --image=busybox:1.36 --restart=Never -- /bin/sh -c "while true; do wget -q -O /dev/null http://podinfo:9898/; wget -q -O /dev/null http://podinfo:9898/status/500; sleep 0.2; done"
```
📸 `35-app-metrics.png` — HTTP req/s, 5xx ratio, p95 latency panels. Then remove it:
```powershell
kubectl -n podinfo-dev delete pod loadgen
```

---

## Phase 6 — Report

Suggested structure (each section uses the screenshots above):
1. Introduction & objectives
2. Architecture (diagram from README) & technical choices (application, Kustomize, App-of-Apps, AppProjects)
3. Partie A — Setup (repo, ArgoCD install, bootstrap) — 📸 01–10
4. Partie A — Scenarios 1 → 4 — 📸 18–29
5. Partie B — Prometheus/Grafana stack, ServiceMonitors, PromQL, alerts — 📸 11–17
6. Partie B — Scenario 5 correlation — 📸 30–35
7. Critical analysis: security (pull model, AppProjects, no secrets in Git, pinned versions, hardened containers), limits (public repo, admin password, polling vs webhook, single cluster) and improvements (Sealed Secrets, Argo Rollouts canary on Prometheus metrics, Image Updater, Alertmanager → Slack/email, branch protection & PR review)
8. Conclusion

Final step: `git add docs/screenshots` + commit + push, so the repo contains the README, manifests, dashboard JSON and the screenshots.

---

## Cleanup (after the defense)
```powershell
kubectl delete -f argocd/root-app.yaml
```
(The finalizers delete all child apps and their resources.) Then `kubectl delete namespace argocd`, or simply reset the Kubernetes cluster in Docker Desktop.
