# Decisions

This page records the load-bearing choices, the alternatives considered and why
they were rejected. Each entry has the same shape: context, decision,
alternatives, consequences. Dates are the decision date, not a claim that the
choice is final.

## D1. Two planes, with Cloudflare as the control plane

**Date:** 2026-10-29

**Context.** The requirement is that the dashboard and manager stay on
Cloudflare while compute runs on owned machines. A naive design would put the
control plane in the request path of every workload, which makes a Cloudflare
outage a workload outage.

**Decision.** Split the system into a Cloudflare-hosted control plane and a
private data plane. The control plane owns identity, inventory, desired state,
policy, audit and the API. The data plane owns execution and local state. The
control plane is not in the request path of a tenant workload, except for
optional public ingress through Tunnel.

**Alternatives.**

- Put everything on the machines, including the dashboard. Rejected: violates
  the requirement and loses Cloudflare's identity and global ingress.
- Put everything on Cloudflare. Rejected: that is the system being replaced.
- Put the control plane on the machines and expose it through a tunnel.
  Rejected: a machine failure takes down the manager, and bootstrapping becomes
  circular.

**Consequences.** The control plane is a single point of failure for management,
not for execution. That is the right trade. It also means the agent must be able
to reconcile from a local cache, and the desired state protocol must be
level-triggered.

## D2. Outbound-only connectivity with one tunnel per machine

**Date:** 2026-10-29

**Context.** Machines sit behind NAT or a firewall. Opening inbound ports is not
acceptable. The control plane must reach an agent, and optionally a public
workload, without a VPN appliance on every site.

**Decision.** One Cloudflare Tunnel per machine, remotely managed, with
`cloudflared` as the only ingress path. The control plane reaches the agent
through Workers VPC with the account-wide `cf1:network` binding. That binding
reaches every tunnel and Mesh route in the account, so a Gateway policy denies
public Internet destinations for Workers VPC traffic and all VPC traffic is
logged. Verify that the policy can be scoped to this binding.
A relay Durable Object is a fallback transport. No inbound ports, no SSH control
path.

**Alternatives.**

- One tunnel per site. Rejected: a compromised machine can reach every route in
  the site, and revoking one machine means disrupting the site.
- A self-hosted WireGuard mesh (Headscale, NetBird). Rejected as the default
  because it adds a second control plane and a second identity system, and it
  does not integrate with Access. Kept as a fallback for sites where Cloudflare
  is not allowed.
- frp or rathole with a public relay. Rejected as the default: you operate the
  relay, its TLS, its auth and its DDoS posture. Kept as a fallback for a
  Cloudflare-free site.
- SSH over a bastion. Rejected: a remote shell is not a control protocol, it is
  unauditable and hard to make idempotent.

**Consequences.** The fleet is capped by the tunnel limit (1,000 free per
account) unless Tunnels for Platforms is used at $4.50 per additional tunnel per
month. Every machine needs a tunnel token and a machine key. The account-wide
binding means a control-plane compromise can reach every machine route, and a
misconfigured Gateway policy can reach public destinations. Constrain it and
log it. Losing Cloudflare loses the control path, but not the workloads.

## D3. Workers VPC first, relay second

**Date:** 2026-10-29

**Context.** Workers VPC was in beta at the time of writing. Depending on a beta
feature for the only control path is fragile.

**Decision.** Abstract the control transport behind a `PrivateTransport`
interface. Use Workers VPC `fetch()`/`connect()` as the default. Implement a
relay Durable Object that accepts an agent-initiated WebSocket as the fallback.
Both speak the same request/response and stream protocol.

**Alternatives.**

- Only the relay. Rejected: it adds a hop and a Durable Object in the path for
  every command, and it is more code to operate.
- Only VPC. Rejected: a beta change or a regional issue would leave no control
  path.
- Public hostname plus Access service token for the agent. Rejected as the
  default because Access service tokens are capped at 50 per account, and a
  machine token in an HTTP header is weaker than a signed request over the
  tunnel.

**Consequences.** Two transports to test. The relay becomes mandatory if
WebSocket upgrades do not survive VPC `fetch()`, which is an open question.

## D4. Incus as the default site runtime

**Date:** 2026-10-29

**Context.** The data plane needs containers and VMs, a local API, clustering,
tenant separation and a licence that permits a product.

**Decision.** Use Incus as the default runtime for containers and VMs. Use
containerd only when the site already standardizes on OCI images and does not
need VMs. Use Kubernetes only as an optional escape hatch for container-heavy
sites.

