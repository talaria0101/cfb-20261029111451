# Decisions

Each entry records a load-bearing choice, the alternatives considered and the
consequence. Dates are the decision date, not a claim that the choice is final.

## D1. Control plane on Cloudflare, data plane on the node

**Context.** The dashboard and manager must stay on Cloudflare while compute runs
on owned machines. A control plane in the request path would make a Cloudflare
outage a workload outage.

**Decision.** The control plane owns identity, inventory, desired state, policy,
audit and the API. The node owns execution and local state. The control plane is
not in the request path of a workload except for optional ingress through the
relay or a tunnel.

**Alternatives.** Everything on the machines (violates the requirement, loses
Access and global ingress). Everything on Cloudflare (that is the system being
replaced). Control plane on a machine behind a tunnel (a machine failure takes
down the manager, and bootstrap becomes circular).

**Consequence.** The control plane is a single point of failure for management,
not execution. The broker must reconcile from local state and the protocol must
be level-triggered.

## D2. One broker binary, runtime children

**Context.** A tunnel client, host agent, reverse proxy and metrics process per
machine is four binaries, four configs and four upgrade paths.

**Decision.** `cfbd` is one static Rust binary. It contains the transport, router,
supervisor, workerd config generation, S3 gateway, state and metrics. It
supervises workerd, rootless Podman, Incus, Firecracker and CI runners as child
processes because they are separate security and failure domains.

**Alternatives.** Keep separate binaries (more surface, more drift). Embed V8 and
a container runtime (larger binary, larger attack surface, duplicate maintenance).
Use a sidecar per feature (same problem as separate binaries).

**Consequence.** `cfbd` becomes the only CFB process and the only upgrade unit.
A crash of a child is contained; a crash of `cfbd` stops control and ingress but
not already running children unless the supervisor kills them.

## D3. WebSocket relay is the default transport

**Context.** Constrained nodes have no inbound TCP, no root, no UDP 7844 and no
TUN. `cloudflared` needs a tunnel token, a process and a network path that some
hosts do not allow.

**Decision.** `cfbd` dials `wss://relay.<domain>/agent` on 443. A Worker upgrades
to a per-machine Durable Object, which opens streams to the node. Native
Cloudflare Tunnel over QUIC or HTTP/2 is optional and requires the missing cfrs
registration and serve loop. Workers VPC is optional and beta.

**Alternatives.** Native tunnel only (blocked on UDP and on unfinished cfrs).
VPC only (beta, account-wide binding, not portable). `cloudflared` per host
(extra binary, root or token management, no single control path).

**Consequence.** The relay DO is in the data path and bills per message. Bulk
traffic should use S3 or a native tunnel. The relay is the portable path and the
one that works on a 512 MB node with no root.

## D4. No inbound listener

**Context.** A public port needs a firewall rule, TLS, authentication and
patching. A constrained host may have no public address at all.

**Decision.** The broker listens on a Unix socket only, mode 0660, with
`SO_PEERCRED`. `lite` and `strict` have no TCP listener. `standard` may bind a
child runtime to loopback when the runtime cannot use a Unix socket. Nothing
binds `0.0.0.0` or `::`.

**Alternatives.** A loopback-only TCP API for the broker (simpler clients, weaker
peer identity). An authenticated TCP API on a private network (still a listener
and a firewall rule).

**Consequence.** Local clients need Unix-socket support. Workerd receives an
inherited socket file descriptor. The broker never exposes an admin port.

## D5. Profiles, with `lite` and `strict`

**Context.** A 512 MB VPS with no root cannot run Podman, Incus, KVM or a
privileged helper. Pretending otherwise produces a design that only works on a
large host.

**Decision.** Four profiles: `standard` (workerd plus rootless Podman), `lite`
(workerd and static files, no root, no containers), `strict` (also no loopback
TCP), and `privileged` (optional helper for Incus, KVM and Firecracker).

**Alternatives.** One profile with optional features (users cannot tell what is
supported). A container-only design (excludes the constrained case). A root-only
design (excludes the common VPS).

**Consequence.** `lite` and `strict` are trusted-code only. They cannot isolate
hostile tenants and must say so at enrollment. Capabilities are reported, not
assumed.

## D6. Portability through existing contracts and a generated shim

**Context.** "Run the same code on Cloudflare or a node" fails if the node
invents its own API or if workerd alone is assumed to provide R2, D1, KV and
Queues.

**Decision.** The portable artifact is a Workers bundle and `wrangler.jsonc`, an
OCI image, an S3 request or a CI workflow. On a node, a generated shim maps R2,
D1, KV and Queues bindings to local services. Native workerd provides Workers,
DO, services, sockets and disk. Compatibility is claimed at source, configuration
and protocol layers; semantics are tested.

