# CFB specification

Status: design draft, 2026-10-29. No implementation is claimed or included.

## 1. Scope

### 1.1 Problem

Cloudflare's data-plane products are convenient until a workload needs a
long-lived process, a large local disk, a private network, a GPU, predictable
egress, or a machine the operator already owns. Moving that workload off
Cloudflare usually means rewriting it against a different API. CFB exists to
avoid that rewrite.

The control plane stays on Cloudflare. A single daemon on each machine, `cfbd`,
implements the same data-plane contracts locally. The same Worker bundle, S3
client, OCI image or CI workflow is intended to run on either side. The deploy
target changes; the application source does not.

### 1.2 Invariants

1. **Portable artifact.** The unit of deployment is the artifact Cloudflare
   already accepts: a Workers bundle and `wrangler.jsonc`, an OCI image, an S3
   request, a GitHub Actions workflow. CFB does not define a parallel application
   API.
2. **One CFB daemon per machine.** `cfbd` replaces the tunnel client, host agent,
   local router, metrics agent and local orchestrator. Runtime children are
   processes it supervises, not separate CFB control planes.
3. **Outbound only.** The node initiates every connection. There is no inbound
   listener on a public interface. The default transport is WebSocket over 443.
4. **Control plane optional for execution.** A running workload continues when
   Cloudflare is unreachable. The node reconciles its last desired state.
5. **Least privilege by default.** `cfbd` runs as an unprivileged user. Root and
   kernel features are opt-in and reported as capabilities.
6. **Fail closed.** An unsupported binding, missing capability or failed
   signature stops the deployment with a named error. It does not silently
   degrade.
7. **One authority per fact.** D1 owns inventory. A per-machine Durable Object
   owns the lease. The node's SQLite owns observed runtime state. Local storage
   owns object bytes. R2 owns the offsite copy.
8. **Testable compatibility.** Compatibility claims are backed by a conformance
   suite that runs the same artifact against Cloudflare and a node. A claim
   without a test is a target, not a result.

### 1.3 Non-goals

- A global anycast edge or a multi-region active-active control plane.
- Reimplementing Cloudflare's proprietary services: Images, Stream, AI, Vectorize,
  Hyperdrive, Browser Rendering, R2 event notifications, D1 read replication.
- A new container runtime, hypervisor, S3 server or CI engine written from
  scratch. CFB supervises existing ones.
- Running hostile multi-tenant code in a shared-kernel container or in workerd
  alone.
- Exact behavioural parity for every Cloudflare API. The supported subset is
  explicit.

### 1.4 Terms

- **Node**: one machine running `cfbd`.
- **Broker**: the `cfbd` process and its supervised children.
- **Relay**: the Cloudflare Worker and Durable Object that terminate the node's
  outbound WebSocket.
- **Profile**: the capability and privilege set a node runs with: `standard`,
  `lite` or `privileged`.
- **Artifact**: a Workers bundle, OCI image, S3 object, or CI workflow.
- **Capability**: a runtime feature the node reports, such as `workerd`,
  `oci_rootless`, `kvm` or `s3_local`.

## 2. Portability contract

### 2.1 Layers

Compatibility is claimed at four layers, weakest first:

| Layer | Meaning | How it is checked |
| --- | --- | --- |
| Source | Application source compiles and imports without edits | Build the same bundle for both targets |
| Configuration | The same `wrangler.jsonc` binding names and values are accepted | Parse and validate the config |
| Protocol | The wire contract is the same: S3, OCI, HTTP, WebSocket, GitHub runner | Run the same client or image |
| Semantics | Behaviour matches: SQL, DO alarms, R2 consistency, queue delivery | Conformance suite against both targets |

CFB targets source, configuration and protocol compatibility. Semantic
compatibility is a measured result per primitive, not a blanket promise.

### 2.2 Matrix

| Contract | Cloudflare | Node | Same artifact | Boundary |
| --- | --- | --- | --- | --- |
| Workers module | Workers V8 isolate | workerd | Yes | `fetch`, `env`, `ctx`, service bindings, DO |
| Durable Objects | Cloudflare DO with SQLite | workerd DO with SQLite | Yes | Alarms and hibernation; no location hints |
| R2 binding | `env.BUCKET` | S3 gateway through a generated shim | Yes | S3 operations in the supported subset |
| External S3 | R2 S3 API | S3 gateway or upstream S3 | Yes | AWS SDK endpoint changes only |
| D1 binding | `env.DB` | SQLite through a generated shim | Yes | SQL subset; no read replication |
| KV binding | `env.KV` | SQLite through a generated shim | Yes | Local strong reads; no global eventual model |
| Queues binding | `env.QUEUE` | SQLite queue through a generated shim | Yes | At-least-once; no global ordering |
| Containers | Cloudflare Containers | Rootless Podman or Docker | Image yes | Cloudflare container API is not emulated |
| CI | GitHub Actions | GitHub runner or Fireactions | Workflow yes | Runner binary and container runtime required |
| CI | Gitea or Forgejo | Woodpecker | Workflow yes | Different pipeline syntax from GitHub |
| Tunnel | Cloudflare Tunnel | WSS relay or native tunnel | Route config translates | No `cloudflared` process |
| Access | Cloudflare Access | Cloudflare Access | Yes | Control plane stays on Cloudflare |

The shims are generated at deploy time. They wrap the user's module and populate
`env` with objects that implement the binding methods over the node's local
services. The user's source is not edited.

### 2.3 Supported Workers subset

Native in workerd:

- ES module `export default { fetch }`, `scheduled`, `queue` handlers where
  workerd exposes them.
- `env` plain values and secrets.
- Service bindings between Workers in the same node.
- Durable Object classes, SQLite storage, alarms, hibernating WebSockets.
- Outbound `fetch` through the node's egress policy.

Generated shim:

