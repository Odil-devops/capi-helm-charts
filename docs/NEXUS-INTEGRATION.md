# Wahda MKaaS — Nexus Registry Mirror Integration

Every image tenant clusters pull is proxied through the in-region Nexus. Purpose: **fast cluster creation, low egress cost, resilience during upstream registry outages.**

This doc covers the runbook for **wiring Nexus into a new region** (e.g. Uzbekistan). The Hyderabad case is the reference deployment.

---

## 1. Architecture

```
     tenant worker VM (10.0.0.x on K8S-Network)
              │  containerd hosts.toml mirror lookup
              ▼
     10.121.50.22:{8083..8087}   Nexus proxy repos (docker-hub, quay, ghcr, k8s, nvcr)
              │  cache-miss → upstream fetch
              ▼
       upstream registries (docker.io, quay.io, ...)
```

**Key design decisions:**

- **No HAProxy / VIP proxy** — tenant workers on Neutron `K8S-Network` reach the Nexus VM directly via the underlay's routing. The K8S-Network router already has a path to `10.121.50.0/24` (Kolla mgmt subnet).
- **Nexus lives outside OpenStack** — Proxmox VM (Hyderabad: node 106, `10.121.50.22`). This puts it on the same L2 as the Kolla mgmt subnet without competing for OpenStack quota.
- **Proxy repos, not group repos** — one Nexus docker-proxy connector per upstream, on its own port. Simpler containerd `hosts.toml` (one host per file); no aggregation headaches.
- **Chart applies mirrors by default** — since chart `0.20.0-hcp.6`, `values.yaml` sets `registryMirrors` to the Nexus URLs. Any cluster using this chart auto-uses Nexus. No per-template label required.

## 2. Nexus VM sizing (per region)

Reference (Hyderabad):
- **CPU**: 4 vCPU
- **RAM**: 4 GB (`INSTALL4J_ADD_VM_PARAMS: "-Xms1500m -Xmx1500m -XX:MaxDirectMemorySize=1500m"`, leaves ~1 GB for OS + Docker daemon)
- **Disk**: 200 GB (blob store grows with cached images; monitor `/data/nexus/blobs`)
- **OS**: Ubuntu 22.04 LTS
- **Container**: `sonatype/nexus3:latest` via `docker-compose`
- **Data volume**: `/data/nexus:/nexus-data`

Deploy manifest: see `~/nexus/docker-compose.yml` on the Nexus VM. Ports `8081` (UI/API), `8082` (docker group — reserved, not yet used), `8083-8087` (per-registry proxies).

## 3. Bootstrapping a new Nexus (fresh region)

### 3.1 Provision the VM

Create Proxmox VM per §2. Ensure it has:
- Static IP on the Kolla mgmt subnet (e.g. `10.121.50.22` in Hyderabad, pick equivalent in the new region).
- Docker + docker-compose installed.
- No firewall blocking `8081-8087` inbound.

### 3.2 Start Nexus

```bash
mkdir -p /opt/nexus /data/nexus
# copy docker-compose.yml from Hyderabad reference
cd /opt/nexus && docker compose up -d
```

Wait ~2 minutes for first-boot init. Grab the auto-generated admin password:

```bash
docker exec nexus cat /nexus-data/admin.password
```

Save it — it's deleted on first admin login.

### 3.3 Complete the Sign-In wizard

Access `http://<nexus-vm-ip>:8081` (via SSH tunnel or corp VPN). Log in as `admin` with the file password. Choose:

- New admin password (record in your password manager or the ops vault).
- **Enable anonymous access** — required so tenant containerd pulls without credentials.

### 3.4 Create proxy repositories

Do this via REST API (idempotent; suitable for automation):

