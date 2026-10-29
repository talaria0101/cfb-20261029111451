# Prior art

Read date for every metadata field: 2026-10-29. Stars, licence, archived flag,
language, creation date and last push come from `gh api repos/OWNER/REPO` and are
stored in `evidence/repos.tsv`. Feature claims come from a saved README in
`evidence/reads/`; where no capture exists, the claim is labelled relayed.
Nothing here was installed, built or benchmarked.

Labels:

- **Metadata verified**: the API fields above were read on 2026-10-29.
- **Feature relayed**: behaviour described by the project itself, not run.
- **Pattern only**: the project is not a broker or a Cloudflare replacement. It
  contributes a design idea.

## 1. Pattern sources

These six projects answer narrow questions. Four are not brokers, and none is a
drop-in backend.

| Project | Licence | Stars | Shape | Use in CFB | Trap |
| --- | --- | --- | --- | --- | --- |
| Pouzor/homelable | MIT | 4,785 | LAN dashboard, FastAPI, SQLite, React | Status envelope and probe taxonomy | No agent, tunnel or multi-tenant path |
| crocofied/CoreObs | MIT | 1,296 | Next.js UI, Go agent, Postgres, Glances | Inventory and history data model | Agent writes directly to Postgres |
| timhartmann7/omnyssh | Apache-2.0 | 1,246 | Rust SSH manager, TUI and Tauri | Host inventory and reconnect backoff | Client-side SSH; no broker or relay |
| ZingerLittleBee/ServerBee | AGPL-3.0 | 339 | Hub-and-spoke VPS monitoring | Capability handshake and enrollment protocol | Server must be reachable; root installer |
| metalbear-co/mirrord | MIT | 5,391 | Local process in a Kubernetes pod | Protocol versioning and stream multiplexing | Kubernetes and elevated capabilities |
| talaria0101/cfrs | 0BSD | 0 | Client-side Cloudflare tunnel connector | Outbound Unix-socket origin and splice | No registration or serving loop |

Metadata verified for all six. Feature claims relayed from READMEs captured in
`evidence/reads/`.

### 1.1 homelable

FastAPI and uvicorn, SQLite, a React SPA and nginx. Three processes, LAN-only
listening, no agent and no outbound path. Status is pushed over WebSocket keyed
by `device_id` and `node_ids`. Probes are typed: `ping`, `http`, `https`, `tcp`,
`ssh`, `prometheus`, `health`, `none`.

Reuse the status envelope and the probe type taxonomy. Do not reuse the
in-process broadcast list: it cannot span replicas. Do not claim homelable is a
broker.

### 1.2 CoreObs

The released product is on branch `corecontrol`; `main` is an unfinished
Bun/Elysia/Prisma rewrite with only `POST /auth/register`. The released stack is
Next.js, a Go agent, Postgres and Glances.

Reuse the `server`, `application` and history model: a denormalized current row
plus a time series, with retention and pruning. Its agent connects straight to
Postgres with `DATABASE_URL`, bypassing its own API. Do not copy that. Its app
probes are SSRF-shaped, and the web process runs as root.

### 1.3 omnyssh

Rust SSH client and server manager: `omnyssh-core`, a TUI (`omny`) and a
Tauri/Svelte GUI. Native SSH over TCP, optional `ProxyJump`, local `ssh -L`
forwarding only. No WebSocket, agent or relay.

Reuse the host-centric model and the reconnect ladder: 1, 2, 5, 10, 30 seconds,
reset after 60 seconds stable. Reuse the one-shot inventory probe with section
markers. Do not copy its client-side credential store into a server: a broker
needs a secret store, not a local TOML file.

### 1.4 ServerBee

Hub-and-spoke VPS monitoring. The server must be reachable and agents dial
`ws://host/api/agent/ws`. This is not zero-inbound.

The protocol is the best part. The agent proposes its own run token and the
server stores only an argon2 hash. Enrollment offers are single use and last
about ten minutes. `Welcome` carries `protocol_version` and `report_interval`.
Messages carry `msg_id` and are acknowledged. Admission is preflight-then-lock
with `connection_id` fencing. One JSON WebSocket multiplexes metrics, PTY, exec,
files and Docker. The capability bitmask is owned by the agent host and
reconciled against real host ability; temporary grants are host-local.