- R2 bucket: `head`, `get`, `put`, `delete`, `list`, multipart upload.
- D1: `prepare`, `bind`, `first`, `all`, `run`, `batch`, `exec`.
- KV: `get`, `put`, `delete`, `list`.
- Queues: `send`, `sendBatch`, consumer handler.

Not supported on a node:

- Cloudflare-only bindings: AI, Vectorize, Hyperdrive, Browser Rendering,
  Images, Stream, Analytics Engine SQL, R2 event notifications, D1 read
  replication, DO location hints, Workers Builds API.
- `wrangler` features that require the Cloudflare API: secrets store, tail
  consumers, custom domains managed by Wrangler, gradual deployments.

A config that uses an unsupported binding fails validation before deploy. A
runtime call to an unimplemented method throws `UnsupportedBindingError` with
the binding name and method.

### 2.4 Deploy flow

1. CI builds the Worker with `wrangler deploy --dry-run --outdir dist`. The
   output is the same bundle Cloudflare would receive.
2. The control plane stores the bundle, its SHA-256, the `wrangler.jsonc` and the
   signature.
3. The target node verifies the hash and signature, parses the config, generates
   the binding shim, writes a workerd Cap'n Proto config, and starts or reloads
   workerd.
4. The node runs a smoke request against the new version. On success it moves
   the route pointer atomically. On failure the previous version stays live.
5. The same bundle can be deployed to Cloudflare with `wrangler deploy` without
   rebuilding.

The deploy command changes from `wrangler deploy` to a CFB upload command. The
application source and the bundle do not.

### 2.5 Failure behaviour

- Unsupported config: reject at upload with the binding path and the reason.
- Missing capability: reject at placement; do not schedule to a node that cannot
  run it.
- Shim method not implemented: throw at first call; do not return a plausible
  wrong value.
- Semantic mismatch found by conformance: mark the primitive unsupported for
  that release; do not claim parity.

## 3. Architecture

### 3.1 Diagram

```
+---------------------------- Cloudflare account ----------------------------+
|                                                                           |
|  Browser --HTTPS/Access--> console Worker --service--> api Worker        |
|                                  |            |            |              |
|                                  v            v            v              |
|                                 D1           R2          Queues           |
|                                  |                                       |
|                     +------------+------------+                          |
|                     | coordinator DO per machine |                        |
|                     | lease, fencing, live state |                        |
|                     +------------+------------+                          |
|                                  |                                       |
|                     +------------v------------+                          |
|                     | relay DO per machine    |                          |
|                     | WSS termination, mux    |                          |
|                     +------------+------------+                          |
+----------------------------------|----------------------------------------+
                                   | outbound wss:// 443
                                   v
+------------------------------ Node ---------------------------------------+
|  cfbd (single Rust binary, unprivileged)                                  |
|    transport: relay client / optional native tunnel                       |
|    router: host/path -> stream -> local Unix socket                       |
|    supervisor: workerd, podman, incus, firecracker, runner                |
|    storage: S3 gateway over local FS or upstream S3                      |
|    state: SQLite (desired, observed, leases, events)                      |
|    metrics: Prometheus text and pushed samples                            |
|         |                                                                 |
|         +--> workerd (V8, Unix socket)                                    |
|         +--> rootless Podman (OCI)                                        |
|         +--> Incus / Firecracker (privileged profile only)                |
+---------------------------------------------------------------------------+
```

### 3.2 Planes and trust

| Plane | Trust | Authority | Failure |
| --- | --- | --- | --- |
| Browser | Untrusted | Access session | Dashboard unavailable |
| Control plane | Cloudflare account, operator-controlled | Inventory, desired state, policy, audit | Nodes keep running; writes queue or fail |
| Relay | Cloudflare, trusted for transport | Stream routing | Ingress and control path lost |
| Node | Operator-owned | Observed state, local execution | Workloads continue |
| Workload | Untrusted or semi-trusted | None over the host | Isolated by workerd, container or VM |

A workload must never reach `cfbd`'s Unix socket, its identity key, its SQLite
state, another tenant's runtime, or the host metadata address.

### 3.3 Request flows

**Deploy a Worker**

1. CI uploads the bundle and config to the control plane.
2. The API records a deployment and enqueues `apply` to the target node.
3. The relay opens an `apply` stream to `cfbd`.
4. `cfbd` verifies the signature, generates the shim and workerd config, writes a
   new version directory, and starts workerd on a new Unix socket.
5. A smoke request succeeds; `cfbd` swaps the route pointer and reports observed
   state.

**Read an object**

1. A Worker calls `env.BUCKET.get(key)`.
2. The generated shim calls the node's S3 gateway over the local Unix socket.
3. The gateway reads the local filesystem or proxies to the upstream S3.
4. The response streams back through the shim. No Cloudflare R2 request is made
   unless the object is configured as an R2 replica.

**Start a container**

1. The API writes a workload row and asks the coordinator DO for a lease.
2. The relay opens a `reconcile` stream to `cfbd`.
3. `cfbd` checks the lease epoch, calls the rootless Podman API over its Unix
   socket, and starts the OCI image.
4. `cfbd` reports observed state and streams logs.

**Exec into a workload**

1. The SPA opens a WebSocket to the API.
2. The API authorizes the actor and asks the relay to open an `exec` stream.
3. The relay and `cfbd` carry framed stdin/stdout/stderr with credit-based flow
   control. The node never opens an inbound connection.

## 4. Broker

### 4.1 Single binary

`cfbd` is one statically linked Rust binary. It is the only CFB process on the
node. It does not embed a container runtime, a hypervisor or a V8 engine; it
supervises those as child processes because they are separate security and
failure domains.

Compile-time features:

| Feature | Contents | Profile |
| --- | --- | --- |
| `core` | transport, router, supervisor, SQLite state, metrics | all |
| `workerd` | workerd config generation and supervision | standard, lite |
| `oci` | rootless Podman/Docker client over Unix socket | standard |
| `incus` | Incus client, VM and container driver | privileged |
| `firecracker` | Firecracker API, jailer integration | privileged |
| `s3` | local S3 gateway and upstream S3 client | standard, lite |
| `ci` | runner supervision and log streaming | standard, privileged |
| `tun` | native Cloudflare Tunnel QUIC/HTTP2 client | standard, privileged |

A `lite` build excludes `oci`, `incus`, `firecracker`, `ci` and `tun`. The binary
still contains the relay client, router, workerd supervisor, S3 gateway, state
and metrics.

### 4.2 Modules

- **transport**: one outbound connection, reconnect with jitter, stream
  multiplexing, heartbeat, signed handshake.
- **router**: maps a stream's host and path to a local Unix socket or supervised
  service; enforces per-route auth and body limits.
- **supervisor**: starts, stops, restarts and health-checks children; captures
  stdout/stderr with bounded buffers; applies rlimits and cgroups when delegated.
- **workerd**: parses `wrangler.jsonc`, generates `workerd.capnp`, generates the
  binding shim, manages version directories and atomic route swaps.
- **oci**: talks to the rootless Podman or Docker socket; pulls by digest;
  creates, starts, stops, removes containers; streams logs and exec.
- **s3**: implements the S3 subset over a local filesystem or proxies to an
  upstream S3 endpoint; verifies presigned URLs.
- **state**: SQLite in WAL mode for desired specs, observed state, leases,
  operations, event queue and local metrics buffer.
- **metrics**: Prometheus text endpoint on the Unix socket and a pushed sample
  to the control plane.
- **admin**: local CLI over the Unix socket: status, logs, drain, upgrade,
  rotate-key, diagnose.

### 4.3 Process and privileges

- Runs as a dedicated user, `cfb`, or as the invoking user in `lite` mode.
- No setuid bit. No ambient capabilities by default.
- The systemd unit uses `NoNewPrivileges=yes`, `ProtectSystem=strict`,
  `ProtectHome=yes`, `PrivateTmp=yes`, `RestrictAddressFamilies=AF_UNIX
  AF_INET AF_INET6`, and a read-write path only for the data directory.
- `standard` may use the user's rootless Podman socket at
  `$XDG_RUNTIME_DIR/podman/podman.sock`.
- `privileged` may use a separate, optional helper for Incus, KVM, network
  namespaces and mounts. The helper is not installed by default and is not part
  of the `cfbd` binary. `cfbd` talks to it over a narrow local API, never a shell.
- `lite` uses no helper and no root-owned socket.

### 4.4 Local API

- Default socket: `/run/cfb/cfbd.sock` for a system service, or
  `$XDG_RUNTIME_DIR/cfb/cfbd.sock` for a user service.
- Mode 0660, group `cfb-admin`; peer credentials checked with `SO_PEERCRED`.
- HTTP/1.1 over the Unix socket, with JSON and streamed bodies. This keeps the
  API debuggable with standard tools and avoids a custom RPC.
- No TCP listener by default. `standard` may bind a child runtime to loopback
  when the runtime cannot use a Unix socket. It never binds `0.0.0.0` or `::`.
- `lite` and `strict` allow no TCP listener at all. Workerd receives an inherited
  Unix socket with `--socket-fd`.

### 4.5 Capabilities

`cfbd` reports capabilities at enrollment and on every heartbeat:

```
workerd, do_sqlite, oci_rootless, oci_root, kvm, s3_local, s3_remote,
ci_github, ci_woodpecker, tun, fuse, cgroup_v2, user_ns, tcp_loopback,
unix_socket, tpm
```

The control plane stores the set and schedules only workloads whose required
capabilities are present. A capability is owned by the node, not granted by the
control plane. This follows the ServerBee model: the endpoint is the authority
on what it can actually do.

### 4.6 Local state

- `/var/lib/cfb/state.db` or `$XDG_DATA_HOME/cfb/state.db`: SQLite, WAL.
- `/var/lib/cfb/versions/<sha256>/`: immutable Worker bundles and configs.
- `/var/lib/cfb/workloads/<id>/`: container and VM metadata.
- `/var/lib/cfb/objects/<bucket>/<key>`: local S3 objects when `s3_local` is on.
- `/var/lib/cfb/logs/<workload>/<run>`: bounded log segments.
- `/etc/cfb/cfbd.toml` or `$XDG_CONFIG_HOME/cfb/cfbd.toml`: non-secret config.
- `/etc/cfb/identity/` or `$XDG_STATE_HOME/cfb/identity/`: machine key and
  credential, mode 0600.

### 4.7 Resource envelope

Design targets, not measured results:

| Metric | Target |
| --- | --- |
| Static binary, `lite` | under 20 MiB |
| Static binary, `standard` | under 30 MiB |
| Idle RSS, `lite` | under 30 MiB |
| Idle RSS, `standard` | under 60 MiB |
| Idle CPU | under 1% of one core |
| Reconnect after network loss | under 10 s, with jitter |
| Local control API latency | under 20 ms p95 |

The broker must stay below these targets on a 512 MB, 1 vCPU VPS. Measure before
publishing.

## 5. Transport

### 5.1 Relay default

The default transport is a single outbound WebSocket from `cfbd` to
`wss://relay.<domain>/agent`. The Worker upgrades the request to a per-machine
Durable Object. The DO holds the socket with the hibernation API and opens
streams to the node.

Why this is the default:

- Works through any network that allows HTTPS on 443. No UDP 7844, no
  `cloudflared`, no inbound firewall rule.
- Runs without root, TUN or network namespaces.
- One connection per node, not one per workload.
- The same code path serves control commands, logs, exec and public ingress.
- The relay is a Cloudflare primitive, so the control plane stays on Cloudflare.

Costs and limits:

- Incoming WebSocket messages are billed at a 20:1 ratio, outgoing messages and
  protocol pings are free, and one request creates the connection. Control
  traffic is cheap; sustained bulk traffic still costs more than a native tunnel
  because of the extra hop.
- A relay DO is a per-machine failure domain. If it is unavailable, that node's
  control and ingress path is unavailable.
- The Worker and DO must enforce frame size, stream count and backpressure.

High-throughput data paths should use the native tunnel or direct S3, not the
relay.

### 5.2 Native tunnel, optional

For nodes that can reach UDP 7844 or TCP 7844, `cfbd` may speak the Cloudflare
Tunnel protocol directly. This removes the relay DO from the data path and
matches `cloudflared` behaviour.

The implementation builds on `talaria0101/cfrs` (0BSD): edge discovery, QUIC and
HTTP/2 transport, header serialization, and the rootless Unix-socket origin.
`cfrs` does not implement tunnel registration or visitor serving; CFB must add
those. Until that exists, the native tunnel is a research track, not the default.

Workers VPC is a third option. It lets a Worker reach a tunnel route without a
relay DO. It is beta, account-wide when bound to `cf1:network`, and not required
by this design. The relay is the portable path; VPC is an optimization.

### 5.3 Framing and multiplexing

One WebSocket carries many logical streams. Frames are binary, not base64 JSON:

| Offset | Size | Field |
| --- | --- | --- |
| 0 | 1 | version, `1` |
| 1 | 4 | stream id, big-endian u32 |
| 5 | 1 | type: `open`, `data`, `credit`, `half_close`, `reset`, `ping` |
| 6 | 1 | flags |
| 7 | 4 | payload length, big-endian u32 |
| 11 | n | payload |

- The control plane opens streams. The node never sends `open`; this prevents a
  compromised node from creating arbitrary server-side streams.
- Each stream has a bounded queue. A slow stream cannot starve another. Use a
  256 KiB default per stream, configurable, following mirrord's tunnel FIFO
  default.
- Credit-based flow control per stream, in bytes. The receiver grants credit;
  the sender stops at zero.
- Half-close is explicit. A request body and a response body may use the same
  stream, as cfrs does for its Unix-socket origin.
- Ping every 20 s; a missed pong after 60 s closes the socket and triggers
  reconnect.
- Maximum frame 64 KiB. Larger payloads are chunked.

### 5.4 Identity and enrollment

- The operator creates a machine and gets a one-time join token, valid for 15
  minutes and bound to the machine id.
- `cfbd` generates an Ed25519 key pair locally. The private key never leaves the
  node.
- `POST /v1/enroll` with the join token and public key. The control plane records
  the key and issues a signed machine credential valid for 24 hours.
- The credential binds machine id, key id and expiry. The lease epoch travels in
  each signed request.
- The node renews by signing the renewal request with its current key.
- Revocation is checked on every request and renewal.
- Every envelope is signed. TLS terminates at Cloudflare, so the signature is
  the end-to-end authentication. No mTLS is used on either Cloudflare path.

### 5.5 Constrained profiles

| Profile | Root | Inbound TCP | Runtime | Transport | Use |
| --- | --- | --- | --- | --- | --- |
| `standard` | No | Loopback only | workerd, rootless Podman | WSS relay, optional native tunnel | Normal VPS or home server |
| `lite` | No | None | workerd, static files | WSS relay on 443 | 512 MB VPS, no root, no container runtime |
| `strict` | No | None | workerd, static files | WSS relay on 443 | No TCP listener at all |
| `privileged` | Optional helper | Loopback only | Incus, Firecracker, workerd, Podman | WSS relay or native tunnel | KVM, VMs, untrusted code |

`lite` constraints:

- No setuid, no capabilities, no TUN, no network namespace, no KVM, no FUSE.
- No Docker or Podman socket. No container workloads.
- Workerd runs as the user and receives a Unix socket.
- Local state under the user's home or a writable data directory.
- Resource limits use rlimits and workerd limits. Cgroup delegation is optional.
- Only trusted code. Workerd is not a hardened sandbox; `lite` must not run
  hostile tenants. Untrusted code requires `privileged` with a microVM.
- Storage is a local directory with the embedded S3 gateway. Optional upstream
  S3 over outbound HTTPS.

`strict` adds:

- No TCP listener, not even loopback. All local services use Unix sockets.
- Static files and workerd only. No child that requires a TCP port.
- Public ingress is the relay only.

### 5.6 Public ingress

- A route maps a hostname and optional path prefix to a local Unix socket or a
  supervised service.
- The relay opens a stream for each request. `cfbd` forwards it to the target.
- Access protects private hostnames. Public hostnames get WAF, rate limits and a
  body limit at Cloudflare.
- TLS terminates at Cloudflare. The origin uses HTTP over a Unix socket. If the
  local network is untrusted, the operator may terminate TLS in `cfbd` with a
  Cloudflare Origin Certificate.
- The SPA never talks to a node directly. It talks to the API; the API talks to
  the relay; the relay talks to `cfbd`.

## 6. Control plane

### 6.1 Cloudflare products

| Need | Product | Constraint |
| --- | --- | --- |
| SPA and API | Workers static assets and Hono | Worker invocations are billed |
| Relational metadata | D1 | 10 GB per database, single primary |
| Object store | R2 | 5 TiB per object, 1 write/s per key |
| Per-machine lease | Durable Objects with SQLite | 10 GB per object, soft 1,000 req/s |
| Relay | Durable Object with WebSocket hibernation | Messages count as requests |
| Command fanout | Queues | At-least-once, unordered |
| Long-running operations | Workflows | Idempotent steps, 25,000 step maximum |
| Human auth | Access | 500 apps, 50 service tokens |
| Optional direct path | Workers VPC | Beta, account-scoped |

### 6.2 Modules

**`console` Worker**

