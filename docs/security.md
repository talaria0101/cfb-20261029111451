# Security model

Status: design draft, 2026-10-29. This is a threat model and a set of controls, not
a certification and not a penetration test.

## 1. Assets

1. Tenant data in local storage, volumes and snapshots.
2. Secrets injected into workloads, CI jobs and the control plane.
3. Machine identity: key, credential and relay session.
4. Control-plane authority to issue commands.
5. Audit integrity.
6. Availability of workloads, dashboard and ingress.
7. Metadata: inventory, topology, labels, usage.

## 2. Trust boundaries

```
Untrusted       browser, tenant code, tenant images, public internet, webhooks
Semi-trusted    SPA bundle, CI job containers
Trusted         Cloudflare account, control plane, relay, cfbd, supervised runtimes
Root            machine kernel, hypervisor, optional privileged helper
```

Two lines matter most:

- **Tenant to broker.** Tenant code must not reach `cfbd`'s Unix socket, SQLite
  state, identity key, logs of another tenant, or another runtime.
- **Node to control plane.** Both sides authenticate. A compromised control plane
  must not silently read tenant bytes without an audited authorization. A
  compromised node must not impersonate another machine.

The relay is inside the trust boundary for transport, not for content. It sees
stream metadata and, unless the payload is end-to-end encrypted, payload bytes.
It does not hold tenant credentials or decryption keys.

## 3. Isolation matrix

| Workload class | Boundary | Use for | Escape impact | Notes |
| --- | --- | --- | --- | --- |
| workerd on host | V8 isolate only | Never for hostile code | Host compromise | workerd is not a hardened sandbox |
| workerd in container | Shared kernel plus V8 | Trusted Workers | Container escape first | `standard` default |
| Rootless Podman | Shared kernel, namespaces, seccomp, cgroups | Trusted services | Host compromise or user namespace escape | `standard` |
| Incus container | Shared kernel, namespaces | Trusted services | Host compromise | `privileged` |
| Incus VM | KVM and QEMU device model | Semi-trusted tenants | Guest escape via QEMU | No PCI passthrough in v1 |
| Firecracker | KVM, minimal device model, jailer | Untrusted code, agent sandboxes | Guest escape via Firecracker or KVM | `privileged`, recommended for untrusted |
| gVisor | User-space kernel | Untrusted code where KVM is absent | Kernel or gVisor bug | Compatibility gaps |
| Dedicated machine | Hardware | High-value tenants | Machine compromise | Highest cost, clearest boundary |

Rules:

- The tenant's isolation class is set by policy in the control plane and verified
  by `cfbd`. A workload cannot request a weaker class.
- `cfbd` refuses privileged containers, host mounts, host PID, host network and
  host IPC.
- `lite` and `strict` run trusted code only. They cannot provide a hostile-code
  boundary and must say so at enrollment.
- One tenant per microVM for untrusted code. Do not pack tenants into one VM.

## 4. Threats and controls

### T1. Tenant escapes its sandbox

**Controls.** MicroVM for untrusted code. Minimal device model. Seccomp,
no-new-privileges, read-only rootfs and a non-root user for containers. No host
mounts. Per-tenant network namespace and default-deny egress. Hypervisor and
kernel patching.

**Residual.** A hypervisor or kernel zero-day. Detect with host integrity
monitoring and egress anomalies. Contain with one tenant per machine for
high-value tenants.

### T2. Tenant reaches the broker

**Controls.**

- `cfbd` listens on a Unix socket only. Mode 0660, group `cfb-admin`, peer
  credentials checked with `SO_PEERCRED`.
- `lite` and `strict` have no TCP listener. `standard` may use loopback for a
  child runtime that cannot use a Unix socket; the broker itself does not listen
  on TCP.
- Tenant network namespaces have no route to the host loopback or the broker
  socket.
- The broker environment is scrubbed. Secrets are fetched per workload, not kept
  in the process environment.
- Tenants cannot mount `/proc`, `/sys`, the Podman socket or the Incus socket.
- Workerd receives an inherited socket file descriptor, not a TCP port.

