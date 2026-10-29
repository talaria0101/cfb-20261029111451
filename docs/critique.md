# Critique

This page argues against the design. Each objection is stated in its strongest
form, then answered. Where the answer is weak, that is said.

The purpose is not to collect objections and dismiss them. It is to find the one
that would make the project fail, before the build does.

## 1. "You are rebuilding a cloud with a small team"

**Objection.** Cloudflare's runtime is not one product. It is a scheduler, a
hypervisor fleet, a global network, a storage system, an identity system, a
billing system, a control plane and years of operational scar tissue. The design
reuses Incus, Firecracker, workerd and SeaweedFS, but the integration, the
leases, the tenant model, the audit, the upgrades and the 3 a.m. incidents are
still yours. A small team is signing up to operate a mini-cloud.

**Response.** The design deliberately narrows the problem. It does not target
global anycast, 100,000-instance bursts, or a general-purpose public cloud.
It targets a known fleet, likely tens to low hundreds of machines, with one
operator and a small number of tenants. The reused components carry the parts
that are years of work: the hypervisor, the runtime, the storage engine, the
container manager. The new code is the control plane and the agent, which is
where the specific requirements live.

That is still a real amount of code. The roadmap estimates 25 to 40 weeks for
one generalist to reach a hardened multi-site system. If the actual requirement
is "run one heavy CI job and a few containers", that estimate is too high and
the answer is a much smaller tool, not this design.

**Residual risk.** High. The estimate is uncertain, and the operator is a single
point of failure.

**What would change the answer.** If the requirement is only "replace Workers CI
for one repo", use Fireactions and stop. If it is "replace R2 for 50 TB", use
SeaweedFS and stop. The full CFB design is justified only by the combination of
containers, sandboxes, Workers, storage and CI under one dashboard.

## 2. "The dashboard is on the thing you are trying to escape"

**Objection.** The requirement says the dashboard must be hosted on Cloudflare.
That keeps a hard dependency on the account, the Access policy, the Workers
runtime, D1, R2 and the tunnel control plane. A Cloudflare outage or account
suspension takes away the manager, and possibly the ingress. This is not
independence; it is a different lock-in.

**Response.** The dependency is real and accepted. The design reduces its blast
radius: the data plane keeps running without the control plane, the agent has a
local desired-state cache, and the public ingress path is separate from the
control path. The control plane is a management dependency, not an execution
dependency.

The lock-in is narrower than it looks. The control plane is a Workers app with
D1 and R2; a port to another host is a rewrite of the hosting layer, not of the
data model or the agent protocol. The data plane is portable: Incus,
Firecracker, SeaweedFS and workerd run anywhere. The evidence is in the
interfaces: `PrivateTransport`, `Driver`, and the OpenAPI surface.

**Residual risk.** Medium. A Cloudflare account compromise is a management
outage, and an account suspension is worse. Keep a documented manual recovery:
the agent can be driven by a local `cfbctl` with a hardware key, and the tunnel
can be replaced by a self-hosted relay.

**What would change the answer.** If the operator cannot accept any Cloudflare
dependency for management, the requirement itself has to change. Then the
dashboard moves to the data plane, and the bootstrap problem gets harder, not
easier.

## 3. "Workers VPC is beta, and the limits are tight"

**Objection.** The control path depends on a beta feature. Tunnels are capped at
1,000 per account unless you pay $4.50 each per month. Access is capped at 500
applications and 50 service tokens. Mesh is capped at 50 nodes. D1 is capped at
10 GB per database. DO is soft-capped at 1,000 requests per second per object.
Any of these can become the wall.

**Response.** Each limit has a design response:

- VPC: a relay Durable Object behind the same interface. If VPC changes, the
  relay is already there.
- Tunnels: one per machine up to 1,000. Beyond that, either pay $4.50 per tunnel
  or move to one tunnel per site with per-machine routes, accepting a wider
  blast radius. The cost is visible, not hidden.
- Access apps: one application for the console, tenant authorization in the API.
  Do not create an app per tenant.
- Service tokens: not per machine. Machine identity is an enrolled key and a
  signed request, not an Access service token.
- Mesh nodes: not used. The control path binds `cf1:network`, which reaches
  tunnel routes without a Mesh node, so the 50-node Mesh cap does not bind. The
  binding is account-wide, which is a blast-radius tradeoff, not a node count.
- D1: shard by account or tenant group before 5 GB. Cross-shard reporting goes
  through R2 exports.
- DO: one object per machine, batched heartbeats, hibernation. A busy machine
  can shard its coordinator if needed.