**Alternatives.**

- Kubernetes as the default. Rejected: it is a distributed system with a large
  operational surface, and it does not give VMs without an extra project. It also
  encourages a control plane that schedules, which this design rejects.
- Nomad. Rejected: BUSL-1.1, which is not suitable for a product that might
  compete with HashiCorp's offering.
- Proxmox. Rejected as the default: AGPL core, an API designed for its own UI,
  and no first-class container/VM object model as clean as Incus.
- Docker Engine plus Compose. Rejected: no VMs, weak multi-tenant story, and a
  daemon with a large root surface.
- Firecracker for everything. Rejected: too low-level for long-running services
  and CI; Firecracker is the sandbox primitive, not the site runtime.

**Consequences.** Incus containers share the host kernel. They are only for
trusted code. Untrusted code goes to Incus VMs or Firecracker. Incus clustering
is per site, not global.

## D5. open-compute as the default Workers platform

**Date:** 2026-10-29

**Context.** The user wants to run Workers on owned machines and reach them
through a tunnel. Raw workerd has no platform layer: no deploy API, no tenant
model, no dashboard, no secret store, no storage bindings.

**Decision.** Use open-compute (`ocd`) as the default per-machine Workers
platform. Use raw workerd plus a small supervisor only where open-compute lacks a
binding. Run workerd inside a VM or a dedicated machine, never as the isolation
boundary for untrusted code.

**Alternatives.**

- Build a platform on raw workerd. Rejected: it is a large, undifferentiated
  project (routing, KV, D1, R2, DO, Queues, Workflows, secrets, deploys).
- celld for everything. Rejected as the default: pre-1.0, one application per fleet,
  no account service, no TLS.
- WDL. Rejected as a dependency: 55 stars, created 2026-06-24, too young. Kept as
  a reference.
- Rivet. Rejected for Workers workloads: not Workers-compatible. Kept for actor
  workloads.
- Cloudflare's own `workerd` + `wrangler` only. Rejected: no runtime platform.

**Consequences.** open-compute is young and single-node. Pin versions, run it
behind the agent, and treat a machine as a failure domain. Keep a raw workerd
mode as a fallback for compatibility gaps.

## D6. celld for distributed Durable Objects, with eyes open

**Date:** 2026-10-29

**Context.** Durable Objects are the hardest piece to replace. They need a single
writer per object, durable storage, alarms, hibernation and global routing.

**Decision.** Use celld for distributed DO when the workload needs them. Accept
its constraints: one application per fleet, pre-1.0, no TLS, Linux only. Use
open-compute DO for single-machine workloads and Rivet for actor-shaped
workloads.

**Alternatives.**

- Build a DO runtime. Rejected: a multi-year project with subtle correctness
  requirements (fencing, storage transactions, alarms, hibernation, migration).
- workerd `ClusterConfig`. Rejected as the default: experimental, requires
  `--experimental`, and the bucket copy semantics replace local state and refuse
  prior local-disk DOs.
- A single open-compute instance per tenant. Rejected at fleet scale: one process
  per tenant is expensive and hard to route.
- celld-operator on Kubernetes. Kept as an option for a k8s site, not the
  default.

**Consequences.** Multi-tenant DO needs one celld fleet per tenant or a wait for
celld 1.0. The control plane must model a DO fleet as a runtime with its own
lifecycle.

## D7. Firecracker as the sandbox primitive

**Date:** 2026-10-29

**Context.** Untrusted code needs a hardware virtualization boundary. Containers
share the kernel and are not sufficient.

**Decision.** Use Firecracker for sandboxes, with the jailer, vsock and snapshot
and restore. Use Incus VMs for semi-trusted workloads where a full machine API is
more useful than a minimal device model. Use bhatti or hyper only as an
orchestration layer if they mature.

**Alternatives.**

- gVisor. Rejected as the default: a user-space kernel is a strong boundary but
  has compatibility gaps and a different performance profile. Keep it as an
  option for a site that cannot use KVM.
- Kata Containers. Rejected as the default: it is a container runtime with a VM
  boundary, but it adds a Kubernetes-shaped stack and a larger dependency set.
- Incus VMs for everything. Rejected for sandboxes: slower start and a larger
  device model than necessary.
- Containers with seccomp. Rejected: not a boundary for hostile code.

**Consequences.** Firecracker needs KVM. Cheap VPS instances without nested
virtualization cannot run it. Sites without KVM use Incus VMs or gVisor, with a
documented isolation downgrade.

