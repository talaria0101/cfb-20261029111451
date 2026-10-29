# Reference deployments

Three concrete shapes, from one machine to a CI-heavy site. These are design
recipes, not tested installers. Every command must be checked against the version
of the tool you actually install.

The common rule: the control plane is created once on Cloudflare, each machine is
enrolled once, and every later change goes through the dashboard or the API.

## 1. Topology A: one machine

Use this to replace a single Cloudflare Containers workload or a single R2 bucket.

```
Cloudflare
  console Worker (SPA + API)
  D1, R2, one DO per machine, Queues, Workflows
  Tunnel: cfb-home-1
        |
        | outbound QUIC 7844
        v
  Machine: 16 vCPU, 64 GiB, 2 TB NVMe
    cloudflared
    cfb-agent
    Incus (containers + VMs)
    SeaweedFS (weed mini, S3 gateway on private net)
    open-compute in an Incus VM (optional)
```

What runs where:

- `cloudflared` and `cfb-agent` are systemd services.
- Incus is the container and VM runtime. The agent talks to the Incus Unix
  socket.
- SeaweedFS runs in `weed mini` mode. Its S3 gateway is reached through a
  dedicated tunnel hostname, for example `s3-<site>.<domain>`, and presigned
  URLs point at that hostname. The agent's own port is never on that hostname.
  Do not put Access in front of the S3 hostname: a presigned request carries no
  Access cookie or service token, so the S3 signature is the authorization.
- open-compute is optional. If present, it runs inside an Incus VM, one VM per
  tenant or trust group, and the agent routes a Worker hostname to the VM's
  address. Do not run it on the host loopback with untrusted tenants.

This machine can host about 8 standard-2 equivalent containers at 80% memory
utilization, and up to 10 at 100%, before storage and headroom. It is a single failure domain.
Do not put irreplaceable data on it without an R2 replica.

## 2. Topology B: a three-machine site

Use this for a small production site with storage redundancy.

```
Site "rack-a"
  machine-a: Incus cluster member, SeaweedFS master + volume, cfb-agent
  machine-b: Incus cluster member, SeaweedFS volume, cfb-agent
  machine-c: Incus cluster member, SeaweedFS volume + S3 gateway, cfb-agent

  each: cloudflared + cfb-agent + one tunnel
  storage: 3 replicas hot, EC 4+2 for warm (needs 6 drives or 6 nodes;
           with 3 nodes use 3-way replication)
  public ingress: one hostname route per workload, Access in front
```

Rules:

- Three machines is the minimum for an Incus cluster with a distributed database
  that can lose one member.
- With three storage nodes, use 3-way replication, not erasure coding. Erasure
  coding needs more failure domains to pay off.
- One tunnel per machine. A site-level tunnel is a fallback for very large sites,
  not the default.
- Put the SeaweedFS master on a stable node and give it a fixed address on the
  private network. Do not put the master on the same node as the only S3 gateway.
- The S3 gateway listens on the private network and is exposed through a
  dedicated tunnel hostname, not the public internet directly. Presigned URLs
  point at that hostname. The S3 signature is the authorization; do not put
  Access in front of it.

Capacity: three 16 vCPU / 64 GiB machines give about 24 standard-2 containers
at 80% memory utilization, and up to 30 at 100%, minus the storage and agent
overhead.

## 3. Topology C: CI-heavy site

Use this when the motivating workload is heavy CI.

```
Site "ci"
  runner-a: Fireactions pool "linux-x64-large", 32 vCPU, 128 GiB, 2 TB NVMe
  runner-b: Fireactions pool "linux-x64-large", 32 vCPU, 128 GiB, 2 TB NVMe
  cache-1:  SeaweedFS S3 for caches and artifacts, 4 TB NVMe
  build-1:  Incus for long-running build services (optional)

  each: cloudflared + cfb-agent + one tunnel
  GitHub App / webhook -> control plane -> pool DO -> runner agent
  logs -> R2, artifacts -> local S3 -> R2 replica
```

Sizing notes:

- A Firecracker runner VM per job needs a warm rootfs and a warm toolchain. Keep
  a pool of at least 2 idle VMs per concurrent job target, or accept a cold-start
  penalty.
- CI is bursty. Size for the burst, not the average, or use a queue with a
  concurrency cap and let jobs wait.
- The cache bucket is the difference between a 4-minute and a 20-minute build.
  Mount it or sync it before the job starts.
