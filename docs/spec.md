# CFB specification

Status: design draft, 2026-10-29. No implementation is claimed or included.
Working name: CFB (Cloudflare Backplane). Rename freely.

## 1. Summary

CFB is a two-plane system.

The **control plane** is a Cloudflare Workers application. It owns identity,
inventory, desired state, policy, audit, the dashboard and the public API. It
uses D1 for relational metadata, R2 for artifacts, one Durable Object per machine
for leases and live state, Queues for command fanout, Workflows for long-running
sagas, Access for human authentication, and Cloudflare Tunnel plus Workers VPC
for private connectivity.

The **data plane** is a set of machines you own. Each machine runs a single
outbound-only host agent, `cloudflared`, and one or more local runtimes: Incus or
containerd for containers and VMs, Firecracker for sandboxes, open-compute or
workerd for Workers-compatible code, SeaweedFS or RustFS for S3, and Woodpecker
or Fireactions for CI.

The control plane stores intent. The data plane stores execution. The data plane
keeps running when the control plane is unavailable. The control plane is never
in the request path of a tenant workload, except for optional public ingress
through Cloudflare Tunnel and Access.

The design is a smaller, narrower system than Cloudflare. It does not aim for a
global anycast edge, 100,000-container bursts, or 648 ms sandbox starts. It
aims for a manageable fleet of machines, with predictable costs, strong local
performance, and a dashboard that works from anywhere.

## 2. Problem and constraints

### 2.1 What is being replaced

| Cloudflare service | Why it becomes expensive or limiting | CFB replacement |
| --- | --- | --- |
| R2 | Storage is cheap, but capacity is rented forever; locality and egress patterns vary | Local S3 (SeaweedFS or RustFS), with R2 as cold tier and offsite backup |
| Containers | Billed per 10 ms of running time; egress charged; instance sizes capped by default and raised on request; HTTP-only ingress; no raw TCP | Incus or containerd on owned machines, with Tunnel ingress and VPC control |
| Sandbox | Billed per 10 ms while running, 6-hour inactivity cap, rootfs-only snapshots, no raw TCP | Firecracker microVMs with snapshots and a small sandbox API |
| Computer | Early preview library, Cloudflare-hosted; a workspace over a container/isolate | A local workspace service over a sandbox or container, with a durable filesystem |
| Workers | CPU and request billing, runtime limits, no long-running processes | open-compute or workerd on a machine, exposed through Tunnel |
| Durable Objects | Request and duration billing, 10 GB per object, single-threaded | open-compute DO for single-node, celld or Rivet for distributed actors |
| Workers CI | Build minutes, 20-minute timeout, 6 concurrent builds, Cloudflare-hosted | Woodpecker or Fireactions on owned runners, with artifacts in local S3 |
| Workers VPC / Tunnel | Free at small scale, but account limits and per-tunnel pricing beyond 1,000 | Tunnel per machine plus an account-wide VPC binding; a relay fallback; optional self-hosted overlay |

The prices and limits above are Cloudflare's list values read on 2026-10-29. The
sources are in section 21 and in `docs/cost.md`.

### 2.2 Hard constraints

1. The dashboard and manager must be hosted on Cloudflare and usable from there.
2. Machines are behind NAT or a firewall. No inbound ports can be opened.
3. The control plane must not be a single point of failure for running workloads.
4. The system must run long-lived containers, heavy CI jobs, sandboxes and
   Workers-compatible code on the same fleet.
5. Multi-tenant isolation must be explicit, not implied by a namespace.
6. The design must be operable by a small team, not a 24/7 platform organization.
7. Licensing must be compatible with self-hosting and, if the system is offered
   to others, with a hosted service.

### 2.3 Soft constraints

- Prefer Apache-2.0 or MIT components.
- Prefer one binary or one daemon per machine over a service mesh.
- Prefer local consistency over global coordination.
- Prefer boring, replaceable components over novel ones.
- Keep the control plane cheap; spend the money on machines and bandwidth.

## 3. Goals and non-goals

### 3.1 Goals

- Provision a new machine into the fleet from the dashboard, with no SSH.
- Create, start, stop, snapshot, migrate and delete containers, VMs, sandboxes,
  Workers and CI runners from the dashboard.
- Keep workloads running through a control-plane outage.
- Reach every machine through an outbound-only connection.
- Isolate untrusted code with a VM boundary.
- Store tenant objects in local S3, with an offsite copy in R2.
- Give every mutation an actor, a request id and an audit record.
- Make cost visible per tenant, machine and workload.
- Allow a component to be replaced without rewriting the control plane.

### 3.2 Non-goals

- Global anycast or a multi-region active-active control plane.
- Exact Cloudflare API compatibility.
- A hypervisor, an S3 server, a CI engine, a Durable Object runtime or a
  container runtime written from scratch.
- 100,000-instance bursts or sub-second cold starts.
- A billing marketplace in the first version.
- Windows guests in the first version.
- Running hostile multi-tenant code in a shared-kernel container.

## 4. Design principles

1. **Split the planes, invert the dependency.** The control plane describes what
   should exist. The data plane decides how to make it exist and keeps doing so
   when the control plane is unreachable.
2. **Outbound only.** Machines initiate every connection. `cloudflared` is the
   only ingress path. No public SSH, no public agent port, no public database.
3. **Desired state, not remote shell.** There is no SSH-based provisioning path.
   The agent reconciles a versioned specification and reports observed state.
4. **One authority per fact.** D1 owns inventory. The per-machine DO owns the
   lease. The local SQLite DB owns runtime state. Local S3 owns object bytes. R2
   owns the offsite copy. No fact is written in two places as a primary.
5. **Isolation by workload class.** Untrusted code gets a microVM. Trusted code
   gets a container. High-trust code gets a dedicated machine. workerd runs
   inside a VM or container because it is not a hardened sandbox.
6. **Replaceable components.** Incus, Firecracker, open-compute, celld, SeaweedFS
   and Woodpecker sit behind driver interfaces. A better component should not
   require a control-plane rewrite.
7. **Fail closed, audit everything.** Deny by default, use short-lived
   credentials, sign artifacts, and record every mutation in a hash chain.
8. **Cost honesty.** Self-hosting is not free. The design makes idle cost,
   redundancy cost and operator time visible instead of hiding them.

## 5. Architecture

### 5.1 Component diagram

```
+------------------------------- Cloudflare account --------------------------------+
|                                                                                    |
|  Browser                                                                           |
|    |  HTTPS + Access cookie                                                         |
|    v                                                                               |
|  +------------------+      +----------------------+      +----------------------+  |
|  | console Worker   |----->| api Worker           |----->| D1 (inventory,       |  |
|  | SPA + /api/*     |      | Hono, OpenAPI 3.1    |      | desired state, audit)|  |
|  | Access JWT check |      | RBAC, idempotency    |      +----------------------+  |
|  +------------------+      +----------+-----------+                               |
|                                      |                                            |
|                    +-----------------+-----------------+                          |
|                    |                 |                 |                          |
|                    v                 v                 v                          |
|          +------------------+  +-------------+  +------------------+              |
|          | DO per machine   |  | Queues      |  | Workflows        |              |
|          | lease, fencing,  |  | commands,   |  | enroll, upgrade, |              |
|          | live events      |  | events      |  | decommission     |              |
|          +--------+---------+  +-------------+  +------------------+              |
|                   |                                                                |
|                   | Workers VPC binding: cf1:network                                |
|                   v                                                                |
|          +------------------+        +------------------------------+              |
|          | Cloudflare       |        | R2 (artifacts, logs,         |              |
|          | Tunnel per host  |        | snapshots, backups)          |              |
|          +--------+---------+        +------------------------------+              |
+-------------------|----------------------------------------------------------------+
                    | outbound QUIC 7844
                    v
+-------------------|------------------------- Machine / site ------------------------+
|  cloudflared (one per machine)                                                    |
|      |                                                                            |
|      v                                                                            |
|  +------------------+     +---------------------------------------------------+    |
|  | host agent        |---->| local runtimes                                    |    |
|  | reconciler        |     |  Incus: containers and VMs                        |    |
|  | local SQLite      |     |  containerd: OCI containers                       |    |
|  | drivers           |     |  Firecracker / bhatti / hyper: sandboxes          |    |
|  | metrics, logs     |     |  open-compute / workerd: Workers and DO           |    |
|  +------------------+     |  celld / Rivet: distributed actors                |    |
|                           |  SeaweedFS / RustFS: S3                           |    |
|                           |  Woodpecker / Fireactions: CI runners             |    |
|                           +---------------------------------------------------+    |
|                                                                                    |
|  local state: SQLite, images, snapshots, logs, S3 data                            |
+------------------------------------------------------------------------------------+
```