**Residual risk.** Medium. The relay is more code, and the 1,000-tunnel wall is a
real cost cliff. A fleet of 2,000 machines pays $4,500/month in tunnels alone,
which changes the economics.

**What would change the answer.** A measured VPC failure rate or a pricing
change. Both are reasons to keep the transport abstraction, not to remove it.

## 4. "workerd is not a sandbox, and the platform layers are young"

**Objection.** The workerd README says it is not a hardened sandbox. open-compute
is a young single-node reimplementation of a large API surface. celld is pre-1.0,
one application per fleet, with no TLS and no account service. Depending on these
for multi-tenant code is reckless.

**Response.** The design does not use workerd as a boundary. It runs workerd
inside a Firecracker VM or a dedicated machine, and it treats open-compute and
celld as replaceable drivers. The isolation matrix is explicit: untrusted code
goes to a microVM, not to an isolate.

The maturity risk is real. The mitigations are version pinning, canaries, a raw
workerd fallback, and a refusal to put two tenants in one open-compute instance
unless they are in the same trust group. celld is limited to one tenant per fleet
until it supports accounts.

**Residual risk.** High for Workers and DO. A bug in open-compute or celld can
take down a tenant's Workers. That is a single-machine or single-fleet failure
domain, not a cross-tenant data breach, if the isolation rules hold.

**What would change the answer.** If open-compute cannot run the required
bindings reliably, the Workers tier is raw workerd plus a small supervisor, which
is more work and less compatible. If celld cannot meet the DO semantics, the
distributed DO tier is deferred.

## 5. "Multi-tenancy is not a feature you bolt on"

**Objection.** The design talks about tenants, quotas, isolation levels and
billing as if they are a schema. Real multi-tenancy is noisy-neighbor control,
per-tenant rate limits, fair scheduling, cross-tenant data isolation, per-tenant
encryption, per-tenant upgrade windows, and a support model. A small team will
get one of these wrong.

**Response.** The design starts with a small number of tenants and an explicit
isolation level per tenant. It does not promise a public multi-tenant cloud. The
controls that matter are in the schema and the agent: tenant-scoped queries,
per-tenant network namespaces, per-tenant buckets and keys, per-tenant quotas,
and a dedicated-machine option for the highest trust.

The weakest part is fair scheduling. A global scheduler is rejected, so a noisy
tenant can still consume a machine's CPU until cgroups and quotas stop it. The
agent enforces cgroups; the control plane enforces quotas at admission. That is
enough for a small fleet, not for a public cloud.

**Residual risk.** High if the product is offered to untrusted third parties. Low
if the tenants are the operator's own projects.

**What would change the answer.** A requirement to host untrusted third-party
tenants at scale. Then the design needs a real scheduler, per-tenant kernels, and
a compliance program. That is a different product.

## 6. "The agent is a root-level daemon on every machine"

**Objection.** A binary that can create VMs, mount volumes and change network
namespaces is a root-level agent. A vulnerability in it is a host compromise. A
compromised control plane can command it. This is a large attack surface, and it
is the same shape as the agents in every self-hosted PaaS that has been breached.

**Response.** The agent is split into an unprivileged reconciler and a narrow
privileged helper. The helper exposes specific operations, not a shell. The agent
verifies signatures, refuses privileged containers and host mounts, and enforces
local policy even when the control plane asks otherwise. Machine identity is
short-lived and revocable. The tunnel is outbound-only.

That reduces the surface but does not eliminate it. A bug in the helper is a host
compromise. The design accepts this and limits blast radius with one machine per
failure domain, per-tenant isolation, and a rebuild-from-image recovery.

**Residual risk.** High. This is the most security-sensitive code in the system.

**What would change the answer.** A hardware root of trust and a formally
verified helper would reduce the risk, at a large cost. For most small fleets,
careful code review and a minimal helper are the practical answer.

## 7. "The cost story is weaker than it looks"

**Objection.** The cost page shows that at low utilization, with operator time
included, self-hosting loses to Cloudflare. Egress is the only clear win, and
only at high volume. The project may cost more than the bill it replaces.

**Response.** That is the honest result, and it is stated in `docs/cost.md`. The
project is justified by control, capability and egress, not by a universal price
win. The break-even formula is explicit: with $200/month hardware, the machine
matches Cloudflare at about 38% utilization before operator time. Add $500/month
of operator time and the threshold rises above 100%, so this shape never breaks
even. Even with $50/month hardware it needs near 100% utilization.

The workloads that win are the ones in the requirement: heavy CI, long-running
containers, sandboxes, and egress-heavy storage. A light dashboard workload does
not win, and should stay on Cloudflare.