## D8. SeaweedFS as the default S3

**Date:** 2026-10-29

**Context.** Tenants expect an S3 API with IAM, presigned URLs, versioning and
lifecycle. The storage must run on one machine and scale to a rack.

**Decision.** Use SeaweedFS as the default: `weed mini` for a single node, a
cluster for a rack, erasure coding for warm data, replication for hot data, and
cross-cluster replication to a second site or R2. Pilot RustFS on non-critical
buckets. Use R2 only for offsite copies and control-plane artifacts.

**Alternatives.**

- MinIO. Rejected: archived upstream and AGPL-3.0.
- Garage. Rejected as a general S3: no versioning, SSE or object ACLs.
- Ceph. Rejected as the default: too heavy for a small site.
- RustFS as the default. Rejected for now: younger, MinIO on-disk compatibility
  is preview, and the operational story is less proven.
- R2 as the primary. Rejected: that is the cost being avoided, and it puts tenant
  data in the control plane's failure domain.

**Consequences.** SeaweedFS has more moving parts than a single binary and needs
tuning. The agent must manage its lifecycle, health and replication lag. RustFS
is a migration path, not a second default.

## D9. Woodpecker and Fireactions for CI

**Date:** 2026-10-29

**Context.** Heavy CI is the motivating workload. Workers CI has a 20-minute
timeout, 6 concurrent builds and 6,000 minutes per month on the paid plan. That
does not fit a heavy build.

**Decision.** Use Fireactions for GitHub Actions on Firecracker. Use Woodpecker
for Gitea or Forgejo. The control plane manages pools, secrets, quotas,
observability and artifacts; it does not run jobs or parse pipelines.

**Alternatives.**

- Build a CI engine. Rejected: years of work, and the pipeline language and plugin
  ecosystem are the value.
- Jenkins. Rejected as the default: plugin sprawl, a large security surface, and a
  UI from another era. It works if the site already runs it.
- Tekton or Argo Workflows. Rejected as the default: Kubernetes-native and heavy
  for a small fleet. Good if the site is already Kubernetes.
- GitLab Runner. Rejected unless the site uses GitLab; the public mirror is not
  the canonical source.
- Actions Runner Controller. Kept for Kubernetes sites.
- Dagger. Kept as a build-step component, not a scheduler.

**Consequences.** Two CI integrations, each with its own webhook and secret
model. The control plane must normalize job status and logs without pretending
the pipeline languages are the same.

## D10. D1 for inventory, a DO per machine for leases

**Date:** 2026-10-29

**Context.** The control plane needs durable relational metadata and a
low-latency, single-writer coordinator per machine. Using one for both leads to
either a hot global object or a relational database used as a lock service.

**Decision.** D1 owns inventory, desired state, audit and usage. One Durable
Object per machine owns the lease, fencing epoch, command queue and live
subscribers. R2 owns artifacts. Local SQLite owns runtime state. No fact has two
primaries.

**Alternatives.**

- One global DO for the fleet. Rejected: a single hot object, a global bottleneck
  and a blast radius that includes every machine.
- D1 as the lease store. Rejected: D1 has no interactive transactions and no
  compare-and-swap across requests that is suitable for fencing.
- KV for inventory. Rejected: no queries, no transactions, eventual consistency.
- Postgres on a machine. Rejected: the control plane must stay on Cloudflare, and
  a database on a machine makes the manager depend on the data plane.

**Consequences.** The DO is the hot path for commands. At 1,000 req/s soft per
object, a busy machine may need sharding or batched heartbeats. D1 has a 10 GB
limit that cannot be raised, so sharding by account is a planned migration.

## D11. No global scheduler; placement is a control-plane decision, execution is local

**Date:** 2026-10-29

**Context.** A global scheduler is the classic distributed-systems trap: it
needs consensus, it becomes a single point of failure, and it fights the local
runtime.

**Decision.** The control plane computes a deterministic candidate set from
labels, capacity and locality. It asks the target machine's DO for a lease. The
agent and the local runtime make the workload real. Incus may place within its
own cluster. There is no continuous global rebalancer.

**Alternatives.**

- A central scheduler with a global state machine. Rejected: complexity and
  failure modes outweigh the utilization gain at this scale.
- Let each machine pull work from a queue. Rejected as the only model: it loses
  placement control and makes anti-affinity and volume locality hard.
