# capi-helm-charts — HCP-support fork

> **This is a fork of [azimuth-cloud/capi-helm-charts](https://github.com/azimuth-cloud/capi-helm-charts)** that
> adds support for **hyperscaler-style hosted control planes**: instead of running the tenant Kubernetes
> control plane as VMs in the customer's OpenStack project, the API server, etcd, scheduler and
> controller-manager run as pods on a shared management cluster — matching the operational model of
> AWS EKS, GCP GKE, Azure AKS and Red Hat HyperShift.
>
> All upstream chart features work unchanged; the HCP branch activates only when
> `controlPlane.provider=hostedControlPlane` is set in values.

## What this fork adds

| Feature | Upstream | This fork |
| --- | --- | --- |
| Traditional CAPI cluster (dedicated CP VMs) | ✅ | ✅ |
| HostedControlPlane rendering (CP as pods on mgmt cluster) | — | ✅ |
| OpenStack CCM on the management cluster | — | ✅ *(EKS/GKE/HyperShift pattern)* |
| CCM/CSI/CNI addons runnable on workers (no CP nodes) | — | ✅ *(via `controlPlaneOnly=false` gates)* |
| Autoscaler runaway guardrails (`ok-total-unready-count`, `scale-down-utilization-threshold`) | partial | ✅ |
| Cascade delete via CAPI ownerReferences | ✅ | ✅ *(plus hosted CCM/ConfigMap)* |

See [`charts/openstack-cluster/examples/hcp/`](./charts/openstack-cluster/examples/hcp/) for the full HCP-mode
deployment guide and reference manifests (konnectivity-agent, sample values.yaml).

## Prerequisites for HCP mode

On the **management cluster**:

- Kubernetes 1.30+ with Cluster API v1.13+ and the OpenStack infrastructure provider (CAPO v0.14+)
- [teutonet/cluster-api-provider-hosted-control-plane (CAPHCP)](https://github.com/teutonet/cluster-api-provider-hosted-control-plane) v1.7+
- Gateway API v1.3+ with a TLS-passthrough Gateway listening on `*.<your-domain>`
- A public SNI-passthrough edge in front of the Gateway VIP (nginx `ssl_preread` or haproxy TCP+SNI)

Traditional kubeadm mode has no extra prerequisites over upstream.

## Installation

```sh
helm repo add wahda-mkaas https://odil-devops.github.io/capi-helm-charts
helm repo update
helm search repo wahda-mkaas --versions
```

For an HCP-mode install:

```sh
helm install my-cluster wahda-mkaas/openstack-cluster \
  --namespace hcp-tenants --create-namespace \
  --timeout 15m \
  -f charts/openstack-cluster/examples/hcp/values.yaml
```

For a traditional kubeadm-mode install (fully upstream-compatible):

```sh
helm install my-cluster wahda-mkaas/openstack-cluster \
  --namespace magnum-<project> \
  -f values.yaml
```

## Available charts

| Chart | Description |
| --- | --- |
| [cluster-addons](./charts/cluster-addons) | Deploys addons into a Kubernetes cluster, e.g. CNI. |
| [etcd-defrag](./charts/etcd-defrag/) | Installs a `CronJob` for running [etcd defragmentation](https://etcd.io/docs/v3.5/op-guide/maintenance/#defragmentation). |
| [openstack-cluster](./charts/openstack-cluster) | Deploys a Kubernetes cluster on an OpenStack cloud. |

## Attribution and license

Fork of [azimuth-cloud/capi-helm-charts](https://github.com/azimuth-cloud/capi-helm-charts), licensed under Apache-2.0.
All added HCP-mode work is offered under the same license. Upstream commits are periodically merged into
this fork's `main`; HCP-mode changes live in `feat/hosted-control-plane` and are released under tags
prefixed `hcp-v`.
