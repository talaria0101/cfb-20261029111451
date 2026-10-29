# Security

Status: design draft, 2026-10-29. This is a threat model and a set of controls,
not a certification.

## 1. Assets

What an attacker wants, in rough order of value:

1. **Tenant data** in local S3, volumes and snapshots.
2. **Secrets** injected into workloads, CI jobs and the control plane.
3. **Machine identity**: the agent credential, the tunnel token and the
   underlying private key.
4. **Control-plane authority**: the ability to issue commands to the fleet.
5. **Audit integrity**: the ability to change or delete the record of an action.
6. **Availability**: the ability to stop workloads or the dashboard.
7. **Metadata**: inventory, topology, labels and usage.

## 2. Trust boundaries

```
Untrusted:  browser, tenant code, tenant images, public internet, webhooks
Semi:       the SPA bundle in the browser, CI job containers
Trusted:    control plane (Cloudflare account), host agent, local runtimes
Root:       machine kernel, hypervisor, cloudflared process, agent key
```

The most important line is between **tenant code** and **the host agent**. A
tenant workload must not be able to reach the agent's API, its local SQLite
database, its identity key, the cloudflared token, or another tenant's runtime.

The second line is between **the agent** and **the control plane**. The agent
authenticates the control plane, not just the reverse. A compromised control
plane should not be able to silently exfiltrate tenant data without leaving an
audit trail, and a compromised agent should not be able to impersonate another
machine.

## 3. Isolation matrix

| Workload class | Boundary | Use for | Escape impact | Notes |
| --- | --- | --- | --- | --- |
| Incus container | Shared host kernel, namespaces, seccomp, cgroups | Trusted services, internal tools, trusted CI | Host compromise | Not for hostile code |
| containerd + runc | Shared host kernel, namespaces, seccomp, cgroups | OCI services, trusted CI | Host compromise | Same as above |
| Incus VM | KVM, QEMU device model | Semi-trusted tenants, dev environments | Guest escape to host via QEMU | Disable unneeded devices; no PCI passthrough in v1 |
| Firecracker | KVM, minimal device model, jailer | Untrusted code, agent sandboxes, CI steps | Guest escape to host via Firecracker or KVM | Requires KVM; jailer required |
| gVisor | User-space kernel, syscall interception | Untrusted code where KVM is unavailable | Kernel or gVisor bug; weaker than KVM | Compatibility gaps |
| workerd in a Firecracker VM | VM plus V8 isolate | Untrusted Workers | VM escape first | Default for untrusted Workers |
| workerd in an Incus container | Container plus V8 isolate | Trusted Workers | Container escape first | Acceptable for trusted code |
| workerd on the host | V8 isolate only | Never for untrusted code | Host compromise | The project says it is not a hardened sandbox |
| Dedicated machine | Hardware | High-trust or regulated tenants | Machine compromise | Highest cost, clearest boundary |

Rules:

- A tenant's isolation class is a property of the tenant, not a per-request
  choice by the tenant. The control plane enforces it, and the agent verifies it.
- A workload cannot request a weaker isolation class than its tenant policy.
- A container running as root, privileged, or with a host mount is a host
  compromise waiting to happen. The agent refuses those configurations.
- `workerd` is never the boundary between tenants. It runs inside a VM or a
  dedicated machine.

## 4. Threat model

Each entry names the threat, the control, and the residual risk. "Residual" is
what remains after the control, not a claim that the control is perfect.

### T1. Tenant escapes its sandbox

**Threat.** A tenant exploits the container runtime, the hypervisor, a device
model or a shared kernel to reach the host or another tenant.

**Controls.**

- MicroVM for untrusted code; no shared kernel.
- Minimal device model (Firecracker) or no PCI passthrough (Incus VM).
- Seccomp, no-new-privileges, read-only rootfs, non-root user for containers.
- Kernel and hypervisor patching on a schedule.
- No host mounts; shared volumes are explicit and scoped.
- Per-tenant network namespace and default-deny egress.
- Agent refuses privileged containers and host mounts.

