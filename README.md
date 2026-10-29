# CFB: a portable Cloudflare-compatible broker for machines you own

CFB keeps the control plane on Cloudflare and runs one daemon, `cfbd`, on each
machine. The daemon exposes Cloudflare's data-plane contracts locally: Workers
bundles run on workerd, object storage speaks S3, containers run as OCI images,
and the host reaches the control plane over an outbound WebSocket on port 443.
For the supported subset, the same artifact and binding names run on Cloudflare
or on a node with no application source change and at most a deploy-target
change. Anything outside that subset fails at deploy with a named error.

This repository is a design and evidence record. It contains no implementation.
External numbers and repository metadata are tied to captures under `evidence/`.
Feature claims are labelled by evidence type. The verification boundary is in
`docs/research.md`.

## The shape

| Plane | Where it runs | What it does |
| --- | --- | --- |
| Control plane | Cloudflare Workers, D1, R2, Durable Objects, Queues, Access | Identity, inventory, desired state, policy, audit, dashboard |
| Relay | Cloudflare Worker and one Durable Object per machine | Terminates the outbound WebSocket and multiplexes streams |
| Node | One `cfbd` process per machine, plus optional runtime children | Tunnel client, router, supervisor, S3 gateway, metrics |
| Runtime | workerd, rootless Podman, Incus, Firecracker, storage server | Executes the portable artifact |

The node never listens for inbound traffic. `cfbd` dials `wss://` on 443 and the
relay opens streams to a local Unix socket. Native Cloudflare Tunnel over
QUIC/HTTP2 is an optional transport, not a requirement. This is what makes the
same design work on a 512 MB VPS with no root and no inbound firewall rule.

## Portability contract

CFB does not invent a second application API. It implements existing contracts:

- Workers source, `wrangler.jsonc` binding names and Durable Object classes.
- S3 for object storage, so an AWS SDK client can switch endpoints.
- OCI images for containers, so the same image runs under Podman or a
  Cloudflare container.
- GitHub Actions or Woodpecker workflows for CI, with the same YAML.
- HTTP and WebSocket for application ingress.

Compatibility is layered: source, configuration, protocol, then semantics. The
design targets the first three; semantic equivalence is a test result, not a
slogan. Unsupported bindings fail at deploy or at first use with a named error.
The matrix and the supported subset are in `docs/spec.md`.

## Repository map

| File | Contents |
| --- | --- |
| `docs/spec.md` | Architecture, broker, transport, portability, orchestration, API, protocol |
| `docs/security.md` | Threat model, identity, isolation, constrained-mode limits |
| `docs/cost.md` | Cloudflare list prices, data-plane model, break-even |
| `docs/reference-deployment.md` | Standard, constrained and storage/CI profiles |
| `docs/prior-art.md` | Evaluated projects, licences, reusable parts, traps |
| `docs/research.md` | Evidence log, verification labels, open questions |
| `docs/decisions.md` | Decision records with rejected alternatives |
| `docs/roadmap.md` | Phases and exit criteria |
| `evidence/` | Captured Cloudflare docs, repository metadata and READMEs |

## Status

Design draft, 2026-10-29. No implementation is claimed. Licence: 0BSD.
