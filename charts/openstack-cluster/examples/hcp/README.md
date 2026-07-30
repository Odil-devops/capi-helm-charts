# HCP-mode deployment example

`controlPlane.provider=hostedControlPlane` runs the tenant apiserver / etcd
/ scheduler / controller-manager as pods on the management cluster via
[CAPHCP](https://github.com/teutonet/cluster-api-provider-hosted-control-plane)
instead of dedicated CP VMs.

## Prerequisites (on the management cluster)

- CAPHCP installed
- A TLS-passthrough Gateway (Gateway API) with a wildcard listener,
  e.g. `hostname: "*.k8s.example.com"`
- DNS wildcard `*.k8s.example.com` pointing to an edge that does SNI passthrough
  to the Gateway VIP

## Known gap — `konnectivity-agent`

CAPHCP wires up the server-side of konnectivity (egress selector on the
apiserver + `s-<cluster>` LoadBalancer) but does **not** deploy the agent
DaemonSet on the workload cluster. Without the agent, apiserver → kubelet
tunneling is broken: `kubectl logs`, `exec`, `port-forward`, and any
webhook that goes apiserver→pod all fail with `No agent available`.

Until [teutonet/cluster-api-provider-hosted-control-plane#TBD] lands,
deploy the agent yourself using [`konnectivity-agent.yaml`](./konnectivity-agent.yaml)
after the workload cluster reaches `Provisioned` and the node is Ready:

```bash
# On mgmt cluster: extract kubeconfig + CA that CAPHCP already generated.
kubectl -n hcp-tenants get secret <cluster>-konnectivity-client -o jsonpath='{.data.ca\.crt}' | base64 -d > /tmp/konn-ca.crt
kubectl -n hcp-tenants get secret <cluster>-konnectivity-client -o jsonpath='{.data.tls\.crt}' | base64 -d > /tmp/konn-client.crt
kubectl -n hcp-tenants get secret <cluster>-konnectivity-client -o jsonpath='{.data.tls\.key}' | base64 -d > /tmp/konn-client.key

# Apply to the workload cluster.
kubectl --kubeconfig=<workload-kubeconfig> -n kube-system create secret generic konnectivity-agent-certs \
  --from-file=ca.crt=/tmp/konn-ca.crt \
  --from-file=tls.crt=/tmp/konn-client.crt \
  --from-file=tls.key=/tmp/konn-client.key

# Substitute your FQDN for konnectivity (from the CAPHCP-managed TLSRoute)
# and apply konnectivity-agent.yaml.
KONN_HOST=konnectivity.<cluster>.<hcp-namespace>.k8s.example.com
sed "s|KONNECTIVITY_HOST|$KONN_HOST|g" konnectivity-agent.yaml \
  | kubectl --kubeconfig=<workload-kubeconfig> apply -f -
```