**Residual.** A hypervisor or kernel zero-day. Detect by host integrity
monitoring and unusual egress; contain by one tenant per machine for high-value
tenants.

### T2. Tenant reaches the host agent

**Threat.** Tenant code calls the agent's loopback API, reads its environment, or
reaches its Unix socket.

**Controls.**

- Agent listens on a loopback address in the host network namespace, not in the
  tenant's.
- Tenant network namespace has no route to the host loopback.
- The local admin socket `/run/cfb/agent.sock` is mode 0660, owned by group
  `cfb-admin`, and requires a local peer credential check. The TCP listener on
  `127.0.0.1:9443` has no peer credential facility; it accepts only signed
  control-plane requests and is bound to loopback.
- Agent environment is scrubbed; secrets are in a separate credential store.
- Tenant cannot mount `/proc`, `/sys`, the Docker socket or the Incus socket.

**Residual.** A kernel or runtime bug that lets a tenant escape the namespace.
The microVM boundary is the answer.

### T3. Stolen machine identity

**Threat.** An attacker copies the agent credential, the tunnel token or the
private key from a machine.

**Controls.**

- Private key in a TPM or a 0600 file owned by the agent user; never in the
  workload's mount namespace.
- Short-lived machine credentials (24 hours) with automatic renewal.
- Machine public key pinned to the machine id; the 24-hour credential is bound
  to that key. A copied credential alone cannot sign a request.
- Tunnel token stored as a systemd credential, not an environment variable.
- Revocation on every request, not only at renewal.
- Anomaly detection: two connections with the same identity, a connection from a
  new ASN, or a command from an unexpected location.
- One tunnel per machine, so revocation is narrow.

**Residual.** An attacker with root on the machine can read the key. Detect by
revocation and rebuild; assume the machine is compromised.

### T4. Compromised control plane

**Threat.** An attacker gains a Cloudflare account token, a Worker binding or a
D1 credential and issues commands to the fleet.

**Controls.**

- Least-privilege API tokens; no global key in a Worker.
- Access for human login, with MFA at the IdP.
- Two-person approval for destructive operations (delete machine, delete bucket,
  rotate fleet credentials, change isolation policy).
- Every command is signed with a short-lived control-plane key and recorded in
  the audit chain.
- The agent enforces local policy: it refuses to delete a volume with a
  retention hold, to run a privileged container, or to exceed a quota, even if
  the control plane asks.
- Rate limits and anomaly alerts on command volume.
- Break-glass: a local `cfbctl` with a hardware key can revoke the control plane's
  access on a machine.

**Residual.** A control-plane compromise can still stop or start workloads and
read metadata. It cannot read tenant object bytes without the per-tenant storage
credential held by the separate `signer` Worker; a compromise of both the API
and the signer can read them.

### T5. Malicious image or bundle

**Threat.** A tenant or an attacker publishes an image or Worker bundle that
contains malware or exploits the runtime.

**Controls.**

- Digest pinning; no floating tags in production.
- Signature verification (cosign or Sigstore) before pull or deploy.
- SBOM generation and storage; admission policy on critical packages.
- Image scanning in CI; a failing scan blocks promotion.
- Agent verifies the signature and digest locally, not only in the control
  plane.
- Worker bundles are signed and content-addressed.

**Residual.** A signed image can still be malicious if the signer is compromised.
Use per-tenant signing keys and audit who signed.

### T6. Secret leakage

**Threat.** A secret is exposed in D1, R2, logs, an error message, the SPA, a
container environment or a CI log.

**Controls.**

- Secrets are envelope-encrypted: a per-tenant data key wrapped by a KEK in the
  Cloudflare secret store or a local KMS. D1 stores only ciphertext and metadata.
- Secrets are never returned by the API after creation; the SPA shows only the
  name, version and last rotation.
