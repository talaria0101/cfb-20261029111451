# Prior art: what already exists on GitHub

Read date for every metadata field: 2026-10-29. Stars, licence, archived flag,
language, creation date and last push come from the GitHub REST API through
`gh api repos/OWNER/REPO` and are stored in `evidence/repos.tsv`. Feature claims
come from a saved README capture in `evidence/reads/` where one exists, otherwise
from the project's GitHub description or a search result. The second is weaker
evidence and is labelled relayed. Feature claims are **not tested** unless a
section says otherwise.

Two labels are used throughout:

- **Metadata verified**: star count, licence, archived flag and last push were
  read from the API on 2026-10-29.
- **Feature relayed**: the behaviour described comes from the project's own
  documentation and was not run.

## 1. The short list

If you build CFB and want the smallest set of dependencies that covers the
requirements:

| Role | Pick | Licence | Why |
| --- | --- | --- | --- |
| Site runtime | Incus | Apache-2.0 | Containers and VMs, REST API, clustering, projects, active |
| Workers platform per machine | open-compute | Apache-2.0 | One binary, workerd, Workers/KV/D1/R2/DO/Queues/Workflows, dashboard |
| Distributed Durable Objects | celld | Apache-2.0 | Closest DO model on private hardware; pre-1.0, one app per fleet |
| Sandbox primitive | Firecracker | Apache-2.0 | Small device model, per-VM API, snapshots |
| S3 | SeaweedFS | Apache-2.0 | S3, IAM/STS, erasure coding, replication, single-node path |
| GitHub CI | Fireactions | Apache-2.0 | Ephemeral Firecracker runners for GitHub Actions |
| Gitea/Forgejo CI | Woodpecker | Apache-2.0 | Server and agents, autoscaler, simple operations |
| Tunnel | cloudflared | Apache-2.0 | The client Cloudflare supports; one per machine |
| Control-plane reference | amika | Apache-2.0 | Host daemon and SDKs; control plane is closed |

Everything else on this page is either a reference, a component for a specific
niche, or a trap.

## 2. amika

Repository: `gofixpoint/amika`. Apache-2.0. 295 stars. Created 2026-02-05.
Last push 2026-10-28. Metadata verified.

This is the closest published system to the user's stated shape: a hosted
control plane, a BYOC daemon on machines you own, sandboxes, and a tunnel-based
connection. It is worth reading before writing a line of CFB.

The open/closed split and the self-hosting limits were checked against the
public repository at commit `fe5accb095bffdc2a560706ce92764114dfcfb1f`
(2026-10-28), plus `docs/self-hosting.md` and `js/amika-hostd`. Those checks are
marked verified below. Everything else about amika is from its README.

### 2.1 What is open

| Path | What it is |
| --- | --- |
| `cmd/amika` | CLI |
| `cmd/amikad` | Daemon |
| `sandbox-image/` | Image build |
| `js/amika-hostd` | Host daemon that runs sandboxes |
| `js/sandbox` | Sandbox SDK |
| `js/service-auth` | Service authentication |
| `js/tunnel-protocol` | Tunnel protocol |
| `sdks/` | Client SDKs |
| `docs/self-hosting.md` | Self-hosting notes |

### 2.2 What is closed or missing

- `amika-control-plane`: not in the public org.
- `amika-ui`: not in the public org.
- `amika-relay`: not in the public org.
- `amika-gateway` and `sandbox-provisioner`: described as "will be open sourced
  soon" and absent from the org's repository list.

The self-hosting document says there is no supported local CLI path. The future
self-hosted gateway is described as single-player with no IAM. So amika is not a
self-hostable replacement today. It is a reference for the shape, not a
dependency.

### 2.3 What to reuse or copy

- **`amika-hostd` as a design reference.** It runs smolvm VMs, dials out to the
  gateway, optionally listens behind ngrok or Cloudflare Tunnel, stores secrets
  in the keychain or a 0600 file, supports three credential types, and uses
  credit-based flow control on the tunnel. Its constant-time credential checks
  are a good default.
- **The split between hostd and the control plane.** The daemon owns local
  execution; the control plane owns identity and routing. That is the same split
  CFB uses.