- Serves the SPA from static assets with SPA fallback.
- Serves `/api/*` on the same origin to avoid CORS and keep the Access cookie.
- Validates `Cf-Access-Jwt-Assertion` on `/api/*` against the team JWKS, with
  issuer and audience checks.
- Applies CSP, HSTS, frame denial, CSRF tokens and per-user rate limits.

**`api` Worker**

- Owns the REST API, OpenAPI 3.1 document and domain services.
- Writes mutations and audit rows in one D1 `batch()`.
- Uses `Idempotency-Key`, `ETag`/`If-Match`, and RFC 9457 errors.
- Writes an outbox row in the same batch as each state change. A queue consumer
  publishes it. No dual write.
- Never calls a node directly for a long operation. It enqueues a command and
  returns `202`.

**`signer` Worker**

- Holds per-tenant storage credentials in Secrets Store.
- Mints short-lived presigned URLs after an authorization check.
- Has no public route; reachable only through a service binding.
- Records every mint in the audit chain.

**`coordinator` Durable Object, one per machine**

- Holds `machine_id`, `lease_epoch`, `key_id`, `last_seen`, desired and observed
  digests, pending commands and live subscribers.
- Serializes reservations and command delivery for one machine.
- Rejects a command with an older `lease_epoch`.
- Runs a lease alarm and marks the machine offline after four missed heartbeats.
- Fans out state to the SPA over a hibernating WebSocket.

**`relay` Durable Object, one per machine**

- Accepts the outbound WebSocket from `cfbd`.
- Verifies the signed hello and maps the socket to a machine id.
- Opens streams for control requests, logs, exec and ingress.
- Enforces per-stream credit, frame size and stream count.
- Never accepts an `open` frame from the node.

**`provisioner` Workflow**

- `enroll_machine`: validate join token, create machine row, render install
  command, wait for first heartbeat.
- `upgrade_broker`: drain, verify signed binary, restart, health check, roll back.
- `decommission`: drain, stop workloads, revoke credential and key, delete local
  keys, mark deleted.
- Every step is idempotent and keyed by operation id.

**`janitor` Cron**

- Expires leases, operations and stale relay sockets.
- Deletes expired snapshots and orphaned R2 objects.
- Rotates credentials before expiry.
- Re-enqueues drift between desired and observed state.
- Emits usage summaries.

**`notifier`**

- Sends alerts to email, Discord or Slack. Content is rendered from D1 and never
  includes secrets.

### 6.3 State ownership

| Fact | Primary | Projection | Consistency |
| --- | --- | --- | --- |
| Tenant, user, role | D1 | SPA cache | Read-after-write |
| Machine inventory | D1 | Coordinator DO, SPA | Eventual |
| Lease and fencing epoch | Coordinator DO | D1 audit | Strong in one object |
| Desired workload spec | D1 | Node SQLite | Eventual; generation idempotent |
| Observed workload state | Node SQLite | Coordinator DO, D1 | Eventual; last writer with generation |
| Object bytes | Node storage | R2 replica | Async, checksum-verified |
| Audit log | D1 hash chain | R2 WORM export | Append-only |
| Job logs | Node disk, then R2 | D1 index | Chunked, checksummed |
| Metrics | Node buffer | DO aggregate, D1/R2 | Sampled; not billing |

Rules:

- D1 is not a queue. Every state change writes an outbox row.
- A DO is not inventory. It holds hot state and a bounded command queue.
- R2 is not mutable state. Objects are immutable and content-addressed.
- The node's SQLite is authoritative for what is running. The control plane
  reconciles to it; it does not overwrite it blindly.

### 6.4 Dashboard

- Vite and React or Svelte, built to static assets. No SSR.
- Views: overview, sites, machines, workloads, storage, CI, routes, operations,
  audit, usage, settings.
- Topology graph with live status, following homelable's node/edge model and
  WebSocket status envelope.
- Server and application cards with current state plus history, following
  CoreObs's model. Do not copy its database-credential agent.
- Host inventory from a one-shot probe, following omnyssh's sectioned command
  output. Do not store plaintext passwords.
- Live updates over one WebSocket with a cursor. Reconnect on gap.
- Destructive actions require typing the resource name; production actions need
  a second approver. The approval is a Workflow step.
- Every resource has a "why is this here" panel: desired state, observed state,
  last reconcile, lease epoch, recent events.

### 6.5 Deployment

- `wrangler versions upload`, then `wrangler versions deploy`, tagged with the
  git SHA. Promotion is a separate auditable action.
- D1 migrations are expand/contract and forward-only. A rollback restores code,
  not data.
- Secrets are Workers secrets or Secrets Store bindings. Never in D1, R2, logs or
  the SPA.
- One Cloudflare account per control plane. Multi-account is out of scope.

## 7. Execution backends

### 7.1 Workers

`workerd` is the runtime for both Cloudflare and the node. The node generates a
Cap'n Proto config from `wrangler.jsonc` and starts one workerd process per
tenant or trust group. A generated shim maps R2, D1, KV and Queues to local
services.

Two modes:

- **direct**: workerd plus the generated shim. This is the default for
  `standard` and `lite`. It covers the supported subset in section 2.3.
- **open-compute**: supervise `ocd` when the application needs the broader
  Workers API surface that open-compute implements. This adds a second binary
  and is optional. open-compute is single-node and not HA.

Raw workerd is not a hardened sandbox. Run it inside a VM or dedicated machine
for untrusted code. `lite` runs trusted code only.

### 7.2 Containers

Use rootless Podman by default, Docker if the operator already runs it, and Incus
containers in `privileged`. The artifact is an OCI image, so the same image runs
on Cloudflare Containers and on the node.

- Read-only root filesystem; writable volume for `/tmp` and app data.
- No host mounts except an explicit volume.
- One network namespace per workload; default-deny egress.
- CPU and memory limits through cgroups v2 when delegated; rlimits otherwise.
- Seccomp profile, no-new-privileges, non-root user inside the container.
- Image signed and pinned by digest.

