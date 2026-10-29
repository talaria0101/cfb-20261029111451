# CFB: a Cloudflare-hosted control plane for machines you own

CFB is a design for replacing Cloudflare's paid runtime services (R2, Containers,
Sandbox, Computer, Workers, Durable Objects, Workers CI) with workloads that run
on hardware you control, while the dashboard, API, identity and audit plane stay
on Cloudflare.

The short version:

- The control plane is a Workers app with a SPA, a Hono API, D1, R2, Durable
  Objects, Queues, Workflows and Access.
- The data plane is one or more machines you own. Each runs a single outbound-only
  host agent, `cloudflared`, and one or more runtimes (Incus, containerd,
  Firecracker, open-compute/workerd, SeaweedFS, Woodpecker or Fireactions).
- There are no inbound ports on the machines. Cloudflare Tunnel carries control
  traffic and optional public ingress. The control plane reaches agents through
  Workers VPC, with a relay Durable Object as a fallback.
- The control plane stores intent. The data plane stores execution state. If the
  control plane is down, workloads keep running and the last desired state keeps
  being reconciled.

The design is deliberately not a Cloudflare clone. It is a smaller system with a
clear split: Cloudflare is good at identity, global ingress, durable metadata and
a dashboard; your machines are good at long-running compute, local storage and
egress-heavy work.

## Read this first

| Document | What it answers |
| --- | --- |
| [`docs/spec.md`](docs/spec.md) | The architecture, components, data model, API, agent protocol, operations and SLOs |
| [`docs/reference-deployment.md`](docs/reference-deployment.md) | Three concrete topologies, the software matrix, bootstrap order, and a workerd-on-a-machine recipe |
| [`docs/prior-art.md`](docs/prior-art.md) | The tools on GitHub that already solve parts of this, with verified metadata and a build-vs-buy verdict |
| [`docs/decisions.md`](docs/decisions.md) | The load-bearing choices, the alternatives, and why they were rejected |
| [`docs/security.md`](docs/security.md) | Threat model, isolation matrix, identity, secrets, supply chain, audit |
| [`docs/cost.md`](docs/cost.md) | Control-plane list prices, a parametric data-plane model, break-even, hidden costs |
| [`docs/roadmap.md`](docs/roadmap.md) | Phases, exit criteria, MVP scope, and what not to build |
| [`docs/critique.md`](docs/critique.md) | The strongest objections to this design, with responses and residual risk |
| [`docs/research.md`](docs/research.md) | How the research was done, what was verified directly, what was not |

## The one-paragraph architecture

A browser loads a Vite SPA from a Workers static-assets deployment. Cloudflare
Access authenticates the user; the Worker validates the Access JWT and loads the
user's role from D1. The API writes desired state to D1 and a per-machine Durable
Object, then enqueues a reconcile command. The machine's host agent, which has
already dialed out through a per-machine Cloudflare Tunnel, receives the command
over Workers VPC, applies it to Incus, containerd, Firecracker, open-compute or a
Woodpecker runner, and reports observed state back. Artifacts, logs, images and
backups live in local S3 (SeaweedFS or RustFS) and are replicated to R2. Public
ingress goes through the same Tunnel and an Access policy. Nothing on the machine
listens on a public port.

## How to use this repository

1. Read `docs/spec.md` sections 1 to 5 for the shape, then section 9 for
   orchestration and section 14 for the agent protocol.
2. Pick the closest topology in `docs/reference-deployment.md` and read its
   verification checklist.
3. Run the Phase 0 spikes in `docs/roadmap.md` before writing code. They are
   designed to kill the design cheaply if a load-bearing assumption is wrong.
4. Use `docs/prior-art.md` to choose components, and `docs/decisions.md` to see
   what was rejected and why.
5. Read `docs/critique.md` before committing to the build. It contains the
   strongest arguments against it.

## Status

Design draft, 2026-10-29. No implementation is claimed. Every external number and
repository metadata field in the documents carries a capture date and a source;
see [`docs/research.md`](docs/research.md) for the verification boundary.

License: 0BSD. See [`LICENSE`](LICENSE).