### 5.2 Planes and trust boundaries

| Plane | Trust boundary | Authority | Failure behavior |
| --- | --- | --- | --- |
| Browser | Untrusted | None; holds an Access session | Dashboard unavailable |
| Control plane | Cloudflare account, operator-controlled | Inventory, desired state, policy, audit | Data plane continues; writes queue or fail |
| Transport | Cloudflare Tunnel and VPC | Route and identity | Private control path lost; public ingress may continue if the tunnel is up |
| Machine | Operator-owned hardware | Local execution state | Workloads continue; control plane marks machine offline |
| Tenant workload | Untrusted or semi-trusted | None over the host | Isolated by VM or container; cannot reach the agent |

The critical trust rule: **a tenant workload must never be able to reach the host
agent, the local SQLite state, the cloudflared token, the machine identity key,
or another tenant's runtime.** That is a design requirement, not a hardening
task.

### 5.3 Request flows

**Create a container**

1. Browser sends `POST /api/v1/workloads` with an `Idempotency-Key`.
2. API validates the tenant, quota, image signature and placement labels.
3. API writes the workload row in D1 with `generation=1` and `desired=running`.
4. API asks the candidate machine's DO to reserve capacity and issue a lease.
5. DO increments `lease_epoch` and returns a signed reservation.
6. API writes the reservation and enqueues `reconcile` with the workload id and
   generation.
7. The agent receives the command over VPC, checks the lease epoch, pulls the
   image, creates the Incus instance, starts it, and reports observed state.
8. The DO updates the live state and fans out an event to the SPA. D1 is updated
   asynchronously from the event, not as a second primary.
9. The API returns `202` with an operation id. The SPA follows the operation over
   SSE.

**Run a CI job**

1. A Git provider webhook hits `POST /api/v1/ci/hooks/{provider}`.
2. API verifies the webhook signature, maps the repo to a tenant and a pool.
3. API creates a job row and asks the pool's DO for a runner slot.
4. The agent starts an ephemeral Incus container or Firecracker VM from a warm
   image, injects a short-lived job token, mounts a cache volume, and starts the
   runner.
5. Logs stream to the agent, then to R2 in chunks; the SPA tails them over SSE.
6. Artifacts are written to local S3; a manifest is written to D1.
7. On exit, the agent destroys the runner, uploads the final log, and reports the
   result. The control plane stores the job's secrets envelope-encrypted and
   never holds them in plaintext.

**Deploy a Worker**

1. CI builds a bundle and uploads it to R2 with a content hash and a signature.
2. API creates a deployment row and enqueues `deploy` to the target machine.
3. The agent verifies the signature and hash, writes the bundle to
   `/var/lib/cfb/bundles/<sha256>`, and calls the open-compute API or swaps a
   workerd config symlink.
4. The agent runs a health check against the local route.
5. On success, the route pointer moves atomically. On failure, the previous
   pointer stays.

## 6. Control plane

### 6.1 Cloudflare product choices

| Need | Product | Why | Constraint |
| --- | --- | --- | --- |
| SPA and API | Workers static assets + Hono | One deploy, one origin, free static requests | Worker invocations are billed; keep navigation off the Worker |
| Relational metadata | D1 | SQL, batch transactions, cheap | 10 GB per database, no interactive transactions, single primary |
| Object store | R2 | Cheap, no egress fees, strong read-after-write | 5 TiB per object, 1 write/s per key |
| Per-machine coordination | Durable Objects (SQLite) | Single writer, alarms, WebSocket fanout | 10 GB per object, soft 1,000 req/s per object |
| Command fanout | Queues | Decoupling, retries, DLQ | At-least-once, unordered |
| Long-running sagas | Workflows | Durable steps, sleeps, retries | Steps must be idempotent; 10,000 steps default, up to 25,000 |
| Human auth | Access | OIDC, policy, audit | 500 apps, 50 service tokens per account |
| Private transport | Tunnel + Workers VPC | Outbound-only, account-scoped | VPC is beta; 1,000 tunnels free, then $4.50 each |
| Scheduled jobs | Cron + DO alarms | Coarse dispatch plus per-resource timers | Cron resolution 1 minute; one alarm per DO |
| Observability | Workers Logs, Analytics Engine, Logpush | Native | Not a full APM; export to the data plane if needed |

### 6.2 Modules

**`console` Worker**

- Serves the Vite SPA from static assets with SPA fallback.
- Serves `/api/*` from the same origin. This avoids CORS and keeps the Access
  cookie on one origin.
- Validates `Cf-Access-Jwt-Assertion` on `/api/*` against the team JWKS, checking
  issuer and audience. Static asset requests do not invoke the Worker; Access
  authorizes them at the edge. It does not trust `ctx.access` across service
  bindings because the documentation says the context does not survive them.
- Applies CSP, HSTS, frame denial, CSRF tokens for state-changing browser calls,
  and per-user rate limits.
- Holds no domain logic beyond auth and request shaping.

**`api` Worker**

- Owns the REST API, OpenAPI 3.1 document and domain services.
- Writes D1 mutations and their audit rows in one `batch()` so the mutation and
  the audit record commit together.
- Uses `Idempotency-Key` for POST, `If-Match`/`generation` for optimistic
  concurrency, and RFC 9457 `application/problem+json` for errors.
- Emits an outbox row in the same batch as each state change. A queue consumer
  publishes the event. This avoids a dual write between D1 and Queues.
- Never calls a machine directly for a long operation. It enqueues a command and
  returns `202` with an operation id.

**`signer` Worker**

- Holds the per-tenant storage credentials in Secrets Store.
- Mints short-lived presigned URLs after an authorization check from the API.
- Never returns a credential to the API, the SPA or a tenant.
- Has no public route; it is reachable only through a service binding.
- Records every mint in the audit chain.

**`coordinator` Durable Object (one per machine)**

- Holds `machine_id`, `lease_epoch`, `agent_key_id`, `last_seen`,
  `desired_digest`, `observed_digest`, `pending_commands` and live subscribers.
- Serializes reservations and command delivery for that machine.
- Rejects a command whose `lease_epoch` is older than the current epoch.
- Runs a lease alarm. If no heartbeat arrives before expiry, it marks the machine
  offline and emits an event.
- Fans out state changes to the SPA over WebSocket using the hibernation API.

**`relay` Durable Object (fallback transport)**

- Accepts an outbound WebSocket from an agent when Workers VPC is unavailable or
  unsuitable.
- Authenticates the machine's signed handshake against its pinned public key,
  maps the connection to a machine id, and routes control-plane requests over
  the socket.
- Uses credit-based flow control for log and exec streams. The agent never sends
  `OPEN`; the relay opens streams. This prevents a compromised agent from
  initiating arbitrary streams.
- Is not the default path. It exists so the design does not depend on a beta VPC
  feature.

**`provisioner` Workflow**

- `enroll_machine`: validate the one-time join token, create the machine row,
  create a tunnel, fetch the tunnel token, render the install command, wait for
  the first heartbeat.
- `join_site`: add the machine to an Incus cluster or a site pool, apply labels,
  run a smoke test.
- `upgrade_agent`: drain, download the signed agent, verify, restart, health
  check, roll back on failure.
- `decommission`: drain, migrate or stop workloads, delete the tunnel, revoke the
  machine credential and key, delete local keys, mark deleted.
- Every step is idempotent and keyed by the operation id.

**`janitor` Cron**

- Expires stale leases and operations.
- Deletes expired snapshots and orphaned R2 objects.
- Rotates tunnel tokens and agent credentials before expiry.
- Detects drift between desired and observed state and re-enqueues reconcile.
- Enforces quotas and emits usage summaries.

**`notifier`**

- Sends alerts to email, Discord or Slack. Alert content is rendered from D1 and
  never includes secrets.

### 6.3 State ownership and consistency