**Alternatives.** A CFB-specific application API (breaks portability). A fork of
workerd with built-in bindings (large maintenance and divergence). Claim
zero-change parity (false for unsupported bindings).

**Consequence.** The shim is the highest-risk component and needs a conformance
suite. Unsupported bindings fail at deploy or first use with a named error.

## D7. workerd is the runtime; open-compute is optional

**Context.** workerd is Cloudflare's runtime, so it is the least divergent
choice. It has no deploy API, tenant model, dashboard or storage bindings.

**Decision.** Use workerd directly with a generated config and shim for the
supported subset. Run open-compute `ocd` only when the application needs its
broader API surface. Raw workerd is never the isolation boundary for hostile
code.

**Alternatives.** open-compute as the only platform (single-node, large API
surface, more bugs). celld for everything (pre-1.0, one app per fleet). WDL
(too young). Rivet (not Workers-compatible).

**Consequence.** Two runtime modes to test. open-compute adds a second binary and
is not part of `lite`.

## D8. S3 is the storage contract

**Context.** Tenants expect an S3 API, presigned URLs and a client that can
switch endpoints without code changes.

**Decision.** S3 is the contract. The node ships a narrow gateway over local
filesystem for `lite`; multi-node sites use SeaweedFS or RustFS. R2 is the offsite
copy and the control-plane store, not the primary for tenant reads.

**Alternatives.** A custom object API (breaks SDK compatibility). MinIO (archived,
AGPL). Garage (no versioning, SSE or ACLs). Ceph (too heavy for a small site).
R2 as primary (that is the cost being avoided).

**Consequence.** The local gateway must be correct for the narrow subset and
delegate the rest. Presigned URL verification is part of the broker, not the
shim.

## D9. OCI images, rootless Podman by default

**Context.** The same container image should run on Cloudflare Containers and on
a node.

**Decision.** Use OCI images and rootless Podman as the default `standard`
runtime. Use Docker only if the operator already runs it. Use Incus for VMs and
for `privileged` containers.

**Alternatives.** A custom image format (breaks portability). Docker root daemon
by default (large root surface). Containerd directly (more moving parts for the
same result).

**Consequence.** Containers share the host kernel and are for trusted code only.
Untrusted code needs a microVM.

## D10. Firecracker for untrusted code

**Context.** A container is not a boundary for hostile tenants.

**Decision.** Use Firecracker with the jailer, vsock and snapshot for sandboxes
and untrusted tenants. Use Incus VMs for semi-trusted workloads. gVisor is an
option where KVM is unavailable.

**Alternatives.** Containers with seccomp (not a boundary). Kata (Kubernetes-shaped
and heavier). Incus VMs for all sandboxes (slower, larger device model).

**Consequence.** Firecracker needs KVM. Cheap VPS instances without nested
virtualization cannot run it and must downgrade with a documented limit.

## D11. Generations and leases, no global scheduler

**Context.** Commands can be duplicated by Queues, retried, or delivered after a
partition. A stale node must not overwrite a newer one. A global scheduler needs
consensus and fights the local runtime.

**Decision.** Every workload has a monotonically increasing `generation`. Every
machine has a monotonic `lease_epoch`. Apply only if the generation is newer and
the epoch is current. The loop is level-triggered. The control plane computes a
candidate set and asks the target's coordinator DO for a lease; it does not
continuously rebalance.

**Alternatives.** Exactly-once delivery (not achievable). A distributed lock
service (another dependency). Last-writer-wins on timestamps (clock skew).
Central scheduler (complexity and failure modes).

**Consequence.** Lower utilization than a global optimizer. Rebalancing is
explicit. Duplicate commands are harmless.

## D12. D1 for inventory, one Durable Object per machine

**Context.** The control plane needs relational metadata and a low-latency
single-writer coordinator per machine. One store for both creates a hot global
object or a database used as a lock service.

**Decision.** D1 owns inventory, desired state, audit and usage. One coordinator
DO per machine owns the lease, fencing epoch, command queue and live subscribers.
One relay DO per machine owns the socket. R2 owns artifacts. Node SQLite owns
runtime state.

**Alternatives.** One global DO (hot, global blast radius). D1 as a lease store
(no suitable compare-and-swap). KV for inventory (no queries). Postgres on a node
(control plane would depend on the data plane).

**Consequence.** A DO is a per-machine failure domain. At the soft 1,000 req/s
limit, a busy machine needs batched heartbeats or sharding. D1's 10 GB limit
requires planned sharding.

## D13. Access for humans, own RBAC in D1

**Context.** Access gives authentication and coarse policy. It does not model
tenants, roles and resource scopes.

