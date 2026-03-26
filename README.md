# cilium-strict-policies

A collection of `CiliumClusterwideNetworkPolicy` resources for hardening a self-hosted Kubernetes cluster with Cilium in strict enforcement mode.

## Overview

These policies are designed for a multi-node cluster running Cilium with:

- `policyEnforcementMode: always`
- `hostFirewall: enabled: true`
- `policyAuditMode: false` (audit mode used only during development)

## Cluster Topology

| Role | Count | IP Range |
|------|-------|----------|
| Control Plane | 3 | `192.168.4.180`, `192.168.4.188`, `192.168.4.189` |
| Workers | 4 | `192.168.4.181` – `192.168.4.184` |
| External VIP (HAProxy) | 1 | `192.168.4.190` |

Access to the cluster is provided via SSH port forwarding through the gateway to the first master node.

## Policy Structure

```
.
└── cluster-policies/
    ├── etcd.yaml               # etcd peer and client traffic (control-plane only)
    ├── vxlan.yaml              # VXLAN overlay traffic (port 8472/UDP)
    ├── icmp.yaml               # ICMP health probes between nodes and endpoints
    ├── kube-api.yaml           # kube-apiserver and kubelet access (port 6443, 10250)
    ├── hubble.yaml             # Hubble relay, UI, and peer service
    ├── health.yaml             # Cilium health checks (port 4240)
    ├── coredns.yaml            # CoreDNS cluster and external resolution
└── host-policies/
    ├── ssh.yaml            # SSH access to nodes (port 22)
    ├── ntp.yaml            # NTP time sync (port 123/UDP)
    ├── dns.yaml            # DNS egress from host (port 53)
    └── updates.yaml        # Package updates egress (port 80, 443)
```

## Key Design Decisions

**`nodeSelector: {}`** is used for host-level firewall rules (applied to the node itself), while **`endpointSelector`** is used for pod-level policies.

**etcd** traffic is split by port: `2380` for peer-to-peer replication between control-plane nodes (`remote-node` entity), and `2379` for client access (`kube-apiserver` entity only).

**VIP traffic** to the external HAProxy (`192.168.4.190`) is explicitly allowed via CIDR egress rules since Cilium classifies it as a `world` identity rather than a known cluster entity.

**Hubble relay** requires both `remote-node` and `host` entities in its egress policy on port `4244` to reach Cilium agents on all nodes including the local one.

## Prerequisites

- Cilium >= 1.14
- `enable-remote-node-identity: "true"` in `cilium-config` ConfigMap
- `hostFirewall.enabled: true` in Helm values

## Usage

### Apply all policies

```bash
kubectl apply -f cluster-policies/
kubectl apply -f host-policies/
```

### Recommended rollout procedure

1. Deploy policies with `policyAuditMode: true`
2. Monitor for would-be drops:
   ```bash
   kubectl exec -it <cilium-pod> -n kube-system -- \
     cilium-dbg monitor --type policy-verdict
   ```
3. Verify no critical traffic is audited
4. Disable audit mode and restart Cilium daemonset:
   ```bash
   kubectl rollout restart ds/cilium -n kube-system
   ```
5. Monitor for actual drops:
   ```bash
   kubectl exec -it <cilium-pod> -n kube-system -- \
     cilium-dbg monitor --type drop
   ```

## Roadmap

- [ ] Wrap policies in a Helm chart with configurable values
- [ ] Add support for toggling individual policy groups via `values.yaml`
- [ ] Parameterize node CIDRs and VIP address
- [ ] Add IPv6 ICMP rules