Containers share the host kernel. They are not a boundary for hostile code.

### 7.3 MicroVMs

`privileged` only. Use Firecracker directly or Incus VMs. One VM per tenant or
trust group; own kernel; virtio devices only; no PCI passthrough in v1.
Snapshot and restore through the driver. A sandbox is an ephemeral microVM with
a TTL and an idle timeout.

### 7.4 Storage

The node exposes an S3 gateway over a local filesystem. The gateway implements a
narrow subset: `HeadObject`, `GetObject`, `PutObject`, `DeleteObject`,
`ListObjectsV2`, and multipart upload. It verifies presigned URLs minted by the
control plane. For multi-node durability, the backend can be SeaweedFS or RustFS
instead of the local filesystem; the S3 contract does not change.

R2 remains the offsite copy and the control-plane store. Tenant reads do not go
through R2 unless the bucket is configured that way.

### 7.5 CI

CI is optional and not part of `lite`. The node supervises one of:

- the official GitHub Actions runner, in a rootless container or as a user
  process, for GitHub workflows;
- a Woodpecker agent, for Gitea or Forgejo.

The broker does not parse pipeline YAML. It starts the runner, injects a
short-lived job token, streams logs and uploads artifacts. The workflow file is
unchanged.

### 7.6 Driver interface

Each backend implements a small interface:

```
Capabilities() -> CapabilitySet
Reconcile(desired Workload, lease Lease) -> Observed
Start(id), Stop(id, grace), Destroy(id)
Snapshot(id) -> SnapshotID, Restore(id, snapshot)
Exec(id, argv) -> Stream, Logs(id, since) -> Stream
Stats(id) -> Stats
```

The driver maps a portable workload spec to native objects and reports drift.
The control plane never knows a Podman container id or a Firecracker socket path.

## 8. Orchestration

### 8.1 Placement

The control plane computes a candidate set from capabilities, labels, free
capacity, volume locality, isolation level, anti-affinity and drain state. It
asks the target machine's coordinator DO for a lease. The node makes the
workload real. The control plane does not continuously rebalance; it moves
workloads only on an explicit operation, drain or failed lease.

Preference order:

1. Node has the image in cache.
2. Same site and rack as the volume.
3. Lowest projected utilization.
4. Not draining.

### 8.2 Generations and leases

Every workload has `spec`, `generation`, `observed`, `observed_generation` and
`lease_epoch`.

- The node applies a spec only if `generation > observed_generation` and
  `lease_epoch >= local_epoch`.
- The same generation twice is a no-op.
- The lease epoch is a monotonic fencing token. It does not expire.
- `liveness_expires_at` is for placement and offline detection, not agent
  authorization.
- A node that cannot reach the control plane keeps reconciling its last desired
  state and refuses a lower epoch.
- Reinstalling a node revokes the old credential and key and increments the
  epoch.

### 8.3 Reconciliation

1. Read local desired and observed state.
2. If the control plane is reachable, fetch a desired digest and commands. If
   not, use the local cache.
3. For each workload with a newer generation, validate, check the lease, call
   the driver.
4. Record an operation entry before and after each external call.
5. Report observed state and events. On failure, keep a bounded queue and retry
   with exponential backoff and jitter.
6. Sleep until the next heartbeat or event.

The loop is level-triggered. A lost command is recovered by the next digest.

### 8.4 Failure handling

| Failure | Detection | Response |
| --- | --- | --- |
| Control plane unavailable | Request timeout | Keep reconciling; queue events; dashboard unavailable |
| Relay unavailable | WebSocket close | Reconnect with jitter; public ingress down; workloads keep running |
| Node offline | Lease expiry, missed heartbeats | Mark offline; do not delete workloads; alert |
| Broker crash | systemd restart | Restart; reconcile from SQLite; do not recreate healthy workloads |
| Runtime crash | Health check | Restart the runtime; recreate only the affected workload |
| Duplicate command | Idempotency key, generation | No-op |
| Split brain | Two brokers claim a machine | Credential and key revocation, epoch bump, relay close |
| D1 write failure | API error | 503; retry with the same idempotency key |
| DO unavailable | API error | Queue command in D1 outbox; data plane unaffected |
| R2 unavailable | Upload error | Buffer locally; retry; do not block workload start |
| Disk failure | SMART, checksum, scrub | Mark degraded; rebuild from replica; alert |
| Site partition | Heartbeats, replication lag | Keep the site running; do not auto-fail over stateful workloads |

## 9. Storage

### 9.1 Tiers

| Tier | Backend | Use | Durability |
| --- | --- | --- | --- |
| Hot | Local NVMe plus S3 gateway | Workload data, images, caches, CI artifacts | Replicas or erasure coding within the site |
| Warm | R2 Standard | Offsite copy, control-plane artifacts, logs, snapshots | R2 durability, versioning |
| Cold | R2 Infrequent Access | Backups and old logs | 30-day minimum, retrieval fee |

### 9.2 Buckets and tenancy

- One bucket per tenant per site, or one bucket with a tenant prefix.
- Per-tenant credentials; the `signer` Worker holds them and mints presigned
  URLs after authorization.
- Presigned URLs point at `s3-<site>.<domain>` through the relay. The S3
  signature is the authorization; do not put Access in front of that hostname.
- Server-side encryption with per-tenant keys. Keys live in the control plane's
  secret store or a local KMS, never in object metadata.
- Versioning for backups and audit artifacts; lifecycle rules to R2 IA.
- Object lock or a WORM bucket for audit exports.

### 9.3 Replication and restore

- A replication worker on each site walks new objects, verifies the checksum and
  uploads to R2 with the same key and metadata.
- Replication lag is a metric. Alert when it exceeds the RPO.
- Restore is tested: restore a snapshot or prefix to a new bucket, verify
  checksums, then cut over.
- A monthly drill restores a random object and a D1 export and compares hashes.