- **The tunnel protocol.** Credit-based flow control, where the control plane
  opens streams and the agent never initiates one, is the right default for a
  daemon that might be compromised.
- **The sandbox image and SDK.** Useful if the API shape fits; check the licence
  and the API stability before depending on it.

### 2.4 What not to copy

- The closed control plane. You cannot self-host it, and the public pieces are
  not a complete system.
- The single-player gateway. It has no IAM, which is not enough for multi-tenant
  use.
- The reliance on smolvm. It is an amika-specific VM layer; CFB uses Incus and
  Firecracker, which have larger communities and clearer operational stories.

Verdict: **read it, borrow the protocol ideas, do not depend on it.**

## 3. Workers-compatible platforms

This is the most important category, because "run Cloudflare's own products on
my machine" is the user's explicit question.

### 3.1 Comparison

| Project | Licence | Stars | Shape | Multi-node | Workers | DO | Storage | Dashboard | Verdict |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| open-compute | Apache-2.0 | 2,436 | One Rust binary `ocd`, pinned workerd | No (single machine) | Yes | Yes | SQLite, local or S3 | Yes | Default per machine |
| celld | Apache-2.0 | 6,169 | Fleet runtime, S3-backed leases | Yes | Yes | Yes | S3/GCS/Azure bucket | No | Default distributed DO |
| WDL | Apache-2.0 | 55 | Multi-tenant platform on stock workerd | Yes | Yes | Yes | Workerd storage | Yes | Reference, too young |
| Rivet | Apache-2.0 | 6,488 | Actor platform, own control plane | Yes | No | Actors | SQLite + bucket | Yes | Actor workloads only |
| workerd | Apache-2.0 | 9,093 | The runtime itself | No | Yes | Yes, via config | Local disk | No | Primitive |
| Wrangler / workers-sdk | Apache-2.0 | 4,680 | Build and deploy tooling | n/a | n/a | n/a | n/a | No | Tooling |
| self-workerd | MIT | 141 | FaaS on workerd | No | Yes | No | Local | No | Stale since 2025-06 |
| vorker | NONE | 204 | Simple worker host | No | Yes | No | Local | Yes | Licence unclear |
| groundflare | MIT | 5 | Wrangler-like DX for self-hosted workerd | No | Yes | No | Local | No | Toy |
| OpenWorkers runner | MIT | 237 | Self-hosted Workers runner | No | Yes | Partial | Local | No | Early |
| durable-git | MIT | 269 | Git on Durable Objects | n/a | n/a | Yes | DO | No | Reference |

Metadata verified for all rows. Feature claims relayed from READMEs.

### 3.2 open-compute

Repository: `elliothux/open-compute`. Apache-2.0. 2,436 stars. Created
2026-08-23. Last push 2026-10-28. Metadata verified.

Claimed surface, from the README: Workers, KV, D1, R2, Durable Objects, Queues,
Workflows, Cron, Cache, Images, Vectorize, AI Search, Artifacts, Static Assets,
Service Bindings, Dynamic Workers, LogTail and Gateway in one Rust binary. It
uses a pinned workerd fork, stores state in SQLite with optional S3, ships a
dashboard and a `/client/v4` API, and reports 2,256 API members and 90%+ test
coverage. Feature relayed.

The README is explicit that it is single-machine and not a multi-replica HA
cluster. Tenant isolation is capability-based. Containers are partial and depend
on an operator-owned Linux broker. Feature relayed.

Why it is the default: it is the only project that gives a single machine a
complete Workers-compatible surface with a dashboard and an API the agent can
drive. It is also the project most likely to have edge-case bugs, because it
reimplements a large API surface. Pin a version, run it behind the agent, and
treat a machine as a failure domain.

### 3.3 celld

Repository: `denoland/celld`. Apache-2.0. 6,169 stars. Created 2025-04-25. Last
push 2026-10-21. Version 0.6.3 (pre-1.0) at the time of reading. Metadata verified.

Claimed shape: a fleet runtime for Workers and Durable Objects, backed by an
S3-compatible bucket for ownership and conditional writes. It runs Workers, DO,
KV, D1, R2, Queues, Workflows and Cron from a `wrangler.json`. Feature relayed.

