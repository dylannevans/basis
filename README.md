# basis

Infrastructure for a healthy LAN, subscribed via Flux GitOps. Extracted from
`simplesalt/base-stack` so LAN-level networking (MetalLB LoadBalancer IPs +
DNS/DHCP) lives in one place.

## Cluster-independence

`network/` has no hardcoded FQDNs and no hardcoded addresses — every hostname
is `${DOMAIN}` and every IP is a named var (see "Addressing"), substituted
at reconcile time via Flux `postBuild.substituteFrom` (wired in
`flux/wiring.example.yaml`) against the `basis-vars` ConfigMap
(`network/vars.yaml`). The committed default is `DOMAIN: famevans.win`; point
basis at a different LAN by overriding that ConfigMap's value in the
consuming repo (e.g. a Kustomize patch layered after `network/`), no edits to
basis itself required.

`cloud/10-access/cf-access-famevans.yaml`'s Cloudflare Zero Trust team domain
(`${CF_ACCESS_TEAM}`) and account ID (`${CF_ACCOUNT_ID}`) are likewise
substituted, via the `basis-cloud` Kustomization in `flux/wiring.example.yaml`
against a dedicated `basis-cloud-vars` ConfigMap (`cloud/vars.yaml`) — kept
separate from `basis-vars` because `cloud/` and `network/` are independent
layers with no data dependency on each other. Committed defaults
(`evans-home.cloudflareaccess.com` / `ed503c805407090970caf579da8193a8`)
match the previous hardcoded values byte-for-byte. The `crossplane.io/
external-name` UUIDs in that file (and the IdP/policy UUIDs it references)
are deliberately left hardcoded — they identify specific LIVE, orphan-
protected crossplane objects, and a misconfigured substitution path for an
identity value risks crossplane re-pointing or failing to adopt the live
resource in a way a plain data field (domain/account ID) does not.

## What's here

| Path | Contents |
|---|---|
| `metallb/` | MetalLB install: `metallb-system` namespace, HelmRepository, HelmRelease. |
| `policy/` | Kyverno `ClusterPolicy` that defaults `loadBalancerClass` onto `LoadBalancer` Services — see "MetalLB loadBalancerClass policy" below. |
| `network/` | MetalLB address pools + L2 (ARP) advertisement, and DNS: pihole (DNS+DHCP, terminates its own TLS) + k8s-gateway (in-cluster upstream). No ingress controller involved — see below. |
| `network/vars.yaml` | `basis-vars` ConfigMap — `DOMAIN` (default `famevans.win`) plus every address `network/` uses (see "Addressing"). Substituted into `network/` at reconcile time. Override to point basis at a different LAN. |
| `cloud/` | Personal (evans-home) Cloudflare Crossplane resources — Zero Trust identity/Access apps and R2 storage. See "Cloudflare Crossplane resources (`cloud/`)" below. |
| `flux/wiring.example.yaml` | Reference GitRepository + Kustomizations to add to **base-stack** to subscribe to this repo. |

## Cloudflare Crossplane resources (`cloud/`)

`cloud/` holds every Crossplane-managed resource for the **personal
(evans-home) Cloudflare account**, accountId
`ed503c805407090970caf579da8193a8`. This is the sole home for personal
Cloudflare Crossplane resources — a separate `dylannevans/cloud-basis` repo
was drafted for this content but is **not used**; everything was consolidated
here instead (`simplesalt/projects#98`, `#182`) because the deploy token used
for this repo has no push access to `cloud-basis`, and a two-repo split
wasn't worth the coordination cost for one personal account.

