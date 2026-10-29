# Roadmap

The roadmap is ordered by risk and by the first bill it removes. Each phase has
an exit criterion that can be tested, not a feeling that it is done.

## Phase 0: Spikes and decisions (2 weeks)

**Goal.** Retire the assumptions that would change the architecture.

**Work.**

1. Prove Workers VPC reaches an agent through a tunnel with a `cf1:network`
   binding. Measure request latency and log streaming.
2. Prove or disprove WebSocket upgrades through VPC `fetch()`. If they fail,
   build the relay path first.
3. Measure Incus container create, start and stop on the target hardware, warm
   and cold.
4. Measure Firecracker boot and snapshot restore on the target hardware.
5. Measure `cloudflared` throughput and CPU at 10 Mbit/s, 100 Mbit/s and 1
   Gbit/s.
6. Install open-compute on one machine, deploy a hello-world Worker, and
   measure idle memory and cold start.
7. Install celld on two machines with an S3 bucket and verify DO failover.
8. Confirm the target machines have KVM and the required kernel modules.

**Exit criteria.**

- A written result for each spike with a command, a measurement and a date.
- A decision to keep or replace Workers VPC, Incus, Firecracker, open-compute
  and celld.
- No open question in `docs/spec.md` section 19 that blocks Phase 1.

**If a spike fails.**

- VPC fails: implement the relay first and treat VPC as an optimization.
- WebSocket fails: exec goes through the relay from day one.
- KVM is unavailable: sandboxes use Incus VMs or gVisor, with a documented
  isolation downgrade.
- open-compute is unusable: raw workerd plus a supervisor, or defer Workers to
  Phase 5.

## Phase 1: Control plane skeleton (4 to 6 weeks)

**Goal.** A deployable Cloudflare app with identity, inventory and audit. No
machines yet.

**Deliverables.**

- Vite SPA served from Workers static assets with SPA fallback.
- Hono API with OpenAPI 3.1, problem+json, idempotency and ETag.
- Access JWT validation with issuer and audience checks.
- D1 schema: accounts, users, memberships, tenants, sites, machines, audit,
  outbox.
- RBAC and tenant scoping enforced in every query.
- Hash-chained audit with a verification job.
- A fake machine driver for local development and tests.
- CI: type check, unit tests, a D1 migration test, a deploy to a staging account.

**Exit criteria.**

- A user can log in through Access, create a tenant and a site, and see an audit
  record for each action.
- A second tenant cannot read the first tenant's resources, including by direct
  id access.
- The audit chain verifies, and a tampered row fails verification.
- A staging deploy is reproducible from a clean account with a documented
  command.

**What not to build yet.** Real machines, real runtimes, billing, a tenant
portal, multi-region.

## Phase 2: One machine, containers (6 to 8 weeks)

**Goal.** Enroll one machine and manage Incus containers from the SPA.

**Deliverables.**

- Host agent with signed enrollment and requests, heartbeat and a local SQLite
  cache. Signatures authenticate both paths; no mTLS on the Cloudflare paths.
- Incus driver: create, start, stop, delete, stats, logs.
- Per-machine DO with lease, fencing epoch, command queue and live events.
- One-time join token and a signed install script.
- Tunnel creation and a `cf1:network` binding.
- Desired-state reconciliation with generations.
- Dashboard: machine detail, workload list, create/stop/delete, live status.
- A relay transport behind the same interface (even if unused).

**Exit criteria.**

- A new machine is enrolled from the dashboard in under 10 minutes with no SSH.
- A container is created, stopped, started and deleted from the SPA.
- Killing the control plane does not stop a running container; restarting the
  control plane reconciles state without duplicating it.
- Revoking the machine credential and key blocks the agent within one heartbeat.
- A duplicate `reconcile` command is a no-op.
- A stale lease epoch is rejected.

**This is the first usable product.** It replaces "SSH and docker run" with a
dashboard, an audit trail and a lease.

## Phase 3: Storage and volumes (4 to 6 weeks)

**Goal.** Tenant buckets and volumes, with an offsite copy.

**Deliverables.**

- SeaweedFS install and lifecycle on a machine.
- Bucket create/delete, per-tenant credentials, presigned URLs.
- Volume create/attach/detach/snapshot on Incus.
- Replication worker from local S3 to R2 with checksums.
- Lifecycle policy: hot to R2 IA, retention, delete.
- Restore drill script and a dashboard action.
- Usage metering per tenant.

**Exit criteria.**

- A tenant can upload and download through a presigned URL without seeing the
  root credential.
- A deleted object is gone from the local store and the replica according to
  policy.
- A restore of a random object from R2 matches the local checksum.
- Replication lag is visible and alerts above the RPO.
- A disk failure on a three-node cluster does not lose data (test with one node
  stopped).

## Phase 4: Sandboxes (4 to 6 weeks)

**Goal.** Firecracker sandboxes with a small API and a warm pool.

**Deliverables.**

- Firecracker driver: create, exec, file read/write, port forward, snapshot,
  restore, destroy.
- Jailer configuration and a minimal rootfs.
- Warm pool per image with a target size.
- TTL and idle timeout enforcement.
- Egress proxy and default-deny policy.
- SDK endpoint compatible with the shape in `docs/spec.md` section 11.3.
- Dashboard: sandbox list, TTL, live CPU/memory, kill.

**Exit criteria.**

- A sandbox starts from a warm pool in under 1 second p95 on the target
  hardware, and cold in under 3 seconds p95.
- A sandbox cannot reach the host agent, the metadata address, or another
  sandbox.
