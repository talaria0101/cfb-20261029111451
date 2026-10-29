# Roadmap

Phases are ordered by risk. Each phase has an exit criterion that can be tested,
not a feeling that it is done. No estimates are given: a date without a measured
velocity would be invented.

## Phase 0: Spikes

**Goal.** Retire the assumptions that would change the architecture.

1. Measure relay WebSocket throughput and latency for 1 MiB responses, exec
   streams and 1,000 nodes. Record message counts and DO duration.
2. Measure workerd idle memory and CPU per tenant on a 512 MB and a 1 GB node.
3. Build a throwaway shim for R2 `get`/`put` and D1 `prepare`/`all` and run a real
   Worker against both Cloudflare and the shim. Record the semantic gaps.
4. Measure rootless Podman cold and warm container start on target hardware.
5. Measure Firecracker boot and snapshot restore on a KVM host.
6. Confirm which constrained hosts have KVM, user namespaces, cgroup v2 and
   FUSE. Record the profile each can support.
7. Test the relay through an HTTPS-only proxy and through a network that breaks
   long-lived sockets.
8. Test cfrs edge discovery and header serialization against a real tunnel
   endpoint. Confirm that registration and visitor serving are the only missing
   pieces.

**Exit criteria.** A written result for each spike with a command, a measurement
and a date. A keep-or-replace decision for the relay, workerd shim, Podman,
Firecracker and native tunnel.

**If a spike fails.** Relay fails: investigate native tunnel or a self-hosted
relay. workerd shim fails: narrow the supported subset and fail closed. KVM is
absent: no untrusted sandboxes on that host. Podman is unavailable: `lite` only.

## Phase 1: Control plane skeleton

**Goal.** A deployable Cloudflare app with identity, inventory, audit and a fake
machine driver. No real nodes yet.

- Vite SPA on Workers static assets with SPA fallback.
- Hono API with OpenAPI 3.1, problem+json, idempotency and ETag.
- Access JWT validation with issuer and audience checks.
- D1 schema: accounts, users, memberships, tenants, sites, machines, audit,
  outbox.
- RBAC and tenant scoping in every query.
- Hash-chained audit with a verification job.
- Fake machine driver for local development.
- CI: type check, unit tests, migration test, staging deploy.

**Exit criteria.** A user logs in through Access, creates a tenant and a site, and
sees an audit record. A second tenant cannot read the first tenant's resources by
id. The audit chain verifies and a tampered row fails. A clean staging deploy is
reproducible from one documented command.

## Phase 2: Broker transport and enrollment

**Goal.** One `cfbd` binary connects to the relay, enrolls, heartbeats and stores
local state. No workloads yet.

- Ed25519 key generation, one-time join token, 24-hour credential.
- Signed envelope on every request, replay cache, clock-skew check.
- Relay Worker and per-machine DO with WebSocket hibernation.
- Stream framing, credit flow control, half-close, reconnect with jitter.
- Local SQLite: desired, observed, operations, events.
- Local Unix socket API with peer credential checks.
- `cfbd status`, `cfbd logs`, `cfbd drain`, `cfbd rotate-key`.
- Profile detection and capability report.

**Exit criteria.** A node enrolls in under 10 minutes with no SSH. A revoked key
cannot reconnect. Killing the control plane does not stop the broker. A duplicate
command is a no-op. A stale lease epoch is rejected. The broker has no TCP
listener in `lite`.

## Phase 3: workerd and the binding shim

**Goal.** Deploy a Worker bundle to a node and reach it through the relay.

- Parse `wrangler.jsonc`, reject unsupported bindings with a named error.
- Generate `workerd.capnp` and the R2/D1/KV/Queues shim.
- Version directories, content-addressed bundles, signature verification.
- Atomic route swap after a smoke request; keep the previous version.
- Relay ingress: host and path to a local Unix socket.
- Logs and metrics into the dashboard.
- Conformance suite: the same Worker on Cloudflare and the node.

**Exit criteria.** A Worker deployed from CI is reachable through the relay with
Access in front. A rollback restores the previous bundle in under 30 seconds. A
tampered bundle is rejected. An unsupported binding is rejected at upload. The
conformance suite reports a per-primitive result, including failures.

**This is the first useful product for a `lite` node.** It replaces a small
Workers project and a static site with no root and no inbound port.

## Phase 4: Containers

**Goal.** Rootless OCI containers on a `standard` node.

- Podman driver over the user socket: create, start, stop, delete, stats, logs,
  exec.
- Image pull by digest, signature verification, admission policy.
- Per-workload network namespace and default-deny egress.
- CPU and memory limits through cgroups v2 when delegated, rlimits otherwise.
- Route integration for container HTTP services.
- Warm image cache for the top images per site.

**Exit criteria.** A container is created, stopped, started and deleted from the
SPA. A container cannot reach the broker socket, the metadata address or another
tenant. Killing the control plane does not stop it. Restarting the broker
reconciles without duplicating it. A bad signature is rejected.