Copy the protocol shape and the capability ownership. Do not copy the code:
AGPL-3.0. Do not copy the root installer or the `--privileged` Docker agent.

### 1.5 mirrord

Rust workspace that runs a local process as if it were in a Kubernetes pod.
Transport is a binary protocol over a TCP port-forward or a WebSocket through the
Kubernetes API server.

Reuse three ideas: protocol versioning rules (renaming is compatible, appending
an enum variant is compatible, adding a field is not; a separate crate version,
`SwitchProtocolVersion`, and per-feature `VersionReq`), one multiplexed agent
connection with bounded per-stream FIFOs (the tunnel default is 256 KiB), and
Prometheus gauges for queue depth.

Do not copy the Kubernetes assumptions or the targeted-mode privileges:
`NET_ADMIN`, `SYS_PTRACE`, `SYS_ADMIN`, host PID and host mounts. The operator
controller is not in the repository.

### 1.6 cfrs

A client-side Rust connector for Cloudflare: QUIC with `quinn` and ALPN
`argotunnel`, HTTP/2 with `h2`, SSH remote forward with `russh`, HTTP CONNECT,
and a userspace TCP/IP stack (`smoltcp`) behind an `LD_PRELOAD` shim.

Reuse the outbound-only origin as a Unix socket, so there is no listening TCP
port; one channel per visitor; the `copy_bidirectional` half-close discipline; a
plain byte pipe from `proxy::connect_tunnel`; and the WebSocket transport
wrapper. The rootless, no-TUN design is the right model for `lite`.

Do not claim it is a working tunnel. Its own README says `cfrs tunnel` does not
serve traffic, registration and visitor serving are not implemented, and the
HTTP/2 and QUIC transports build connections and frame messages but nothing
consumes them. A native tunnel needs that missing registration and serve loop.

## 2. Workers platforms

| Project | Licence | Stars | Shape | Multi-node | Verdict |
| --- | --- | --- | --- | --- | --- |
| open-compute | Apache-2.0 | 2,436 | One Rust binary `ocd`, pinned workerd | No | Optional full API surface |
| celld | Apache-2.0 | 6,169 | Fleet runtime, S3-backed leases | Yes | Distributed DO track |
| WDL | Apache-2.0 | 55 | Multi-tenant platform on stock workerd | Yes | Reference, too young |
| Rivet | Apache-2.0 | 6,488 | Actor platform, own control plane | Yes | Actor workloads only |
| workerd | Apache-2.0 | 9,093 | The runtime | No | Primitive |
| workers-sdk | Apache-2.0 | 4,680 | Wrangler build and deploy | n/a | Tooling |
| self-workerd | MIT | 141 | FaaS on workerd | No | Stale since 2025-06 |
| vorker | NONE | 204 | Simple worker host | No | Licence unclear |
| groundflare | MIT | 5 | Wrangler-like DX | No | Toy |
| OpenWorkers runner | MIT | 237 | Self-hosted runner | No | Early |

Metadata verified. Feature claims relayed.

**open-compute.** Claims Workers, KV, D1, R2, Durable Objects, Queues,
Workflows, Cron, Cache, Images, Vectorize, AI Search, Artifacts, Static Assets,
Service Bindings, Dynamic Workers, LogTail and Gateway in one binary. Uses a
pinned workerd fork, SQLite with optional S3, a dashboard and a `/client/v4` API.
The README says it is single-machine, not an HA cluster, capability-based for
tenant isolation, and partial for containers. It reimplements a large API, so
pin a version and treat a machine as a failure domain.

**celld.** A fleet runtime for Workers and Durable Objects backed by an
S3-compatible bucket for ownership and conditional writes. Runs from a
`wrangler.json`. Documented limits: one application per fleet, no multi-tenant
account service, no TLS termination, Linux only, resident-cell and outbound
WebSocket limits, and migration only for hibernated cells. Pre-1.0. Use one
tenant per fleet, or wait.

**Rivet.** Durable actors with SQLite, queues, scheduling, hibernation and a
self-hostable control plane. One Rust binary plus an S3 bucket. It is an actor
platform, not a Workers runtime; a Worker bundle does not run unchanged.

