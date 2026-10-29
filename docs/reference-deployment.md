# Reference deployments

Status: design recipes, not tested installers. Check every command against the
version you install. The common rule: create the control plane once on
Cloudflare, enroll each machine once, and change everything else through the
dashboard or API.

## 1. Profiles

| Profile | Root | Inbound TCP | Containers | VMs | Transport | Target |
| --- | --- | --- | --- | --- | --- | --- |
| `standard` | No | Loopback only | Rootless Podman | No | WSS relay | 4 GB+ VPS or home server |
| `lite` | No | None | No | No | WSS relay | 512 MB to 1 GB VPS |
| `strict` | No | None | No | No | WSS relay | Network that forbids all inbound |
| `privileged` | Optional helper | Loopback only | Incus, Podman | Incus, Firecracker | WSS relay or native tunnel | KVM host, untrusted code |

`lite` and `strict` run trusted code only. They cannot isolate hostile tenants.
`privileged` is the only profile that can.

## 2. Topology A: one small machine (`lite`)

Use this to replace one Workers project and one small bucket on a 512 MB VPS
with no root.

```
Cloudflare
  console Worker (SPA + API)
  D1, R2, coordinator DO, relay DO, Queues
        |
        | outbound wss:// on 443
        v
  Machine: 1 vCPU, 512 MB to 1 GB, 20 GB disk
    cfbd (unprivileged, one binary)
      workerd child (Unix socket)
      S3 gateway (local directory)
      metrics and logs
```

What runs:

- `cfbd` as a user service, not root.
- One workerd process per tenant or trust group, receiving an inherited Unix
  socket.
- Static files served by workerd.
- A local directory for objects, with an optional upstream S3 over outbound 443.
- No Podman, no Docker socket, no TUN, no netns, no FUSE.

What does not run: containers, microVMs, CI, native tunnel, privileged helpers.

Resource budget: keep `cfbd` under 30 MiB RSS and workerd under 80 MiB per idle
tenant, leaving headroom on a 512 MB machine. These are targets, not measured.

## 3. Topology B: one standard machine

Use this to replace a Cloudflare Containers workload, a tunnel and a local
reverse proxy.

```
Cloudflare
  console Worker, D1, R2, coordinator DO, relay DO, Queues, Access
        |
        | outbound wss:// on 443
        v
  Machine: 16 vCPU, 64 GiB, 2 TB NVMe
    cfbd (systemd, user cfb)
      workerd children
      rootless Podman
      SeaweedFS or RustFS (optional, local S3)
      GitHub runner or Woodpecker agent (optional)
```

- `cfbd` talks to the rootless Podman socket at
  `$XDG_RUNTIME_DIR/podman/podman.sock`.
- Storage is the embedded S3 gateway over local NVMe. For multi-node durability,
  point it at SeaweedFS or RustFS.
- One relay socket per machine. No `cloudflared`.
- Public ingress routes through the relay to a workload's Unix socket or
  loopback port. No host port is bound to a public interface.

This machine can host about 8 standard-2 equivalent containers at 80% memory
utilization. It is one failure domain. Do not put irreplaceable data on it
without an R2 replica.

## 4. Topology C: three-machine site (`privileged`)

```
Site "rack-a"
  machine-a: Incus member, storage master, cfbd
  machine-b: Incus member, storage volume, cfbd
  machine-c: Incus member, storage volume and S3 gateway, cfbd

  each: one cfbd, one relay socket
  storage: 3-way replication for 3 nodes
  ingress: one route per workload, Access in front
```

Rules:

- Three machines is the minimum for a cluster that can lose one member.
- With three storage nodes use 3-way replication. Erasure coding needs more
  failure domains to pay off.
- One socket per machine. A site-level socket is a fallback for very large
  sites, not the default.
- Put the storage master on a stable node with a fixed private address.
- The S3 gateway listens on the private network and is exposed as
  `s3-<site>.<domain>`. Presigned URLs point there. Do not put Access in front of
  that hostname; the S3 signature is the authorization.
- The privileged helper is installed only if Incus or Firecracker is needed. It
  is not part of `cfbd` and is not installed by default.

Capacity: three 16 vCPU / 64 GiB machines give about 24 standard-2 containers at
80% memory utilization, minus storage and broker overhead.

## 5. Topology D: CI-heavy site

