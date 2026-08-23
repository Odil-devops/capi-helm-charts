# Wahda MKaaS — Chart Fork Changelog (HCP releases)

This tracks the hosted-control-plane (`hcp.N`) release cadence of the `openstack-cluster` and `cluster-addons` charts on this fork (`Odil-devops/capi-helm-charts`, branch `feat/hosted-control-plane`).

Charts are consumed by the `magnum-capi-helm` driver on Wahda's OpenStack. Pin the version per cluster template via the label:

```yaml
labels:
  capi_helm_chart_version: 0.20.0-hcp.5
```

Rollout for a new version:

1. Bump `Chart.yaml` + `Chart.lock` + `charts/cluster-addons/Chart.yaml` to the new version.
2. Commit + tag `hcp-vX.Y.Z-hcp.N` + push tag → triggers `.github/workflows/release-hcp.yaml`.
3. If the CI hits the chart-releaser race (see [docs/TROUBLESHOOTING.md](docs/TROUBLESHOOTING.md#chart-releaser-race)), manually publish (steps in that doc).
4. `PATCH /v1/clustertemplates/{uuid}` on every HCP template to update the `capi_helm_chart_version` label. Existing clusters keep their pinned chart (`helm.sh/resource-policy: keep`); only new / upgraded clusters pick up the new chart.

---

## 0.20.0-hcp.5 — 2026-08-23

**Fix — flavor lookup by ID for numeric flavor identifiers.**

- Charts: `openstack-cluster` (both `templates/node-group/openstack-machine-template.yaml` and `templates/control-plane/openstack-machine-template.yaml`).
- Behaviour: if `machineFlavor` matches `^[0-9]+$` (a legacy Nova numeric ID such as `3`), render `flavorID: "3"`; otherwise render `flavor: "<name>"`.
- Rationale: CAPO's `OpenStackMachineTemplate.spec.template.spec.flavor` is resolved as a Nova **name**, not ID. Passing an all-digit legacy ID caused `no flavors were found: name=3` and workers never provisioned. CAPO v0.14.6 supports `spec.flavorID` (takes precedence over `flavor`) which does a by-ID lookup.
- Impact: regions whose Nova has legacy numeric flavor IDs (Kolla default `1/2/3/4/...`) now work through the Skyline UI, where the flavor picker returns `{id: "3"}` (the Nova record's `id` field) instead of the flavor name.

## 0.20.0-hcp.4 — 2026-08-23

**Fix — quote the flavor value so YAML doesn't coerce it to integer.**

- Same two templates as above.
- Behaviour: added `| quote` to `machineFlavor` output.
- Rationale: helm text-template rendering of a bare numeric-string value produced `flavor: 42` (unquoted), which the k8s API server parses as a JSON integer and rejects with `spec.template.spec.flavor: Invalid value: "integer": ... must be of type string`.
- Impact: superseded by hcp.5 (which changes the field, not just the quoting). Left here for historical context.
- This bug is present upstream (`azimuth-cloud/capi-helm-charts`) as well.

## 0.20.0-hcp.3 — 2026-08 (earlier)

**Cleanup — drop the CoreDNS UDP:53 NetworkPolicy workaround.**

- Chart: `cluster-addons`.
- Rationale: CAPHCP previously had a NetworkPolicy gap for CoreDNS on UDP 53 in-cluster. That was fixed upstream; the local workaround (`fix(cluster-addons): work around CAPHCP v1.7.2 CoreDNS NetworkPolicy UDP:53 gap`, commit `bacec8cf`) is no longer needed.

## 0.20.0-hcp.2 — 2026-07

Bump to align with upstream 0.20.0 base.

## 0.20.0-hcp.1 — 2026-07

Initial hosted-control-plane variant.
