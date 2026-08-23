# Wahda MKaaS — Troubleshooting

Known failure modes discovered in production. Each has: **symptom → root cause → fix**.

---

## Cluster stuck in `CREATE_IN_PROGRESS`, no worker VMs in Nova

### Symptom

`openstack coe cluster show <name>` returns `CREATE_IN_PROGRESS` for > 10 min.
`nova server list --name '<cluster-name>-*'` shows zero worker VMs.

### Diagnostic

Tunnel to the management k8s cluster and check:

```bash
ssh -p <PORT> -N -L 6443:<MGMT-CPLANE-IP>:6443 <USER>@<BASTION>
export KUBECONFIG=~/mgmt-cluster.kubeconfig

NS=magnum-<project_id>
kubectl -n $NS get cluster,openstackcluster,machinedeployment,machine,openstackmachine
```

Look for a `Machine` stuck in `Provisioning` with `OpenStackMachine.status.conditions[].reason=WaitingForBootstrapData` **or** the CAPO controller logging `no flavors were found: name=<value>`.

CAPO controller logs:

```bash
kubectl -n capo-system logs deploy/capo-controller-manager --tail=200 | grep -i <cluster-name>
```

### Root cause A — chart version mismatch

Symptom: `Error: chart "openstack-cluster" version "X.Y.Z" not found in ... repository`.

Chart was tagged and manually pushed to `gh-pages` as a `.tgz` but the `index.yaml` was not updated (chart-releaser CI race — see below).

### Fix A

Regenerate `index.yaml` and push manually:

```bash
git clone --branch gh-pages https://github.com/Odil-devops/capi-helm-charts.git /tmp/pages
cd /tmp/pages
helm repo index . --url https://odil-devops.github.io/capi-helm-charts --merge index.yaml
git add index.yaml && git commit -m "manual: refresh index for X.Y.Z" && git push
# Wait 30-60s for GitHub Pages CDN
curl -sS https://odil-devops.github.io/capi-helm-charts/index.yaml | grep 'X.Y.Z'
```

### Root cause B — numeric flavor identifier + old chart

Symptom in CAPO logs: `Reconciler error err="no flavors were found: name=3"`.

Chart `< 0.20.0-hcp.5` writes `spec.template.spec.flavor: "3"` on `OpenStackMachineTemplate`. CAPO treats that as a Nova flavor **name**, not ID. Kolla's default flavors are named `m1.small/medium/large`, so the by-name lookup fails.

### Fix B

Pin `capi_helm_chart_version=0.20.0-hcp.5` (or later) on every HCP template. Chart hcp.5 emits `flavorID: "3"` for all-digit values which CAPO looks up by ID.

Steps to update a live template that's already in use:

1. Delete any dependent cluster (or leave them; they keep their own chart snapshot).
2. `PATCH /v1/clustertemplates/{uuid}` with `[{"op":"replace","path":"/labels/capi_helm_chart_version","value":"0.20.0-hcp.5"}]`.
3. Retry the failed cluster (must be delete + create; the failed one can't be repaired in-place).

---

## Cluster `CREATE_FAILED`, `status_reason` mentions helm

### Symptom

`openstack coe cluster show <name>` shows `CREATE_FAILED` and `status_reason` contains:
`Unexpected error while running command. Command: helm upgrade ...`

### Diagnostic

Read the `Stderr:` line in `status_reason`. Common patterns:

| Stderr fragment | Cause | Fix |
|-----------------|-------|-----|
| `chart "openstack-cluster" version "X.Y.Z" not found` | Bad chart pin or GH Pages index missing entry | Fix A above |
| `OpenStackMachineTemplate ... is invalid: [spec.template.spec.flavor: Invalid value: "integer"]` | Chart < 0.20.0-hcp.4 with numeric flavor | Upgrade chart pin to hcp.4+ (hcp.5 preferred) |
| `no flavors were found: name=<digits>` | Chart < 0.20.0-hcp.5 with numeric flavor | Fix B above |
| `context deadline exceeded` / `timeout` | Mgmt cluster overloaded or Envoy Gateway route not applied | Check mgmt cluster health |

---

## Chart-releaser race — GH Actions `Release HCP charts` fails with "non-fast-forward"

<a id="chart-releaser-race"></a>

### Symptom

Push a tag `hcp-vX.Y.Z-hcp.N` → `.github/workflows/release-hcp.yaml` runs → `Publish via chart-releaser` step fails:

```
! [rejected]        HEAD -> gh-pages (non-fast-forward)
Error: exit status 1
```

### Root cause

`helm/chart-releaser-action` fetches `gh-pages`, packages the chart, updates `index.yaml`, and pushes back. In parallel, GitHub's own `pages build and deployment` workflow can push to `gh-pages` (SHA `pages-build-deployment@...`), invalidating the chart-releaser's parent SHA. Chart-releaser's push is then rejected.

### Fix (manual publish)

Chart-releaser DID upload the tgz(s) to `.cr-release-packages/`. But because the `git push` failed, they're not on `gh-pages`. Publish them yourself:

```bash
# On the local machine (has helm)
mkdir -p /tmp/pkg
helm package /path/to/openstack-cluster -d /tmp/pkg
helm package /path/to/cluster-addons    -d /tmp/pkg

git clone --branch gh-pages https://<PAT>@github.com/Odil-devops/capi-helm-charts.git /tmp/pages
cd /tmp/pages
cp /tmp/pkg/*.tgz .
helm repo index . --url https://odil-devops.github.io/capi-helm-charts --merge index.yaml
git add *.tgz index.yaml
git commit -m "manual: publish <chart>-<version> (chart-releaser race workaround)"
git push origin gh-pages
```

### Long-term fix (TODO)

Wrap the chart-releaser step in retry-on-conflict, or move to a self-hosted helm-repo (S3-backed) — GitHub Pages is not really designed for concurrent writers.

---

## Skyline shows "The resource has been deleted" on Cluster Detail

### Symptom

Cluster is `CREATE_COMPLETE + HEALTHY` in Magnum but Skyline UI shows spooky `The resource has been deleted` cards on the Cluster Detail page.

### Root cause

Skyline's Cluster Detail page tries to resolve `flavor_id`, `master_flavor_id`, `fixed_network`, `fixed_subnet` by ID via the corresponding OpenStack service. When the value is a Nova **name** (`m1.medium`) instead of an ID, or when the network is on a foreign project, the lookup returns 404 and the UI renders "deleted".

### Fix

Skyline UI patch is pending (see `thewahdaa/skyline-console`, TODO): try ID → fall back to name, and don't render "deleted" for empty lookups.

---

## Cluster stuck in `DELETE_IN_PROGRESS`

### Symptom

`openstack coe cluster delete <name>` accepted (HTTP 204) but the cluster stays in `DELETE_IN_PROGRESS` forever.

### Root cause

Common on dev-openstack where the image is fake and no real Nova VMs exist — CAPO's finalizer on `OpenStackCluster` cannot complete because there's no cloud state to clean up.

On prod this rarely happens; when it does, it usually indicates the mgmt cluster lost track of the helm release.

### Fix

```bash
NS=magnum-<project_id>
# Strip finalizers on OpenStackCluster — this cascades into Cluster deletion
kubectl -n $NS patch openstackcluster <name> --type=merge -p '{"metadata":{"finalizers":[]}}'
# Sometimes secrets stick around; label-select to clean up
kubectl -n $NS delete secret -l cluster.x-k8s.io/cluster-name=<name>
```

If Magnum itself doesn't clear the cluster row, use the DB fallback (Kolla-side):

```bash
docker exec -it mariadb mysql -u root -p<password> magnum -e "DELETE FROM cluster WHERE name='<name>';"
```

**Never** do this on a healthy cluster; the row deletion bypasses the driver, so any live CAPI resources will be orphaned.

---

## Certificate rotation confusion (10-year certs)

### Symptom

Etcd/apiserver certificates in the HostedControlPlane were regenerated with a longer lifetime, and now two etcd nodes have different CA fingerprints — `tls: bad certificate` in etcd peer logs.

### Root cause

CAPHCP has **no CA-rotation state machine**. Bumping cert lifetimes on an existing HCP via `Values.controlPlane.certificatesDuration` does not gracefully re-issue peer certs — some nodes get new certs, others don't.

### Fix

Recreate the cluster. Any HCP created **after** the CAPHCP fork's 10-year default (`pkg/hostedcontrolplane/controller.go: caCertificatesDuration = 10 * 365 * 24 * time.Hour`) will have consistent long-lived certs from birth. See CAPHCP fork's changes in the summary; do not attempt in-place rotation on existing clusters.

---

## Skyline dev/prod flavor picker sends numeric ID

### Symptom

Cluster create through Skyline UI, cluster ends up with `flavor_id: "3"` even though the user picked "m1.medium" in the dropdown.

### Root cause

Skyline's `select-table` component returns the `rowKey` of the selected row, which is the record's `id` field. Nova returns `{id: "3", name: "m1.medium"}` — Skyline forwards the id.

### Fix

Not a bug in Skyline — Nova legitimately treats "3" as the flavor's canonical identifier. Chart `0.20.0-hcp.5+` handles this correctly (routes numeric to `flavorID`, name to `flavor`).

If you want the user to see and pick by name in the wizard, the UX-only fix is to display `name` in the dropdown and translate to `id` on submit (Skyline already does this). No change needed.

---

## Getting to a mgmt cluster shell from a fresh session

Reproducibility for future ops folks:

```bash
# 1. Tunnel through bastion to mgmt cluster kube-apiserver
ssh -p 2024 -N -L 6443:10.121.55.10:6443 -i ~/.ssh/id_ed25519 \
    ubuntu@165.99.104.126

# 2. Use the mgmt cluster kubeconfig
export KUBECONFIG=~/mgmt-cluster.kubeconfig
kubectl get ns

# 3. Look at the magnum tenant namespace
NS=magnum-<project_id>
kubectl -n $NS get cluster,openstackcluster,machinedeployment,machine,openstackmachine
```

Prod OpenStack API is reachable from Skyline VM (thewahda-main, port 2026 on bastion). Fetch admin credentials from `docker exec skyline-apiserver cat /etc/skyline/skyline.yaml` (search for `system_user_password`).