Documented limits: one application per fleet, no account service or
multi-tenant support, no TLS termination, Linux only, node limits on resident
cells and outbound WebSockets, and migration only for hibernated cells. Feature
relayed.

Why it matters: it is the closest thing to a distributed DO runtime that runs on
hardware you own, and it is the piece that would otherwise be hardest to build.
Why it is not the default: pre-1.0, single-app, no TLS, and no multi-tenant account
model. Use it for one tenant per fleet, or wait for 1.0.

### 3.4 WDL

Repository: `wdl-dev/wdl`. Apache-2.0. 55 stars. Created 2026-06-24. Last push
2026-10-22. Metadata verified.

Claimed shape: a multi-tenant platform on stock workerd with multi-replica
failover. It is the best public reference for the platform layer that open-compute
and celld each solve differently. It is too young and too small to depend on.
Feature relayed.

### 3.5 Rivet

Repository: `rivet-dev/rivet`. Apache-2.0. 6,488 stars. Created 2023-06-16. Last
push 2026-10-29. Metadata verified.

Claimed shape: durable actors with SQLite, queues, scheduling, hibernation and a
self-hostable control plane. One Rust binary plus an S3-compatible bucket, no
external database or message bus. Claims 6.9 ms cold starts and 72 KB per actor.
Feature relayed.

Why it is not the default: it is an actor platform, not a Workers runtime. A
Worker bundle does not run unchanged. Use it when the workload is actor-shaped
(per-session state, long-lived agents, game rooms, collaborative documents) and
when the Workers API compatibility does not matter.

### 3.6 workerd

Repository: `cloudflare/workerd`. Apache-2.0. 9,093 stars. Last push 2026-10-29.
Metadata verified.

This is the actual runtime. It is the least risky component to depend on and the
most work to operate. Three facts drive the design:

1. The README says workerd is **not a hardened sandbox**. Untrusted code must run
   inside a VM or an equivalent boundary. Feature relayed, but stated plainly by
   the project.
2. `workerd serve config.capnp` runs a server; systemd examples use
   `--socket-fd`. There is no built-in deploy API, dashboard, tenant model or
   secret store.
3. `src/workerd/server/workerd.capnp` contains a `ClusterConfig` that distributes
   Durable Objects across instances with S3-backed leases and conditional writes.
   It requires `--experimental`, and a bucket copy replaces local state, so prior
   local-disk DOs are refused at startup. Verified from the source file saved at
   `evidence/reads/cloudflare_workerd.capnp`, read 2026-10-29.

The `ClusterConfig` is a research track. It is not a supported multi-tenant
platform, and the migration semantics are hostile to a mixed local/S3 fleet.

### 3.7 Tooling

- `cloudflare/workers-sdk` (Apache-2.0, 4,680 stars): Wrangler. Use it to build
  and bundle, not as a server.
- `cloudflare/terraform-provider-cloudflare` (Apache-2.0, 1,351 stars): manage
  the Cloudflare side declaratively.
- `cloudflare/agents` (MIT, 5,912 stars): agent framework on Workers and DO. A
  reference for the agent UX, not a runtime.
- `cloudflare/computer` (MIT, 9,752 stars, resolves from `cloudflare/workspace`):
  the Workspace library behind Cloudflare Computer. Early preview, 0.8.0. A
  reference for the workspace abstraction.

## 4. Sandboxes and microVMs

| Project | Licence | Stars | Role | Verdict |
| --- | --- | --- | --- | --- |
| Firecracker | Apache-2.0 | 37,695 | MicroVM primitive | Use |
| bhatti | Apache-2.0 | 270 | MicroVM orchestrator, libkrucible/libkrun | Pilot |
| hyper | MIT | 171 | Distributed Firecracker orchestrator in Elixir | Reference |
| Fireactions | Apache-2.0 | 200 | GitHub runners on Firecracker | Use for GitHub CI |
| E2B runtime | Apache-2.0 | 1,781 | Sandbox runtime and API | Reference |
| microsandbox | Apache-2.0 | 8,861 | Self-hosted sandbox platform | Evaluate |
| gVisor | Apache-2.0 | 19,818 | User-space kernel | Niche |
| Kata | Apache-2.0 | 9,044 | VM-isolated containers | Niche |
| Coder | AGPL-3.0 | 17,220 | Cloud dev environments | Licence risk |
| Daytona | NONE (archived) | 71,441 | Dev environments | Avoid |
| Firecracker + jailer | Apache-2.0 | n/a | Process isolation for the VMM | Use with Firecracker |

