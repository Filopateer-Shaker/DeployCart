# Worker 4 — Monitoring, Security & Observability

Updated for the current stack: **GitHub Actions** (not Jenkins) for CI, **MongoDB
Atlas** (not an in-cluster StatefulSet) for the database, everything deployed
via **Helm**, with Argo CD watching the gitops repo.

Everything here belongs in the **gitops repo**, under these same paths, so
Argo CD picks it up automatically:

```
gitops-repo/
  monitoring/
    prometheus-values.yaml
    grafana-dashboards/
      kubernetes-cluster-dashboard.yaml   # ConfigMap-wrapped, grafana_dashboard: "1"
      application-dashboard.yaml          # ConfigMap-wrapped, grafana_dashboard: "1"
      cd-pipeline-dashboard.yaml          # ConfigMap-wrapped, grafana_dashboard: "1"
    servicemonitors/
      frontend.yaml
      backend.yaml
    alerts/
      basic-alert-rules.yaml
    grafana-ingress.yaml
  security/
    security-checklist.md
    rbac.yaml
    network-policy.yaml
  argocd/
    monitoring-app.yaml
    security-app.yaml
```

## What changed from the original Jenkins/in-cluster-MongoDB version
- **No Jenkins scrape target** — GitHub Actions runs on GitHub's own SaaS
  runners, so there's nothing in your VPC to point Prometheus at. Instead,
  the CD half of the pipeline (Argo CD syncing what GitHub Actions pushed)
  is tracked via Argo CD's own Prometheus metrics — see
  `cd-pipeline-dashboard.json`.
- **No `database` namespace** — MongoDB is Atlas, fully managed and outside
  the cluster. There's no StatefulSet/PVC/Secret/NetworkPolicy for it in
  this repo anymore. The backend's NetworkPolicy instead allows egress to
  the internet on port 27017 (Atlas resolves to Atlas-managed IPs, not a
  fixed CIDR you can pin).
- **RBAC has no `database` namespace role** for the same reason.

## Before you apply anything
Review these settings:
- GitOps repository URLs in `argocd/*-app.yaml` are configured as:
  `https://github.com/Filopateer-Shaker/DeployCart.git`
- Replace the `team-viewers` / `team-admins` group names in `security/rbac.yaml`
  to match the groups configured for your EKS cluster.

And set up outside this repo:
- A Kubernetes Secret in the `backend` namespace holding the Atlas
  `mongodb+srv://` connection string (referenced by the backend Deployment
  as an environment variable; that manifest is Worker 3's, this repo just documents the
  expectation in `security-checklist.md`)
- Atlas project Network Access List entry for the EKS node's egress IP
- GitHub repository secrets: Docker Hub creds, the gitops-repo push token,
  and (if Terraform runs from Actions) AWS OIDC role, not static keys

## Install order (depends on Worker 1 and Worker 3 finishing first)
1. Worker 1's EKS cluster and node group must be `Ready`.
2. Worker 3's namespaces (`frontend`, `backend`, `argocd`) and Argo CD itself
   must already exist, installed via Helm with `metrics.enabled: true` so
   its Prometheus metrics are exposed.
3. Add the `monitoring` namespace if it isn't already in Worker 3's
   `namespaces.yaml`.
4. Commit everything in this folder to the gitops repo.
5. Apply the two Argo CD Application manifests once (after that, a `git push` is enough):
   ```
   kubectl apply -f argocd/monitoring-app.yaml
   kubectl apply -f argocd/security-app.yaml
   ```
6. Watch Argo CD sync:
   ```
   kubectl get applications -n argocd
   ```

## Grafana dashboards — how they get loaded
`sidecar.dashboards.enabled: true` (plus `defaultDashboardsEnabled: true`) in
`prometheus-values.yaml` makes Grafana watch for ConfigMaps labeled
`grafana_dashboard: "1"` in any namespace (`searchNamespace: ALL`). Each
dashboard already ships pre-wrapped as a ConfigMap in
`monitoring/grafana-dashboards/` - no manual pasting needed, they're applied
as-is by the `monitoring-extras` Argo CD Application (`path: monitoring`).

## Verifying it worked
```bash
kubectl get pods -n monitoring
# Expect: prometheus, grafana, alertmanager, node-exporter, kube-state-metrics all Running

kubectl get ingress -n monitoring
# open http://<AWS_LOAD_BALANCER_DNS>/grafana (same LB as the ingress-nginx controller,
# no custom domain needed - see monitoring/grafana-ingress.yaml)
# fall back to port-forward if the ingress/LB isn't ready yet:
#   kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
# check all three dashboards under Dashboards

kubectl get prometheusrule -n monitoring
# Expect: basic-alert-rules present

# In Prometheus > Alerts: NodeNotReady, PodCrashLooping, FrontendDown,
# BackendDown, FrontendDeploymentReplicasUnavailable,
# BackendDeploymentReplicasUnavailable, ArgoCDAppOutOfSync,
# ArgoCDAppUnhealthy should all be listed (green/inactive is fine before the demo)

kubectl get servicemonitor -n monitoring
# Expect: frontend, backend, and ingress-nginx-controller (from the ingress-nginx
# Helm release) all present
```

## Tracking the GitHub Actions side
GitHub Actions build/test/scan status is visible on the repo's **Actions**
tab and via status badges in the README — Prometheus isn't the right tool
for that since there's no long-running process to scrape. What Prometheus
*does* give visibility into is the outcome that actually matters for the
demo: did the image that GitHub Actions built actually make it onto EKS
healthy — that's what `cd-pipeline-dashboard.json` and the
`ArgoCDAppOutOfSync` / `ArgoCDAppUnhealthy` alerts cover.

## Definition of done for this scope
See the bottom of `security-checklist.md` — mirrors Section 23 of the main plan.