- The agent injects secrets at workload start through a file or a local socket,
  not an environment variable, where the runtime allows it.
- Log redaction at the agent and at the API; a deny-list of key names and
  patterns.
- CI masking, plus a post-job scan for secret patterns in logs.
- R2 buckets for logs are private and encrypted.

**Residual.** A workload can always print a secret it was given. Limit the blast
radius: short-lived secrets, per-job scoping, and rotation.

### T7. Replay and request forgery

**Threat.** An attacker replays a command, a webhook or an API request.

**Controls.**

- `Idempotency-Key` for POST, with a body hash.
- Request id, timestamp and nonce on agent messages; reject older than 60
  seconds and reuse.
- Webhook signature verification with a per-provider secret and a timestamp
  tolerance.
- CSRF tokens for browser mutations; SameSite cookies.
- Signed agent traffic; the control plane verifies the machine key signature.
  Neither Cloudflare path uses mTLS because Cloudflare terminates TLS.

**Residual.** A compromised client can replay within the window. Keep the window
short and make every mutation idempotent.

### T8. SSRF and metadata access

**Threat.** A tenant workload or a Worker fetches an internal address, the
Cloudflare metadata endpoint, or a local service.

**Controls.**

- Tenant egress goes through a proxy with an allowlist; link-local and metadata
  addresses are denied.
- The Worker does not fetch user-supplied URLs without an allowlist and a
  response size limit.
- Webhook URLs are validated and resolved to public addresses only.
- DNS rebinding protection: resolve, pin the address, connect to the pinned
  address.

**Residual.** An allowlisted destination can be compromised. Scope egress to the
minimum set of hosts.

### T9. Lateral movement inside a site

**Threat.** A compromised workload reaches another tenant's workload, the
storage cluster or the Incus API.

**Controls.**

- Per-tenant network namespace and bridge; no flat L2 network.
- Default-deny between tenants; explicit service routes only.
- Storage access through presigned URLs or per-tenant credentials, not a shared
  root key.
- Incus API is a local Unix socket, not a TCP endpoint on the LAN.
- Firecracker VMs have a single virtio-net interface with a host-side tap and a
  per-VM MAC; no promiscuous bridging.

**Residual.** A shared storage cluster is a shared failure domain. Encrypt
per tenant and isolate the metadata path.

### T10. Audit tampering

**Threat.** An attacker with database or account access edits or deletes audit
records.

**Controls.**

- Append-only enforced by SQLite triggers that reject UPDATE and DELETE on
  `audit_events`. The API binding is otherwise unrestricted, so this is
  tamper-evident, not tamper-proof.
- Hash chain: `hash = H(canonical_json(event without the hash field))`. The
  event body includes `prev_hash`.
- Daily anchor: the latest hash is written to an R2 object with object lock and
  to an external timestamp or a second account.
- A verification job recomputes the chain and alerts on a mismatch.
- Export to a WORM bucket with a retention period.

**Residual.** An attacker with both D1 and R2 write access can rewrite history
before the anchor. The external anchor is the backstop.

### T11. Denial of service

**Threat.** A tenant floods the API, the queue, the agent or the network.

**Controls.**

- Per-token and per-tenant rate limits at the Worker.
- Queue depth limits and per-tenant concurrency.
- Agent backpressure: bounded log buffers, drop oldest with a counter, never
  unbounded memory.
- Egress and ingress quotas per workload.
- Cloudflare WAF and DDoS protection on public hostnames.
- A circuit breaker that stops accepting new work when the control plane is
  unhealthy.

**Residual.** A determined tenant can consume its quota and degrade itself. That
is acceptable. Cross-tenant impact is the thing to prevent.

### T12. Supply-chain compromise of CFB itself

**Threat.** A dependency, a CI runner or a maintainer account is compromised and
ships a malicious agent or control-plane build.

**Controls.**