- Kubernetes as the scheduler. Rejected as the default: it requires a cluster per
  site and a control plane that watches it.

**Consequences.** Utilization is lower than a global optimizer. Rebalancing is
explicit. The design is easier to reason about and debug.

## D12. Desired state with generations and leases

**Date:** 2026-10-29

**Context.** Commands can be duplicated by Queues (at-least-once), retried by the
API, or delivered after a network partition. A stale agent must not overwrite a
new one.

**Decision.** Every workload has a monotonically increasing `generation`. Every
machine has a monotonically increasing `lease_epoch`. The agent applies a spec
only if the generation is newer and the epoch is current. Applying the same
generation twice is a no-op. The loop is level-triggered: it compares desired and
observed digests rather than relying on every command arriving.

**Alternatives.**

- Exactly-once command delivery. Rejected: not achievable across a queue and a
  network partition.
- A distributed lock service. Rejected: another control plane dependency.
- Last-writer-wins on timestamps. Rejected: clock skew and retries make it
  unsafe.

**Consequences.** The agent needs durable local state and an operation log. The
control plane must tolerate duplicate commands and stale reports.

## D13. Build the agent and the control plane, reuse everything below

**Date:** 2026-10-29

**Context.** amika publishes a host daemon and SDKs but keeps the control plane,
UI and relay closed. Coolify, Dokploy and Portainer are SSH- or Docker-centric.
No public project provides a Cloudflare-native multi-tenant control plane for
Incus, Firecracker and workerd.

**Decision.** Build the control plane, the agent, the data model, the lease
protocol, the policy engine and the dashboard. Reuse Incus, Firecracker,
open-compute, celld, SeaweedFS, Woodpecker, Fireactions and cloudflared.

**Alternatives.**

- Fork amika and wait for the control plane to open. Rejected: no date, no
  license grant for the missing pieces, and the gateway is single-player.
- Fork Coolify. Rejected: SSH orchestration and a Docker-only model.
- Fork Portainer. Rejected: Docker-centric, and its edge model is not a compute
  control plane.
- Use OpenStack or CloudStack. Rejected: too heavy and the wrong abstraction.

**Consequences.** The build is real: agent, API, schema, leases, UI, audit. That
is the product. Everything below is a dependency with an exit plan.

## D14. Access for humans, own RBAC in D1

**Date:** 2026-10-29

**Context.** Cloudflare Access gives strong authentication and a policy layer,
but it does not model tenants, roles and resource scopes inside the application.

**Decision.** Use Access for human authentication and coarse policy. Validate the
Access JWT at the Worker with issuer and audience checks. Implement fine-grained
RBAC and tenant scoping in D1. Use scoped API tokens for automation. Do not use
Access service tokens as a per-tenant credential, because the account limit is
50.

**Alternatives.**

- Access groups as the only authorization. Rejected: no tenant model, and the
  limit of 500 reusable policies is too coarse.
- A third-party IdP directly in the SPA. Rejected: more moving parts, and Access
  already gives a tested OIDC flow and an audit trail.
- Per-tenant Access applications. Rejected: 500-application limit and a policy
  explosion.

**Consequences.** Two authorization layers to keep consistent. The Worker is the
enforcement point, and D1 is the source of truth for roles.

## D15. 0BSD for this design repository

**Date:** 2026-10-29

**Context.** The user asked for a public 0BSD repository.

**Decision.** License the design documents and any sample code in this repository
under 0BSD. Do not relicense third-party components; their licences still apply
when they are used.

**Alternatives.** MIT or Apache-2.0. Rejected because the user specified 0BSD.

**Consequences.** Anyone can use the design without attribution. That is the
intent. It also means the repository cannot enforce a patent grant; if that
matters later, a separate implementation repository can use Apache-2.0.

## D16. No multi-region control plane in v1

**Date:** 2026-10-29

**Context.** A global control plane needs replicated D1, DO placement, and a
conflict model. Cloudflare's primitives are regional in ways that are not fully
transparent.

**Decision.** One control-plane deployment in one Cloudflare location strategy,
with D1 read replication if needed. The data plane can span sites. The control
plane is not active-active across regions in v1.

**Alternatives.**

- Multi-region active-active from day one. Rejected: complexity, and the failure
  budget of a small team.
- One control plane per site. Rejected: N dashboards and N identity systems.

**Consequences.** A regional Cloudflare incident can make the dashboard
unavailable. Workloads keep running. The RTO for management is documented, not
zero.

