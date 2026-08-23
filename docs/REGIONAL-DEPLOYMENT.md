# Wahda MKaaS — Regional Deployment Runbook

Bring up **Managed Kubernetes as a Service** (MKaaS) in a new OpenStack region — e.g. Uzbekistan / Tashkent AZ. Follow steps in order; each references the config we've already proven in Hyderabad (`in-north-az1`).

**Pre-audit before starting:** Kolla-Ansible deployment is done, Nova/Neutron/Glance/Cinder/Octavia are healthy, and you have admin credentials.

**Estimated time end-to-end:** ~1 working day (assuming Kolla is already up).

---

## 0. Region Naming Convention

We use a single, region-scoped subdomain for tenant control planes:

```
<cluster-name>-<random>.<REGION-AZ>.k8s.thewahda.com
```

- `<REGION-AZ>`: e.g. `in-north-az1` (Hyderabad), `uz-north-az1` (Tashkent).
- Wildcard DNS `*.<REGION-AZ>.k8s.thewahda.com` points to the edge nginx public IP.
- Skyline / Magnum are region-agnostic; each region has its own OpenStack API endpoint set.

Pick the name up front — it appears in DNS, Envoy Gateway routes, cert templates, and Skyline's `default_region`.

---

## 1. Prerequisites (OpenStack Side)

### 1.1 Nova flavors

Verify at least these exist:

```bash
openstack flavor list
```

Wahda-style defaults (any subset is fine; wire what the templates reference):
- `m1.small`  (1 vCPU / 2 GB / 20 GB) — dev/test workers
- `m1.medium` (2 vCPU / 4 GB / 40 GB) — HCP requires ≥ 2 vCPU per worker
- `m1.large`  (4 vCPU / 8 GB / 80 GB)
- `m1.xlarge` (8 vCPU / 16 GB / 160 GB)

**Important:** Kolla creates flavors with **numeric IDs** (`1`, `2`, `3`...). Chart `0.20.0-hcp.5+` handles this correctly (see [CHANGELOG-HCP.md](../CHANGELOG-HCP.md)). Older chart versions will fail; pin `capi_helm_chart_version=0.20.0-hcp.5` (or later) on every template.

### 1.2 Glance images