### 9.4 Control-plane backups

- D1: daily export to R2 plus Time Travel. Encrypted and versioned.
- DO: reconstructible from D1 and node SQLite; daily snapshot for faster recovery.
- R2: versioning and object lock on audit and backup prefixes.
- Configuration: Git, with secrets in the secret store.

RPO and RTO:

- Control-plane metadata: RPO 15 minutes, RTO 2 hours.
- Tenant objects: RPO 15 minutes offsite, RTO 4 hours for a site loss.
- Audit: RPO 5 minutes, immutable export.

## 10. Security summary

The full threat model is in `docs/security.md`. Load-bearing controls:

- Access JWT validation at the Worker with issuer and audience checks.
- Signed machine envelopes, pinned keys, 24-hour credentials and revocation.
- One outbound WebSocket and one identity per node. No inbound port, no SSH.
- Default-deny egress per workload.
- MicroVM isolation for untrusted code; containers for trusted code; workerd
  only inside a VM or dedicated machine for hostile code.
- Signed images and bundles, digest pinning, SBOM.
- Plaintext secrets never in D1, R2, logs or the SPA.
- Hash-chained audit with a WORM export.
- Two-person approval for destructive operations.

## 11. Operations and SLOs

### 11.1 Enrollment and upgrades

- Enrollment is a Workflow with a visible checklist.
- Broker upgrades are staged: one canary per site, then 10%, then the rest.
- Binaries are signed and verified before execution. Keep the last two versions.
- A failed health check rolls back.

### 11.2 Observability

- OpenTelemetry in the broker and control plane.
- Metrics: local Prometheus text on the Unix socket; pushed aggregates to the
  control plane. Do not send raw samples to D1.
- Logs: local files, uploaded to R2 in chunks; index in D1.
- Traces: OTLP endpoint, adaptive sampling.
- Alerts: lease expiry, node offline, disk > 85%, replication lag, credential
  expiry, failed reconciles, queue depth, unusual egress, audit chain failure.

### 11.3 SLOs

| SLO | Target | Measurement |
| --- | --- | --- |
| Control-plane API availability | 99.9% monthly | Successful requests, excluding 4xx |
| Dashboard availability | 99.9% monthly | Synthetic check from two regions |
| Heartbeat freshness | 95% under 30 s | Broker and DO timestamps |
| Container start, cached image | p95 under 10 s | Broker operation log |
| Worker start, warm | p95 under 1 s | workerd log |
| Sandbox start, warm pool | p95 under 1 s | Broker operation log |
| New node provisioning | p95 under 10 min | Workflow timestamps |
| Log delivery to R2 | p95 under 10 s | Chunk timestamp vs upload |
| Control-plane metadata RPO | 15 min | Backup timestamps |
| Tenant object offsite RPO | 15 min | Replication lag |
| Audit export RPO | 5 min | Export cursor |

These are targets, not measured results.

### 11.4 Capacity

- Reserve 20% headroom per node for the broker, workerd, images and snapshots.
- Cap containers by CPU and memory, not by count.
- Keep a warm image cache for the top 10 images per site.
- Track storage growth per tenant; alert at 70% and 85%.
- On `lite`, reserve 128 MiB for the broker and workerd; reject workloads that
  do not fit.

### 11.5 Incident response

1. Revoke the machine credential and key; close the relay socket.
2. Cordon the node; do not place new workloads.
3. Snapshot local state and logs before rebuilding.
4. Rotate any secret the node could reach.
5. Rebuild from a known image; re-enroll with a new identity.
6. Record timeline, root cause and the control that prevents recurrence.

## 12. Data model

Logical schema, not final DDL. Every mutable table has `created_at`,
`updated_at`, `created_by` and `generation` where relevant.

```
accounts(id, name, plan, created_at)
users(id, account_id, email, name, status)
memberships(user_id, account_id, role, tenant_scope)
tenants(id, account_id, name, isolation_level, quota_json)
sites(id, account_id, name, region, network_cidr, storage_class)
machines(id, site_id, name, hostname, profile, labels_json, capacity_json,
         capabilities_json, status, transport, relay_id, broker_version,
         key_id, last_seen_at, drain_state, created_at)
runtimes(id, machine_id, kind, version, capabilities_json, status)
workloads(id, tenant_id, site_id, machine_id, name, kind, artifact,
          config_json, resources_json, volumes_json, network_json,
          placement_json, desired_state, generation, observed_state,
          observed_generation, lease_epoch, created_at)
volumes(id, tenant_id, site_id, machine_id, name, size_gb, storage_class,
        snapshot_id, attached_to, status)
buckets(id, tenant_id, site_id, name, backend, replication_target, status)
routes(id, tenant_id, machine_id, hostname, path_prefix, target, access_app_id,
       tls_mode)
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

Indexes:

- `workloads(tenant_id, desired_state)`
- `workloads(machine_id, observed_generation)`
- `machines(site_id, status)`
- `audit_events(account_id, created_at)`
- `outbox(published_at)` partial index for unpublished rows
- `jobs(tenant_id, status, created_at)`

D1 size strategy: one database until 5 GB, then shard by account or tenant
group. Cross-shard reporting uses R2 exports, not cross-database joins. The 10 GB
per-database limit cannot be raised.

## 13. API

### 13.1 Conventions

- Base path: `/api/v1`.
- JSON, UTF-8, `snake_case`.
- OpenAPI 3.1 at `/api/v1/openapi.json`.
- Auth: Access JWT for the SPA, bearer token for API clients, signed envelopes
  for nodes.
- Errors: `application/problem+json` with `type`, `title`, `status`, `detail`,
  `instance`, `code`, `retryable`, `request_id`.
- Idempotency: `Idempotency-Key` on POST; key and body hash stored for 24 hours.
- Concurrency: `ETag` and `If-Match`; stale writes return 409.
- Pagination: `?limit=50&cursor=...`, response `next_cursor`.
- Async: `202` with `operation_id` and `Location`.
- Events: SSE for operations and logs; WebSocket for exec.
- Rate limits per token and tenant; `429` with `Retry-After`.

### 13.2 Resources

```
GET    /api/v1/sites
POST   /api/v1/sites
GET    /api/v1/sites/{id}
PATCH  /api/v1/sites/{id}