## Phase 5: Storage

**Goal.** Tenant buckets and volumes with an offsite copy.

- Local S3 gateway: Head, Get, Put, Delete, ListV2, multipart.
- Presigned URL minting in the signer Worker and verification in the broker.
- SeaweedFS or RustFS backend for multi-node sites.
- Volume create, attach, detach and snapshot.
- Replication to R2 with checksums and lag metrics.
- Lifecycle to R2 IA, retention and delete.
- Restore drill script and a dashboard action.
- Usage metering per tenant.

**Exit criteria.** A tenant uploads and downloads through a presigned URL without
seeing a root credential. A deleted object is gone per policy. A restore of a
random object matches the checksum. A stopped storage node does not lose data.
Replication lag is visible and alerts above the RPO.

## Phase 6: Sandboxes and VMs

**Goal.** Firecracker sandboxes and Incus VMs in the `privileged` profile.

- Firecracker driver: create, exec, file read/write, port forward, snapshot,
  restore, destroy.
- Jailer configuration and a minimal rootfs.
- Warm pool per image.
- TTL and idle timeout enforcement.
- Egress proxy and default-deny policy.
- Incus VM driver for semi-trusted workloads.
- Dashboard: sandbox list, TTL, live CPU and memory, kill.

**Exit criteria.** A warm sandbox starts in under 1 second p95 and cold in under 3
seconds p95 on target hardware. A sandbox cannot reach the host, the broker, the
metadata address or another sandbox. A snapshot restores to a working filesystem.
An expired sandbox is destroyed. A broker restart cleans orphan VMs.

## Phase 7: CI

**Goal.** Replace Workers CI for one real repository.

- GitHub Actions runner or Fireactions pool: config, JIT tokens, lifecycle.
- Woodpecker agent for Gitea or Forgejo.
- Webhook verification and job records.
- Log streaming to R2 and the SPA.
- Artifact manifest and retention.
- Concurrency, timeout and quota enforcement.
- Cache bucket per tenant.

**Exit criteria.** A real repository runs a job longer than 20 minutes on an owned
runner. Logs are live and retained, and a secret in a log is masked. A cancelled
job destroys its runner within 60 seconds. A job cannot read another tenant's
cache or artifacts.

## Phase 8: Hardening

**Goal.** Operable by someone who did not build it.

- Multi-site placement, site-loss drills, cross-site restore.
- Broker self-update with canary and rollback.
- Break-glass path, tested quarterly.
- Backup and restore automation with a monthly drill.
- Load test at 10x the expected fleet.
- Penetration test and a fix round.
- Runbooks for the top 10 incidents.
- Cost dashboard and quota alerts.
- Licence review of every shipped dependency.
- Install, upgrade, recover, rotate and decommission documentation.

**Exit criteria.** A new operator enrolls a machine and deploys a workload using
only the documentation. A site is lost and restored within the documented RTO. A
failed broker upgrade rolls back. The audit chain survives a D1 restore and
verifies against the R2 anchor.

## MVP

Phase 1 plus Phase 2 plus Phase 3 plus the smallest slice of Phase 4:

- One Cloudflare account, one tenant, one site, up to 10 nodes.
- Access login, RBAC, audit.
- Enroll a node with no SSH.
- Deploy a Worker bundle through the relay with the supported shim.
- Create, start, stop and delete a rootless container.
- One bucket per tenant with presigned access and an R2 replica.
- Live status, logs and a basic cost view.

That replaces a small Workers project, a container and a bucket. It does not
replace sandboxes, CI or a global edge.

## Sequence rationale

1. Control plane first: the runtimes are replaceable, the protocol and data model
   are not.
2. Broker before workloads: nothing else can be trusted until enrollment,
   transport and local state work.
3. Workers before containers: `lite` is the hardest constraint and the clearest
   differentiator.
4. Storage before sandboxes: bundles and logs need a place to live, and storage is
   the easiest bill to replace.
5. CI last: Fireactions and Woodpecker run standalone, so CI is not blocked by
   the control plane.

## What not to build

- A hypervisor, S3 server, CI engine, Durable Object runtime, container runtime,
  service mesh or global scheduler.
- A billing marketplace, tenant self-signup or public status page in v1.
- A Kubernetes distribution.
- A custom VPN or tunnel protocol.
- A Windows guest story in v1.
- A multi-region active-active control plane in v1.

## Success metrics

- Enroll a node in under 10 minutes.
- First workload in under 5 minutes after enrollment.
- Control-plane availability 99.9% monthly.
- Workload survival during a control-plane outage: 100%.
- Restore drill success: 100% of monthly drills.
- Audit chain verification: 100% of daily checks.
- Cost per workload lower than the Cloudflare equivalent for the target shape,
  including operator time, or the design says plainly that it is not.