Metadata verified. Feature claims relayed.

### 4.1 Firecracker

The primitive. One process per microVM, a small device model, a REST API on a
Unix socket, snapshot and restore, and a jailer for host-side isolation. It needs
KVM. It does not schedule, network, image, or meter by itself. That is the work
CFB's agent does.

Do not build a Firecracker orchestrator from scratch if bhatti or hyper fits. Do
not depend on bhatti v1 (Firecracker) when v2 uses libkrucible; the project says
v1 is frozen. Feature relayed.

### 4.2 bhatti

Repository: `sahil-shubham/bhatti`. Apache-2.0. 270 stars. Created 2026-03-11.
Last push 2026-10-27. Metadata verified.

Claimed shape: self-hostable microVMs for agents, with a daemon and an API,
Linux and macOS hosts, and a v2 built on `libkrucible` and libkrun. It is a good
candidate for the sandbox driver if the API matches and the project matures.
Feature relayed.

### 4.3 hyper

Repository: `harmont-dev/hyper`. MIT. 171 stars. Created 2026-06-19. Last push
2026-10-29. Metadata verified.

Claimed shape: a distributed Firecracker orchestrator in Elixir with clustering,
scheduling and a control API. It is a reference for the orchestration side, not
a dependency, because it is early and its operational model (BEAM distribution)
is different from CFB's.

### 4.4 E2B runtime

Repository: `e2b-dev/runtime`. Apache-2.0. 1,781 stars. Last push 2026-10-29.
Metadata verified.

The open-source runtime behind E2B's sandboxes. It is a good reference for
sandbox lifecycle, snapshotting and the API shape. It is not a drop-in for
Firecracker on your own fleet without the E2B control plane.

### 4.5 microsandbox

Repository: `superradcompany/microsandbox`. Apache-2.0. 8,861 stars. Last push
2026-10-29. Metadata verified.

A self-hosted sandbox platform with a CLI and an API. Worth evaluating if you
want a higher-level product than raw Firecracker. It is younger than Firecracker
and adds its own runtime assumptions; test it before trusting it.

## 5. Container and VM platforms

| Project | Licence | Stars | Role | Verdict |
| --- | --- | --- | --- | --- |
| Incus | Apache-2.0 | 6,524 | Containers + VMs, REST API, clustering | Default site runtime |
| LXD | AGPL-3.0 | n/a | Incus predecessor | Avoid for a product |
| Proxmox | NONE in the mirror; AGPL-3.0 upstream | 108 (mirror) | VM platform | Only if already operated |
| Harvester | Apache-2.0 | 5,228 | HCI on Kubernetes | Heavy |
| OpenStack | Apache-2.0 | 6,070 | IaaS | Too heavy |
| CloudStack | Apache-2.0 | 3,120 | IaaS | Too heavy |
| OpenNebula | Apache-2.0 | 1,762 | IaaS | Too heavy |
| MicroCloud | AGPL-3.0 | 545 | Incus-based small cloud | Licence risk |
| Talos | MPL-2.0 | 11,429 | Kubernetes OS | Good for k8s sites |
| Omni | NOASSERTION | 1,419 | Talos fleet management | Niche |
| Rancher | Apache-2.0 | 26,020 | Kubernetes management | Optional |
| Portainer | Zlib | 38,744 | Container UI | Reference |
| Kubernetes | Apache-2.0 | 128,596 | Orchestrator | Optional for k8s sites |

Metadata verified. Feature claims relayed.

### 5.1 Why Incus