GET    /api/v1/machines
POST   /api/v1/machines
GET    /api/v1/machines/{id}
PATCH  /api/v1/machines/{id}
DELETE /api/v1/machines/{id}
POST   /api/v1/machines/{id}/actions

GET    /api/v1/workloads
POST   /api/v1/workloads
GET    /api/v1/workloads/{id}
PATCH  /api/v1/workloads/{id}
DELETE /api/v1/workloads/{id}
POST   /api/v1/workloads/{id}/actions
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

### 13.4 Example: unsupported binding

```http
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/problem+json

{
  "type": "https://cfb.example/errors/unsupported-binding",
  "title": "Unsupported binding",
  "status": 422,
  "detail": "Binding AI is not supported on profile lite.",
  "instance": "/api/v1/workloads/w_789",
  "code": "unsupported_binding",
  "retryable": false,
  "request_id": "req_..."
}
```

## 14. Broker protocol

### 14.1 Envelope

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
`metric`, `error`, `exec_open`, `exec_data`, `exec_close`, `apply`, `route`.

### 14.2 Endpoints over the relay

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
POST /v1/apply
POST /v1/route
```

### 14.3 State machine

```
pending -> applying -> running -> stopping -> stopped -> deleting -> deleted
                  \-> failed
                  \-> degraded
```

Transitions are idempotent. A reconcile for an already applied generation returns
the current observed state without calling the runtime.

### 14.4 Heartbeats and leases

- Heartbeat every 15 seconds.
- Liveness deadline 60 seconds; four missed heartbeats marks the node offline.
- The fencing epoch does not expire.
- Heartbeat payload: broker version, profile, capabilities, uptime, load,
  capacity, observed digest, runtime health, storage health, clock skew, last
  command id.
- The heartbeat carries a digest, not the full workload list.

### 14.5 Versioning

- `protocol_version` and `broker_version` in every handshake.
- The control plane supports the current and previous protocol major versions.
- An old broker is drained and upgraded, not silently ignored.
- Upgrades are signed, canary-first and reversible.

## 15. Risks and open questions

| Risk | Impact | Mitigation | Status |
| --- | --- | --- | --- |
| Relay DO message billing | High-volume logs and ingress cost more than a tunnel | Native tunnel option; batch frames; keep bulk data on S3 | Open |
| WSS throughput and latency | Interactive exec and large responses are slower than QUIC | Measure; cap frame size; credit flow control; native tunnel | Open |
| workerd is not a sandbox | Host compromise from hostile code | MicroVM for untrusted; `lite` trusted only | Accepted |
| Binding shim semantics | D1/KV/Queues behave differently from Cloudflare | Conformance suite; fail closed; mark unsupported | Open |
| Native tunnel incomplete | No `cloudflared` replacement yet | Relay default; cfrs research; keep cloudflared as fallback | Open |
| Rootless limits | No netns, KVM or cgroups on some hosts | Capability negotiation; `lite` profile | Accepted |
| S3 gateway correctness | Data loss or incompatible clients | Narrow subset; checksum tests; delegate to SeaweedFS/RustFS | Open |
| Cloudflare API drift | Wrangler config or binding changes | Pin supported versions; conformance gate | Open |
| D1 10 GB per database | Metadata outgrows one DB | Shard by tenant; export reporting to R2 | Open |
| DO soft 1,000 req/s | Busy coordinator or relay saturates | Shard by machine; aggregate heartbeats; backpressure | Open |
| Access limits | 500 apps, 50 service tokens | One console app; tenant auth in API | Open |
| Licensing | AGPL/BUSL/SSPL blocks a hosted service | Apache/MIT/0BSD first; legal review | Open |
| Operator time | Self-hosting becomes a second job | Automation, SLOs, start with one site | Open |
| Data durability | Site loss loses data | 3-2-1, R2 offsite, restore drills | Open |
| Supply chain | Malicious image or bundle | Signatures, digests, SBOM, admission | Open |

Open questions that need a prototype:

1. What is the throughput and latency of the relay WebSocket for 1 MiB
   responses and for exec streams?
2. Does a WSS relay survive Cloudflare's idle and message limits for a fleet of
   1,000 nodes?
3. Can the generated binding shim match D1 SQL semantics closely enough for
   real Workers code?
4. How much memory and CPU does workerd use per idle tenant on a 512 MB node?
5. What is the cold and warm start time for rootless Podman on the target
   hardware?
6. Can `cfrs` be extended to a full tunnel registration and serve loop without
   the Cloudflare-internal roots?
7. What is the S3 gateway throughput on local NVMe versus SeaweedFS?
8. Does the native tunnel work through a proxy that only allows HTTP CONNECT on
   443?
9. What is the restore time for a 1 TB bucket from R2?
10. Which Workers APIs used by real applications are missing from the supported
    subset?

## 16. References

Cloudflare, read 2026-10-29. Captures in `evidence/cf-docs/`.

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

Projects, read 2026-10-29. Captures in `evidence/reads/`.

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
- cfrs: https://github.com/talaria0101/cfrs
- mirrord: https://github.com/metalbear-co/mirrord
- ServerBee: https://github.com/ZingerLittleBee/ServerBee
- homelable: https://github.com/Pouzor/homelable
- CoreObs: https://github.com/crocofied/CoreObs
- omnyssh: https://github.com/timhartmann7/omnyssh
- Pangolin: https://github.com/fosrl/pangolin
- NetBird: https://github.com/netbirdio/netbird