**Residual risk.** Medium. The model depends on inputs the reader supplies, and
the operator-time estimate is easy to understate.

**What would change the answer.** If the measured utilization is below 30%, the
project is a hobby, not a cost saving. Say so and keep the workload on
Cloudflare.

## 8. "Incus is not Kubernetes, and you will miss Kubernetes"

**Objection.** Incus is excellent for machines and containers, but the ecosystem
for CI, service discovery, ingress, secrets, operators and autoscaling is built
around Kubernetes. Choosing Incus means building integrations that already exist.
Incus clustering is per site, so multi-site placement is manual.

**Response.** Incus is the default because it gives containers and VMs with a
clean API and a small operational surface. Kubernetes is an optional escape
hatch for a site that already runs it. The design does not need a service mesh,
an ingress controller or an operator ecosystem; it needs a runtime the agent can
drive.

The integrations that matter are narrow: Fireactions for GitHub runners, a
Woodpecker agent, a load balancer or tunnel route, and metrics. Those do not
require Kubernetes.

**Residual risk.** Medium. Teams with Kubernetes skills will find Incus unfamiliar,
and some workloads will need Kubernetes. The escape hatch exists but adds a
second operational model.

**What would change the answer.** A requirement for Kubernetes-native tooling
across the fleet. Then k3s per site is the default, and Incus is the VM layer
under it.

## 9. "Storage is the hardest operational problem"

**Objection.** SeaweedFS is not a single binary. It has a master, volume servers,
a filer, an S3 gateway and a metadata store. Erasure coding, replication,
rebalancing, disk replacement and cross-cluster replication all have failure
modes. R2's durability is not reproduced by "three replicas and a backup".

**Response.** The design starts with `weed mini` on one machine, which is a
documented single-node production path, and grows to a cluster only when needed.
It keeps R2 as the offsite copy and runs monthly restore drills. It does not
claim R2 durability; it claims a documented, tested RPO and RTO.

The hard part is not the first machine. It is the first disk failure, the first
rebalance and the first restore. Those are in the roadmap and in the runbooks.

**Residual risk.** High. A storage cluster is a distributed system, and data loss
is the one failure that cannot be apologized for.

**What would change the answer.** If the data is irreplaceable and the team
cannot run a storage cluster, keep the primary in R2 and use local storage only
for cache. That is a smaller, safer design.

## 10. "You are replacing one bill with several"

**Objection.** The Cloudflare bill is one line. The self-hosted bill is hardware,
power, bandwidth, colocation, spare parts, monitoring, backups, licenses and
operator time. It is more work to forecast and easier to hide.

**Response.** The cost page makes each line explicit and gives formulas. The
dashboard shows cost per tenant and per machine. The design does not pretend the
bill disappears; it moves it.

**Residual risk.** Medium. Small costs hide in power, replacement and time.

**What would change the answer.** If the operator does not want to forecast
infrastructure, a committed Cloudflare plan is simpler and possibly cheaper.

## 11. "amika already exists, and you are ignoring it"

**Objection.** amika is the closest system to this design, and the user pointed
at it. The design says to read it and not depend on it, which sounds like
not-invented-here.

**Response.** amika's control plane, UI and relay are closed, and its
self-hosting document says there is no supported local CLI path and the future
gateway is single-player with no IAM. That makes it a reference, not a
foundation. The open pieces (`amika-hostd`, the tunnel protocol, the SDKs) are
worth reusing or copying where the licence allows.

The design also reuses far more than it builds: Incus, Firecracker, workerd,
open-compute, celld, SeaweedFS, Woodpecker, Fireactions and cloudflared. The
"build" is the control plane and the agent, which amika does not publish.

**Residual risk.** Low. If amika opens the control plane under a permissive
licence, the decision should be revisited.

**What would change the answer.** A public, Apache-2.0 control plane and relay
from amika with a multi-tenant IAM model. Then forking it is cheaper than
building.

## 12. "CI is a product, not an integration"

**Objection.** Fireactions is GitHub-specific. Woodpecker's autoscaler is not
complete for every backend. CI users expect pipeline syntax, caching, matrix
builds, artifacts, annotations, retries, approvals, and a UI that developers
already know. A thin layer over two runners will not satisfy them.

**Response.** The design does not build a CI engine or a pipeline language. It
integrates the runners people already use and owns the parts Cloudflare's CI
does not give: long jobs, custom hardware, private network access, pool
management, quotas, logs and artifacts. Fireactions runs GitHub Actions YAML
unchanged; Woodpecker runs its own pipelines.