**Residual.** A kernel or runtime escape. The microVM is the answer.

### T3. Stolen machine identity

**Controls.**

- Ed25519 private key in a 0600 file owned by the `cfb` user, or a TPM when
  available. Never mounted into a workload.
- 24-hour credential bound to the key. A copied credential cannot sign.
- One-time join token, 15 minutes, bound to the machine id.
- Relay session bound to machine id and key id; a second session closes the
  first and raises an alert.
- Revocation checked on every request.
- Anomaly alerts: new ASN, concurrent sessions, command bursts.

**Residual.** Root on the machine can read the key. Revoke, rebuild, rotate every
secret the machine could reach.

### T4. Compromised control plane

**Controls.**

- Least-privilege API tokens. No global key in a Worker.
- Access with MFA at the IdP.
- Two-person approval for destructive operations.
- Commands signed with a short-lived control-plane key and written to the audit
  chain.
- `cfbd` enforces local policy even when the control plane asks otherwise:
  retention holds, quotas, isolation class, no privileged containers.
- Rate limits and alerts on command volume.
- Break-glass: a local `cfbctl` with a hardware key can revoke the control
  plane's access to one machine.
- Tenant object bytes need a presigned URL from the separate `signer` Worker, so
  API compromise alone does not grant bulk reads.

**Residual.** A control-plane compromise can stop workloads and read metadata. A
compromise of both API and signer can read object bytes, and the audit chain
records the reads.

### T5. Malicious image or bundle

**Controls.** Digest pinning, signature verification at admission and on the
node, SBOM, critical-finding scan gate, content-addressed bundles. Per-tenant
signing keys.

**Residual.** A compromised signer key. Audit who signed and rotate.

### T6. Secret leakage

**Controls.**

- Envelope encryption: per-tenant data key wrapped by a KEK in Secrets Store or a
  local KMS. D1 stores ciphertext and metadata only.
- The API never returns plaintext after creation.
- The broker writes a secret to a tmpfs file, mode 0400, owned by the workload
  user, and deletes it when the workload stops. Environment injection only when
  the runtime cannot use a file.
- Redaction at the broker and the API. CI masking plus a post-job scan.
- Short-lived, per-workload secrets.

**Residual.** A workload can print a secret it was given. Scope and rotate.

### T7. Replay and request forgery

**Controls.** `Idempotency-Key` with a body hash. Timestamp and nonce on broker
messages; reject older than 60 seconds and reject reuse. Webhook signatures with
a timestamp tolerance. CSRF tokens and SameSite cookies. Signed broker
envelopes.

**Residual.** Replay inside the window. Keep it short and mutations idempotent.

### T8. SSRF and metadata access

**Controls.** Egress through a proxy with an allowlist; link-local, metadata and
private ranges denied by default. No user-supplied URL fetch without an allowlist
and size limit. Resolve, pin and connect to the pinned address to stop DNS
rebinding. The broker's own egress is limited to the relay, the upstream S3 and
the configured image registries.

**Residual.** An allowlisted destination can be compromised. Keep the set small.

### T9. Lateral movement inside a site

**Controls.** Per-tenant network namespace. Default-deny between tenants.
Storage through presigned URLs or per-tenant credentials, never a shared root
key. Incus and Podman sockets are local, not LAN endpoints. One virtio-net
interface per VM, no promiscuous bridging.

**Residual.** Shared storage is a shared failure domain. Encrypt per tenant and
isolate the metadata path.

### T10. Audit tampering

**Controls.** Append-only trigger rejects UPDATE and DELETE on `audit_events`.
Hash chain with `prev_hash`. Daily anchor to an R2 object with object lock and to
an external timestamp. A verifier recomputes the chain and alerts on mismatch.
WORM export.

**Residual.** An attacker with both D1 and R2 write access can rewrite history
before the anchor. The external anchor is the backstop.

### T11. Denial of service

**Controls.** Per-token and per-tenant rate limits. Queue depth and concurrency
limits. Bounded log buffers with drop-oldest counters. Egress and ingress quotas.
WAF and DDoS protection on public hostnames. A circuit breaker that stops
accepting new work when the control plane is unhealthy. Per-stream credit and
frame caps on the relay.