- Signed releases; reproducible builds where possible.
- Dependency pinning, lockfiles and a software bill of materials.
- CI runs on ephemeral runners with least privilege.
- Protected branches and required review.
- The agent verifies its own update signature and refuses an unsigned binary.
- Two-person approval for a release that changes the agent's privileged helper.

**Residual.** A compromised maintainer with signing access. Use hardware-backed
keys and a transparency log.

## 5. Identity and access

### 5.1 Humans

- Access with an OIDC IdP and MFA.
- The Worker validates `Cf-Access-Jwt-Assertion` against the team's JWKS,
  checking signature, issuer, audience, expiry and not-before.
- The Worker does not trust `Cf-Access-Authenticated-User-Email` alone, and does
  not rely on Access context surviving a service binding.
- D1 holds users, memberships, roles and tenant scopes.
- Roles: owner, admin, operator, developer, viewer, auditor, approver.
- A tenant-scoped role cannot see another tenant's resources, even by guessing an
  id. Every query includes the tenant scope.
- Destructive actions need a second approver with the `approver` role. That
  role can approve or reject a pending action; it cannot create one.
- Sessions are short; re-authentication is required for destructive actions.

### 5.2 Machines

- One-time join token, 15-minute TTL, bound to a machine id.
- The agent generates a key pair locally; the private key never leaves.
- The agent enrolls an Ed25519 public key and receives a 24-hour signed machine
  credential.
- Every request is signed. The agent pins the control-plane public key; the
  control plane pins the machine public key. Neither Cloudflare path uses mTLS
  because Cloudflare terminates TLS. A self-hosted relay that terminates TLS
  may add it as an optional deployment.
- Revocation is checked on every request; a revoked machine cannot renew.
- The tunnel token is separate from the credential and can be rotated
  independently.

### 5.3 Workloads

- A workload gets a short-lived identity token scoped to its tenant and
  workload, not a machine credential.
- The token is audience-bound to the service it calls.
- The agent injects it at start; it expires with the workload.
- Workloads cannot mint new tokens without an explicit grant.

### 5.4 Automation

- API tokens are scoped to a tenant and a set of actions, with an expiry.
- Tokens are shown once and stored hashed.
- CI jobs get a job-scoped token, not a tenant-wide token.
- Access service tokens are used only where a non-Worker client must cross an
  Access application, and they are not per-tenant because the account limit is
  50.

## 6. Secrets

### 6.1 Storage

```
KEK (Cloudflare Secrets Store or local KMS)
  +-- DEK per tenant (wrapped, stored in D1)
        +-- secret versions (ciphertext in D1, metadata in D1)
```

- The API never logs a plaintext secret.
- The SPA never receives a plaintext secret after creation.
- The agent receives a secret only for a workload that is allowed to use it,
  over the authenticated transport, and writes it to a tmpfs file with mode 0400
  owned by the workload user.
- The local copy is deleted when the workload stops.
- Rotation creates a new version; old versions have a grace period and an expiry.

### 6.2 Never in

- D1 plaintext.
- R2 object bodies or metadata.
- Logs, traces or error messages.
- The SPA bundle, localStorage or sessionStorage.
- A container environment variable, when a file or socket is possible.
- A Git repository, including this one.

### 6.3 Detection

- A pre-commit and CI secret scanner.
- A log scanner for high-entropy strings and known key prefixes.
- Alerts on a secret read by an unexpected workload or actor.
- A quarterly rotation for long-lived credentials, and immediate rotation on
  suspicion.

## 7. Network security

- Default deny ingress and egress.
- One tunnel per machine; no inbound ports.
- `cloudflared` runs as an unprivileged user with a systemd credential.
- The agent control API is loopback-only and accepts signed requests only. The
  local admin API is a Unix socket with a peer credential check.
- Tenant networks are namespaced and isolated; no flat L2.
- Egress through a proxy with an allowlist; metadata and link-local denied.
- DNS through a local resolver that logs and enforces the allowlist.
- Public hostnames get WAF, rate limits, TLS and Access where appropriate.
- Admin endpoints are not exposed on public hostnames.