| Fact | Primary | Projection | Consistency |
| --- | --- | --- | --- |
| Tenant, user, role | D1 | SPA cache | Read-after-write with `first-primary` session |
| Machine inventory | D1 | DO, SPA | Eventual; DO is live, D1 is durable |
| Lease and fencing epoch | DO per machine | D1 audit | Strong within one object |
| Desired workload spec | D1 | Agent local SQLite | Eventual; generation makes it idempotent |
| Observed workload state | Agent local SQLite | DO, then D1 | Eventual; last writer wins with generation |
| Object bytes | Local S3 | R2 replica | Async, checksum-verified |
| Audit log | D1 hash chain | R2 WORM export | Append-only; verified on export |
| Job logs | Local disk then R2 | D1 index | Chunked, checksummed |
| Metrics | Agent buffer | DO aggregate, then D1/R2 | Sampled; not a billing source |

Rules:

- D1 is not a message queue. Every state change also writes an outbox row.
- A DO is not a database for inventory. It holds only the per-machine hot state
  and a bounded command queue.
- R2 is not a mutable state store. Objects are immutable and content-addressed;
  the pointer lives in D1.
- The agent's local SQLite is authoritative for what is actually running. The
  control plane must reconcile to it, not overwrite it blindly.

### 6.4 Dashboard

- Vite + React or Svelte SPA, built to static assets. No SSR, so page views do
  not invoke the Worker.
- Views: overview, sites, machines, workloads, storage, CI, routes, operations,
  audit, usage, settings.
- Live updates over a single WebSocket to the coordinator or an SSE stream from
  the API. The SPA reconnects with a cursor.
- Destructive actions require typing the resource name and, for production
  resources, a second approver. The approval is a Workflow step, not a UI flag.
- Every list view has a "why is this here" panel showing the desired state,
  observed state, last reconcile, lease epoch and recent events.
- The dashboard is an operator tool, not a tenant portal, in v1. A tenant portal
  can be added later with the same API and narrower scopes.

### 6.5 Control-plane deployment

- `wrangler versions upload` then `wrangler versions deploy`, tagged with the git
  SHA. Promotion is a separate auditable action.
- D1 migrations are expand/contract and forward-only. A rollback restores code,
  not data, so every migration must be compatible with the previous code.
- Secrets are Workers secrets or Secrets Store bindings. They are never in D1,
  R2, logs or the SPA.
- The control plane is deployed to one Cloudflare account. Multi-account is a
  later feature; the tunnel and VPC model is account-scoped.

## 7. Data plane

### 7.1 Host agent

One static binary per machine, installed as a systemd service. It runs as an
unprivileged user. A small privileged helper, or a set of specific sudo rules,
handles only the operations that need root: network namespace setup, mounting
volumes, creating Incus instances, and reading the machine identity key. The
helper exposes a narrow local API, not a shell.

Responsibilities:

- Enroll once with a one-time token, then authenticate with a machine key and a
  short-lived credential.
- Maintain an outbound connection through `cloudflared`; the agent listens only
  on loopback.
- Reconcile desired state from a local SQLite cache and the control plane.
- Drive local runtimes through driver interfaces.
- Report heartbeats, observed state, events, metrics and logs.
- Enforce local policy: resource limits, network rules, image signature checks.
- Survive control-plane outages and restart safely.

Local state:

- `/var/lib/cfb/state.db`: SQLite with desired specs, observed state, operation
  log, lease epoch, and a bounded event queue.
- `/var/lib/cfb/bundles/<sha256>`: immutable Worker bundles.
- `/var/lib/cfb/images/<sha256>`: image manifests and content-addressed layers.
- `/var/lib/cfb/snapshots/<id>`: local snapshots, with a retention policy.
- `/var/lib/cfb/logs/<workload>/<run>`: local log segments before upload.
- `/etc/cfb/agent.toml`: non-secret configuration.
- `/etc/cfb/identity/`: machine credential and key, mode 0600, preferably
  backed by a TPM.

### 7.2 Runtimes and drivers

| Driver | Runtime | Workload type | Notes |
| --- | --- | --- | --- |
| `incus` | Incus | container, vm | REST API over a local Unix socket; projects for tenant separation; clustering for a site |
| `containerd` | containerd + runc | container | OCI-native; good for CI runners and simple services |
| `firecracker` | Firecracker | sandbox | One microVM per sandbox; jailer; vsock; snapshot/restore |
| `bhatti` | bhatti (libkrucible) | sandbox | Higher-level microVM orchestrator; Linux and macOS; pilot |
| `open-compute` | open-compute (`ocd`) | worker, durable | Workers-compatible platform on one machine; SQLite and optional S3 |
| `workerd` | workerd | worker | Raw runtime; needs a supervisor and bindings; run inside a VM |
| `celld` | celld | durable | Distributed Durable Objects; one application per fleet; pre-1.0 |
| `rivet` | Rivet | durable | Actor platform with its own control plane; alternative to celld |
| `seaweedfs` | SeaweedFS | storage | S3, erasure coding, replication, IAM/STS |
| `rustfs` | RustFS | storage | S3, erasure coding, site replication; younger |
| `woodpecker` | Woodpecker agent | ci | Gitea/Forgejo-style pipelines; ephemeral runners |
| `fireactions` | Fireactions | ci | GitHub Actions runners in Firecracker VMs |

Each driver implements the same small interface:

```
type Driver interface {
    Capabilities(ctx) (Capabilities, error)
    Reconcile(ctx, desired Workload, lease Lease) (Observed, error)
    Start(ctx, id) error
    Stop(ctx, id, grace) error
    Destroy(ctx, id) error
    Snapshot(ctx, id) (SnapshotID, error)
    Restore(ctx, id, snapshot) error
    Exec(ctx, id, cmd) (Stream, error)
    Logs(ctx, id, since) (Stream, error)
    Stats(ctx, id) (Stats, error)
}
```

The driver is responsible for mapping a portable workload spec to the runtime's
native objects and for reporting drift. The control plane never knows an Incus
container name or a Firecracker socket path.

### 7.3 Site model

A **site** is a trust and failure domain: a rack, a home, a colo cage, or a cloud
region. A site has one network boundary, one storage policy, and one latency
budget. Machines belong to exactly one site.

Within a site:

- **Standalone mode**: each machine runs its own agent and runtimes. No cluster.
  This is the default for home labs and small sites.
- **Incus cluster mode**: three or more machines form an Incus cluster with a
  distributed database. The agent talks to the local Incus daemon; the cluster
  handles placement and migration. Use this for a rack with reliable networking.
- **Kubernetes mode**: optional. If the site already runs k3s or Kubernetes, the
  agent can use it as the container runtime, and Fireactions or Actions Runner
  Controller can manage CI runners. This is an escape hatch, not the default.

The control plane does not require a cluster. It works with standalone machines
and treats a cluster as one placement target with multiple machines.

### 7.4 Provisioning a machine

1. Operator creates a machine record in the dashboard. The API returns a one-time
   join token, a tunnel token and an install command.
2. Operator boots the machine with a base image (Ubuntu, Debian, Talos, or a
   custom image) and runs the install command. For bare metal, cloud-init or a
   USB image can carry the same command.
3. The installer creates the `cfb` user, installs `cloudflared` and the agent,
   writes the tokens to systemd credentials, and starts both services.
4. The agent enrolls: it presents the join token, proves possession of its key,
   and receives a short-lived credential.
5. The provisioner creates the tunnel route, waits for the first heartbeat, runs
   a smoke test (create a network namespace, pull a tiny image, run `true`), and
   marks the machine `ready`.
6. The machine appears in the dashboard with its capabilities, labels and
   capacity.

For racks with BMCs, MAAS or Tinkerbell can automate step 2. For a home lab,
cloud-init on a USB image is enough. Do not build a PXE stack for one machine.

## 8. Connectivity

### 8.1 One tunnel per machine

Each machine gets one remotely managed Cloudflare Tunnel:

- Name: `cfb-<site>-<machine>`.
- `config_src=cloudflare`.
- A hostname route for the agent, for example
  `machine-<id>.internal.<domain>`, and optional hostname routes for public
  workloads.
- A token delivered once to the machine, stored in a systemd credential, never
  in a file readable by workloads.

Why one tunnel per machine:

- One revocable credential per machine.
- One route set per machine.
- A compromised machine can be cut off by deleting its tunnel and its
  connections.
- It fits the 1,000-tunnel free tier. Beyond that, Tunnels for Platforms costs
  $4.50 per additional tunnel per month, and inactive tunnels count.

Do not use one tunnel for the whole fleet. A single tunnel shared by machines
makes revocation and blast-radius control much harder.

### 8.2 Control path

The control plane binds the account-wide network once:

```jsonc
{
  "vpc_networks": [
    { "binding": "FLEET", "network_id": "cf1:network" }
  ]
}
```