```
cloud/
├── kustomization.yaml       # aggregates the three subdirectories below
├── 00-provider/             # ProviderConfig + credentials plumbing
│   ├── cf-provider-config.yaml     # Secret/cloudflare-credentials (empty), both ProviderConfig/default objects
│   └── cf-creds-assembler.yaml     # SA/Role/RoleBinding + Secret/cloudflare-api-token (empty) + assembler Job
├── 10-access/               # Zero Trust org + Access apps
│   ├── cf-zero-trust-org.yaml       # TrustOrganization/cloudflare-zero-trust-org
│   ├── cf-zero-trust-apps.yaml      # TrustAccessApplication/cloudflare-app-launcher
│   └── cf-access-famevans.yaml      # TrustAccessApplication/cloudflare-app-warp-login (+ quarantined famevans IdP)
└── 20-storage/              # R2 storage
    └── cf-bucket-ss-testing.yaml    # Bucket/ss-testing
```

**Ordering matters.** Everything in `10-access/` and `20-storage/` sets
`providerConfigRef: {name: default}`, resolving against the two
`ProviderConfig/default` objects (namespaced `.m.upbound.io` for R2, and
cluster-scoped `upjet-cloudflare.upbound.io` for the Zero Trust apps/org)
created in `00-provider/`, which in turn read `Secret/cloudflare-credentials`
(also created in `00-provider/`). Directories are numbered so the dependency
is visible in a file listing — this alone doesn't guarantee apply order
(Flux applies everything in one Kustomization via server-side apply, and
Crossplane's controllers retry until a referenced `ProviderConfig` exists
either way), but it keeps the intent legible.

### Out-of-band secrets

Two `Secret` objects are checked in as **empty placeholders**, both annotated
`kustomize.toolkit.fluxcd.io/ssa: merge` so Flux creates them without ever
overwriting data written some other way. Neither ships credential material —
this repo is public.

- **`Secret/cloudflare-api-token`** (`cloud/00-provider/cf-creds-assembler.yaml`)
  — populate on a fresh cluster with the raw Cloudflare API token:
  ```
  kubectl create secret generic cloudflare-api-token \
    -n crossplane-system --from-literal=api_token=YOUR_TOKEN_HERE
  ```
- **`Secret/cloudflare-credentials`** (`cloud/00-provider/cf-provider-config.yaml`)
  — populated automatically by `Job/assemble-cloudflare-credentials`
  (`cloud/00-provider/cf-creds-assembler.yaml`) once the token above exists.
  Re-trigger after token rotation by deleting the Job; Flux recreates it on
  the next reconcile.

### `deletionPolicy: Orphan`