**Residual.** A tenant can exhaust its own quota. Cross-tenant impact is
prevented.

### T12. Supply-chain compromise of CFB

**Controls.** Signed releases, reproducible builds where possible, lockfiles,
SBOM, ephemeral CI, protected branches, self-update signature verification,
two-person approval for changes to the privileged helper.

**Residual.** A compromised signing key. Hardware-backed keys and a transparency
log.

### T13. Relay compromise or abuse

**Threat.** The relay DO, the Worker, or a Cloudflare account insider reads or
injects stream traffic.

**Controls.**

- TLS to Cloudflare plus signed envelopes for control messages.
- Per-machine relay DO and a signed hello that binds socket to machine id.
- The node accepts `open` only from the relay, never from a workload.
- Frame and stream limits; the relay cannot request a local socket outside the
  route table.
- Optional payload encryption for control messages. Public ingress stays
  plaintext at the relay because the edge must route HTTP.
- Access logs and audit on route changes.

**Residual.** The relay sees ingress payloads in plaintext. A tenant that needs
end-to-end confidentiality must terminate TLS at the origin with an Origin
Certificate and accept that Cloudflare still sees metadata.

### T14. Constrained node compromise

**Threat.** A `lite` node with no root, no container runtime and no inbound port
is compromised through workerd or a dependency.

**Controls.**

- Trusted code only. No hostile tenants on `lite`.
- No setuid, no capabilities, no TUN, no netns, no Docker socket.
- Read-only application files; writable paths limited to the data directory.
- systemd sandboxing: `NoNewPrivileges`, `ProtectSystem=strict`, `ProtectHome`,
  `PrivateTmp`, `RestrictAddressFamilies`, `SystemCallFilter`.
- rlimits on address space, file size, open files and tasks.
- Outbound egress limited to the relay and configured upstreams.

**Residual.** A workerd escape gives the `cfb` user, not root. Damage is limited
to that user's files, the machine key and the workloads that user runs. Revoke
and rebuild.

## 5. Identity and access

### 5.1 Humans

- Access with an OIDC IdP and MFA.
- The Worker validates `Cf-Access-Jwt-Assertion` against the team JWKS, checking
  signature, issuer, audience, expiry and not-before.
- Do not trust `Cf-Access-Authenticated-User-Email` alone. Do not assume Access
  context survives a service binding.
- Roles: owner, admin, operator, developer, viewer, auditor, approver.
- Tenant-scoped queries always include the tenant scope. A guessed id is not
  access.
- Destructive actions need a second approver with the `approver` role. That role
  can approve or reject, not create.
- Re-authenticate for destructive actions.

### 5.2 Machines

- One-time join token, 15 minutes, bound to machine id.
- Key generated on the node; private key never leaves.
- Enrollment binds the public key. Credential valid 24 hours, renewed by
  signature.
- Every request signed. The node pins the control-plane key; the control plane
  pins the machine key.
- No mTLS on Cloudflare paths because Cloudflare terminates TLS. A self-hosted
  relay may add mTLS as an option.
- Revocation on every request. A revoked node cannot renew or reconnect.

### 5.3 Workloads

- Short-lived token scoped to tenant and workload, audience-bound.
- Injected at start, expires with the workload.
- No minting without an explicit grant.

### 5.4 Automation

- API tokens scoped to tenant and actions, with expiry, stored hashed, shown
  once.
- CI jobs get a job-scoped token.
- Access service tokens only where a non-Worker client must cross an Access app.
  The account limit is 50, so they are not per tenant.

## 6. Secrets

```
KEK (Secrets Store or local KMS)
  +-- DEK per tenant (wrapped, in D1)
        +-- secret versions (ciphertext and metadata in D1)
```

- The API never logs plaintext.
- The SPA never receives plaintext after creation.
- The broker fetches only for an authorized workload over the signed transport.
- The local copy is deleted when the workload stops.
- Rotation creates a new version with a grace period and an expiry.

