# basis

Infrastructure for a healthy LAN, subscribed via Flux GitOps. Extracted from
`simplesalt/base-stack` so LAN-level networking (MetalLB LoadBalancer IPs +
DNS/DHCP) lives in one place.

## Cluster-independence

`network/` has no hardcoded FQDNs — every hostname is `${DOMAIN}`, substituted
at reconcile time via Flux `postBuild.substituteFrom` (wired in
`flux/wiring.example.yaml`) against the `basis-vars` ConfigMap
(`network/vars.yaml`). The committed default is `DOMAIN: famevans.win`; point
basis at a different LAN by overriding that ConfigMap's value in the
consuming repo (e.g. a Kustomize patch layered after `network/`), no edits to
basis itself required.

`cloud/cf-access-famevans.yaml` still hardcodes the Cloudflare Zero Trust team
domain (`evans-home.cloudflareaccess.com`) — it's not wired to a Flux
Kustomization yet (no `basis-cloud` stanza in `wiring.example.yaml`), so it
wasn't templated here to avoid shipping an unsubstituted `${VAR}` into a live,
orphan-protected crossplane resource.

## What's here

| Path | Contents |
|---|---|
| `metallb/` | MetalLB install: `metallb-system` namespace, HelmRepository, HelmRelease. |
| `network/` | MetalLB address pools + L2 (ARP) advertisement, and DNS: pihole (DNS+DHCP, terminates its own TLS) + k8s-gateway (in-cluster upstream). No ingress controller involved — see below. |
| `network/vars.yaml` | `basis-vars` ConfigMap — the single `DOMAIN` value (default `famevans.win`) substituted into every `${DOMAIN}` in `network/`. Override it to point basis at a different LAN. |
| `flux/wiring.example.yaml` | Reference GitRepository + Kustomizations to add to **base-stack** to subscribe to this repo. |

## MetalLB vs kube-vip — they are NOT the same VIP

The multi-server k3s image (`simplesalt/oci` `cluster/`) runs a **kube-vip**
DaemonSet from the k3s static-manifests dir. That provides the **control-plane
VIP** (a floating apiserver address) via ARP/L2 across the HA server nodes.

**MetalLB is unrelated to that.** It hands out **`type: LoadBalancer` service
IPs** — here the dedicated, dual-stack addresses that LAN clients depend on:

| Pool | IPv4 | Consumer |
|---|---|---|
| `dns-pool` | `192.168.1.2` | pihole DNS/DHCP + pihole web UI (shared IP, different ports) |
| `fe-apps` | `192.168.1.32-64` | front-end app LoadBalancers |

kube-vip in this configuration does **not** provide these, so MetalLB stays.
(Moving to the new image is additive: you gain control-plane HA and keep
MetalLB unchanged.) MetalLB here already runs in **L2/ARP mode**
(`L2Advertisement`) — the "BGP" note in `network/metallb-pools.yaml` is a
long-term aspiration, not the running config.

## Traefik/metallb are decoupled

There is no `traefik` IPAddressPool and no `HelmChartConfig` pinning k3s's
built-in traefik to a metallb VIP. Traefik doesn't need one: cf-tunnel (in
base-stack) reaches it via its ClusterIP Service DNS name, not a LAN IP. The
pihole web UI — previously the only thing routed through traefik on the LAN —
now gets its own dedicated metallb LoadBalancer IP (`serviceWeb` in
`network/dns.yaml`) and terminates TLS itself in-pod (lighttpd, via the
mounted `famevans-tls` secret), so it doesn't need an ingress controller
either. If something in the consuming cluster needs its ingress controller
LAN-reachable, give it its own Service/IP off `fe-apps` — don't reintroduce a
basis -> traefik coupling here.

## Dependencies that remain in base-stack

basis assumes these already exist on the cluster (provided by base-stack). All
are *soft* runtime references except cert-manager/metallb CRDs, which are the
only *hard* (apply-order) requirement — so basis must reconcile after
base-stack's `1.basis` layer.

- **Flux controllers** (source-controller, helm-controller, kustomize-controller).
- **cert-manager** controller + CRDs. The `famevans` ClusterIssuer itself now
  lives in basis (`network/famevans-issuer.yaml`); only the operator stays in
  base-stack. The `simplesalt` ClusterIssuer (public domain) also stays.
- Nothing traefik-related — see "Traefik/metallb are decoupled" above.

One fragile hardcode remains: `network/dns.yaml` pins k8s-gateway's Service
`clusterIP` (`10.43.171.237`) and references that same literal in pihole's
dnsmasq forwarder config (`server=/${DOMAIN}/10.43.171.237`). dnsmasq can only
forward to a literal IP — it can't resolve a Kubernetes Service DNS name
itself, since it *is* the resolver — so this is the one value in `network/`
that's tied to a specific cluster's Service CIDR and isn't behind `${DOMAIN}`.
Update both occurrences together if you fork this for another cluster.

## Cross-repo consumers left in base-stack

These base-stack resources reference basis-owned objects by name/annotation
(soft; they just need basis reconciled):

- `3.infra/local-fs.yaml` — the `fs` SMB LoadBalancer consumes the basis
  `fe-apps` pool (stays `Pending` if basis isn't applied).
- `3.infra/mcp-k8s.yaml` (`famevans-tls-svcs`) — references the basis
  `famevans` ClusterIssuer; LAN-only/disposable (not in any CF tunnel).

(`4.ss/cal.yaml` previously also referenced this issuer via `cal-tls` — it was
deprovisioned in base-stack independently of this migration, superseded by
`ssint-main-cal`, so it's no longer a consumer.)

## Cutover (test plan)

1. Push this repo's `main`.
2. In **base-stack**, add `flux/wiring.example.yaml` (a `basis` GitRepository +
   `basis-metallb` / `basis-network` Kustomizations).
3. Remove the now-duplicated resources from base-stack so Flux doesn't fight
   for ownership of the same objects:
   - `1.basis/controllers.yaml` — the `metallb-system` Namespace + `metallb`
     HelmRelease.
   - `1.basis/helmrepos.yaml` — the `metallb`, `pihole`, `k8s-gateway`
     HelmRepositories.
   - `1.basis/ns.yaml` — the `cluster-named-dns` Namespace.
   - `2.access/metallb-pools.yaml` — the whole file (pools, L2Advertisement).
     Note the `traefik` pool + `HelmChartConfig` this file used to carry are
     gone entirely, not moved — traefik no longer gets a metallb VIP at all.
   - `3.infra/dns.yaml` — the whole file (pihole + k8s-gateway + the old
     traefik Ingress).
   - `3.infra/certs.yaml` — the `famevans` ClusterIssuer, the `fe-acme-cf-token`
     Secret, and the `famevans-tls-dns` Certificate. Keep the `simplesalt`
     ClusterIssuer + `ss-acme-cf-token`. `famevans-tls-svcs` (mcp-k8s) stays,
     referencing the now-basis issuer.
   - Drop the `metallb` HelmRelease healthCheck on the `access` Kustomization
     in `order.yaml` (metallb readiness is now gated by `basis-metallb`).
4. Reconcile and confirm: the pihole service (DNS/DHCP + web UI, both off
   `dns-pool`) and `fe-apps` consumers come up; `dns.${DOMAIN}` (default
   `dns.famevans.win`) serves HTTPS directly from pihole (no ingress hop);
   traefik's Service is ClusterIP-only and cf-tunnel still reaches it fine.

> Objects are moved between Flux Kustomizations, not deleted — comment out the
> base-stack copy in the **same commit** that adds the basis subscription so
> there's no window where both (or neither) own an object.