- A snapshot restores to a working filesystem.
- An expired sandbox is destroyed and its resources are reclaimed.
- Killing the agent does not leave orphan VMs after restart (reconcile cleans
  them).

## Phase 5: Workers and Durable Objects (4 to 6 weeks)

**Goal.** Deploy a Worker bundle to a machine and reach it through the tunnel.

**Deliverables.**

- open-compute install, lifecycle and health.
- Bundle upload to R2, signature verification, local content-addressed store.
- Deploy, promote and rollback through the agent.
- Route creation on the tunnel and Access policy.
- Logs and metrics from open-compute into the dashboard.
- celld evaluation for distributed DO; a single-tenant fleet if it passes.
- A raw workerd fallback path for a binding that open-compute lacks.

**Exit criteria.**

- A Worker deployed from CI is reachable at a hostname with Access in front.
- A rollback restores the previous bundle in under 30 seconds.
- A tampered bundle is rejected by the agent.
- A Worker cannot reach another tenant's KV or R2 namespace.
- DO state survives an agent restart (single node) and a celld node loss (fleet),
  with the documented caveats.

## Phase 6: CI (3 to 4 weeks)

**Goal.** Replace Workers CI for one real repository.

**Deliverables.**

- Fireactions integration for GitHub: pool config, JIT tokens, runner lifecycle.
- Woodpecker integration for Gitea/Forgejo.
- Webhook verification and job records.
- Log streaming to R2 and the SPA.
- Artifact manifest and retention.
- Concurrency, timeout and quota enforcement.
- A cache bucket per tenant.

**Exit criteria.**

- A real repository runs a job longer than 20 minutes on an owned runner.
- Logs are visible live and after the job, and a secret in a log is masked.
- A cancelled job destroys its runner within 60 seconds.
- A job cannot read another tenant's cache or artifacts.
- Queue wait time and run time are visible per repo.

## Phase 7: Hardening and multi-site (ongoing)

**Goal.** Make it operable by someone who did not build it.

**Work.**

- Multi-site placement, site loss drills, and cross-site restore.
- Agent self-update with canary and rollback.
- Break-glass path, tested quarterly.
- Backup and restore automation with a monthly drill.
- Load tests at 10x the expected fleet size.
- Penetration test and a fix round.
- Runbooks for the top 10 incidents.
- Cost dashboard and quota alerts.
- Licence review of every dependency that ships.
- Documentation: install, upgrade, recover, rotate, decommission.

**Exit criteria.**

- A new operator can enroll a machine and deploy a workload using only the
  documentation.
- A site can be lost and restored within the documented RTO.
- An agent upgrade rolls back automatically on a failed health check.
- The audit chain survives a D1 restore and verifies against the R2 anchor.

## MVP definition

The MVP is Phase 1 plus Phase 2 plus the smallest useful slice of Phase 3:

- One Cloudflare account, one tenant, one site, up to 10 machines.
- Access login, RBAC, audit.
- Enroll a machine with no SSH.
- Create, start, stop and delete Incus containers from the SPA.
- One bucket per tenant with presigned access and an R2 replica.
- Live status, logs and a basic cost view.

That MVP replaces a meaningful part of Containers and R2 for a small fleet. It
does not replace Workers, DO or CI yet.

## Sequencing rationale

1. **Control plane first, because it is the product.** The runtimes are
   replaceable; the control plane, the agent protocol and the data model are not.
2. **Containers before sandboxes, because containers are the common case.**
   Sandboxes are the hardest isolation story and should not block the first
   useful release.
3. **Storage before Workers, because Workers need a place to put bundles and
   logs.** It is also the easiest bill to replace.
4. **Workers before CI, because CI can run without the control plane.** Fireactions
   and Woodpecker work standalone; Workers on owned hardware needs the agent,
   routing and bundle pipeline.
5. **Hardening last, but not optional.** A system that works in a demo and fails
   at 3 a.m. is not a replacement for Cloudflare.

## What not to build

- A hypervisor, an S3 server, a CI engine, a Durable Object runtime, a container
  runtime, a service mesh, or a global scheduler.
- A billing marketplace, a tenant self-service signup, or a status page in v1.
- A Kubernetes distribution.
- A custom VPN or tunnel protocol. Use `cloudflared`; add a relay, not a new
  protocol.
- A Windows guest story in v1.
- A multi-region active-active control plane in v1.

## Staffing and calendar

These are estimates, not commitments. They assume one experienced generalist
working full time, with access to at least two machines and a Cloudflare account.

| Phase | Estimate | Confidence |
| --- | --- | --- |
| 0 | 2 weeks | Medium |
| 1 | 4 to 6 weeks | Medium |
| 2 | 6 to 8 weeks | Low |
| 3 | 4 to 6 weeks | Medium |
| 4 | 4 to 6 weeks | Low |
| 5 | 4 to 6 weeks | Low |
| 6 | 3 to 4 weeks | Medium |
| 7 | ongoing | n/a |

The low-confidence phases depend on the spikes in Phase 0. If Firecracker or
celld does not behave, the phase grows.

## Success metrics

- Time to enroll a machine: under 10 minutes.
- Time to first workload: under 5 minutes after enrollment.
- Control-plane availability: 99.9% monthly.
- Workload survival during a control-plane outage: 100%.
- Restore drill success: 100% of monthly drills.
- Audit chain verification: 100% of daily checks.
- Cost per workload: lower than the Cloudflare equivalent for the target shape,
  including operator time, or the project is honest that it is not.