Then a Worker calls:

```ts
const res = await env.FLEET.fetch(
  `https://machine-${id}.internal:9443/v1/reconcile`,
  { method: "POST", body: JSON.stringify(command), headers: authHeaders }
);
```

`cf1:network` is the Cloudflare Mesh binding. It reaches every Mesh node,
subnet route and hostname route announced through Tunnel or Mesh in the account,
and public Internet destinations through Gateway. It is account-wide, not
scoped to one tunnel, and it requires at least one active Tunnel, Mesh node or
WAN on-ramp in the account. Adding a machine does not require a Worker redeploy.
That convenience is a deliberate blast-radius tradeoff: a Gateway policy denies
public Internet destinations for Workers VPC traffic in this account, and all
traffic is logged. Scope the policy to this binding if the Gateway model allows
it; verify that in Phase 0.
The narrower alternative is one `tunnel_id` binding per machine. It scopes reach
to one tunnel, but each new machine needs a Worker configuration change. Use it
for small or high-value fleets. Workers VPC is a beta feature and is free on all
Workers plans as of 2026-10-29.

If VPC is unavailable, a feature flag switches the transport to the relay:

- The agent opens `wss://relay.<domain>/agent` and signs the handshake with its
  machine key.
- The relay DO verifies the signature against the pinned key, maps the
  connection, and exposes the same request/response and stream semantics.
- The agent initiates the connection; the control plane never dials the machine.

The transport is behind a `PrivateTransport` interface. The rest of the system
does not know which one is in use.

### 8.3 Public ingress

- A public workload gets a hostname route on the machine's tunnel.
- The route points to a local address, for example `127.0.0.1:8080` or an Incus
  instance's proxy address.
- Access protects private applications. Public applications get WAF, rate limits
  and a request-size limit.
- TLS terminates at Cloudflare. The origin can use HTTP on loopback, or TLS with
  a Cloudflare Origin Certificate if the local network is untrusted. On the
  control path this means the Worker does not see a client certificate from the
  agent; machine and control-plane authentication use the request signatures in
  section 14.2.
- The SPA never talks to a machine directly. It talks to the API; the API talks
  to the agent; the agent talks to the runtime. This is the same-origin BFF
  pattern and it keeps service tokens out of JavaScript.

### 8.4 Human and service access

- Humans authenticate through Access with an OIDC identity provider.
- The Worker validates the Access JWT on every `/api/*` request. Static asset
  requests are authorized by Access at the edge and do not invoke the Worker.
  It does not trust a header alone.
- Service-to-service automation uses scoped API tokens issued by the control
  plane, not Access service tokens, except where a non-Worker client must cross
  an Access application. Access service tokens are limited to 50 per account, so
  they are not per-tenant.
- SSH is not a control path. If an operator needs a shell, the dashboard opens
  an audited exec session through the agent. For break-glass access, use Access
  short-lived certificates or a Teleport-style proxy, and record the session.

### 8.5 Egress

- Each workload gets a network namespace and a default-deny egress policy.
- Allowed destinations are configured per tenant: CIDRs, hostnames, or a proxy.
- DNS goes through a local resolver that enforces the allowlist and logs queries.
- A per-tenant egress proxy can inject credentials so that workload code never
  holds the upstream token.
- Egress volume is metered per tenant and reported in the dashboard.

## 9. Orchestration

### 9.1 What orchestrates what

Cloudflare is transport, identity and metadata. It is not the scheduler.

The scheduler is the **host agent plus the site runtime**. The control plane
computes a candidate set and records the decision; the agent and the runtime
make it real. This keeps the data plane independent and avoids a global
consensus system.

For a single site, Incus can make placement decisions within its cluster. For
multiple standalone machines, the control plane computes a candidate set from
labels, free capacity and locality, then asks the target machine's DO for a
lease. If the machine cannot take the workload, the agent returns a structured
failure and the control plane tries the next candidate.

### 9.2 Desired state and generations

Every workload has:

- `spec`: the desired configuration.
- `generation`: a monotonically increasing integer, incremented on every spec
  change.
- `observed`: the last state reported by the agent.
- `observed_generation`: the generation that produced `observed`.
- `lease_epoch`: the fencing token for the machine that owns the workload.

The agent applies a spec only if `generation > observed_generation` and
`lease_epoch >= local_epoch`. Applying the same generation twice is a no-op.
This makes retries and duplicate queue deliveries safe.

### 9.3 Leases and fencing

A lease carries a monotonic fencing token and a separate liveness deadline:

```
lease = {
  machine_id,
  epoch,          // monotonic, incremented on every new owner
  holder,         // agent instance id
  liveness_expires_at,
  signature
}
```

- The DO issues the lease and increments the epoch.
- The agent includes the epoch in every mutation and heartbeat.
- `epoch` is the fencing token. It never expires and is the only thing that
  decides whether a mutation is stale.
- `liveness_expires_at` is used by the control plane for placement and offline
  detection. It is not an authorization deadline for the agent.
- An agent that cannot reach the control plane keeps reconciling local desired
  state with its last epoch. It refuses a command carrying a lower epoch. This
  is what lets the data plane keep running through an outage longer than 60
  seconds.
- The DO rejects a stale epoch.
- The local runtime is told the epoch; a volume or snapshot operation that
  carries an older epoch is refused.
- If an agent is reinstalled, the old credential and key are revoked and the
  epoch is incremented. The old agent cannot act even if it still has a network
  path.

A lease is not a distributed lock across machines. It is a per-machine fencing
token. Cross-machine operations, such as moving a volume, use a two-phase
reservation: reserve on the destination, detach from the source, attach on the
destination, then release the source.

### 9.4 Placement

Placement inputs:

- Workload class (container, vm, sandbox, worker, ci).
- Required runtime and image architecture.
- CPU, memory, disk and GPU requirements.
- Volume locality and storage class.
- Site, rack and network constraints.
- Tenant isolation level (dedicated machine, dedicated VM, shared container).
- Anti-affinity and spread rules.
- Cost and power state.

Placement output is a machine id and a lease. The algorithm is deterministic for
a given input and a given inventory snapshot, so a retry does not flap. It
prefers:

1. A machine that already has the image and a warm cache.
2. A machine in the same site and rack as the volume.
3. A machine with the lowest projected utilization.
4. A machine that is not draining.

The control plane does not continuously rebalance. It rebalances only on an
explicit operation, a machine drain, or a failed lease. Avoiding a global
optimizer is deliberate: it is a source of instability and hard-to-debug
movement.

### 9.5 Reconciliation loop

The agent loop:

1. Read local desired state and the last observed state.
2. If the control plane is reachable, fetch a digest of desired state and any
   new commands. If not, continue with the local cache.
3. For each workload whose desired generation is ahead, validate the spec, check
   the lease, and call the driver.
4. Record an operation log entry before and after each external call.
5. Report observed state and events. If the report fails, keep them in a bounded
   queue and retry with exponential backoff and jitter.
6. Sleep until the next heartbeat or an event.

The loop is level-triggered, not edge-triggered. It does not depend on receiving
every command. A lost command is recovered by the next digest comparison.

### 9.6 Failure handling

| Failure | Detection | Response |
| --- | --- | --- |
| Control plane unavailable | Agent request timeout | Keep reconciling from local cache; queue events; dashboard unavailable |
| Cloudflare Tunnel down | `cloudflared` metrics, VPC error | Agent retries; public ingress down; alert; no workload restart |
| Machine offline | Lease expiry, missed heartbeats | Mark offline; do not delete workloads; alert; reschedule replicated workloads elsewhere |
| Agent crash | systemd restart, missed heartbeat | Restart; reconcile from local SQLite; do not recreate healthy workloads |
| Runtime crash | Driver health check | Restart the runtime; recreate only the affected workload |
| Queue duplicate | Idempotency key, generation | No-op |
| Split brain | Two agents claim the same machine | Credential and key revocation, epoch bump, tunnel deletion; the stale agent cannot act |
| D1 write failure | API error | Return 503; retry with the same idempotency key; do not acknowledge |
| DO unavailable | API error | Queue the command in D1 outbox; retry; the data plane is unaffected |
| R2 unavailable | Upload error | Buffer locally; retry; do not block workload start |
| Storage disk failure | SMART, checksum, scrub | Mark the disk degraded; rebuild from EC or replica; alert |
| Site network partition | Heartbeats, storage replication lag | Keep the site running; control plane sees stale state; do not auto-fail over stateful workloads |