```
Site "ci"
  runner-a: 32 vCPU, 128 GiB, 2 TB NVMe, cfbd with the ci feature
  runner-b: 32 vCPU, 128 GiB, 2 TB NVMe, cfbd with the ci feature
  cache-1:  S3 for caches and artifacts, 4 TB NVMe
  build-1:  Incus for long-running build services (optional)

  GitHub App or webhook -> control plane -> coordinator DO -> relay -> cfbd
  logs -> R2, artifacts -> local S3 -> R2 replica
```

Sizing:

- One Firecracker VM per job needs a warm rootfs and toolchain. Keep two idle
  VMs per concurrent job target, or accept cold starts.
- CI is bursty. Size for the burst or cap concurrency and let jobs wait.
- The cache bucket is the difference between a 4-minute and a 20-minute build.
- Secrets are envelope-encrypted, injected at start, masked in logs.
- This site exists because Workers Builds has a 20-minute job timeout, a
  6-concurrent-build limit and a 4 vCPU / 8 GB / 20 GB build machine.
- CI is not available in `lite`.

## 6. Software matrix

| Role | Software | Listen | Exposure |
| --- | --- | --- | --- |
| Every node | `cfbd` | `/run/cfb/cfbd.sock` or `$XDG_RUNTIME_DIR/cfb/cfbd.sock` | Local socket, mode 0660 |
| Workers | workerd child | Inherited Unix socket | Local socket |
| Containers | rootless Podman | `$XDG_RUNTIME_DIR/podman/podman.sock` | Local socket |
| Containers and VMs | Incus | `/var/lib/incus/unix.socket` | Local socket |
| Sandboxes | Firecracker and jailer | vsock, Unix socket | Local |
| Storage | SeaweedFS or RustFS | private network ports | Private network and S3 hostname |
| CI (GitHub) | Actions runner | outbound 443 | Outbound only |
| CI (Gitea) | Woodpecker agent | outbound 443 | Outbound only |
| Observability | OTel collector | private network | Private network |

No tenant workload listens on a public host port. Ingress is a relay stream to
`cfbd`, which proxies to a local socket or loopback port.

## 7. Bootstrap

1. **Cloudflare.** Workers Paid, one D1 database, one R2 bucket, one Queue, the
   console Access application, and the Durable Object namespaces.
2. **Control plane.** Deploy the SPA and API. Set `ACCOUNT_ID`, `TEAM_DOMAIN`,
   `ACCESS_AUD`, the D1, R2, Queue and DO bindings. Store secrets as Workers
   secrets or Secrets Store bindings.
3. **Machine.** Create the `cfb` user, a data directory, and install `cfbd` from a
   signed release. Verify the signature and the SHA-256 before execution.
4. **Enroll.** Create a machine in the dashboard, copy the one-time join token
   (15 minutes), run `cfbd enroll --token ... --profile lite`. The broker
   generates its key, enrolls the public key, and receives a 24-hour credential.
5. **Smoke test.** The broker writes a minimal workerd config, starts workerd on
   a Unix socket, requests `/`, and reports success. The machine becomes `ready`.
6. **First workload.** Deploy a Worker bundle or a container from the dashboard.
   Watch the operation, the event stream and the audit record.
7. **Backup.** Enable replication to R2 and run a restore drill before calling the
   machine production.

The install command is a shape, not a tested script. The broker CLI and flags do
not exist yet.

## 8. Running workerd

### 8.1 Standard mode

1. `cfbd` parses `wrangler.jsonc` and rejects unsupported bindings.
2. It generates `workerd.capnp` with one socket inherited from the broker.
3. It generates the binding shim for R2, D1, KV and Queues and writes it into the
   version directory.
4. It starts `workerd serve workerd.capnp --socket-fd http=3` as the `cfb` user.
5. It waits for a health request, then swaps the route pointer atomically.
6. It keeps the previous version until the next successful deploy.

`workerd` is not a hardened sandbox. For hostile code, run it inside a
Firecracker VM in the `privileged` profile. For `lite`, run trusted code only.

### 8.2 Optional open-compute mode

If the application needs the broader Workers API surface that open-compute
implements, run `ocd` inside a VM and let `cfbd` supervise it as a child. This
adds a second binary and is not part of the `lite` build. open-compute is
single-node and not HA.

### 8.3 Unit shape

```ini
[Unit]
Description=CFB broker
After=network-online.target
Wants=network-online.target

[Service]
Type=exec
ExecStart=/usr/local/bin/cfbd run --config /etc/cfb/cfbd.toml
Restart=always
User=cfb
Group=cfb
NoNewPrivileges=true
ProtectSystem=strict
ProtectHome=true
PrivateTmp=true
RestrictAddressFamilies=AF_UNIX AF_INET AF_INET6
ReadWritePaths=/var/lib/cfb
StateDirectory=cfb
RuntimeDirectory=cfb
```