- Keep job secrets envelope-encrypted in the control plane, inject them at
  start, and mask them in logs. Never bake them into the runner image.
- A 20-minute Cloudflare Workers Builds timeout is the reason this site exists.
  Set a longer timeout here, with a hard cap and a disk quota.

## 4. Software matrix

| Machine role | Software | Port | Exposure |
| --- | --- | --- | --- |
| Every machine | `cloudflared` | outbound 7844/UDP, 443/TCP fallback | Outbound only |
| Every machine | `cfb-agent` | 127.0.0.1:9443 and /run/cfb/agent.sock | Loopback and local socket |
| Container/VM host | Incus | Unix socket | Local only |
| Container host | containerd | Unix socket | Local only |
| Sandbox host | Firecracker + jailer | vsock, Unix socket | Local only |
| Storage | SeaweedFS master/volume/S3 | 9333, 8080, 8333 on private net | Private net and S3 hostname route |
| Workers | open-compute `ocd` (in an Incus VM) | VM loopback HTTP | Tunnel route to the VM |
| Workers (minimal) | workerd (in an Incus VM or dedicated machine) | VM loopback HTTP | Tunnel route to the VM |
| CI (GitHub) | Fireactions | loopback API | Loopback, tunnel route |
| CI (Gitea) | Woodpecker server + agent | 8000/9000 on private net | Tunnel route for UI |
| Observability | OTel collector, Prometheus, Loki | private net | Private net |

No tenant workload listens on a host port. Ingress is a tunnel route to the
agent, and the agent proxies to the workload's loopback or private address.

## 5. Bootstrap order

1. **Cloudflare account.** Create the Workers Paid plan, a D1 database, an R2
   bucket, a Queue, and the Access application for the console.
2. **Control plane.** Deploy the SPA and API. Set `ACCOUNT_ID`, `TEAM_DOMAIN`,
   `ACCESS_AUD`, `D1`, `R2`, `QUEUE` and the DO bindings as secrets or vars.
3. **Tunnel.** Create one tunnel per machine with `config_src=cloudflare`. Store
   the token as a systemd credential on the machine, not in a file.
4. **Machine base image.** Install the OS, a non-root `cfb` user, `cloudflared`,
   the agent, and the runtime (Incus, Firecracker, SeaweedFS, open-compute).
5. **Enroll.** Generate a one-time join token in the dashboard, run the install
   command, wait for the first heartbeat.
6. **Smoke test.** The agent creates a network namespace, starts a tiny container,
   runs `true`, and reports success. The machine becomes `ready`.
7. **First workload.** Create a container or a Worker from the dashboard. Watch
   the operation, the event stream and the audit record.
8. **Backup.** Enable replication to R2 and run a restore drill before calling the
   machine production.

## 6. workerd on a machine, exposed through the tunnel

There are two supported modes.

### 6.1 open-compute mode (recommended)

1. Create an Incus VM for the tenant or trust group, with a default-deny egress
   policy that blocks the host loopback and the metadata address.
2. Install `ocd` from a pinned release inside that VM.
3. Create one open-compute instance per tenant or trust group with its own data
   directory.
4. Deploy a bundle with the open-compute CLI or its `/client/v4` API.
5. Bind the instance to the VM's loopback address.
6. Add a tunnel hostname route to the VM address, with Access in front.
7. The agent records the deployment id, the bundle hash and the health check.

The control plane does not need to know how open-compute stores state. It calls
the documented API and stores the deployment metadata.

### 6.2 raw workerd mode (fallback)

Use this only if you need a binding that open-compute does not support.

1. Write a Cap'n Proto config that defines the Worker and a socket on
   `127.0.0.1:8080`.
2. Run `workerd serve /etc/workerd/config.capnp`.
3. If you need a privileged port, use the systemd socket-activation pattern from
   the workerd README: `--socket-fd http=3` with a `.socket` unit. For this
   design, bind loopback and let the tunnel route to it, so no privileged port is
   needed.
4. Run it inside an Incus VM or a dedicated machine. workerd is not a hardened
   sandbox; do not run hostile code directly on the host.
5. Add a tunnel route and an Access policy.

The systemd unit is:

```ini
[Unit]
Description=workerd runtime
After=network-online.target
Wants=network-online.target

[Service]
Type=exec
ExecStart=/usr/bin/workerd serve /etc/workerd/config.capnp
Restart=always
User=workerd
Group=workerd
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true
ReadWritePaths=/var/lib/workerd
```