## 10. Storage

### 10.1 Tiers

| Tier | Backend | Use | Durability |
| --- | --- | --- | --- |
| Hot | Local NVMe + SeaweedFS or RustFS | Workload data, images, caches, CI artifacts | 3 replicas or EC 4+2 within the site |
| Warm | R2 Standard | Offsite copy, control-plane artifacts, logs, snapshots | R2 durability claim, versioning |
| Cold | R2 Infrequent Access | Backups and old logs | 30-day minimum, retrieval fee |

R2 is not the primary data path for tenant workloads. It is the offsite copy and
the control-plane store. This keeps egress free for reads from Cloudflare and
keeps tenant data local.

### 10.2 Why SeaweedFS first

SeaweedFS is Apache-2.0, active, and covers the S3 surface plus IAM/STS, erasure
coding, cross-cluster replication and an admin API. `weed mini` is documented as
a single-node production path, which makes the first machine simple. It scales
out by adding volume servers and stateless S3 gateways. The tradeoff is more
moving parts than a single binary and a metadata store choice that matters.

RustFS is the second choice: Apache-2.0, MinIO-compatible layout, erasure coding,
site replication, IAM/OIDC and a console. It is younger and its own compatibility
matrix is honest about gaps. Pilot it on a non-critical bucket before trusting a
tenant.

MinIO is not a choice for new deployments. The upstream repository is archived,
and the licence is AGPL-3.0. Use it only as a compatibility reference.

Garage is a good small geo-distributed store, but it does not aim for R2 parity:
no versioning, no SSE, no object ACLs, and replication rather than erasure
coding. Use it for a small home lab, not as a general R2 replacement.

Ceph is the right answer only if you already operate Ceph. It is too heavy for a
one-machine site.

### 10.3 Buckets and tenancy

- One bucket per tenant per site, or one bucket per tenant with a key prefix if
  bucket count is a problem.
- IAM policies per tenant. A dedicated `signer` Worker holds the per-tenant
  storage credential and mints presigned URLs after an authorization check. The
  API calls it over a service binding, so the API Worker does not hold the
  storage credential.
- Presigned URLs point at a dedicated S3 hostname route through the site's
  tunnel, for example `s3-<site>.<domain>`. The S3 signature is the
  authorization. Do not put an Access application in front of that hostname: a
  presigned request carries no Access cookie or service token.
- Server-side encryption with per-tenant keys. Keep the keys in the control
  plane's secret store or a local KMS; never in object metadata.
- Versioning on for backups and audit artifacts; lifecycle rules to R2 IA.
- Object lock or a WORM bucket for audit exports.
- A `cfb-manifests` bucket holds content hashes, signatures and restore indexes.

### 10.4 Replication and restore

- A replication worker runs on each site. It walks new objects, verifies the
  checksum, and uploads to R2 with the same key and metadata.
- Replication lag is a metric. A alert fires when lag exceeds the RPO.
- Restore is a documented, tested operation: pick a snapshot or a prefix,
  restore to a new bucket, verify checksums, then cut over.
- A monthly restore drill restores a random object and a random D1 export into a
  scratch environment and compares hashes.

### 10.5 Control-plane backups

- D1: daily export to R2, plus Time Travel for point-in-time recovery. The export
  is encrypted and versioned.
- DO: the authoritative per-machine state is reconstructible from D1 and the
  agent's local SQLite. A daily snapshot is still taken for faster recovery.
- R2: versioning and object lock on audit and backup prefixes.
- Configuration: everything in Git (Terraform or Wrangler config), with secrets
  in the secret store.

RPO and RTO:

- Control-plane metadata: RPO 15 minutes, RTO 2 hours.
- Tenant objects: RPO 15 minutes for the offsite copy, RTO 4 hours for a site
  loss.
- Audit: RPO 5 minutes, immutable export.

## 11. Compute classes

### 11.1 Containers (trusted)

Use Incus containers or containerd. Default for long-running services, internal
tools, and CI runners for trusted code.

- Read-only root filesystem; a writable volume for `/tmp` and app data.
- No host mounts except an explicit volume.
- One network namespace per workload; default-deny egress.
- CPU and memory cgroups; disk quota; PID limit.
- Seccomp profile and no-new-privileges.
- A non-root user inside the container.
- Image must be signed and pinned by digest.

Containers share the host kernel. They are not a boundary for hostile code. Use
them only for code you trust at least as much as the host.

### 11.2 Virtual machines (semi-trusted)

Use Incus VMs (QEMU/KVM) for a stronger boundary with a familiar API. Use
Firecracker for a smaller device model and faster start.

- One VM per tenant or per trust group.
- Own kernel; no shared kernel with the host.
- Virtio devices only; no PCI passthrough in v1.
- Cloud-init for first boot; Ignition optional.
- Snapshot and restore through the driver.
- Live migration only within an Incus cluster and only on shared storage.

### 11.3 Sandboxes (untrusted)

Use Firecracker directly, or bhatti/hyper as an orchestrator. A sandbox is an
ephemeral microVM for agent code, CI steps, or user-supplied code.

API:

```
POST /v1/sandboxes                 create
POST /v1/sandboxes/{id}/exec       exec, stream stdout/stderr
GET  /v1/sandboxes/{id}/files      read a file
PUT  /v1/sandboxes/{id}/files      write a file
POST /v1/sandboxes/{id}/ports      expose a local port through the agent
POST /v1/sandboxes/{id}/snapshot   snapshot the rootfs
POST /v1/sandboxes/{id}/restore    restore a snapshot
DELETE /v1/sandboxes/{id}          destroy
```

Properties:

- TTL and idle timeout. The default is 30 minutes; the maximum is configurable.
- No host mounts. A shared volume is a virtio-fs or 9p mount with a narrow path,
  or an S3 mount.
- Egress through the workload proxy only.
- Snapshot is rootfs only, like Cloudflare's Containers. Memory and processes do
  not survive.
- A warm pool per image to hide image pull and boot time.
- One microVM per sandbox; do not multiplex tenants in one VM.

Do not claim Cloudflare's 648 ms p50 or 100,000-container burst. Target 1 to 3
seconds for a cold start and 200 to 500 ms from a warm pool, and measure it.

### 11.4 Workers (stateless and durable)

Use open-compute as the default per-machine Workers platform. It is one binary,
runs workerd, and provides Workers, KV, D1, R2, DO, Queues, Workflows, Cron,
Cache, Vectorize and AI Search on one machine with SQLite and optional S3. It has
a dashboard and a `/client/v4` API, which makes it easy to drive from the agent.
It is explicitly single-node and not HA, which is the right scope for one machine.

Two deployment modes:

- **open-compute mode**: install `ocd` inside an Incus VM, one VM per tenant or
  trust group. Create one open-compute instance in that VM, deploy bundles
  through `ocd cf deploy --prebuilt`, and expose routes through the tunnel. The
  agent stores the deployment metadata and health. Do not run `ocd` directly on
  the host when tenants are not mutually trusted: workerd is not a hardened
  sandbox, and open-compute's own threat model says tenant egress can reach host
  loopback and metadata addresses.
- **raw workerd mode**: run `workerd serve` under systemd with `--socket-fd`,
  generate a Cap'n Proto config, and supervise it yourself. Use this only if you
  need a binding that open-compute does not support. Raw workerd is not a
  hardened sandbox; run it inside a VM or a dedicated machine.

Distributed Durable Objects:

- **celld** is the closest thing to a distributed DO runtime on private hardware.
  It uses an S3-compatible bucket for ownership records and conditional writes,
  and runs Workers, DO, KV, D1, R2, Queues, Workflows and Cron from a
  `wrangler.json`. It is pre-1.0 (v0.6.3), runs one application per fleet, has no
  account service, and does not terminate TLS. Use it for one tenant per fleet,
  or wait.
- **workerd ClusterConfig** is a new experimental mode that distributes DOs
  across workerd instances using leases in an S3 bucket. It requires
  `--experimental` and a private network. Treat it as a research track, not a
  production default.
- **Rivet** is an actor platform, not a Workers runtime. It is a good fit for
  per-session or per-agent actors with SQLite, queues, scheduling and hibernation.
  It runs its own control plane. Use it only if the workload is actor-shaped, not
  Worker-shaped.

### 11.5 CI (heavy, long-running)

Two supported shapes:

**GitHub Actions: Fireactions**

