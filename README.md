# k0s platform

Open source Kubernetes platform on k0s. GitHub is the only source of truth. Flux reconciles
everything. Teams request an environment by pull request and own it from there. LLM agents
operate day two changes by opening pull requests, never by writing to the cluster.

## Layout

| Path | Purpose |
|------|---------|
| `bootstrap/` | k0sctl cluster definition. Run once, then the cluster manages itself from Git. |
| `clusters/home/` | Flux entry point. One Kustomization per layer, ordered by `dependsOn`. |
| `infra/` | cert-manager, Sealed Secrets, Kyverno, Capsule, OTel operator and collector, kube prometheus stack. |
| `policies/` | Kyverno ClusterPolicies. Tenant scaffolding, workload baseline, GitOps only enforcement. |
| `tenants/` | One file per team. Adding that file is the entire provisioning request. |
| `apps/` | Team owned overlays. Workloads, alert routing, alert rules. |
| `agents/` | Agent roster, merge authority guardrails, prompts. |

## Reconciliation order

```
infra  ->  policies  ->  tenants  ->  apps
```

Each layer declares `dependsOn` the previous one, so Kyverno exists before any tenant is created,
and tenant namespaces exist before workloads target them.

## Bootstrap

```bash
# 1. Cluster
k0sctl apply --config bootstrap/k0sctl.yaml
k0sctl kubeconfig --config bootstrap/k0sctl.yaml > ~/.kube/k0s-home
export KUBECONFIG=~/.kube/k0s-home

# 2. Flux, pointed at this repo
flux bootstrap github \
  --owner=<your-github-user> \
  --repository=k0s-platform \
  --branch=main \
  --path=clusters/home \
  --personal

# 3. Watch it converge
flux get kustomizations --watch
```

## Adding a team

1. Copy `tenants/team-a/tenant.yaml` to `tenants/<team>/tenant.yaml`, set the name, owner group, and tier.
2. Add the path to `tenants/kustomization.yaml`.
3. Open a pull request.

On merge, Flux picks it up within one minute, Capsule creates the bounded namespace, and Kyverno
stamps the default network policy, limit range, and the alerting Role. No ticket, no manual step.

## Alerting ownership

Teams commit an `AlertmanagerConfig` into their own namespace carrying the label
`alertmanager-config: enabled`. Alertmanager merges it automatically. The Prometheus Operator injects
a namespace matcher, so a team route cannot capture another team's alerts. Platform rules
(node down, disk pressure, Flux reconciliation failure) stay in `infra/` and cannot be silenced by a tenant.

Every `PrometheusRule` and `ServiceMonitor` must carry a `team` label. Admission rejects it otherwise,
because an unlabelled rule is an unroutable alert.

## Agent boundary

Agents read cluster and telemetry state and write pull requests. They hold no kubeconfig. Merge
authority is tiered in `agents/config/guardrails.yaml`: low risk changes auto merge after CI and a
Kyverno dry run, everything else needs an independent reviewing agent, and RBAC, policy, secret, and
bootstrap paths are human only. CI blocks agent authored pull requests that touch those paths, and
CODEOWNERS enforces the same boundary a second way.

Rollback is the safety net: every agent commit is atomic and never squash merged, so a bad decision
is one `git revert` away from being reconciled out.

## Verification

```bash
flux get all -A
kubectl get tenant
kubectl get policyreport -A
kubectl get alertmanagerconfig -A
```