**workerd.** The actual runtime and the least risky primitive. Three facts:

1. Its README says it is not a hardened sandbox. Untrusted code needs a VM.
2. `workerd serve config.capnp` has no deploy API, dashboard, tenant model or
   secret store. `--socket-fd` supports socket activation.
3. `src/workerd/server/workerd.capnp` defines a `ClusterConfig` with S3-backed
   leases and conditional writes for Durable Objects. It requires
   `--experimental`, and a bucket copy replaces local state, so prior local-disk
   DOs are refused at startup. Verified from
   `evidence/reads/cloudflare_workerd.capnp`, read 2026-10-29.

The `ClusterConfig` is a research track, not a supported multi-tenant platform.

## 3. Sandboxes and microVMs

| Project | Licence | Stars | Role | Verdict |
| --- | --- | --- | --- | --- |
| Firecracker | Apache-2.0 | 37,695 | MicroVM primitive | Use for untrusted code |
| bhatti | Apache-2.0 | 270 | MicroVM orchestrator, libkrucible/libkrun | Pilot |
| hyper | MIT | 171 | Distributed Firecracker orchestrator in Elixir | Reference |
| Fireactions | Apache-2.0 | 200 | GitHub runners on Firecracker | Use for GitHub CI |
| E2B runtime | Apache-2.0 | 1,781 | Sandbox runtime and API | Reference |
| microsandbox | Apache-2.0 | 8,861 | Self-hosted sandbox platform | Evaluate |
| gVisor | Apache-2.0 | 19,818 | User-space kernel | Niche |
| Kata | Apache-2.0 | 9,044 | VM-isolated containers | Niche |
| Coder | AGPL-3.0 | 17,220 | Cloud dev environments | Licence risk |
| Daytona | NONE, archived | 71,441 | Dev environments | Avoid |

Metadata verified. Feature claims relayed.

Firecracker gives one process per microVM, a small device model, a REST API on a
Unix socket, snapshots and a jailer. It needs KVM and provides no scheduler,
network, image store or metering. bhatti is the closest orchestrator but its v1
is frozen and v2 uses libkrucible. Do not build an orchestrator from scratch if
one of these fits.

## 4. Container and VM platforms

| Project | Licence | Stars | Role | Verdict |
| --- | --- | --- | --- | --- |
| Incus | Apache-2.0 | 6,524 | Containers and VMs, REST API, clustering | Default privileged runtime |
| LXD | AGPL-3.0 | n/a | Incus predecessor | Avoid for a product |
| Proxmox | AGPL-3.0 upstream | 108 (mirror) | VM platform | Only if already operated |
| Harvester | Apache-2.0 | 5,228 | HCI on Kubernetes | Heavy |
| OpenStack | Apache-2.0 | 6,070 | IaaS | Too heavy |
| CloudStack | Apache-2.0 | 3,120 | IaaS | Too heavy |
| OpenNebula | Apache-2.0 | 1,762 | IaaS | Too heavy |
| MicroCloud | AGPL-3.0 | 545 | Incus-based small cloud | Licence risk |
| Talos | MPL-2.0 | 11,429 | Kubernetes OS | For k8s sites |
| Kubernetes | Apache-2.0 | 128,596 | Orchestrator | Optional |

Metadata verified. Feature claims relayed.

Incus is one daemon with a Unix-socket REST API, containers and VMs in one object
model, clustering with a distributed database, projects, storage pools,
snapshots and migration. Apache-2.0 and active. Containers share the host
kernel, so use Incus VMs for semi-trusted workloads and Firecracker for
untrusted ones.

## 5. Storage

| Project | Licence | Stars | Role | Verdict |
| --- | --- | --- | --- | --- |
| SeaweedFS | Apache-2.0 | 35,799 | S3, EC, replication, IAM | Default multi-node |
| RustFS | Apache-2.0 | 35,526 | S3, EC, site replication, console | Pilot |
| Garage | AGPL-3.0 | 4,784 | Small geo-distributed S3 | Home lab |
| Ceph | LGPL/GPL mix | 17,178 | Unified storage | Only if operated |
| MinIO | AGPL-3.0 | 61,300 | S3 server, archived | Avoid |
| VersityGW | Apache-2.0 | 3,079 | S3 gateway over POSIX | Niche |
| Zenko CloudServer | Apache-2.0 | 1,955 | S3 server | Niche |
| JuiceFS | Apache-2.0 | 14,586 | POSIX over S3 | Niche |