- One daemon with a REST API over a local Unix socket.
- Containers and VMs with the same object model.
- Clustering with a distributed database, so a site can survive one machine loss.
- Projects and profiles for tenant separation.
- Storage pools, snapshots, migration, PCI and GPU passthrough.
- Apache-2.0, active, no CLA.
- A CLI that is easy to script for break-glass operations.

The tradeoff: containers share the host kernel, so they are not a hostile
multi-tenant boundary. Use Incus VMs for semi-trusted workloads and Firecracker
for untrusted ones.

LXD is the predecessor and is AGPL-3.0. Do not build a product on it. Proxmox is
a fine platform if you already run it, but its core is AGPL and its API is not
designed for a multi-tenant control plane.

### 5.2 Why not a full IaaS

OpenStack, CloudStack and OpenNebula solve a larger problem: virtual data centers,
networking, quotas, images and identity. They bring control-plane services,
databases, message queues and operational burden that dwarf a small fleet. Use
one only if the site already runs it and the agent can call its API.

## 6. Storage

| Project | Licence | Stars | Role | Verdict |
| --- | --- | --- | --- | --- |
| SeaweedFS | Apache-2.0 | 35,799 | S3, EC, replication, IAM | Default |
| RustFS | Apache-2.0 | 35,526 | S3, EC, site replication, console | Pilot |
| Garage | AGPL-3.0 | 4,784 | Small geo-distributed S3 | Home lab only |
| Ceph | NOASSERTION (LGPL/GPL mix) | 17,178 | Unified storage | Only if operated |
| MinIO | AGPL-3.0 | 61,300 | S3 server, archived | Avoid |
| VersityGW | Apache-2.0 | 3,079 | S3 gateway over POSIX | Niche |
| Zenko CloudServer | Apache-2.0 | 1,955 | S3 server | Niche |
| JuiceFS | Apache-2.0 | 14,586 | POSIX over S3 | Niche |

Metadata verified.

### 6.1 SeaweedFS

Claimed: S3 API, IAM and STS, erasure coding for warm data, replication for hot
data, active-active cross-cluster replication, an admin UI and API, and a
`weed mini` single-node production mode. Feature relayed from the README.

Why it wins: it covers the S3 surface a tenant expects (IAM, STS, bucket
policies, presigned URLs), it has a single-node path, and it scales out without
a separate metadata database cluster in the small case. The cost is more moving
parts than a single binary and a set of tuning knobs that matter at scale.

### 6.2 RustFS

Claimed: S3 core, erasure coding, bucket and site replication, IAM and policies,
OIDC/SSO, KMS, console, SNS, ILM tiering. MinIO on-disk compatibility is
preview, not the default. Standalone mode is single-node. Feature relayed.

Why it is the pilot: it has a broader enterprise surface than SeaweedFS
(OIDC/SSO, KMS, console) and an Apache-2.0 licence. It is younger, and its own
compatibility notes are honest about gaps. Run it on a non-critical bucket
first.

### 6.3 MinIO

The repository is archived (`archived=true`, last push 2026-04-24) and the
licence is AGPL-3.0. Do not start a new deployment on it. Use it only as an API
compatibility reference.

### 6.4 Garage

AGPL-3.0, 4,784 stars, active. It is a good small geo-distributed store, but it
does not aim for R2 parity: no versioning, no SSE, no object ACLs, and
replication rather than erasure coding. Use it for a home lab where simplicity
matters more than the S3 surface.

### 6.5 Ceph

The right answer if you already run Ceph and have the staff. RGW gives S3, and
the same cluster gives block and file. It is too heavy for a one-machine site and
its licence is a mix that needs review.

## 7. CI