```bash
NX=http://<nexus-vm-ip>:8081
AUTH="admin:<password>"

# docker.io (port 8083)
curl -s -u "$AUTH" -X POST "$NX/service/rest/v1/repositories/docker/proxy" \
  -H 'Content-Type: application/json' -d '{
    "name": "docker-hub",
    "online": true,
    "storage": {"blobStoreName": "default", "strictContentTypeValidation": true},
    "proxy": {"remoteUrl": "https://registry-1.docker.io", "contentMaxAge": 1440, "metadataMaxAge": 1440},
    "negativeCache": {"enabled": true, "timeToLive": 1440},
    "httpClient": {"blocked": false, "autoBlock": true},
    "docker": {"v1Enabled": false, "forceBasicAuth": false, "httpPort": 8083},
    "dockerProxy": {"indexType": "HUB", "cacheForeignLayers": false}
  }'

# quay.io (port 8084)
curl -s -u "$AUTH" -X POST "$NX/service/rest/v1/repositories/docker/proxy" \
  -H 'Content-Type: application/json' -d '{
    "name": "quay",
    "online": true,
    "storage": {"blobStoreName": "default", "strictContentTypeValidation": true},
    "proxy": {"remoteUrl": "https://quay.io", "contentMaxAge": 1440, "metadataMaxAge": 1440},
    "negativeCache": {"enabled": true, "timeToLive": 1440},
    "httpClient": {"blocked": false, "autoBlock": true},
    "docker": {"v1Enabled": false, "forceBasicAuth": false, "httpPort": 8084},
    "dockerProxy": {"indexType": "REGISTRY"}
  }'

# ghcr.io (port 8085)
curl -s -u "$AUTH" -X POST "$NX/service/rest/v1/repositories/docker/proxy" \
  -H 'Content-Type: application/json' -d '{
    "name": "ghcr",
    "online": true,
    "storage": {"blobStoreName": "default", "strictContentTypeValidation": true},
    "proxy": {"remoteUrl": "https://ghcr.io", "contentMaxAge": 1440, "metadataMaxAge": 1440},
    "negativeCache": {"enabled": true, "timeToLive": 1440},
    "httpClient": {"blocked": false, "autoBlock": true},
    "docker": {"v1Enabled": false, "forceBasicAuth": false, "httpPort": 8085},
    "dockerProxy": {"indexType": "REGISTRY"}
  }'

# registry.k8s.io + k8s.gcr.io (port 8086)
curl -s -u "$AUTH" -X POST "$NX/service/rest/v1/repositories/docker/proxy" \
  -H 'Content-Type: application/json' -d '{
    "name": "k8s",
    "online": true,
    "storage": {"blobStoreName": "default", "strictContentTypeValidation": true},
    "proxy": {"remoteUrl": "https://registry.k8s.io", "contentMaxAge": 1440, "metadataMaxAge": 1440},
    "negativeCache": {"enabled": true, "timeToLive": 1440},
    "httpClient": {"blocked": false, "autoBlock": true},
    "docker": {"v1Enabled": false, "forceBasicAuth": false, "httpPort": 8086},
    "dockerProxy": {"indexType": "REGISTRY"}
  }'

# nvcr.io (port 8087) — NVIDIA GPU operator images
# NOTE: if the region will deploy paid NVIDIA products, add "authentication"
# block with the enterprise NGC api-key. For open OSS images (gpu-operator, etc.)
# anonymous proxy is enough.
curl -s -u "$AUTH" -X POST "$NX/service/rest/v1/repositories/docker/proxy" \
  -H 'Content-Type: application/json' -d '{
    "name": "nvcr",
    "online": true,
    "storage": {"blobStoreName": "default", "strictContentTypeValidation": true},
    "proxy": {"remoteUrl": "https://nvcr.io", "contentMaxAge": 1440, "metadataMaxAge": 1440},
    "negativeCache": {"enabled": true, "timeToLive": 1440},
    "httpClient": {"blocked": false, "autoBlock": true},
    "docker": {"v1Enabled": false, "forceBasicAuth": false, "httpPort": 8087},
    "dockerProxy": {"indexType": "REGISTRY"}
  }'
```

Each `POST` returns 201 on create, 400 if name collision.

### 3.5 Verify

```bash
# List repos
curl -s -u "$AUTH" "$NX/service/rest/v1/repositories" | jq '.[].name'

# Trigger a pull-through to prime the cache
curl -s "http://<nexus-ip>:8083/v2/library/hello-world/manifests/latest" -o /dev/null -w '%{http_code}\n'
# Expect 200
```

---

## 4. Wiring the chart (region-specific)

The chart's `values.yaml` (from `0.20.0-hcp.6`) has Hyderabad's Nexus baked in:

```yaml
registryMirrors:
  docker.io:     [{url: http://10.121.50.22:8083}]
  quay.io:       [{url: http://10.121.50.22:8084}]
  ghcr.io:       [{url: http://10.121.50.22:8085}]
  registry.k8s.io: [{url: http://10.121.50.22:8086}]
  k8s.gcr.io:    [{url: http://10.121.50.22:8086}]
  nvcr.io:       [{url: http://10.121.50.22:8087}]
```

**For a new region**, either:

- **A. Fork values-per-region** — clone this chart, replace the IP with the new region's Nexus IP, release as `0.20.0-hcp.6-uz` or similar. Pin the region's cluster templates to that version via `capi_helm_chart_version`.
- **B. Neutron IP overlap trick** — assign the new region's Nexus VM the SAME IP `10.121.50.22`. Networks are per-region isolated so no collision. Chart values.yaml stays unchanged. This is what we recommend for launch — one chart across regions.

Either way, chart's `_helpers.tpl` writes `/etc/containerd/certs.d/<upstream>/hosts.toml` on every worker VM via `KubeadmConfigSpec.files`, and `preKubeadmCommands` restarts containerd.

---

## 5. Verifying pulls are proxied through Nexus

On a running tenant worker (SSH via `k8s-jump` keypair):

```bash
# Watch containerd resolve the mirror
sudo journalctl -u containerd -f | grep -i registry

# Look at hosts.toml
sudo cat /etc/containerd/certs.d/docker.io/hosts.toml
# Expect: [host."http://10.121.50.22:8083"]

# Force a pull and see it hit Nexus
sudo ctr -n=k8s.io images pull docker.io/library/nginx:latest
```

On the Nexus VM:

```bash
docker logs -f nexus 2>&1 | grep -i 'GET /v2'
# You should see requests for /v2/library/nginx/manifests/latest etc.
```

---

## 6. Operations

### 6.1 Blob store size

Monitor with:

```bash
du -sh /data/nexus/blobs
```

Grows ~5-15 GB per cluster's initial image pulls. Set a Prometheus alert at 80 % of disk.

### 6.2 Cache eviction

Nexus can be configured with a "Cleanup Task" — image blobs unused for > N days get pruned. Recommended: 90 days for prod, 30 for dev.

Admin UI → *Tasks → Create task → Admin — Cleanup repositories using their associated policies*.

### 6.3 Upstream registry outage

Nexus keeps serving from cache while the upstream is down (assuming the image was pulled at least once before). Alerts on cache-miss + upstream fail:

```
Nexus admin.log:
  ...WARN...HttpClient...connect timed out
```

Route to alerting.

### 6.4 What happens if Nexus itself is down

`containerd` falls back to the upstream when the mirror returns a non-2xx or times out — that's the `server = "..."` line at the top of each `hosts.toml`. **Cluster creation continues, just slower.** So a Nexus outage does not block MKaaS.

---

## 7. Roadmap items

- [ ] **Nexus HA** — single VM is a single point of latency spike. Move to a two-node Nexus with shared blob storage (S3) once traffic warrants it.
- [ ] **TLS on Nexus** — currently HTTP for internal traffic. If any hop of the underlay is untrusted, wrap Nexus in nginx TLS terminator. Update chart values with `skipVerify` handling.
- [ ] **Per-registry auth** — nvcr enterprise, ghcr private repos, etc. Add `authentication` blocks to proxy repos; store creds in Vault; sync via automation.
- [ ] **Chart HTTPS mirror URLs** — CAPO's `KubeadmConfigSpec.files` supports contentFrom.secret — pass mirror URLs as a secret if we ever want per-cluster customization without values change.

---

## 8. Related docs

- [REGIONAL-DEPLOYMENT.md](REGIONAL-DEPLOYMENT.md) — sections 1-2 (prerequisites) reference Nexus as a mandatory item.
- [../CHANGELOG-HCP.md](../CHANGELOG-HCP.md#0200-hcp6---2026-08-24) — chart release that introduced Nexus by default.
- [TROUBLESHOOTING.md](TROUBLESHOOTING.md) — see "container image pulls slow / cluster take > 20 min" section (to be added after benchmarking).