Fireactions runs GitHub runners in ephemeral Firecracker VMs, creates JIT runner
tokens, scales pools on `workflow_job` events, and destroys the VM after the job.
It is Apache-2.0 and BYOM. Use it when the repository is on GitHub and the job
needs a real VM.

**Gitea or Forgejo: Woodpecker**

Woodpecker has an Apache-2.0 server and agent, a gRPC boundary, a Kubernetes
backend and an autoscaler. Run the server on a machine (or behind the tunnel) and
agents on the site. Use ephemeral Incus containers or Firecracker VMs per job.

**What the control plane does**

- Stores repository, pool, image, secret reference, concurrency and timeout
  configuration.
- Receives webhooks, verifies signatures, and creates a job row.
- Asks the pool's DO for a slot and enqueues a start command.
- Streams logs to R2 and artifacts to local S3.
- Shows queue depth, wait time, run time, failures and cost per repo.
- Does not run the job, hold the job's secrets, or parse the pipeline language.

Heavy CI needs:

- A warm image cache on each runner machine.
- A shared cache bucket (S3) mounted or synced into the job.
- CPU pinning or a dedicated runner machine for large builds.
- A concurrency limit per tenant to avoid one repo starving the fleet.
- A hard timeout and a disk quota.
- Log chunking with backpressure; a chatty job must not fill the agent's memory.

### 11.6 Computer-like workspace

Cloudflare Computer is an early-preview library (`@cloudflare/computer` 0.8.0,
MIT) that gives an agent a SQLite-backed workspace and switches between an
isolate and a container. It is not a separate compute product. The CFB equivalent
is a **workspace service**:

- A durable filesystem per workspace, stored in local S3 or a volume.
- Two execution backends: an open-compute/worker backend for fast file and code
  operations, and a sandbox backend for Linux commands.
- A sync daemon that keeps the two views consistent.
- Tools for read, write, edit, bash, and port forwarding.
- Every operation is audited and attributable to a workspace and an actor.

This is a product layer on top of the sandbox and Worker primitives, not a new
runtime.

## 12. Data model

The following is a logical schema, not final DDL. All tables have `created_at`,
`updated_at`, `created_by` and `generation` where mutation matters.

```
accounts(id, name, plan, created_at)
users(id, account_id, email, name, status)
memberships(user_id, account_id, role, tenant_scope)
tenants(id, account_id, name, isolation_level, quota_json)
sites(id, account_id, name, region, network_cidr, storage_class)
machines(id, site_id, name, hostname, labels_json, capacity_json,
         status, tunnel_id, tunnel_name, agent_version, agent_fingerprint,
         last_seen_at, drain_state, created_at)
runtimes(id, machine_id, kind, version, capabilities_json, status)
workloads(id, tenant_id, site_id, machine_id, name, kind, image, command_json,
          resources_json, volumes_json, network_json, placement_json,
          desired_state, generation, observed_state, observed_generation,
          lease_epoch, created_at)
volumes(id, tenant_id, site_id, machine_id, name, size_gb, storage_class,
        snapshot_id, attached_to, status)
buckets(id, tenant_id, site_id, name, backend, replication_target, status)
routes(id, tenant_id, machine_id, hostname, target, access_app_id, tls_mode)
jobs(id, tenant_id, provider, repo, ref, sha, pool_id, status, started_at,
     finished_at, exit_code, log_key, artifact_manifest_key)
operations(id, tenant_id, kind, status, steps_json, requested_by, started_at,
           finished_at, error)
secrets(id, tenant_id, name, provider, current_version, metadata_json)
secret_versions(id, secret_id, version, ciphertext, nonce, key_id, created_at,
                expires_at)
audit_events(id, account_id, tenant_id, actor, action, resource_type,
             resource_id, before_json, after_json, request_id, prev_hash, hash,
             created_at)
usage_events(id, tenant_id, machine_id, workload_id, kind, quantity, unit,
             period_start, period_end)
outbox(id, topic, payload_json, created_at, published_at)
agents(id, machine_id, credential_id, public_key, token_hash, version,
       last_seen_at, revoked_at)
```

Indexes that matter:

- `workloads(tenant_id, desired_state)`
- `workloads(machine_id, observed_generation)`
- `machines(site_id, status)`
- `audit_events(account_id, created_at)`
- `outbox(published_at)` partial index for unpublished rows
- `jobs(tenant_id, status, created_at)`

D1 size strategy: one database for the control plane until 5 GB, then shard by
account or tenant group. Cross-shard reporting goes through R2 exports, not
cross-database joins. The 10 GB per-database limit cannot be raised.

## 13. API

### 13.1 Conventions

- Base path: `/api/v1`.
- JSON, UTF-8, `snake_case` fields.
- OpenAPI 3.1 at `/api/v1/openapi.json`.
- Auth: Access JWT for the SPA, `Authorization: Bearer <token>` for API clients,
  signed requests with `X-CFB-Agent`, `X-CFB-Key-Id` and `X-CFB-Signature`
  for agents.
- Errors: `application/problem+json` with `type`, `title`, `status`, `detail`,
  `instance`, `code`, `retryable`, `request_id`.
- Idempotency: `Idempotency-Key` on POST. The key and a hash of the body are
  stored for 24 hours. A replay with a different body returns 409.
- Concurrency: `ETag` on reads, `If-Match` on writes. A stale write returns 409
  with the current generation.
- Pagination: `?limit=50&cursor=...`, response `next_cursor`.
- Async: `202` with `operation_id` and `Location: /api/v1/operations/{id}`.
- Events: `GET /api/v1/events?since=...` as SSE; `GET /api/v1/workloads/{id}/logs`
  as SSE; `GET /api/v1/workloads/{id}/exec` upgrades to WebSocket.
- Rate limits: per token and per tenant. `429` with `Retry-After`.

### 13.2 Resources

```
GET    /api/v1/sites
POST   /api/v1/sites
GET    /api/v1/sites/{id}
PATCH  /api/v1/sites/{id}

GET    /api/v1/machines
POST   /api/v1/machines            create a join token and install command
GET    /api/v1/machines/{id}
PATCH  /api/v1/machines/{id}       labels, drain state
DELETE /api/v1/machines/{id}       decommission, requires approval
POST   /api/v1/machines/{id}/actions  cordon, drain, reboot, upgrade,
                                      rotate-token, reprovision

GET    /api/v1/workloads
POST   /api/v1/workloads
GET    /api/v1/workloads/{id}
PATCH  /api/v1/workloads/{id}      desired spec, bumps generation
DELETE /api/v1/workloads/{id}
POST   /api/v1/workloads/{id}/actions  start, stop, restart, snapshot,
                                       restore, migrate, exec
GET    /api/v1/workloads/{id}/logs
GET    /api/v1/workloads/{id}/events

GET    /api/v1/volumes
POST   /api/v1/volumes
POST   /api/v1/volumes/{id}/attach
POST   /api/v1/volumes/{id}/detach
POST   /api/v1/volumes/{id}/snapshot

GET    /api/v1/buckets
POST   /api/v1/buckets
POST   /api/v1/buckets/{id}/replicate
GET    /api/v1/buckets/{id}/usage

GET    /api/v1/pools
POST   /api/v1/pools
GET    /api/v1/jobs
POST   /api/v1/jobs
GET    /api/v1/jobs/{id}
GET    /api/v1/jobs/{id}/logs
GET    /api/v1/jobs/{id}/artifacts

GET    /api/v1/routes
POST   /api/v1/routes
DELETE /api/v1/routes/{id}

GET    /api/v1/operations
GET    /api/v1/operations/{id}
GET    /api/v1/audit
GET    /api/v1/usage
GET    /api/v1/health
```

### 13.3 Example: create a sandbox

```http
POST /api/v1/workloads
Authorization: Bearer ...
Idempotency-Key: 9f2c...
Content-Type: application/json

{
  "tenant_id": "t_123",
  "name": "agent-run-42",
  "kind": "sandbox",
  "image": "ghcr.io/acme/agent-base@sha256:...",
  "resources": { "vcpu": 2, "memory_mb": 4096, "disk_gb": 20 },
  "network": { "egress": ["api.github.com", "registry.npmjs.org"] },
  "ttl_seconds": 3600,
  "placement": { "site_id": "s_home", "isolation": "microvm" }
}
```

```http
HTTP/1.1 202 Accepted
Location: /api/v1/operations/op_456
Content-Type: application/json

{
  "operation_id": "op_456",
  "status": "pending",
  "resource": { "type": "workload", "id": "w_789" }
}
```