| Project | Licence | Stars | Fit | Verdict |
| --- | --- | --- | --- | --- |
| Fireactions | Apache-2.0 | 200 | GitHub Actions on Firecracker | Use for GitHub |
| Woodpecker | Apache-2.0 | 8,067 | Gitea/Forgejo CI | Use for Gitea/Forgejo |
| actions-runner-controller | Apache-2.0 | 6,623 | GitHub runners on Kubernetes | Use if k8s |
| GitHub Actions runner | MIT | 6,364 | The runner itself | Component |
| Buildkite agent | MIT | 1,101 | Agent/job API design | Reference |
| GitLab Runner | MIT | 248 (mirror) | GitLab CI | Only if GitLab |
| Tekton | Apache-2.0 | 9,094 | Kubernetes-native pipelines | Heavy |
| Argo Workflows | Apache-2.0 | 17,067 | Kubernetes workflows | Heavy |
| Dagger | Apache-2.0 | 16,544 | Reproducible build steps | Component |
| Jenkins | MIT | 26,702 | General CI | Legacy |
| Earthly | MPL-2.0 | 12,054 | Build automation | Niche |
| Harness | Apache-2.0 | 38,757 | Platform | Too heavy |
| act | MIT | 72,478 | Run GitHub Actions locally | Testing only |
| runs-on | MIT | 1,389 | Self-hosted GitHub runners on AWS | Reference |

Metadata verified.

### 7.1 Fireactions

Claimed: BYOM GitHub runners in ephemeral Firecracker VMs, JIT runner tokens,
pool scaling on `workflow_job` events, and VM destruction after each job. It is
GitHub-specific by design. Feature relayed.

This is the shortest path to replacing Workers CI for a GitHub-heavy project. It
does not need the CFB control plane to run, which is a feature: the control plane
observes and configures, but a Fireactions outage does not stop CI if the pool is
already warm.

### 7.2 Woodpecker

