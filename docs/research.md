# Research record

Read date: 2026-10-29. This page states what was verified, what was relayed and
what remains unknown. It is an evidence log, not a narrative.

## 1. Method

- Candidate projects were found by search, then every repository fact was read
  from `gh api repos/OWNER/REPO` and stored in `evidence/repos.tsv`.
- Cloudflare numbers were read from the documentation pages cited in
  `docs/cost.md` and saved under `evidence/cf-docs/`.
- Project behaviour comes from saved READMEs under `evidence/reads/`. No project
  was cloned, built, installed or benchmarked.
- No Cloudflare product was provisioned. No penetration test, load test or
  restore drill was run.
- Six projects were read specifically for design patterns:
  homelable, CoreObs, omnyssh, ServerBee, mirrord and cfrs. README captures are
  `evidence/reads/<owner>_<repo>.md`.

## 2. Labels

| Label | Meaning |
| --- | --- |
| Verified metadata | Read from the GitHub REST API on 2026-10-29 |
| Verified doc | Read from the vendor's page on 2026-10-29 |
| Verified source | Read from a file in the project's repository at a named path |
| Relayed | Reported by a project README or an unsaved research summary; not run |
| Derived | Arithmetic or inference from named verified inputs |
| Unknown | Not established; listed as an open question |

## 3. Load-bearing claims

| Claim | Evidence | Label |
| --- | --- | --- |
| Cloudflare Container memory is $0.0000025/GiB-s, CPU $0.000020/vCPU-s, disk $0.00000007/GB-s | Containers pricing capture | Verified doc |
| Containers include 25 GiB-h memory, 375 vCPU-min and 200 GB-h disk per month | Containers pricing capture | Verified doc |
| Tunnels are free up to 1,000 per account, then $4.50 each | Tunnels for Platforms capture | Verified doc |
| D1 is capped at 10 GB per database | D1 limits capture | Verified doc |
| A Durable Object is soft-capped at 1,000 req/s | DO limits capture | Verified doc |
| Workers VPC is beta and free | Workers VPC capture | Verified doc |
| Access is capped at 500 apps and 50 service tokens | Cloudflare One limits capture | Verified doc |
| `cf1:network` is the account-wide Mesh binding, not a tunnel binding | Mesh capture | Verified doc |
| workerd is not a hardened sandbox | workerd README | Verified doc |
| workerd has an experimental S3-backed DO cluster config | `src/workerd/server/workerd.capnp` | Verified source |
| open-compute is single-node and not HA | open-compute README | Relayed |
| celld is pre-1.0, one app per fleet, no TLS | celld README and release API | Verified doc |
| MinIO is archived and AGPL-3.0 | GitHub API: `archived=true`, `AGPL-3.0` | Verified metadata |
| cfrs does not implement tunnel registration or visitor serving | `src/tunnel/mod.rs` at commit `0cbf27d84c2f3091f69a83e2a75076b7738b278d` | Verified source |
| ServerBee stores an argon2 hash of an agent-proposed token | `crates/server/src/service/agent_authority/mod.rs` and `auth.rs` at commit `ff409d112ab5808d541f225d23e6aa4689cdc34b` | Verified source |
| mirrord caps per-stream queues at 16 KiB and uses bincode over WebSocket | mirrord README | Relayed |
| homelable has no agent or tunnel; it is LAN-only | homelable README | Relayed |
| CoreObs agents write directly to Postgres | CoreObs README and agent docs | Relayed |
| omnyssh uses native SSH and local `-L` forwarding only | omnyssh README | Relayed |

## 4. Findings that changed the design

1. **A relay is the portable default.** cfrs shows an outbound Unix-socket origin
   with no listening TCP port, but it has no registration or serve loop. The
   WebSocket relay is the path that works today with no root, no UDP and no
   `cloudflared`. The native tunnel is Phase 2.
2. **The node owns capabilities.** ServerBee reconciles a capability bitmask
   against the real host and disables high-risk bits by default. CFB uses the
   same ownership: the control plane schedules only what the node reports.
3. **One binary, runtime children.** The earlier design had a tunnel client, an
   agent, a proxy and a metrics process. `cfbd` replaces those. workerd, Podman,
   Incus, Firecracker and storage servers remain child processes because they are
   separate security domains.
4. **The relay must not be the only data path.** mirrord's versioning and
   per-stream queues inform the protocol, but the relay DO bills per message.
   Bulk data belongs on S3 or a native tunnel.