Metadata verified. Feature claims relayed.

SeaweedFS claims S3, IAM and STS, erasure coding for warm data, replication for
hot data, cross-cluster replication, an admin UI and a `weed mini` single-node
mode. RustFS claims S3 core, erasure coding, bucket and site replication, IAM,
OIDC/SSO, KMS, console, SNS and ILM tiering; MinIO on-disk compatibility is
preview. MinIO is archived (last push 2026-04-24) and AGPL. Do not start a new
deployment on it.

CFB's local S3 gateway covers a narrow subset for `lite`; full IAM, STS and
policy support is delegated to SeaweedFS or RustFS.

## 6. CI

| Project | Licence | Stars | Fit | Verdict |
| --- | --- | --- | --- | --- |
| Fireactions | Apache-2.0 | 200 | GitHub Actions on Firecracker | Use for GitHub |
| Woodpecker | Apache-2.0 | 8,067 | Gitea/Forgejo CI | Use for Gitea |
| actions-runner-controller | Apache-2.0 | 6,623 | GitHub runners on Kubernetes | If k8s |
| GitHub Actions runner | MIT | 6,364 | The runner itself | Component |
| Buildkite agent | MIT | 1,101 | Agent/job API design | Reference |
| GitLab Runner | MIT | 248 (mirror) | GitLab CI | If GitLab |
| Tekton | Apache-2.0 | 9,094 | Kubernetes pipelines | Heavy |
| Argo Workflows | Apache-2.0 | 17,067 | Kubernetes workflows | Heavy |
| Dagger | Apache-2.0 | 16,544 | Reproducible steps | Component |
| Jenkins | MIT | 26,702 | General CI | Legacy |
| act | MIT | 72,478 | Run Actions locally | Testing only |

Metadata verified. Feature claims relayed.

Fireactions gives ephemeral Firecracker runners for GitHub, JIT runner tokens
and pool scaling on `workflow_job` events. Woodpecker gives an Apache-2.0 server
and agent with a gRPC boundary and a Kubernetes backend. Integrate a runner, own
the scheduling, secrets and observability around it, and do not write a CI
engine.

## 7. Connectivity

| Project | Licence | Stars | Role | Verdict |
| --- | --- | --- | --- | --- |
| cloudflared | Apache-2.0 | 16,337 | Tunnel client | Native tunnel fallback |
| Pangolin | AGPL + commercial | 23,263 | Self-hosted tunnel | Reference |
| NetBird | BSD core + AGPL mgmt | 30,316 | WireGuard overlay | Alternative |
| Headscale | BSD-3-Clause | 45,164 | Tailscale control server | Single-tailnet alternative |
| Tailscale | BSD-3-Clause | 38,286 | Mesh VPN | Hosted |
| frp | Apache-2.0 | 110,283 | Reverse proxy tunnel | Simple alternative |
| rathole | Apache-2.0 | 14,382 | Reverse proxy tunnel | Simple alternative |
| Nebula | MIT | 18,481 | Overlay network | Alternative |
| ZeroTier | MPL core + BSL | 17,221 | Overlay network | Licence risk |
| Teleport | AGPL-3.0 | 21,044 | Access plane | Licence risk |
| OpenZiti | Apache-2.0 | 4,458 | Zero-trust network | Alternative |
| Octelium | AGPL-3.0 | 4,134 | Zero-trust/PaaS | Licence risk |

Metadata verified. Feature claims relayed.

CFB does not require a self-hosted overlay. The default is a WebSocket relay in
the Cloudflare account, which works through HTTPS-only networks without root.
Native Cloudflare Tunnel over QUIC or HTTP/2 is the optional high-throughput
path. A `PrivateTransport` interface keeps frp, Headscale or NetBird available
for sites that forbid Cloudflare.

## 8. Control-plane UX references