The control plane does not need to be the CI UI. It can link to the runner's UI
and store the job record, logs and artifacts. That is a smaller promise and a
more honest one.

**Residual risk.** Medium. Two integrations, two secret models, two status
models.

**What would change the answer.** A requirement for a unified pipeline language
across providers. That is a product decision, not an infrastructure one.

## 13. "The security model assumes a competent operator"

**Objection.** The controls are only as good as the person running them. Key
rotation, patching, restore drills, audit review and incident response are
operational disciplines. A small team will skip them under pressure.

**Response.** The design makes the disciplines structural where it can: short
credential lifetimes with automatic renewal, a hash-chained audit that fails
closed, a restore drill that is a scheduled Workflow, a janitor that rotates
credentials, and an agent that refuses unsafe configurations. The rest is
documentation and runbooks.

**Residual risk.** High. A missed patch on a hypervisor is a bad month.

**What would change the answer.** A compliance or insurance requirement that
demands a certified operator. Then the design needs an external audit and a
support contract, which is a different cost.

## 14. "The requirement itself may be the wrong shape"

**Objection.** Why must the dashboard be on Cloudflare? If the goal is to escape
Cloudflare's prices, keeping the manager there preserves the dependency and
creates a split-brain. A dashboard on the data plane, reachable through a
tunnel, would be simpler and more independent.

**Response.** This is the user's constraint, and it has a real benefit: the
dashboard is reachable when the machines are not, identity is handled by Access,
and there is no bootstrap problem for the UI. The split is a deliberate
trade: management depends on Cloudflare, execution does not.

If the constraint were removed, the design would put the API and UI on a small
control-plane machine, use the tunnel only for ingress, and keep D1-like metadata
in Postgres. That is a different design with a different failure model.

**Residual risk.** The user owns this trade. It should be re-examined if
Cloudflare pricing or policy changes.

## 15. The objection that would actually kill it

The strongest objection is not any single technical risk. It is the combination:

- a small team,
- a multi-tenant control plane,
- five runtime classes,
- a storage cluster,
- a security boundary that must hold against hostile code,
- and a cost model that only wins at high utilization.

Each is manageable. Together they are a platform organization's workload. The
design survives only if the scope stays narrow: one operator, a known set of
tenants, one or two sites, and a first release that replaces one bill rather than
all of them.

If the project tries to replace all of Cloudflare at once, it will finish none of
it. The roadmap is ordered to avoid that: containers first, storage second,
sandboxes third, Workers fourth, CI fifth.

## 16. What would make me recommend against building it

- The fleet is fewer than five machines and utilization is below 30%. Use
  Cloudflare and a single Fireactions runner.
- The tenants are untrusted third parties at scale. This design is not a public
  cloud.
- The team cannot commit an on-call rotation. A storage cluster and a hypervisor
  fleet need one.
- The workload is bursty and short-lived. Scale-to-zero is Cloudflare's strength.
- The requirement is only "cheaper R2". Use SeaweedFS or RustFS directly; do not
  build a control plane.
- The requirement is only "longer CI". Use Fireactions or Woodpecker directly.

## 17. What would make me recommend building it

- The workload is long-running, egress-heavy, and needs custom hardware or
  private network access.
- The operator already runs machines and accepts the on-call burden.
- The tenants are the operator's own projects, or a small number of trusted
  teams.
- The dashboard-on-Cloudflare requirement is firm, and the value is one place to
  manage a heterogeneous fleet.
- The first release can be scoped to containers plus storage, with Workers and CI
  as later phases.

## 18. Cheapest experiments that would falsify the design

1. Run one heavy CI job on Fireactions for a week. If it is enough, stop; the
   control plane is unnecessary for that requirement.
2. Put 10 TB in SeaweedFS for a month and measure the operational time. If it is
   more than the R2 bill, the storage thesis is weak.
3. Deploy open-compute and run a real Worker with KV and DO. If the compatibility
   gaps are fatal, the Workers tier is not viable.
4. Measure the control-plane cost with 100 machines and hibernating DOs. If DO
   duration dominates, redesign the heartbeat path.
5. Run a restore drill from R2 to local storage. If it takes a day, the RTO is a
   fiction.
6. Try to escape an Incus container from inside, with a known technique, on the
   target kernel. If it succeeds, containers are not a tenant boundary; they were
   never meant to be.
7. Measure operator time for one month. If it exceeds the Cloudflare bill, the
   project is a choice, not a saving.

Each experiment is cheaper than building the wrong system.