## D17. No SSH, including break-glass by default

**Date:** 2026-10-29

**Context.** SSH is the default operational path for most self-hosted systems and
the main reason they cannot be managed from a browser.

**Decision.** The control path is the agent. Shell access is an audited exec
session through the agent, recorded and time-limited. Break-glass access uses
Access short-lived certificates or a Teleport-style proxy on a separate network
path, and it is logged. There is no standing SSH key on the machine.

**Alternatives.**

- SSH with a bastion. Rejected: unauditable, hard to make idempotent, and a
  standing credential.
- A web terminal that runs a shell on the host as root. Rejected: a remote root
  shell is the thing being avoided.
- No break-glass at all. Rejected: a machine with a broken agent needs a way in.

**Consequences.** The agent must be robust enough that a broken agent is rare, and
the break-glass path must be tested. The exec path must enforce authorization,
recording and time limits.

## D18. The workspace is a product layer, not a runtime

**Date:** 2026-10-29

**Context.** Cloudflare Computer is a library (`@cloudflare/computer`, early
preview) that gives an agent a SQLite-backed workspace over an isolate and a
container. It is not a separate compute product.

**Decision.** Implement a workspace service on top of the sandbox and Worker
primitives: a durable filesystem in local S3 or a volume, a worker backend for
fast file and code operations, a sandbox backend for Linux commands, and a sync
daemon. Do not build a new runtime for it.

**Alternatives.**

- Treat Computer as a product to clone. Rejected: it is a preview library, and
  the value is the workspace abstraction, not a new VM.
- Skip workspaces. Rejected: agent workloads need a durable filesystem and a
  shell, and that is a common request.

**Consequences.** A sync protocol and a consistency model to design. The
workspace is where the "computer" experience lives, and it is worth the work.

## D19. Cost model is parametric, not a fake quote

**Date:** 2026-10-29

**Context.** Hardware prices vary by country, power contract and second-hand
market. A single "cheaper than Cloudflare" number would be invented.

**Decision.** Use Cloudflare list prices with a read date, and a parametric model
for the data plane: hardware amortization, power, bandwidth, storage media,
replacement and operator time. Publish a worked example with explicit
assumptions and a break-even formula. Do not invent hardware prices.

**Alternatives.**

- A single TCO number. Rejected: it would be wrong for almost every reader.
- Ignore cost. Rejected: cost is the reason for the project.

**Consequences.** The reader has to supply their own inputs. The model is more
useful and more honest.

## D20. Keep the 0BSD design repo separate from the implementation

**Date:** 2026-10-29

**Context.** A design repository and an implementation have different licensing,
review and release needs.

**Decision.** This repository holds the design, the evidence and the decision
record. An implementation repository can choose Apache-2.0 for a patent grant and
can carry the code, tests and releases.

**Alternatives.**

- One repository for design and code. Rejected: the design would be tied to one
  implementation and its release cadence.

**Consequences.** The design can be reused by anyone, including a competing
implementation. That is the point of 0BSD.

## D21. Sign every control-plane request; do not rely on mTLS through a terminating proxy

**Date:** 2026-10-29

**Context.** The default control path is Workers VPC `fetch()` through a
Cloudflare Tunnel. TLS terminates at Cloudflare, so the Worker does not see a
client certificate from the agent and cannot pin it. The relay path sees a
WebSocket handshake, not the TLS peer either.

**Decision.** Authenticate both directions with Ed25519 request signatures
independent of TLS. The agent pins the control-plane public key at enrollment;
the control plane pins the machine public key. The signed tuple is method, path,
body hash, timestamp, nonce, machine id and lease epoch. No Cloudflare path uses
mTLS: TLS terminates at Cloudflare on both the VPC and relay paths, so the
Worker never sees a client certificate. A self-hosted relay that terminates TLS
may add mTLS as a deployment option, but it is not part of the default design.

**Alternatives.**

- Client certificates through Workers VPC. Rejected: the Worker does not receive
  the peer certificate, so it cannot verify or pin it.
- A bearer token in a header. Rejected: replayable within the window, and it
  does not authenticate the control plane to the agent.
- Terminate mTLS at a self-hosted relay. Kept as an optional deployment for
  sites that need it, but it moves the control path off Cloudflare and adds a
  second ingress.

**Consequences.** The protocol needs key rotation, a replay cache and a signature
verification step per request. The 24-hour machine credential is an identity and
revocation artefact checked alongside the signed request, not a TLS client
certificate.