This is a shape, not a tested unit. Check it against the workerd version you
install.

### 6.3 What the tunnel exposes

- The agent, on a machine-internal hostname, for control traffic.
- Each public Worker hostname, routed to the open-compute VM or the local workerd port.
- Optional Access policies per hostname.
- No direct access to the open-compute dashboard, the VM's SSH port, or the
  machine's Incus socket.

## 7. Network and firewall

Outbound only:

| Destination | Port | Purpose |
| --- | --- | --- |
| Cloudflare Tunnel edge | 7844/UDP (QUIC) | Tunnel transport |
| Cloudflare Tunnel edge | 443/TCP | Fallback if QUIC is blocked |
| Container registries | 443/TCP | Image pulls |
| OS and package mirrors | 443/TCP | Patches |
| Tenant egress allowlist | 443/TCP | Workload egress via proxy |

Inbound from the internet: none. Listeners are loopback, the local agent
socket, and the private site network for SeaweedFS, Woodpecker and
observability. Restrict those private ports to the site subnet and block tenant
namespaces.

If the network blocks UDP 7844, `cloudflared` falls back to HTTP/2 over 443. Test
both paths. A network that breaks long-lived connections will break the tunnel;
set `--protocol http2` explicitly if needed.

## 8. Verification checklist

Before calling a site production:

- [ ] The machine enrolls in under 10 minutes with no SSH.
- [ ] The agent heartbeats every 15 seconds; the dashboard shows fresh state.
- [ ] Stopping the control plane does not stop a running container.
- [ ] Restarting the control plane reconciles without duplicating workloads.
- [ ] Revoking the machine credential and key blocks the agent within one
  heartbeat.
- [ ] A container cannot reach the agent, the metadata address or another tenant.
- [ ] A sandbox cannot reach the host or another sandbox.
- [ ] A Worker bundle with a bad signature is rejected.
- [ ] A deleted object is gone locally and from the replica per policy.
- [ ] A restore from R2 matches the checksum.
- [ ] A disk failure (simulated by stopping a node) does not lose data.
- [ ] A CI job longer than 20 minutes completes.
- [ ] A cancelled CI job destroys its runner within 60 seconds.
- [ ] The audit chain verifies, and a tampered row fails.
- [ ] A second tenant cannot read the first tenant's resources by id.
- [ ] The tunnel has no inbound port open on the machine.

## 9. Upgrade order

1. **Control plane**, expand-only migration. Deploy the new code, keep the old
   code compatible with the new schema.
2. **Agent**, canary on one machine. Health check and rollback on failure.
3. **Runtime** (Incus, Firecracker, SeaweedFS, open-compute), one machine at a
   time. Drain first if the runtime restart affects workloads.
4. **Contract migration**, after all agents and the control plane are on the new
   version. Never drop a column the old code still reads.
5. **Storage**, one node at a time, wait for replication to catch up before the
   next.

## 10. Failure drills

Run these before production, then quarterly:

1. **Control-plane outage.** Block the agent's egress to Cloudflare for 15
   minutes. Containers keep running, the agent queues events, and the dashboard
   shows stale state. Restore and verify reconciliation.
2. **Tunnel outage.** Stop `cloudflared` for 5 minutes. Public ingress fails,
   workloads keep running, the agent reconnects.
3. **Machine power loss.** Pull power on one node. The Incus cluster and storage
   survive; the control plane marks the machine offline. Restart and verify no
   duplicate workloads.
4. **Disk failure.** Remove one storage node. Reads and writes continue; rebuild
   completes; no checksum mismatch.
5. **Restore.** Restore a random object and a D1 export into a scratch
   environment. Compare hashes.
6. **Revocation.** Revoke a machine credential and key. Verify the agent cannot
   reconnect, even with a valid tunnel token.
7. **Break-glass.** Use the audited exec path to a machine with a stopped agent.
   Verify the session is recorded and expires.

Each drill produces a record: what was done, what happened, how long it took, and
what changed as a result.

## 11. What this reference does not include

- Terraform or Wrangler configuration. The control plane's infrastructure should
  be codified, but that code does not exist yet.
- An installer. The install script is described, not written.
- Tested versions. Pin versions and read the release notes.
- Hardware prices. See `docs/cost.md` for the parametric model.
- A multi-site setup. Start with one site; add a second only after the first
  survives the drills.