5. **Four of the six pattern projects are not brokers.** homelable, CoreObs,
   omnyssh and mirrord do not provide zero-inbound brokering. Claiming otherwise
   would be false.
6. **The binding shim is the weak point.** workerd gives Workers, DO, services,
   sockets and disk, but not R2, D1, KV or Queues bindings. Those need a generated
   shim, a supported subset and a conformance suite. workerd alone is not
   zero-change compatibility.

## 5. Repository metadata

`evidence/repos.tsv` columns:

```
repo  stars  license  pushed_at  archived  language  created_at  description  topics  read_at
```

Notes:

- `license` is the SPDX id from the GitHub API, or `NONE` when GitHub reports no
  licence file. `NOASSERTION` means GitHub could not classify the licence; it
  does not mean there is no licence.
- `archived` is the API boolean. MinIO is the only archived repository in the
  catalog.
- Star counts are a snapshot and move daily. They are not a quality measure.
- `cloudflare/computer` resolves from `cloudflare/workspace`; the catalog uses
  the name the API returned.
- `cloudflare/sandbox` is a seccomp rules project, not the Sandbox SDK. The
  catalog uses `cloudflare/sandbox-sdk`.

## 6. Captures

### 6.1 Cloudflare documentation

`evidence/cf-docs/` contains the Markdown source, or a text extract where no
Markdown exists, for:

- Containers: limits, pricing, architecture, Durable Object container API.
- Sandbox: reference, lifetime.
- Workers VPC, Workers static assets.
- Workers platform pricing, Workers Builds limits and pricing.
- Cloudflare One: account limits, Mesh.
- Tunnels for Platforms.
- D1 limits and pricing.
- Durable Objects limits and pricing.
- R2 pricing and limits.
- Queues pricing.
- Client certificates.
- Access pricing.
- Blog: Sandbox GA, faster agent sandboxes, Cloudflare Computer.

The client certificates capture records an alternative that was rejected:
Cloudflare terminates TLS, so the Worker does not receive the peer certificate.
Signed request envelopes carry authentication instead.

### 6.2 Project READMEs

`evidence/reads/` holds one file per project, named `owner_repo.md`. These are
the project's own claims, not tests. Performance claims in a README are not
repeated as measured results.

## 7. Open questions

1. What is the throughput and latency of the relay WebSocket for 1 MiB responses
   and for exec streams?
2. Does a WSS relay stay within Cloudflare limits for 1,000 nodes with ingress
   and log traffic?
3. Can the generated D1 shim match enough SQL semantics for real Workers code?
4. What is workerd's idle memory and CPU per tenant on a 512 MB node?
5. What is rootless Podman cold and warm start time on the target hardware?
6. Can cfrs be extended to a full tunnel registration and serve loop without the
   embedded Cloudflare-internal roots?
7. What is the S3 gateway throughput on local NVMe versus SeaweedFS?
8. Does the native tunnel work through a proxy that allows only HTTP CONNECT on
   443?
9. What is the restore time for a 1 TB bucket from R2?
10. Which Workers APIs used by real applications are missing from the supported
    subset?
11. Can the audit hash chain be implemented with D1 triggers?
12. Does `cf1:network` reach a route created after the binding without a Worker
    redeploy?

None is answered by reading. Each needs a prototype and a measurement.

## 8. Limits of this research

- **Recency.** The ecosystem moves weekly. Re-check metadata before depending on
  a project.
- **Documentation bias.** A README describes intent, not behaviour. Those claims
  are labelled relayed.
- **Selection bias.** The catalog favours projects that surfaced in search. A
  better project with no SEO may be missing.
- **No benchmarks.** Performance numbers in the design are targets.
- **No legal review.** NOASSERTION and mixed licences need a human.
- **No live Cloudflare account.** Product behaviour is from documentation.

## 9. Reproducing the checks

Repository metadata:

```bash
gh api repos/OWNER/REPO \
  --jq '[.full_name, (.stargazers_count|tostring), (.license.spdx_id // "NONE"),
         .pushed_at, (.archived|tostring), (.language // "NONE"), .created_at,
         (.description // "" | gsub("[\t\r\n]+"; " ")),
         ((.topics // []) | join(","))] | @tsv'
```

Cloudflare documentation:

```bash
curl -qfsSL https://developers.cloudflare.com/containers/platform/pricing/ \
  -o evidence/cf-docs/containers-pricing.md
```

Prices and limits change. Re-read before quoting them.
