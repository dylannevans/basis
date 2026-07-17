# basis

Infrastructure for a healthy LAN, subscribed via Flux GitOps. Extracted from
`simplesalt/base-stack` so LAN-level networking (MetalLB LoadBalancer IPs +
DNS/DHCP) lives in one place.

## What's here

| Path | Contents |
|---|---|
| `metallb/` | MetalLB install: `metallb-system` namespace, HelmRepository, HelmRelease. |
| `network/` | MetalLB address pools + L2 (ARP) advertisement, the traefik VIP pin, and DNS: pihole (DNS+DHCP) + k8s-gateway (in-cluster upstream) + the `dns.famevans.win` Ingress/cert. |
| `flux/wiring.example.yaml` | Reference GitRepository + Kustomizations to add to **base-stack** to subscribe to this repo. |

## MetalLB vs kube-vip — they are NOT the same VIP

The multi-server k3s image (`simplesalt/oci` `cluster/`) runs a **kube-vip**
DaemonSet from the k3s static-manifests dir. That provides the **control-plane
VIP** (a floating apiserver address) via ARP/L2 across the HA server nodes.

**MetalLB is unrelated to that.** It hands out **`type: LoadBalancer` service
IPs** — here the dedicated, dual-stack addresses that LAN clients depend on:

| Pool | IPv4 | Consumer |
|---|---|---|
| `dns-pool` | `192.168.1.2` | pihole DNS/DHCP (clients hardcode this resolver) |
| `traefik` | `192.168.1.3` | k3s built-in traefik ingress |
| `fe-apps` | `192.168.1.32-64` | front-end app LoadBalancers |

kube-vip in this configuration does **not** provide these, so MetalLB stays.
(Moving to the new image is additive: you gain control-plane HA and keep
MetalLB unchanged.) MetalLB here already runs in **L2/ARP mode**
(`L2Advertisement`) — the "BGP" note in `network/metallb-pools.yaml` is a
long-term aspiration, not the running config.

## Dependencies that remain in base-stack

basis assumes these already exist on the cluster (provided by base-stack). All
are *soft* runtime references except cert-manager/metallb CRDs, which are the
only *hard* (apply-order) requirement — so basis must reconcile after
base-stack's `1.basis` layer.

- **Flux controllers** (source-controller, helm-controller, kustomize-controller).
- **cert-manager** controller + CRDs. The `famevans` ClusterIssuer itself now
  lives in basis (`network/famevans-issuer.yaml`); only the operator stays in
  base-stack. The `simplesalt` ClusterIssuer (public domain) also stays.
- **traefik** (k3s built-in). `network/metallb-pools.yaml` only pins traefik's
  LB IP to the `traefik` pool via a `HelmChartConfig`; `network/dns.yaml` also
  hardcodes traefik's clusterIP (`10.43.171.237`) as the pihole upstream for
  `*.famevans.win` — fragile, worth converting to a DNS name later.

## Cross-repo consumers left in base-stack

These base-stack resources reference basis-owned objects by name/annotation
(soft; they just need basis reconciled):

- `3.infra/local-fs.yaml` — the `fs` SMB LoadBalancer consumes the basis
  `fe-apps` pool (stays `Pending` if basis isn't applied).
- `4.ss/cal.yaml` (`cal-tls`) — references the basis `famevans` ClusterIssuer.
  **cal.com's canonical URL IS `cal.famevans.win`** (hardcoded WEBAPP_URL +
  auth callback) and it is NOT published via CF tunnel, so its famevans ingress
  is load-bearing — do not treat it as disposable.
- `3.infra/mcp-k8s.yaml` (`famevans-tls-svcs`) — same issuer ref; this one *is*
  LAN-only/disposable (not in any CF tunnel).

## Cutover (test plan)

1. Push this repo's `main`.
2. In **base-stack**, add `flux/wiring.example.yaml` (a `basis` GitRepository +
   `basis-metallb` / `basis-network` Kustomizations).
3. Comment out the now-duplicated resources in base-stack so Flux doesn't fight
   for ownership of the same objects:
   - `1.basis/controllers.yaml` — the `metallb-system` Namespace + `metallb`
     HelmRelease.
   - `1.basis/helmrepos.yaml` — the `metallb`, `pihole`, `k8s-gateway`
     HelmRepositories.
   - `1.basis/ns.yaml` — the `cluster-named-dns` Namespace.
   - `2.access/metallb-pools.yaml` — the whole file (pools, L2Advertisement,
     traefik HelmChartConfig).
   - `3.infra/dns.yaml` — the whole file (pihole + k8s-gateway).
   - `3.infra/certs.yaml` — the `famevans` ClusterIssuer, the `fe-acme-cf-token`
     Secret, and the `famevans-tls-dns` Certificate. Keep the `simplesalt`
     ClusterIssuer + `ss-acme-cf-token`. `famevans-tls-svcs` (mcp-k8s) can be
     dropped or left referencing the now-basis issuer.
   - Drop the `metallb` HelmRelease healthCheck on the `access` Kustomization
     in `order.yaml` (metallb readiness is now gated by `basis-metallb`).
4. Do NOT drop `4.ss/cal.yaml`'s `cal.famevans.win` ingress/cert as part of this
   — cal.com has no CF route and that host is its live URL. Re-publish cal via
   CF (and update its WEBAPP_URL) as separate work if you want it off famevans.
5. Reconcile and confirm the pihole/traefik/fe-apps service IPs and
   `dns.famevans.win` still resolve/serve.

> Objects are moved between Flux Kustomizations, not deleted — comment out the
> base-stack copy in the **same commit** that adds the basis subscription so
> there's no window where both (or neither) own an object.