## 8. Supply chain

- Pin base images by digest.
- Sign images and bundles with cosign; verify at admission and at the agent.
- Generate an SBOM (SPDX or CycloneDX) per image.
- Scan for vulnerabilities in CI; block critical findings on promotion.
- Pin GitHub Actions by commit SHA, not tag.
- Use ephemeral CI runners with a job-scoped token.
- Sign agent releases; verify before install and before self-update.
- Keep a dependency inventory; review new dependencies for licence and
  maintenance.

## 9. Audit

Every mutation writes an audit event:

```
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

- Canonical JSON (sorted keys, no whitespace) is hashed. The `hash` field is
  excluded from its own input; `prev_hash` is part of the body.
- The chain is per account, with a daily anchor.
- Reads of secrets and exports are audited too, not only writes.
- The audit API is read-only and paginated; there is no delete endpoint.
- An auditor role can read audit but not mutate resources.
- Retention: 1 year hot in D1, 7 years in R2 with object lock, configurable.

## 10. Break-glass

A machine with a broken agent still needs a way in. The break-glass path:

1. An operator requests access in the dashboard, with a reason and a duration.
2. A second approver grants it.
3. The control plane brokers an Access short-lived certificate or a
   Teleport-style proxy session on a separate network path, scoped to one
   machine, or issues a one-time exec token through the agent. It does not issue
   an SSH certificate.
4. The session is recorded (asciinema or a proxy log) and uploaded to R2.
5. The grant expires automatically; a cron job revokes any residual key.

The break-glass path is not a standing key. It is tested quarterly. If Cloudflare
is unavailable, a local hardware key on the machine can enable a recovery user
with a console-only login.

## 11. Incident response

1. **Contain.** Revoke the machine credential and key, delete the tunnel
   connections, and cordon the machine. Rotate any credential the machine could
   reach.
2. **Preserve.** Snapshot local state and logs before rebuilding. Do not destroy
   evidence.
3. **Assess.** Determine the blast radius: which tenants, which buckets, which
   secrets, which audit records.
4. **Notify.** Follow the contractual and legal notification requirements. Do not
   speculate in the notification.
5. **Eradicate.** Rebuild from a known image, re-enroll with a new identity, and
   restore data from a verified backup.
6. **Learn.** Write a post-incident record with a timeline, a root cause and a
   control that prevents recurrence. Add a test that fails without the control.

## 12. Residual risks

These are accepted, with the mitigation named:

- **Cloudflare is a trusted dependency.** A Cloudflare account compromise or a
  regional outage affects management and ingress. Mitigation: data plane
  independence and a documented manual recovery. The relay covers a Workers VPC
  failure, not a Cloudflare account compromise, because the relay is a
  Cloudflare Durable Object in the same account.
- **Containers share the kernel.** Trusted code only. Mitigation: microVM for
  untrusted code.
- **The agent runs with privileges.** Mitigation: a narrow privileged helper, a
  hardened agent, and a signed binary.
- **Tenant code can leak its own secrets.** Mitigation: short-lived, scoped
  secrets and rotation.
- **A single site is a single failure domain.** Mitigation: offsite R2 copies and
  tested restore.
- **Beta dependencies.** Workers VPC and celld may change. Mitigation: interface
  boundaries and fallbacks.
- **The operator is a single point of failure.** Mitigation: runbooks, two-person
  approval, and automation.

## 13. What this document does not cover

- A formal penetration test. None has been run.
- A compliance certification (SOC 2, ISO 27001, HIPAA). The controls are
  designed to be auditable, but no audit exists.
- A cryptographic review of the audit chain or the secret envelope. The
  construction is standard, but it has not been reviewed.
- Physical security of the machines.
- Legal review of the licences of the components in `docs/prior-art.md`.