This is a shape. Check it against your systemd version and the broker's actual
flags. `lite` and `strict` add `IPAddressAllow` for the relay and upstream S3
only, and no `ListenStream`.

## 9. Network and firewall

Outbound only:

| Destination | Port | Purpose |
| --- | --- | --- |
| Relay and API | 443/TCP | WSS transport and enrollment |
| Container registries | 443/TCP | Image pulls |
| OS and package mirrors | 443/TCP | Patches |
| Upstream S3 | 443/TCP | Replication and restore |
| Tenant egress allowlist | 443/TCP | Workload egress through the broker proxy |
| Cloudflare Tunnel edge, optional | 7844/UDP or 443/TCP | Native tunnel only |

Inbound from the internet: none.

Local listeners:

- `cfbd.sock`, mode 0660.
- The Podman or Incus socket, accessible to the `cfb` user only.
- Private network ports for storage and observability, restricted to the site
  subnet and blocked from tenant namespaces.

`lite` and `strict` have no `ListenStream` at all. If a dependency demands a TCP
port, it does not belong in those profiles.

If the network breaks long-lived WebSockets, the broker reconnects with jitter
and resumes streams. Test this before production. If the network blocks
WebSocket upgrade entirely, only a native tunnel over HTTP/2 or a different
network can help.

## 10. Verification checklist

- [ ] The machine enrolls in under 10 minutes with no SSH and no root.
- [ ] The dashboard shows a heartbeat every 15 seconds.
- [ ] Stopping the control plane does not stop a running workload.
- [ ] Restarting the control plane reconciles without duplicating workloads.
- [ ] Revoking the credential and key blocks the broker within one heartbeat.
- [ ] The broker has no TCP listener in `lite` and `strict`.
- [ ] A container cannot reach the broker socket, the metadata address or
      another tenant.
- [ ] A microVM cannot reach the host or another microVM.
- [ ] A bundle with a bad signature is rejected.
- [ ] An unsupported binding is rejected at deploy with a named error.
- [ ] A deleted object is gone locally and from the replica per policy.
- [ ] A restore from R2 matches the checksum.
- [ ] A stopped storage node does not lose data.
- [ ] A CI job longer than 20 minutes completes.
- [ ] A cancelled CI job destroys its runner within 60 seconds.
- [ ] The audit chain verifies, and a tampered row fails.
- [ ] A second tenant cannot read the first tenant's resources by id.
- [ ] The relay socket is the only outbound control connection.

## 11. Upgrade order

1. **Control plane**, expand-only migration.
2. **Broker**, canary on one machine. Health check and roll back on failure.
3. **Runtime** (workerd, Podman, Incus, storage), one machine at a time. Drain
   first if the restart affects workloads.
4. **Contract migration**, after all brokers and the control plane are on the new
   version. Never drop a column the old code still reads.
5. **Storage**, one node at a time, wait for replication before the next.

## 12. Failure drills

1. **Control-plane outage.** Block broker egress for 15 minutes. Workloads keep
   running, events queue, the dashboard shows stale state. Restore and verify
   reconciliation.
2. **Relay outage.** Stop the relay DO or block the socket for 5 minutes. Public
   ingress fails, workloads keep running, the broker reconnects.
3. **Machine power loss.** Pull power on one node. The cluster and storage
   survive; the control plane marks the machine offline. Restart and verify no
   duplicate workloads.
4. **Disk failure.** Remove one storage node. Reads and writes continue; rebuild
   completes; no checksum mismatch.
5. **Restore.** Restore a random object and a D1 export into a scratch
   environment. Compare hashes.
6. **Revocation.** Revoke a credential and key. Verify the broker cannot
   reconnect.
7. **Break-glass.** Use the audited exec path to a machine with a stopped broker.
   Verify the session is recorded and expires.
8. **Constrained node.** Confirm no inbound port, no root, no capability, and
   that a workerd crash does not affect the host.

Each drill produces a record: what was done, what happened, how long it took, and
what changed.

## 13. Not included

- Terraform or Wrangler infrastructure code.
- An installer.
- Tested versions.
- Hardware prices. See `docs/cost.md`.
- A multi-site setup. Start with one site and add a second after the drills.