| Project | Licence | Stars | What to learn |
| --- | --- | --- | --- |
| Coolify | Apache-2.0 | 65,237 | Self-hosted PaaS UX, deployments, backups |
| Dokploy | NOASSERTION | 38,109 | Templates and Docker UX |
| CapRover | NOASSERTION | 15,195 | One-click apps |
| Dokku | MIT | 32,208 | Git-push deploys |
| Kamal | MIT | 14,697 | Explicit container deploys |
| Portainer | Zlib | 38,744 | Edge agents, RBAC, UI |
| Cloudflare Computer | MIT | 9,752 | Workspace abstraction |
| Cloudflare sandbox-sdk | NOASSERTION | 1,156 | Sandbox API |

Metadata verified. Feature claims relayed.

Coolify is the best UX reference for "one dashboard, many machines, deploy
things". It orchestrates over SSH, which CFB rejects, but its information
architecture, backup flow and status model are worth studying. Portainer's edge
agent and RBAC are the closest public analogue to the CFB split: central UI,
outbound agent, scoped permissions.

## 9. Licence traps

| Project | Licence | Why it matters |
| --- | --- | --- |
| MinIO | AGPL-3.0 | Archived; network clause if offered as a service |
| LXD | AGPL-3.0 | Incus is the Apache-2.0 fork |
| Nomad | BUSL-1.1 | Not open source for a competing product |
| Coder | AGPL-3.0 | Network clause |
| Garage | AGPL-3.0 | Network clause |
| Teleport | AGPL-3.0 | Network clause |
| Octelium | AGPL-3.0 | Network clause |
| ServerBee | AGPL-3.0 | Do not copy code into a distributed broker |
| Restate | BSL 1.1 | Competing-service restriction |
| Inngest | SSPL 1.0 | Service restriction |
| Windmill | Apache-2.0 + AGPL-3.0 + proprietary | Mixed |
| ZeroTier | MPL-2.0 core + BSL nonfree | Mixed |
| Pangolin | Fossorial commercial plus other files | Review |
| NetBird | BSD-3-Clause core, AGPL management | Review |
| Ceph | LGPL/GPL mix | Review |

The rule: a component used at arm's length can be copyleft. A component that is
linked, modified or embedded in a distributed product should be Apache-2.0,
MIT or 0BSD. Keep copyleft on the data plane behind a process boundary and
document the obligation.

## 10. Build versus reuse

| Capability | Reuse | Build |
| --- | --- | --- |
| Container and VM lifecycle | Incus, Podman | Driver |
| MicroVM lifecycle | Firecracker | Driver and scheduler |
| Workers runtime | workerd | Config generator and shim |
| Workers platform | open-compute, optional | Supported subset |
| Distributed DO | celld, optional | Placement |
| S3 | Local gateway, SeaweedFS, RustFS | Narrow gateway |
| CI | GitHub runner, Woodpecker, Fireactions | Supervision and secrets |
| Tunnel | Relay, optional native tunnel | Transport client |
| Control plane | None fits | Product |
| Broker | None fits | Product |
| Dashboard | None fits | Product |
| Placement and leases | None fits | Product |
| Audit and policy | None fits | Product |

The control plane, broker, data model and policy engine are the work.
Everything below them is a replaceable dependency.

## 11. Gaps

1. A multi-tenant control plane native to Cloudflare. amika's is closed; Coolify
   and Dokploy are SSH-based.
2. One broker that reconciles workerd, Podman, Incus and Firecracker. Each
   project has its own agent; none spans the classes.
3. Leases and fencing across a control plane and a BYOC daemon. amika has a
   tunnel protocol; its lease model is not public.
4. One verifiable audit chain across control-plane and data-plane actions.
5. Cost attribution per tenant per machine.
6. A tested restore path from local S3 to R2 and back.
7. An honest isolation matrix. Most self-hosted platforms treat a container as a
   security boundary. It is not.

## 12. Not covered

- No project was installed, built or benchmarked.
- Star counts and licences are a snapshot and change.
- The search was not exhaustive. Projects under roughly 50 stars were included
  only when directly relevant.
- Non-GitHub sources are out of scope here; Cloudflare docs are in
  `docs/spec.md` and `docs/cost.md`.
- Licence texts were not read line by line for every row. NOASSERTION and mixed
  rows need a legal read before they become dependencies.

Metadata: `evidence/repos.tsv`. READMEs: `evidence/reads/`.