Claimed: an Apache-2.0 server and agent, a gRPC boundary, a Kubernetes backend
and an autoscaler. Agents are small (roughly 100 MB server, 30 MB agent idle,
from the project's own numbers; relayed). Feature relayed.

The autoscaler is not feature-complete for every backend. For a small fleet,
static agents with ephemeral Incus containers per job are simpler and easier to
debug.

### 7.3 What not to build

Do not write a CI engine. The pipeline language, the plugin ecosystem, the
secret handling, the cache semantics and the UI are years of work. Integrate a
runner and own the scheduling, secrets and observability around it.

## 8. Connectivity and overlay

| Project | Licence | Stars | Role | Verdict |
| --- | --- | --- | --- | --- |
| cloudflared | Apache-2.0 | 16,337 | Tunnel client | Use |
| Pangolin | NOASSERTION (AGPL + commercial) | 23,263 | Self-hosted tunnel/reverse proxy | Reference |
| NetBird | NOASSERTION (BSD core + AGPL mgmt) | 30,316 | WireGuard overlay | Alternative |
| Headscale | BSD-3-Clause | 45,164 | Tailscale control server | Single-tailnet alternative |
| Tailscale | BSD-3-Clause | 38,286 | Mesh VPN | Hosted, not self-hosted |
| frp | Apache-2.0 | 110,283 | Reverse proxy tunnel | Simple alternative |
| rathole | Apache-2.0 | 14,382 | Reverse proxy tunnel | Simple alternative |
| boringproxy | MIT | 1,388 | Tunnel + UI, stale | Avoid |
| Nebula | MIT | 18,481 | Overlay network | Alternative |
| ZeroTier | NOASSERTION (MPL core + BSL nonfree) | 17,221 | Overlay network | Licence risk |
| Teleport | AGPL-3.0 | 21,044 | Access plane | Licence risk |
| OpenZiti | Apache-2.0 | 4,458 | Zero-trust network | Alternative |
| Octelium | AGPL-3.0 | 4,134 | Zero-trust/ZTNA/PaaS | Licence risk |
| WireGuard tools | GPL-2.0 | 733 | VPN primitive | Kernel/user tools |

Metadata verified.

### 8.1 Why Cloudflare Tunnel stays

The user's constraint is that the control plane lives on Cloudflare. Tunnel gives
an outbound-only connection that terminates in the same account as the Worker and
the Access policy. It removes inbound firewall rules, it is free for 1,000
tunnels, and the account-wide `cf1:network` binding lets a Worker reach the machine
without a per-machine route deployment, at the cost of account-wide reach.

The alternatives are for the fallback and for sites where Cloudflare is not
allowed:

- **frp or rathole**: a self-hosted relay with a public IP. Simple, but you now
  operate the relay and its TLS.
- **Headscale**: a self-hosted Tailscale control server. Good for a private mesh,
  but one tailnet and a different identity model.
- **NetBird**: a WireGuard overlay with a management plane; the management plane
  is AGPL, which matters if you distribute.
- **Pangolin**: a self-hosted tunnel and reverse proxy with a UI. Its licence is
  a mix of AGPL and commercial terms; review before embedding.
- **Octelium**: a zero-trust platform that overlaps with Access and Tunnel. AGPL.

The design keeps a `PrivateTransport` interface so the tunnel can be replaced
without touching the rest of the system. That is cheaper than betting on one
transport.

## 9. Control-plane and PaaS references

| Project | Licence | Stars | What to learn | Verdict |
| --- | --- | --- | --- | --- |
| Coolify | Apache-2.0 | 65,237 | Self-hosted PaaS UX, deployments, backups | Reference |
| Dokploy | NOASSERTION | 38,109 | PaaS UX, templates, Docker | Reference |
| CapRover | NOASSERTION | 15,195 | One-click apps, nginx | Reference |
| Dokku | MIT | 32,208 | Git-push deploys | Reference |
| Kamal | MIT | 14,697 | Container deploys over SSH | Reference |
| Portainer | Zlib | 38,744 | Edge agents, RBAC, UI | Reference |
| Cosmos-Server | NOASSERTION | 6,187 | Self-hosted app server | Niche |
| Cloudflare Computer | MIT | 9,752 | Workspace abstraction | Reference |
| Cloudflare sandbox-sdk | NOASSERTION | 1,156 | Sandbox API | Reference |

Metadata verified.

Coolify is the best UX reference for "one dashboard, many machines, deploy
things". It uses SSH for orchestration, which CFB explicitly rejects, but its
information architecture, backup flow and status model are worth studying.
Dokploy and CapRover have similar lessons and less permissive licensing.

Kamal is the best reference for a small, explicit deploy tool. Its "no
Kubernetes, just SSH and Docker" stance is the opposite of CFB's "no SSH, a
reconciling agent" stance, and the contrast is useful.

Portainer's edge agent and its RBAC model are the closest public analogue to the
CFB control-plane split: a central UI, an outbound agent, and scoped permissions.

## 10. Durable execution and workflows

| Project | Licence | Stars | Role | Verdict |
| --- | --- | --- | --- | --- |
| Temporal | MIT | 24,058 | Durable execution | Strong alternative |
| DBOS | MIT | 1,419 | Durable execution in Postgres | Alternative |
| Trigger.dev | Apache-2.0 | 16,650 | Background jobs | Alternative |
| Hatchet | MIT | 8,197 | Task queue and workflows | Alternative |
| Restate | NOASSERTION (BSL reported) | 4,621 | Durable execution | Licence risk |
| Inngest | NOASSERTION (SSPL reported) | 6,001 | Event workflows | Licence risk |
| Windmill | NOASSERTION (AGPL + proprietary reported) | 18,313 | Script workflows | Licence risk |
| Orleans | MIT | 10,918 | Virtual actors | Reference |
| Dapr | Apache-2.0 | 26,169 | Distributed app runtime | Reference |
| Pekko | Apache-2.0 | 1,659 | Actors | Reference |

Metadata verified. Licence descriptions marked "reported" come from the
project's licence files as read by the research pass, not from the GitHub API
(which returns NOASSERTION for a mixed or custom licence). Verify with legal
before shipping.

CFB does not need a durable-execution engine in v1. Cloudflare Workflows covers
the control-plane sagas. If a tenant needs durable execution, offer Temporal or
DBOS as a managed workload on the data plane, not as a control-plane dependency.
This keeps the licence surface clean and the control plane small.

## 11. Licence traps

| Project | Licence | Why it matters |
| --- | --- | --- |
| MinIO | AGPL-3.0 | Archived; AGPL network clause if offered as a service |
| LXD | AGPL-3.0 | Incus is the Apache-2.0 fork |
| Nomad | BUSL-1.1 (verified in LICENSE; API returns NOASSERTION) | Not open source for a competing product |
| Coder | AGPL-3.0 | Network clause |
| Garage | AGPL-3.0 | Network clause |
| Teleport | AGPL-3.0 | Network clause |
| Octelium | AGPL-3.0 | Network clause |
| Restate | BSL 1.1 (verified in LICENSE) | Competing-service restriction |
| Inngest | SSPL 1.0 (verified in LICENSE.md) | Service restriction |
| Windmill | Apache-2.0 + AGPL-3.0 + proprietary (verified in LICENSE) | Mixed |
| ZeroTier | MPL-2.0 core + BSL nonfree (verified in repo) | Mixed; the service and nonfree portions are not open source |
| Dokploy | NOASSERTION | Mixed; review |
| Pangolin | NOASSERTION | Fossorial commercial licence plus other files (verified in LICENSE) |
| NetBird | NOASSERTION | BSD-3-Clause core, AGPL-3.0 management/signal/relay (verified in LICENSE) |
| Proxmox | NONE in the mirror; AGPL-3.0 upstream | Mixed |
| Ceph | NOASSERTION | LGPL/GPL mix |

The rule for CFB: a component that is merely *used* at arm's length can be
AGPL, but a component that is *linked, modified or embedded* in a distributed
product should be Apache-2.0 or MIT. The control plane is distributed as a
Workers bundle; the agent is distributed as a binary. Keep copyleft components on
the data plane, behind a process boundary, and document the obligation.

## 12. Build versus buy

| Capability | Build | Buy or reuse | Verdict |
| --- | --- | --- | --- |
| Container/VM lifecycle | High effort, low differentiation | Incus | Reuse |
| MicroVM lifecycle | High effort | Firecracker + agent driver | Reuse primitive, build driver |
| Workers runtime | Very high effort | workerd | Reuse |
| Workers platform | High effort | open-compute | Reuse with a fallback |
| Distributed DO | Very high effort | celld | Reuse, accept pre-1.0 |
| S3 | High effort | SeaweedFS | Reuse |
| CI engine | Very high effort | Woodpecker / Fireactions | Reuse |
| Tunnel | High effort | cloudflared + VPC | Reuse |
| Control plane | The actual product | None fits | Build |
| Host agent | The actual product | None fits | Build |
| Dashboard | The actual product | None fits | Build |
| Placement and leases | The actual product | None fits | Build |
| Audit and policy | The actual product | None fits | Build |

The control plane, the agent, the data model and the policy engine are where the
value is. Everything below them should be a replaceable dependency.

## 13. Gaps in the prior art

These are the things no public project does well, and therefore where CFB has to
do real work:

1. **A multi-tenant control plane that is native to Cloudflare.** amika's is
   closed. Coolify and Dokploy are SSH-based. Portainer is Docker-centric.
2. **A host agent that reconciles both Incus and Firecracker and open-compute.**
   Each project has its own agent; none spans the classes.
3. **Leases and fencing across a control plane and a BYOC daemon.** amika has a
   tunnel protocol; it does not publish the lease model.
4. **A single audit chain across control-plane and data-plane actions.** Every
   project has logs; none has a verifiable chain that spans both planes.
5. **Cost attribution per tenant per machine.** Cloud providers do this; self-hosted
   platforms mostly do not.
6. **A tested restore path from local S3 to R2 and back.** Most projects document
   backup; few test restore.
7. **A clear isolation matrix.** Most self-hosted PaaS products treat "container"
   as a security boundary. It is not.

## 14. What was not covered

- No project was installed, built or benchmarked for this document. Every
  feature claim is from documentation.
- Star counts and licences are a snapshot; they change.
- The GitHub search was not exhaustive. It covered the categories above and the
  obvious names in each. Projects with fewer than roughly 50 stars were included
  only when they were directly relevant (WDL, groundflare, celld-operator,
  OpenClawMachines).
- Non-GitHub sources (vendor docs, papers, blog posts) are out of scope here;
  Cloudflare's own docs are cited in `docs/spec.md` and `docs/cost.md`.
- Licence texts were not read line by line for every repository. The
  NOASSERTION and "reported" rows need a legal read before they become
  dependencies.

Metadata source: `evidence/repos.tsv`, captured 2026-10-29 with
`gh api repos/OWNER/REPO`. README captures: `evidence/reads/`.