### 13.4 Example: error

```http
HTTP/1.1 409 Conflict
Content-Type: application/problem+json

{
  "type": "https://cfb.example/errors/stale-generation",
  "title": "Stale generation",
  "status": 409,
  "detail": "The workload is at generation 7, not 6.",
  "instance": "/api/v1/workloads/w_789",
  "code": "stale_generation",
  "retryable": false,
  "request_id": "req_..."
}
```

## 14. Agent protocol

### 14.1 Identity and enrollment

- The operator creates a machine and gets a one-time join token, valid for 15
  minutes and bound to the machine id.
- The installer writes the token to a systemd credential.
- The agent generates a key pair locally. The private key never leaves the machine.
- `POST /v1/enroll` with the join token and the agent's Ed25519 public key. The
  control plane verifies the token, records the public key, and issues a signed
  machine credential valid for 24 hours. The credential binds the machine id,
  key id and expiry; the lease epoch travels in each signed request. It is not a
  TLS client certificate.
- The agent renews the credential before expiry by signing the renewal request
  with its current key.
- The control plane can revoke the credential and the public key; revocation is
  checked on every request and on every renewal.

### 14.2 Transport

- Preferred: HTTPS over the machine's Cloudflare Tunnel, reached by the control
  plane through Workers VPC `fetch()`.
- Fallback: WebSocket to the relay DO, initiated by the agent.
- Control path: the agent listens on `127.0.0.1:9443` only. It does not listen
  on a LAN address. The local admin path uses a Unix socket at
  `/run/cfb/agent.sock`, mode 0660, group `cfb-admin`, with a peer credential
  check.
- Authentication is end-to-end and does not depend on TLS termination. On both
  Cloudflare paths, Cloudflare terminates TLS, so the Worker never sees the
  agent's client certificate. Neither path uses mTLS; a self-hosted relay that
  terminates TLS may add it as an optional deployment.
- Both directions sign. The control plane holds an Ed25519 signing key; the agent
  pins its public key at enrollment and verifies every command. The agent signs
  every heartbeat and response with its machine key; the control plane verifies
  against the public key recorded at enrollment.
- The signed input is the canonical tuple `(method, path, body_sha256, ts,
  nonce, machine_id, lease_epoch)`. The signature and key id travel in
  `X-CFB-Signature` and `X-CFB-Key-Id`.
- The receiver rejects a timestamp older than 60 seconds, a reused nonce, or a
  signature from a revoked key. Nonces are kept in a bounded replay cache.

### 14.3 Endpoints

```
POST /v1/enroll
POST /v1/heartbeat
GET  /v1/desired?since=<generation>
POST /v1/reconcile
GET  /v1/state
POST /v1/events
POST /v1/logs
GET  /v1/health
POST /v1/exec
POST /v1/snapshot
POST /v1/upgrade
```

### 14.4 Envelope

```json
{
  "v": 1,
  "id": "msg_...",
  "type": "reconcile",
  "ts": "2026-10-29T11:00:00Z",
  "machine_id": "m_123",
  "lease_epoch": 42,
  "key_id": "cp-2026-10",
  "sig": "ed25519:...",
  "payload": {}
}
```

`sig` covers the canonical envelope without the `sig` field.

Types: `hello`, `heartbeat`, `desired`, `reconcile`, `ack`, `event`, `log`,
`metric`, `error`, `exec_open`, `exec_data`, `exec_close`.

### 14.5 State machine

```
pending -> applying -> running -> stopping -> stopped -> deleting -> deleted
                  \-> failed
                  \-> degraded (running but unhealthy)
```

Transitions are idempotent. A `reconcile` for a generation already applied returns
the current observed state without calling the runtime.

### 14.6 Heartbeats and leases

- Heartbeat every 15 seconds.
- Liveness deadline 60 seconds; the DO marks the machine offline after 4 missed
  heartbeats. The fencing epoch does not expire.
- Heartbeat payload: agent version, uptime, load, capacity, observed digest,
  runtime health, storage health, clock skew, last command id.
- A heartbeat does not carry the full workload list. The digest is enough; the
  control plane asks for a diff when it differs.

### 14.7 Versioning

- `protocol_version` and `agent_version` in every handshake.
- The control plane supports the current and previous protocol major versions.
- An agent that is too old is drained and upgraded, not silently ignored.
- Upgrades are signed, canary-first, and reversible.

## 15. Operations and SLOs

### 15.1 Enrollment and upgrades

- Enrollment is a Workflow with a human-visible checklist.
- Agent upgrades are staged: one canary machine per site, then 10%, then the
  rest. A failed health check rolls back.
- Agent binaries are signed and verified before execution.
- The control plane keeps the last two agent versions.

### 15.2 Observability

- OpenTelemetry in the agent and the control plane.
- Metrics: Prometheus or VictoriaMetrics on a site; remote write to the control
  plane for aggregates. Do not send every raw sample to D1.
- Logs: local Loki or plain files; upload to R2 in chunks; index in D1 by key.
- Traces: Tempo or an OTLP endpoint; sample adaptively.
- Dashboards: SPA for fleet health; Grafana for deep debugging.
- Alerts: lease expiry, machine offline, disk > 85%, replication lag > RPO,
  credential expiry < 7 days, failed reconciles > 3, queue depth > threshold,
  unusual egress, and audit chain failure.

### 15.3 SLOs

| SLO | Target | Measurement |
| --- | --- | --- |
| Control-plane API availability | 99.9% monthly | Successful requests / total, excluding 4xx |
| Dashboard availability | 99.9% monthly | Synthetic check from two regions |
| Machine heartbeat freshness | 95% < 30 s | Agent and DO timestamps |
| Container start (cached image) | p95 < 10 s | Agent operation log |
| Worker start (warm) | p95 < 1 s | open-compute / workerd log |
| Sandbox start (warm pool) | p95 < 1 s | Agent operation log |
| Provisioning a new machine | p95 < 10 min | Workflow timestamps |
| Log delivery to R2 | p95 < 10 s | Chunk timestamp vs upload |
| Control-plane metadata RPO | 15 min | Backup and export timestamps |
| Tenant object offsite RPO | 15 min | Replication lag |
| Audit export RPO | 5 min | Export cursor |

These are design targets, not measured results. Measure them before publishing
them to users.

### 15.4 Capacity planning

- Reserve 20% headroom per machine for the agent, `cloudflared`, images and
  snapshots.
- Cap containers per machine by CPU and memory, not by count.
- Keep one machine per site able to run the control-plane fallback relay? No:
  the relay is a Cloudflare DO, not on the machine.
- Keep a warm image cache sized to the top 10 images per site.
- Track storage growth per tenant and alert at 70% and 85%.

### 15.5 Incident response

1. Revoke the machine credential and key, and delete the tunnel connections.
2. Cordon the machine; do not place new workloads.
3. Snapshot the local state and logs for forensics before rebuilding.
4. Rotate any secret the machine could reach.
5. Rebuild the machine from a known image; re-enroll with a new identity.
6. Write a post-incident record with a timeline, root cause, and the control that
   prevents recurrence.

## 16. Security summary

The full threat model is in `docs/security.md`. The load-bearing controls:

- Access JWT validation at the Worker for `/api/*`, with issuer and audience
  checks.
- Machine key signatures, a short-lived machine credential, key pinning and
  revocation.
- One tunnel and one identity per machine.
- No inbound ports; no SSH control path.
- Default-deny egress per workload.
- MicroVM isolation for untrusted code; containers only for trusted code.
- workerd only inside a VM or a dedicated machine.
- Signed images and bundles; digest pinning; SBOM.
- Plaintext secrets never in D1, R2, logs or the SPA. Secret ciphertext may
  live in D1 under envelope encryption.
- Hash-chained audit log with a WORM export.
- Two-person approval for destructive operations.

## 17. Cost summary

The control plane is cheap at small scale: Workers Paid is $5/month with 10M
requests and 30M CPU-ms; D1, R2, DO and Queues are metered but small for a
fleet dashboard. Tunnels are free up to 1,000 per account, then $4.50 per tunnel
per month. Access is free for teams under 50 users, then $7 per user per month,
paid annually.

The data plane is not free. The costs are hardware amortization, power, bandwidth,
storage media, replacement, and operator time. Self-hosting wins when utilization
is high and egress is expensive; it loses when a machine sits idle or when one
person spends a day a week on it.

The full model, with formulas and a worked example, is in `docs/cost.md`.

## 18. Prior art summary