Never in D1 plaintext, R2 bodies or metadata, logs, traces, errors, the SPA,
localStorage, a container environment when a file is possible, or Git.

Detection: pre-commit and CI scanning, log scanning for key prefixes and high
entropy, alerts on unexpected secret reads, quarterly rotation, immediate
rotation on suspicion.

## 7. Network security

- Default deny ingress and egress.
- No inbound listener. One outbound WebSocket per node.
- The broker's local API is a Unix socket with peer credential checks.
- Tenant networks are namespaced and isolated; no flat L2.
- Egress through a proxy with an allowlist; metadata and link-local denied.
- DNS through a local resolver that logs and enforces the allowlist.
- Public hostnames get WAF, rate limits, TLS and Access where appropriate.
- Admin endpoints are never on public hostnames.
- The relay accepts only signed hellos and closes a socket that sends `open`.

## 8. Supply chain

- Pin base images by digest.
- Sign images and bundles with cosign; verify at admission and on the node.
- SBOM per image, SPDX or CycloneDX.
- Scan in CI; block critical findings on promotion.
- Pin GitHub Actions by commit SHA.
- Ephemeral runners with job-scoped tokens.
- Sign broker releases; verify before install and self-update.
- Review new dependencies for licence and maintenance.

## 9. Audit

Every mutation writes an event:

```json
{
  "id": "evt_...",
  "ts": "2026-10-29T11:00:00.000Z",
  "account_id": "a_...",
  "tenant_id": "t_...",
  "actor": { "type": "user", "id": "u_..." },
  "action": "workload.delete",
  "resource": { "type": "workload", "id": "w_..." },
  "request_id": "req_...",
  "before": { "state": "running", "generation": 7 },
  "after": { "state": "deleted", "generation": 8 },
  "prev_hash": "sha256:...",
  "hash": "sha256:..."
}
```

Canonical JSON with sorted keys is hashed. The `hash` field is excluded from its
own input; `prev_hash` is included. The chain is per account with a daily anchor.
Secret reads and exports are audited. The audit API is read-only. Retention: one
year hot in D1, seven years in R2 with object lock, configurable.

## 10. Break-glass

1. Operator requests access with a reason and duration.
2. A second approver grants it.
3. The control plane issues a one-time exec token through the relay, scoped to
   one machine, or a short-lived Access certificate on a separate path. No
   standing SSH key.
4. The session is recorded and uploaded to R2.
5. The grant expires; a cron job revokes residue.

Tested quarterly. If Cloudflare is unavailable, a local hardware key can enable a
console-only recovery user on the machine.

## 11. Incident response

1. Revoke the machine credential and key; close the relay socket; cordon.
2. Rotate every secret the machine could reach.
3. Snapshot local state and logs before rebuilding.
4. Assess blast radius: tenants, buckets, secrets, audit records.
5. Notify per contract and law. Do not speculate.
6. Rebuild from a known image, re-enroll with a new identity, restore verified
   backups.
7. Record timeline, root cause and a control that prevents recurrence. Add a test
   that fails without it.

## 12. Residual risks

- **Cloudflare is a trusted dependency.** Account compromise or regional outage
  affects management and ingress. Data plane independence and a manual recovery
  path reduce impact. The relay does not survive a Cloudflare account compromise
  because it lives in the same account.
- **Containers share the kernel.** Trusted code only; microVM for untrusted.
- **The privileged profile uses root.** A narrow helper, a signed binary, and no
  default install.
- **Tenant code can leak its own secrets.** Short-lived, scoped, rotated.
- **A single site is a single failure domain.** Offsite R2 and tested restore.
- **Beta dependencies.** Workers VPC and open-compute may change. Interface
  boundaries and fallbacks.
- **The operator is a single point of failure.** Runbooks, two-person approval,
  automation.

## 13. Not covered

- A formal penetration test. None has been run.
- Compliance certification. No audit exists.
- Cryptographic review of the audit chain or the secret envelope.
- Physical security of machines.
- Legal review of component licences.
- End-to-end encryption of public ingress payloads. The relay sees them.