**Decision.** Use Access for human login and policy. Validate the JWT at the
Worker with issuer and audience checks. Implement fine-grained RBAC and tenant
scoping in D1. Use scoped API tokens for automation.

**Alternatives.** Access groups as the only authorization (no tenant model, 500
policy limit). A separate IdP in the SPA (more moving parts). Per-tenant Access
apps (500-app limit).

**Consequence.** Two authorization layers to keep consistent. The Worker is the
enforcement point; D1 is the source of truth for roles.

## D14. CI is optional and supervised, not built

**Context.** Heavy CI is a motivating workload. Workers Builds has a 20-minute
job timeout, six concurrent builds and a 4 vCPU / 8 GB build machine.

**Decision.** Supervise a GitHub Actions runner, Fireactions pool or Woodpecker
agent in the `standard` or `privileged` profile. The broker does not parse
pipelines; it starts the runner, injects a short-lived token, streams logs and
uploads artifacts.

**Alternatives.** Build a CI engine (years of work). Jenkins (plugin sprawl).
Tekton or Argo (Kubernetes-heavy). GitLab Runner (only for GitLab).

**Consequence.** Two integrations with different secret and webhook models. CI
is not available in `lite`.

## D15. No SSH control path

**Context.** SSH is the default operational path for self-hosted systems and the
main reason they cannot be managed from a browser.

**Decision.** The control path is the broker. Shell access is an audited exec
stream through the relay, recorded and time-limited. There is no standing SSH key
on the machine. Break-glass uses a one-time exec token or an Access certificate on
a separate path, and it is logged.

**Alternatives.** SSH with a bastion (unauditable, standing credential). A web
terminal running root (the thing being avoided). No break-glass (a broken broker
needs a way in).

**Consequence.** The broker and relay must be robust, and break-glass must be
tested quarterly.

## D16. No multi-region control plane in v1

**Context.** A global active-active control plane needs replicated D1, DO
placement and a conflict model.

**Decision.** One control-plane deployment with D1 read replication if needed.
The data plane can span sites. The control plane is not active-active across
regions in v1.

**Alternatives.** Multi-region from day one (complexity). One control plane per
site (N dashboards and N identity systems).

**Consequence.** A regional Cloudflare incident can make the dashboard
unavailable. Workloads keep running. The management RTO is documented, not zero.

## D17. Cost is parametric

**Context.** Hardware, power and bandwidth prices vary by country and contract. A
single "cheaper than Cloudflare" number would be invented.

**Decision.** Use Cloudflare list prices with a read date, and a parametric model
for the node: hardware amortization, power, bandwidth, media, replacement and
operator time. Publish a worked example with explicit assumptions and a
break-even formula.

**Alternatives.** A single TCO number (wrong for almost every reader). Ignore
cost (cost is the reason for the project).

**Consequence.** The reader supplies their own inputs. The model is honest and
less quotable.

## D18. 0BSD design repository, separate from an implementation

**Context.** The design and an implementation have different licensing, review
and release needs.

**Decision.** This repository is 0BSD and holds the design, evidence and decision
record. An implementation repository can choose Apache-2.0 for a patent grant.

**Alternatives.** One repository for both (design tied to one implementation).
Apache-2.0 here (the user specified 0BSD).

**Consequence.** Anyone can reuse the design, including a competing
implementation.

## D19. Sign every control request

**Context.** TLS terminates at Cloudflare on both the relay and VPC paths, so the
Worker never sees a client certificate. mTLS cannot authenticate the node end to
end.

**Decision.** Authenticate both directions with Ed25519 request signatures. The
node pins the control-plane key at enrollment; the control plane pins the machine
key. The signed tuple is method, path, body hash, timestamp, nonce, machine id and
lease epoch. A self-hosted relay may add mTLS, but it is not part of the default.

**Alternatives.** Client certificates (not visible to the Worker). A bearer token
(replayable, does not authenticate the server). mTLS at a self-hosted relay
(moves the control path off Cloudflare).

**Consequence.** Key rotation, a replay cache and signature verification per
request. The 24-hour credential is an identity and revocation artefact, not a TLS
certificate.

## D20. Fail closed on unsupported bindings

**Context.** A silent fallback produces a deployment that looks successful and
fails at the first request.

**Decision.** Reject an unsupported binding at upload with the binding path and
reason. Throw `UnsupportedBindingError` at first call if a method is missing.
Mark a primitive unsupported when conformance fails.

**Alternatives.** Warn and continue (hidden data loss). Ignore the binding (crash
later). Emulate everything (unmaintainable and dishonest).

**Consequence.** Some applications cannot move without changes. The supported
subset is explicit, and the failure is named.