Build and upload one Kubernetes worker image per supported version, using [image-builder](https://github.com/kubernetes-sigs/image-builder):

```
ubuntu-2204-kube-v1.33.7
ubuntu-2204-kube-v1.34.9
ubuntu-2204-kube-v1.35.6
```

- Image OS: Ubuntu 22.04.
- Property `os_distro=ubuntu`.
- The Magnum `kube_tag` label MUST match the image's kube version (e.g. `kube_tag: v1.34.9` → image `ubuntu-2204-kube-v1.34.9`).

Uploading an image:

```bash
openstack image create ubuntu-2204-kube-v1.34.9 \
  --disk-format qcow2 --container-format bare \
  --property os_distro=ubuntu \
  --file /path/to/ubuntu-2204-kube-v1.34.9.qcow2
```

### 1.3 Neutron networks

Every workload cluster gets attached to the same shared tenant network (matches Hyderabad `K8S-Network`):

- Network: `K8S-Network` (project-scoped, tagged `mkaas`)
- Subnet: `K8S-Network-Subnet` (CIDR e.g. `10.0.0.0/24`, DHCP on, DNS `10.121.50.51` or local resolver)
- Router: `K8S-Router` connected to external network `Public1`
- Router-gateway: NAT out via external

Workers **do not get floating IPs** — egress via router NAT, ingress via Octavia `type: LoadBalancer` for user services.

### 1.4 Octavia

Verify Octavia has enough capacity to run amphora VMs — the driver creates one Octavia LB per workload cluster's `type: LoadBalancer` Service (and one internal LB on the mgmt cluster per HCP for the tenant apiserver Service).

---

## 2. Management Kubernetes Cluster (CAPI-mgmt)

Bring up a small k8s cluster on this region's OpenStack to run CAPI/CAPHCP/CAPO and the addon provider. In Hyderabad:

- `capi-cplane-1/2/3` (10.121.55.10/11/12) — 3 control plane nodes
- `capi-worker-1/2` (10.121.55.20/21) — 2 workers
- Version: `v1.32.10` (Kubernetes)

Any HA k8s tool works. Suggested: kubeadm-init + `kubeadm join`, or CAPI-managed bootstrap from an existing cluster. Configure:

- **CNI**: Cilium (matches tenant clusters' CNI for consistency).
- **CSI**: cloud-provider-openstack Cinder driver — for PVCs used by CAPHCP etcd.
- **CCM**: cloud-provider-openstack — enables `type: LoadBalancer` for tenant apiserver Services.
- **kubeconfig**: copy off-box; store securely. This is the god-key of the region.

### 2.1 Install the CAPI stack

```bash
export CLUSTER_TOPOLOGY=true
clusterctl init \
  --core cluster-api:v1.13.3 \
  --bootstrap kubeadm:v1.13.3 \
  --control-plane kubeadm:v1.13.3 \
  --infrastructure openstack:v0.14.6
```

### 2.2 Install CAPHCP fork

Our fork lives at `github.com/Odil-devops/cluster-api-provider-hosted-control-plane` (image `ghcr.io/odil-devops/cluster-api-provider-hosted-control-plane:latest`).

Apply its bundle manifest (`config/default`) — includes CRDs, RBAC, controller Deployment. Verify `capi-hosted-control-plane-system/caphcp-controller-manager` is `Running`.

### 2.3 Install Envoy Gateway

Envoy Gateway does TLS passthrough for tenant apiservers. Install upstream chart:

```bash
helm install envoy-gateway oci://docker.io/envoyproxy/gateway-helm \
  --version v1.4.2 --namespace envoy-gateway-system --create-namespace
```

Then apply the shared Gateway + GatewayClass (see `docs/envoy-gateway-config.yaml` in this repo — TODO: commit that file).

### 2.4 Install cert-manager

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.16.2/cert-manager.yaml
```

### 2.5 Install cluster-autoscaler (per-tenant provisioner)

The addon provider (`capi-addons`) installs the autoscaler into each tenant's namespace during helm-upgrade.

### 2.6 Install k-orc (Kubernetes OpenStack Resource Controller)

Handles OpenStack side of the tenant lifecycle:

```bash
kubectl apply -f https://github.com/k-orc/openstack-resource-controller/releases/download/v2.2.0/install.yaml
```

---

## 3. Magnum + magnum-capi-helm Driver

### 3.1 Deploy magnum-capi-helm

Our patches live at `~/Downloads/magnum-capi-helm-patches/magnum-patches/driver.py` (TODO: put these in a proper fork with release tags). Install into `magnum_conductor` container as an out-of-tree driver.

Kolla-side, in `/etc/kolla/globals.yml`:

```yaml
magnum_capi_helm_driver_enabled: true
magnum_capi_helm_default_chart_repo: https://odil-devops.github.io/capi-helm-charts
magnum_capi_helm_default_chart_version: 0.20.0-hcp.5
```

Reconfigure Magnum:

```bash
kolla-ansible -i inventory reconfigure -t magnum
```

### 3.2 Kubeconfig for the driver

The driver invokes `helm upgrade --kubeconfig /var/lib/magnum/kubeconfig` — so the mgmt cluster's kubeconfig must be mounted into `magnum_conductor` at that path. Steps:

- Copy the mgmt-cluster kubeconfig onto every Kolla control host.
- Volume-mount `/etc/kolla/magnum-conductor/kubeconfig` (host) → `/var/lib/magnum/kubeconfig` (container).
- Restart `magnum_conductor`.

### 3.3 Allowed-flavor guard

The driver has `_validate_allowed_flavor` which enforces a per-project whitelist. Configure via magnum policy — see the patches file.

---

## 4. Skyline UI

Deploy the console (fork `github.com/thewahdaa/skyline-console`) on a dedicated VM in the region. Bumped image lands via GHA `deploy.yml` on push to `main`.

### 4.1 skyline.yaml (per-region)

```yaml
openstack:
  interface_type: public
  keystone_url: https://console.<REGION-DOMAIN>:5000/v3/
  nginx_prefix: /api/openstack
  default_region: <REGION-CODE>          # e.g. in-north-1, uz-north-1
  system_admin_roles: [admin, system_admin]
  system_project: admin
  system_project_domain: Default
  system_user_domain: Default
  system_user_name: admin
  system_user_password: <STRONG>
```

### 4.2 Nginx reverse proxy

Reverse-proxy each OpenStack service through the Skyline host to the internal Kolla VIP (in Hyderabad, `10.121.50.50`). See `/etc/nginx/sites-available/console.<REGION-DOMAIN>` on the Skyline host — one `upstream` + `location /api/openstack/<REGION-CODE>/<service>/` block per service. **Magnum needs its own block** or clusters won't show in the UI.

### 4.3 CI/CD wiring

GHA secrets on `thewahdaa/skyline-console` for the region's deploy:

- `DEPLOY_HOST` — public IP / DNS of the Skyline VM
- `DEPLOY_HOST_USER` — deploy user
- `DEPLOY_HOST_PRIVATE_KEY` — SSH key
- `DEPLOY_PORT` — SSH port

For a **new region**, add a second deploy job in `.github/workflows/deploy.yml` (or convert to a matrix over regions) so the same commit deploys everywhere.

---

## 5. DNS + Edge Nginx (Tenant Apiserver Routing)

### 5.1 DNS

Add to Cloudflare zone `thewahda.com`:

```
*.<REGION-AZ>.k8s.thewahda.com   A   <PUBLIC-IP-OF-EDGE-NGINX>   grey cloud (DNS-only)
```

**Grey cloud** (no CF proxy). CF orange cloud would break SNI passthrough — TLS is end-to-end from `kubectl` to the tenant apiserver pod.

### 5.2 Edge nginx

Runs on a small VM with a public IP (Hyderabad: `165.99.104.123`). Config template:

```nginx
stream {
  map $ssl_preread_server_name $upstream {
    default 10.121.55.64:443;  # internal Envoy Gateway LB VIP
  }
  server {
    listen 443;
    proxy_pass $upstream;
    ssl_preread on;
    proxy_protocol on;
  }
}
```

Envoy Gateway on the mgmt cluster then peels SNI and routes to the tenant apiserver Service. Cert issued in-cluster by CAPHCP (10-year lifetime — see fork changes).

---

## 6. Cluster Templates

**MUST include** `capi_helm_chart_version` label — otherwise Magnum uses the driver default which may drift.

Create one template per supported Kubernetes minor version.

### 6.1 v1.34-hcp (example)

```bash
openstack coe cluster template create \
  --coe kubernetes \
  --image ubuntu-2204-kube-v1.34.9 \
  --flavor m1.medium \
  --master-flavor m1.medium \
  --network-driver calico \
  --external-network Public1 \
  --fixed-network K8S-Network \
  --fixed-subnet K8S-Network-Subnet \
  --dns-nameserver 10.121.50.51 \
  --master-lb-enabled \
  --labels capi_cp_provider=hostedControlPlane,kube_tag=v1.34.9,octavia_provider=amphora,master_lb_floating_ip_enabled=false,capi_helm_chart_version=0.20.0-hcp.5 \
  kubernetes-v1.34-hcp
```

Repeat for v1.33, v1.35. **Never omit `capi_helm_chart_version`.**

### 6.2 Verifying labels

```bash
openstack coe cluster template show kubernetes-v1.34-hcp -f value -c labels
```

Confirm `capi_helm_chart_version=0.20.0-hcp.5` is present.

### 6.3 Bumping chart version on existing templates

Templates in use by any cluster refuse updates (`ClusterTemplate ... is referenced by one or multiple clusters`). To update:

1. Delete or migrate any dependent clusters (or spin up a new template with the new version and switch new customers to it).
2. `PATCH /v1/clustertemplates/{uuid}` with JSON Patch:
   ```json
   [{"op": "replace", "path": "/labels/capi_helm_chart_version", "value": "0.20.0-hcp.6"}]
   ```
   (Use `"op": "add"` if the label is not yet set.)

Existing running clusters keep the chart snapshot they were created with (`helm.sh/resource-policy: keep` on OpenStackMachineTemplate).

---

## 7. First Cluster Smoke Test

```bash
openstack coe cluster create smoketest-01 \
  --cluster-template kubernetes-v1.34-hcp \
  --keypair mykeypair \
  --node-count 1 \
  --labels auto_healing_enabled=true,auto_scaling_enabled=true,min_node_count=1,max_node_count=3
```

Watch:

```bash
openstack coe cluster show smoketest-01 -f value -c status
```

Expected: `CREATE_IN_PROGRESS` → `CREATE_COMPLETE` in ~5-8 min. Then health check:

```bash
openstack coe cluster show smoketest-01 -f value -c health_status
# → HEALTHY
```

Retrieve kubeconfig and verify:

```bash
openstack coe cluster config smoketest-01 --dir /tmp/smoketest
export KUBECONFIG=/tmp/smoketest/config
kubectl get nodes    # 1 worker node Ready
kubectl -n kube-system get pods   # CoreDNS, CCM, Cilium all Running
```

Delete the smoketest cluster after verification.

---

## 8. Region-Level Checklist (Copy this into your launch ticket)

- [ ] **1.1** Nova flavors present + tested (`openstack flavor list`)
- [ ] **1.2** Glance images built for each supported K8s minor (`ubuntu-2204-kube-v1.X.Y`)
- [ ] **1.3** Neutron `K8S-Network` + subnet + router + external gateway
- [ ] **1.4** Octavia amphora capacity verified
- [ ] **2** Management k8s cluster on OpenStack, kubeconfig off-box
- [ ] **2.1** CAPI + CAPO + kubeadm bootstrap installed
- [ ] **2.2** CAPHCP fork controller running
- [ ] **2.3** Envoy Gateway installed + Gateway/GatewayClass applied
- [ ] **2.4** cert-manager installed
- [ ] **2.6** k-orc installed
- [ ] **3.1** magnum-capi-helm driver deployed + Kolla globals updated
- [ ] **3.2** Mgmt kubeconfig mounted into magnum_conductor
- [ ] **4** Skyline console VM deployed + CI wired
- [ ] **4.2** Nginx reverse proxy includes Magnum route
- [ ] **5.1** DNS `*.<REGION-AZ>.k8s.thewahda.com` → edge nginx public IP (grey cloud)
- [ ] **5.2** Edge nginx SNI passthrough config live
- [ ] **6** 3 HCP cluster templates created, each with `capi_helm_chart_version=<current>`
- [ ] **7** Smoke-test cluster created + `HEALTHY`
- [ ] **7** Smoke-test cluster deleted

---

## 9. Related Docs

- [CHANGELOG-HCP.md](../CHANGELOG-HCP.md) — chart release history + rollout instructions.
- [TROUBLESHOOTING.md](TROUBLESHOOTING.md) — known failure modes, symptoms, fixes.
- Upstream refs: [azimuth-cloud/capi-helm-charts](https://github.com/azimuth-cloud/capi-helm-charts), [stackhpc/magnum-capi-helm](https://github.com/stackhpc/magnum-capi-helm).