The strongest reusable pieces found on GitHub, with verified metadata:

| Need | Best pick | Why | Main gap |
| --- | --- | --- | --- |
| Site runtime | Incus | Apache-2.0, REST API, containers and VMs, clustering, projects | Shared kernel for containers; cluster is per site |
| Workers platform | open-compute | One binary, workerd fork, Workers/KV/D1/R2/DO/Queues/Workflows/Cron, SQLite/S3 | Single node, not HA; young |
| Distributed DO | celld | Closest DO semantics on private hardware, S3-backed leases | Beta, one app per fleet, no TLS |
| Raw Workers runtime | workerd | The actual runtime, Apache-2.0, active | Not a hardened sandbox; no platform |
| Actor platform | Rivet | Durable actors, SQLite, queues, scheduling, self-hostable | Not Workers-compatible |
| Sandbox | Firecracker | Per-VM API, jailer, snapshots | You build the scheduler |
| Sandbox orchestrator | bhatti / hyper / Fireactions | Higher-level microVM management | Young; Fireactions is GitHub-specific |
| S3 | SeaweedFS | S3, IAM/STS, EC, replication, single-node path | More moving parts |
| S3 alternative | RustFS | Apache-2.0, MinIO-compatible, EC, site replication | Younger; compatibility gaps |
| CI (GitHub) | Fireactions | Ephemeral Firecracker runners, JIT tokens, pool scaling | GitHub-specific |
| CI (Gitea/Forgejo) | Woodpecker | Server + agent, autoscaler, Apache-2.0 | Autoscaler not feature-complete |
| Tunnel | cloudflared + VPC | Outbound-only, identity, dynamic routes | VPC beta; 1,000 tunnel limit |
| Control-plane references | amika, Coolify, Dokploy, Portainer | Host agent, PaaS domain model, RBAC, edge agent | None is a multi-tenant CF-native control plane |

The detailed catalog, licenses, star counts and verdicts are in
`docs/prior-art.md`. Metadata was captured on 2026-10-29; see
`evidence/repos.tsv`.

## 19. Risks and open questions

| Risk | Impact | Mitigation | Status |
| --- | --- | --- | --- |
| Workers VPC is beta and may change | Private control path breaks | `PrivateTransport` abstraction; relay DO fallback; public hostname + Access fallback | Open |
| Tunnel limit and per-tunnel price | Fleet cost grows beyond 1,000 machines | Tunnels for Platforms; site-level tunnel for large sites; measure | Open |
| Access app and service-token limits | Cannot model one app/token per tenant | One app for the console; tenant authorization in the API; service tokens only per environment | Open |
| D1 10 GB per database | Metadata outgrows one DB | Shard by account/tenant; export reporting to R2 | Open |
| DO soft 1,000 req/s per object | A busy machine's coordinator saturates | Shard by machine; aggregate heartbeats; backpressure | Open |
| Incus containers share the kernel | Container escape is a host compromise | MicroVM for untrusted; seccomp; no privileged containers | Accepted |
| Firecracker needs KVM | Not available on cheap VPS or without nested virt | Bare metal or KVM-capable hosts; fall back to Incus VMs | Open |
| open-compute is young | Bugs and compatibility gaps | Pin versions; canary; keep raw workerd fallback | Open |
| celld is pre-1.0 and single-app | No multi-tenant DO fleet | One fleet per tenant; wait; or Rivet | Accepted |
| Licensing | AGPL/BUSL/SSPL blocks a hosted service | Apache/MIT first; legal review; isolate copyleft components | Open |
| Operator time | Self-hosting becomes a second job | SLOs, automation, runbooks, start with one site | Open |
| Data durability | A site loss loses data | 3-2-1, R2 offsite, restore drills | Open |
| Control-plane compromise | Attacker can command the fleet | Least privilege, audit, two-person approval, machine credential and key revocation | Open |
| Supply chain | A malicious image or bundle | Signatures, digests, SBOM, admission | Open |

Open questions that need a prototype, not an opinion:

1. Does `cf1:network` reach a route on a tunnel created after the binding, with
   no Worker redeploy? The docs say it does; test it. If it does not, fall back
   to a `tunnel_id` binding per site.
2. What is the real throughput and latency of Workers VPC `fetch()` to an agent,
   including streaming logs and exec? Measure.
3. Does a WebSocket survive a VPC `fetch()` upgrade? If not, the relay is
   mandatory for exec.
4. How fast is Incus container creation on the target hardware, warm and cold?
5. How fast is Firecracker restore from a snapshot on the target hardware?
6. What is the actual per-machine `cloudflared` throughput and CPU cost?
7. How much does open-compute cost in memory and CPU per idle tenant?
8. What is the real D1 query latency from the control plane's region?
9. Can the audit hash chain be implemented with D1 triggers, or must it be in the
   Worker?
10. What is the restore time for a 1 TB bucket from R2?

## 20. Glossary

- **Agent**: the host process that reconciles desired state on a machine.
- **Control plane**: the Cloudflare-hosted API, dashboard, identity and metadata.
- **Data plane**: the machines and runtimes that execute workloads.
- **DO**: Cloudflare Durable Object; also used for the per-machine coordinator.
- **Fencing epoch**: a monotonic number that invalidates stale writers.
- **Generation**: a monotonic number for a workload spec.
- **Lease**: a time-bounded right to act on a machine.
- **Site**: a trust and failure domain (rack, home, colo, region).
- **Workload**: a container, VM, sandbox, Worker, durable actor, CI runner or
  storage service managed by CFB.

## 21. References

Cloudflare, read 2026-10-29:

- Containers limits: https://developers.cloudflare.com/containers/platform/limits/
- Containers pricing: https://developers.cloudflare.com/containers/platform/pricing/
- Containers architecture: https://developers.cloudflare.com/containers/concepts/architecture/
- Container DO API: https://developers.cloudflare.com/containers/api/durable-object-container/
- Sandbox lifetime: https://developers.cloudflare.com/sandbox/concepts/lifetime/
- Workers VPC: https://developers.cloudflare.com/workers-vpc/
- Cloudflare One limits: https://developers.cloudflare.com/cloudflare-one/account-limits/
- D1 limits: https://developers.cloudflare.com/d1/platform/limits/
- Durable Objects limits: https://developers.cloudflare.com/durable-objects/platform/limits/
- Durable Objects pricing: https://developers.cloudflare.com/durable-objects/platform/pricing/
- R2 pricing: https://developers.cloudflare.com/r2/pricing/
- R2 limits: https://developers.cloudflare.com/r2/platform/limits/
- Workflows pricing: https://developers.cloudflare.com/workflows/reference/pricing/
- Workflows limits: https://developers.cloudflare.com/workflows/reference/limits/
- Workers VPC networks: https://developers.cloudflare.com/workers-vpc/configuration/vpc-networks/
- Workers static assets: https://developers.cloudflare.com/workers/static-assets/
- Cloudflare Mesh: https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-mesh/
- Tunnels for Platforms: https://developers.cloudflare.com/tunnel/features/tunnels-for-platforms/
- Sandbox GA: https://blog.cloudflare.com/sandbox-ga/
- Faster agent sandboxes: https://blog.cloudflare.com/faster-agent-sandboxes/
- Cloudflare Computer: https://blog.cloudflare.com/cloudflare-computer/

Captured copies of these pages are in `evidence/cf-docs/`.

Project sources, read 2026-10-29:

- workerd: https://github.com/cloudflare/workerd
- open-compute: https://github.com/elliothux/open-compute
- celld: https://github.com/denoland/celld
- Rivet: https://github.com/rivet-dev/rivet
- WDL: https://github.com/wdl-dev/wdl
- amika: https://github.com/gofixpoint/amika
- Incus: https://github.com/lxc/incus
- Firecracker: https://github.com/firecracker-microvm/firecracker
- bhatti: https://github.com/sahil-shubham/bhatti
- hyper: https://github.com/harmont-dev/hyper
- Fireactions: https://github.com/hostinger/fireactions
- SeaweedFS: https://github.com/seaweedfs/seaweedfs
- RustFS: https://github.com/rustfs/rustfs
- Woodpecker: https://github.com/woodpecker-ci/woodpecker
- cloudflared: https://github.com/cloudflare/cloudflared
- Pangolin: https://github.com/fosrl/pangolin
- NetBird: https://github.com/netbirdio/netbird

Metadata captures are in `evidence/repos.tsv`; page captures are in
`evidence/reads/`.