Every adopted managed resource in `cloud/` sets `deletionPolicy: Orphan`
instead of the Crossplane default (`Delete`): if the MR is ever pruned —
e.g. moved between Flux Kustomizations, or a Kustomization is deleted —
Crossplane detaches from the live Cloudflare object rather than deleting it.
This matters most for `TrustOrganization/cloudflare-zero-trust-org`: it's a
per-account singleton whose create path fails ("account or zone must be
provided"), so it can only ever be adopted, never recreated — a prune without
Orphan would irrecoverably reset live Zero Trust org configuration. The same
guard is applied to `TrustAccessApplication/cloudflare-app-launcher`,
`Bucket/ss-testing`, and the pre-existing `TrustAccessApplication/cloudflare-app-warp-login`.

### Adopted (not created) objects

Three resources carry `crossplane.io/external-name` annotations, adopting
the existing live Cloudflare object instead of taking the create path:

| Object | `crossplane.io/external-name` |
|---|---|
| `TrustOrganization/cloudflare-zero-trust-org` | `ed503c805407090970caf579da8193a8` (accountId) |
| `TrustAccessApplication/cloudflare-app-launcher` | `8b9603f4-9ac0-4e47-9aad-943fcf4b7959` |
| `TrustAccessApplication/cloudflare-app-warp-login` | `cd1dd3f6-6bfb-44cd-91c8-2a751fae13e3` |

`Bucket/ss-testing` carries no `crossplane.io/external-name` annotation —
none was dropped in any copy, the source manifest simply never had one.

### `Bucket/ss-testing`

Despite the SimpleSalt-sounding name, this bucket's `accountId` is
`ed503c805407090970caf579da8193a8` — the personal evans-home account, not
SimpleSalt's `ssint-main`. Classification here is by **owning cloud account,
not by object name**. If it's actually meant to be a SimpleSalt asset, that's
a *Cloudflare-side* migration (recreate it under `ssint-main`), not a repo-
placement decision.

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

## MetalLB loadBalancerClass policy

`metallb/helmrelease.yaml` sets `loadBalancerClass: metallb.universe.tf/metallb`
(the chart's top-level `loadBalancerClass` value, plumbed to the controller as
`--lb-class`; verified against the upstream chart source — the
`loadBalancerClass: ""` key in `charts/metallb/values.yaml` and its consumption
as `--lb-class={{ .Values.loadBalancerClass }}` in
`charts/metallb/templates/controller.yaml` — rather than assumed from the
chart's docs). Once MetalLB has a class, its service controller filters
*every* `LoadBalancer` Service by that exact class — including Services that
never set one, not just ones with a different class (see
`metallb/helmrelease.yaml`'s comment for the upstream source references).
That makes `loadBalancerClass` an all-or-nothing, cluster-wide contract the
moment MetalLB adopts it: any `LoadBalancer` Service anywhere in the cluster
(this repo, base-stack, any other consumer) that doesn't carry the class
silently gets no IP.

`policy/kyverno-metallb-lb-class.yaml` turns that contract into something the
cluster enforces automatically: a Kyverno mutating `ClusterPolicy` that, on
**CREATE only**, injects `spec.loadBalancerClass: metallb.universe.tf/metallb`
onto any `type: LoadBalancer` Service that doesn't already declare a class.

**Escape hatch (two of them, either is sufficient):**
- Set `spec.loadBalancerClass` yourself in the manifest — the policy only
  fills in an *unset* field, it never overrides one that's already there.
- Label the Service `basis/skip-lb-class: "true"` — an explicit opt-out for a
  Service that intentionally wants to stay classless (e.g. targeting klipper
  or some other LB implementation) even though it sets `type: LoadBalancer`.

**Upward dependency:** Kyverno itself is not installed by basis — it comes
from base-stack's `1.basis` layer, so `policy/` is a *second* upward
dependency of the same kind already documented for cert-manager below (basis
must reconcile after base-stack's `1.basis`, and specifically after Kyverno's
CRDs + admission webhook are Ready). See the header comment in
`policy/kyverno-metallb-lb-class.yaml` for the full reasoning.

**CREATE-only, and why that matters — `loadBalancerClass` is immutable.**
Kubernetes rejects any attempt to set `spec.loadBalancerClass` on a Service
that already exists, so the policy can only affect Services at creation time;
it cannot retroactively fix ones that predate it. On k1 today that's the
`dns` and `pihole-web` Services in `network/dns.yaml` — the LAN's DNS/DHCP
Service. Recreating them is a real, if brief, outage, so it's deliberate and
manual, not automated by this change. See "Migration on k1" in the PR that
introduced this policy (dylannevans/basis#12) for the exact sequenced
procedure — merging the policy does **not** by itself migrate those two
Services; that is a separate, manual second step.

**Bootstrap ordering.** The policy must be admitted (webhook Ready) *before*
the first `LoadBalancer` Service is created in a given Kustomization, or that
Service is created classless and a class-scoped MetalLB silently ignores it —
a fresh-cluster-only failure, the same shape of bug that took a rebuild to
surface in #9. `flux/wiring.example.yaml` expresses this with a `basis-policy`
Kustomization that `basis-network` (and any other Service-creating
Kustomization — see "Cross-repo consumers left in base-stack") must
`dependsOn`, gated by a `healthChecks` entry on the `ClusterPolicy`'s own
`Ready` condition.

**Failure policy.** The webhook uses `failurePolicy: Ignore`, not `Fail` —
see the comment in `policy/kyverno-metallb-lb-class.yaml` for why `Fail`
would make Kyverno's availability a hard gate on *all* Service creation
cluster-wide, which is a worse failure mode than "a Service is occasionally
created classless and needs a reconcile/retry."

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
- **Kyverno** controller + CRDs (`ClusterPolicy`) and its admission webhook.
  A second upward dependency of the same kind as cert-manager above: basis's
  `policy/` only owns the `ClusterPolicy` object
  (`policy/kyverno-metallb-lb-class.yaml`), same pattern as basis owning the
  `famevans` `ClusterIssuer` while cert-manager's operator stays in
  base-stack. See "MetalLB loadBalancerClass policy" above.
- Nothing traefik-related — see "Traefik/metallb are decoupled" above.

`network/` no longer hardcodes any address. Every IP — the DNS/DHCP VIP, the
metallb pool ranges, the DHCP scope and router, and the k8s-gateway ClusterIP —
is a `${VAR:=default}` reference resolved from `basis-vars`
(`network/vars.yaml`) the same way `${DOMAIN}` is. See "Addressing" below.

**k8s-gateway's ClusterIP is now pinned, not observed.** dnsmasq can only
forward to a literal IP — it cannot resolve a Kubernetes Service name, since it
*is* the resolver — so `server=/${DOMAIN}/${K8S_GATEWAY_IP}` needs a real
address. Previously the Service didn't set `spec.clusterIP` at all, so that
address was whatever Kubernetes happened to assign, transcribed into the
forwarder by hand. That drifts: on k1 the Service had been recreated and moved
to a different ClusterIP, leaving the forwarder aimed at an address no Service
owned and the whole `${DOMAIN}` zone black-holed (dylannevans/basis#15). The
Service now pins `clusterIP: ${K8S_GATEWAY_IP}`, so the forwarder and the
Service are two references to one asserted value rather than a value and a copy
of it.

The pinned address must fall inside the cluster's Service CIDR, so it is still
cluster-specific — override `K8S_GATEWAY_IP` in `basis-vars` for a cluster with
a different CIDR, exactly as you would `DOMAIN`.

## Addressing

Every address `network/` uses lives in `basis-vars`:

| Var | Default | What it is |
| --- | --- | --- |
| `DNS_VIP` | `192.168.1.2` | LAN VIP pihole serves DNS/DHCP on. Also the node's upstream resolver, which is what makes #9's cold-start deadlock possible — never point pihole's own upstream here. |
| `DNS_VIP6` | `fdaa:3c:a129:8f42::2` | The v6 half of the same VIP, and what DHCPv6 clients are handed as their resolver. |
| `K8S_GATEWAY_IP` | `10.43.230.226` | k8s-gateway's pinned ClusterIP. Must be inside the Service CIDR. |
| `LAN_ROUTER` | `192.168.1.1` | Default gateway handed to DHCP clients. |
| `LAN_NETMASK` | `255.255.255.0` | Netmask handed to DHCP clients. |
| `DHCP_START` / `DHCP_END` | `192.168.1.100` / `.200` | DHCPv4 scope. |
| `APPS_POOL_V4` / `APPS_POOL_V6` | `192.168.1.32-192.168.1.64` / `fdaa:3c7e:a129:8f42::32-…::64` | metallb pool for application Services. |

Each reference carries an **inline default** (`${VAR:=default}`) rather than
relying only on the committed ConfigMap. This is not belt-and-braces — it is
required for correctness. Flux substitutes the whole build *before* applying
it, and `basis-vars` is applied by the same pass that consumes it, so any newly
added key is undefined on its first reconcile. Flux renders an undefined
`${var}` as the **empty string**, so without the default a new key ships an
empty address to dnsmasq or metallb for one interval. Keep the default in sync
with the ConfigMap when changing a value.

Note `DNS_VIP6` (`fdaa:3c:…`) and `APPS_POOL_V6` (`fdaa:3c7e:…`) are on
different /64s. That is preserved as-is, not endorsed — see
dylannevans/basis#19.

## Cross-repo consumers left in base-stack

These base-stack resources reference basis-owned objects by name/annotation
(soft; they just need basis reconciled):

- `3.infra/local-fs.yaml` — the `fs` SMB LoadBalancer consumes the basis
  `fe-apps` pool (stays `Pending` if basis isn't applied). Since
  dylannevans/basis#12, its Kustomization also needs a `dependsOn` on
  `basis-policy` (see "MetalLB loadBalancerClass policy" above) — otherwise,
  on a fresh cluster, `fs` can be created before the Kyverno policy is
  admitted and end up classless, silently unpicked-up by a class-scoped
  MetalLB.
- `3.infra/mcp-k8s.yaml` (`famevans-tls-svcs`) — references the basis
  `famevans` ClusterIssuer; LAN-only/disposable (not in any CF tunnel).

(`4.ss/cal.yaml` previously also referenced this issuer via `cal-tls` — it was
deprovisioned in base-stack independently of this migration, superseded by
`ssint-main-cal`, so it's no longer a consumer.)

## Why a running cluster doesn't hit the #9 cold-start DNS deadlock

[#9](https://github.com/dylannevans/basis/issues/9) describes an unrecoverable deadlock on a
**fresh** single-node cluster: `network/metallb-pools.yaml` pins `dns-pool` to `192.168.1.2`,
which is also the address DHCP hands the node as its own upstream resolver; pihole's Service
takes that IP but its pod can't start until `famevans-tls` exists; that cert is issued by
cert-manager over ACME DNS-01, which needs outbound DNS; and the node (and therefore CoreDNS)
resolves outbound queries via `192.168.1.2` — the endpoint-less Service pihole hasn't managed
to stand up yet. Nothing breaks the loop on its own.

A cluster that's already running (k1) is immune to that specific deadlock, but not because
anything here defends against it — it's immune because **it never takes the cold-start
branch**. k1 already holds the `famevans-tls` secret, issued 26+ days ago while DNS was still
up, so every restart takes the happy path: pihole mounts the existing cert and starts
immediately, no ACME round-trip required, no dependency on the resolver it's about to become.
The deadlock in #9 is reachable only from a cold start — a fresh cluster with no cert yet —
which is exactly the case a USB reprovision creates and a routine reboot doesn't. Fixed for
the cold-start case itself in #10.

Two more properties worth recording, because they're **implicit** — true today, not asserted
anywhere in this repo, so a rebuild doesn't necessarily inherit them:

- **pihole's own upstream is static public DNS** (`FTLCONF_dns_upstreams` in
  `network/dns.yaml`, pinned to `8.8.8.8;8.8.4.4`), so once pihole is actually running it
  never forwards queries back into the cluster. That matters, but it operates **one level
  above** #9: the deadlock is that pihole's pod never starts in the first place, so nothing
  is listening on `192.168.1.2` at all — the *node's* resolver (not pihole's upstream) is
  what's pointed at a dead Service. A correct upstream setting on a pod that isn't running
  protects nothing. Whether the node's resolver should point at the cluster at all — the
  actual fix for #9 — is a separate, still-open design decision; see #9's suggested
  directions.
- **k3s's built-in `servicelb` must stay disabled** (`base-stack`'s `install.sh`/`create.sh`
  pass `--disable servicelb`) wherever this repo's MetalLB lands, or the two controllers
  fight over the same hostPorts. A cluster built via the USB provisioning path
  (`simplesalt/oci`'s `create-usb.sh`) currently renders no `disable:` line, so it comes up
  with `svclb-*` DaemonSets that MetalLB has to compete with — e.g. `svclb-traefik` can grab
  80/443 before `svclb-*-pihole-web` gets a chance to schedule. Tracked as
  `simplesalt/oci#64`; not a basis bug, but it affects any cluster this repo's pools land on.

Also worth knowing so it doesn't get re-filed: traefik's `LoadBalancer` Service sitting at
`<pending>` (`kubectl get svc -n kube-system traefik`) is **expected**, not a regression.
Both `dns-pool` and `fe-apps` in `network/metallb-pools.yaml` set `autoAssign: false`, and
traefik's Service carries no pool annotation requesting either one, so MetalLB never
allocates it an address — by design, per "Traefik/metallb are decoupled" above. k1 has sat in
this state for its entire life with nothing depending on it.

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
