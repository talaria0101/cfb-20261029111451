# Research method and verification

Read date: 2026-10-29. The machine clock reported `2026-10-29T11:14:51Z` at the
start of the work.

This page says exactly what was verified directly, what was relayed from research
agents, and what remains unknown. It exists so a reader can tell the difference
between a measurement and a claim.

## 1. How the work was done

### 1.1 Discovery

The starting point was `https://web.pkgforge.dev/llms.txt`, a directory of free
web services. It describes a search endpoint:

```
https://search.web.pkgforge.dev/search?q=QUERY&count=10&provider=tinyfish
```

The instruction "pin to tinyfish" was read as: use that search service with
`provider=tinyfish` for discovery. The provider IDs observed in the directory
included `exa`, `parallelai`, `youdotcom`, `tinyfish`, `diffbot`, `firecrawl`,
`octenai`, `serperdev`, `perplexity`, `jina`, `keenable` and `tavily`.
`x-web-provider: tinyfish` was observed on responses.

Search results were used only to find candidate repositories and documents.
Every repository fact in this repository was then read from the GitHub REST API,
and every Cloudflare number was read from a Cloudflare documentation page. A
search snippet was never treated as a source.

### 1.2 Research agents

Two swarms of read-only research agents were run.

**Sweep 1: tool landscape, 8 agents.** Topics: S3 storage, microVM and sandbox
platforms, self-hosted PaaS and panels, CI systems, Durable Objects and durable
execution, tunnels and overlays, workerd self-hosting, and amika plus prior art.

**Sweep 2: Cloudflare and architecture, 6 agents.** Topics: Containers, Sandbox
and Computer; Workers, DO, R2, Queues, D1 and Workflows limits; Tunnel, Mesh,
VPC and Access; dashboard and control-plane patterns; orchestration patterns;
cost and failure modes.

All 14 agents completed. Their outputs were returned in the conversation and
were **not saved as files**. That matters: the exact wording and citations of
those outputs are gone, and the claims that came from them are labelled as
relayed in this repository unless a direct capture exists.

The last cost/failure agent's output was truncated mid-sentence before its final
recommendation section. Its factual statements are used, but its concluding
recommendations are not represented as its conclusions. They are the author's.

### 1.3 Direct verification

After the sweeps, the following were done directly on this machine:

- `gh api repos/OWNER/REPO` for 98 repositories, with fields for full name,
  stars, licence SPDX id, last push, archived flag, language, creation date,
  description and topics. Result: `evidence/repos.tsv`, header plus 98 rows,
  no error rows. Read date 2026-10-29. `daytonaio/daytona` was added later with
  the same query, so the file now has 99 repository rows plus the header.
- `curl` fetches of Cloudflare documentation pages, saved under
  `evidence/cf-docs/`. The pages cover Containers (limits, pricing,
  architecture, Durable Object API), Sandbox (reference, lifetime), Workers VPC,
  Cloudflare One account limits, D1 limits and pricing, Durable Objects limits
  and pricing, R2 pricing and limits, Queues pricing, Workers platform pricing,
  Workers static assets, Workers Builds limits and pricing, client certificates,
  Mesh, Tunnels for Platforms, Workers VPC Networks, Workflows pricing, and
  three Cloudflare blog posts (Sandbox GA, faster agent sandboxes, Cloudflare
  Computer). The Access pricing page is a text extract because it has no
  Markdown source.
- `curl` fetches of project READMEs and docs, saved under `evidence/reads/`.
  These include amika, workerd, celld, Rivet, SeaweedFS, RustFS, Incus,
  Firecracker, Coolify, Dokploy, Woodpecker, Pangolin, OpenClawMachines, WDL,
  open-compute, Fireactions, hyper, bhatti, and the awesome-cloudflare-selfhosted
  list.
- Additional price checks on 2026-10-29 for Workers, Durable Objects, D1,
  Queues, Workflows, Access and Workers Builds. The captures are in
  `evidence/cf-docs/`; the lines behind each number are quoted in
  `docs/cost.md`.
- Licence file checks on 2026-10-29 for the rows marked "reported" in the
  licence table: Nomad (BUSL 1.1), Restate (BSL 1.1), Inngest (SSPL 1.0 with an
  Apache-2.0 future licence), Windmill (Apache-2.0, AGPL-3.0 and a proprietary
  enterprise licence in one tree), NetBird (BSD-3-Clause core with AGPL-3.0
  management, signal and relay directories), ZeroTier (MPL-2.0 core plus
  nonfree service code) and Pangolin (Fossorial commercial licence plus other
  files). The GitHub API returns NOASSERTION for these because the repository is
  mixed; the LICENSE files are the primary source.

### 1.4 Adversarial review of the draft

After the first complete draft, six read-only review agents were run, one per
area: the specification, cost arithmetic, prior-art metadata, security and
decisions, transport feasibility, and house style. Each agent was told to report
only defects with a file and line, and to say "no defect found" rather than pad.

They converged on several real defects: the `cf1:network` binding was described
as a tunnel binding when it is the account-wide Mesh binding; mTLS was claimed
over a path where TLS terminates at Cloudflare; open-compute was placed on the
host loopback despite the workerd isolation rule; six creation dates disagreed
with `evidence/repos.tsv`; the sandbox example used an instance shape that the
API rejects; three cost lines used the wrong rounding or the wrong TB
convention; and the security document granted a D1 role that D1 does not have.

The load-bearing findings were re-checked against the capture or the live page
before any edit. The fixes are in the documents, not recorded only here. The
remaining disagreements are listed as open questions; the largest is whether
`cf1:network` reaches a newly created tunnel route without a redeploy.

### 1.5 What was not done

- No repository was cloned, built, installed or benchmarked. Every feature claim
  about a third-party project is from its documentation.
- No Cloudflare product was provisioned. The VPC, tunnel, D1, DO and R2 behaviour
  described here is from documentation, not from a live account.
- No penetration test, load test or restore drill was run.
- No licence text was read line by line for every dependency. The NOASSERTION
  rows and the "reported" licences need a legal read before they become
  dependencies.
- No hardware prices were collected. The cost model is parametric on purpose.

## 2. Verification labels

| Label | Meaning |
| --- | --- |
| Verified metadata | Read from the GitHub REST API on 2026-10-29 |
| Verified doc | Read from the vendor's documentation page on 2026-10-29 |
| Relayed | Reported by a research agent or a project README, not independently run |
| Derived | Arithmetic or inference from verified inputs, with the inputs named |
| Unknown | Not established; listed as an open question |

In `docs/prior-art.md`, the tables say "metadata verified" and "feature relayed".
In `docs/spec.md`, Cloudflare numbers are cited to a page and a read date. In
`docs/cost.md`, every price is cited and every calculation shows its formula.

## 3. Claims that are load-bearing, and their evidence

| Claim | Evidence | Label |
| --- | --- | --- |
| workerd is not a hardened sandbox | workerd README, `evidence/reads/cloudflare_workerd.md` | Verified doc |
| workerd has an experimental S3-backed DO cluster config | `src/workerd/server/workerd.capnp`, saved as `evidence/reads/cloudflare_workerd.capnp` | Verified source |
| amika's control plane, UI and relay are closed | Public org repository list; `docs/self-hosting.md` | Verified by absence and doc |
| amika has no supported local CLI path | `docs/self-hosting.md` | Verified doc |
| open-compute is single-node, not HA | open-compute README | Relayed |
| celld is pre-1.0 (v0.6.3), one app per fleet, no TLS | celld README, docs and release API | Verified doc |
| Incus has a REST API, containers, VMs and clustering | Incus README and docs | Relayed |
| SeaweedFS has S3, IAM/STS, EC and replication | SeaweedFS README | Relayed |
| MinIO is archived and AGPL-3.0 | GitHub API: `archived=true`, `AGPL-3.0` | Verified metadata |
| Containers are billed per 10 ms with the listed rates | Containers pricing page | Verified doc |
| Tunnels are 1,000 free, then $4.50 each | Tunnels for Platforms page | Verified doc |
| D1 is capped at 10 GB per database | D1 limits page | Verified doc |
| DO is soft-capped at 1,000 req/s per object | DO limits page | Verified doc |
| Workers VPC is beta and free | Workers VPC page | Verified doc |
| Access is capped at 500 apps and 50 service tokens | Cloudflare One account limits | Verified doc |

## 4. Repository metadata

`evidence/repos.tsv` columns:

```
repo  stars  license  pushed_at  archived  language  created_at  description  topics  read_at
```

Notes:

- 99 repositories, one row each, no duplicates, no `ERROR` rows at the last
  check.
- `license` is the SPDX id from the GitHub API, or `NONE` when GitHub reports no
  licence file. `NOASSERTION` means GitHub could not classify the licence; it
  does not mean there is no licence.
- `archived` is the API's boolean. MinIO is the only archived repository in the
  main catalog.
- Star counts are a snapshot and move daily. They are not a quality measure.
- The `cloudflare/computer` API response reports 9,752 stars and an MIT licence;
  the repository resolves from `cloudflare/workspace`. The catalog uses the name
  that the API returned.
- `cloudflare/sandbox` was removed from the catalog because it is a seccomp rules
  project, not the Sandbox SDK. `cloudflare/sandbox-sdk` is the correct
  repository and is included.

## 5. Document captures

### 5.1 Cloudflare documentation

Saved under `evidence/cf-docs/`, filenames derived from the URL. The set covers:

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
- Client certificates (mTLS).
- Access pricing.
- Blog: Sandbox GA, faster agent sandboxes, Cloudflare Computer.

Most captures are the documentation site's own Markdown source. The Access
pricing page is an HTML-to-text extract of its pricing section, because the
marketing site has no Markdown source. All captures preserve the numbers used
in the design. None preserves interactive table or calculator behaviour.

The client certificates capture was taken to check whether Cloudflare could
validate a machine certificate at the edge. The design does not use that path,
because the Worker does not receive the TLS peer and request signatures carry
authentication instead. The capture is kept as the record of the alternative
that was considered.

### 5.2 Project READMEs

Saved under `evidence/reads/`, one file per project, named
`owner_repo.md`. These are the project's own claims, not tests. Where a
README makes a performance claim (for example, a cold-start number), the design
does not repeat it as a measured result.

## 6. What the research changed

Three findings changed the design:

1. **Cloudflare Computer is not a product to clone.** It is an early-preview
   library (`@cloudflare/computer` 0.8.0, MIT) over a Durable Object workspace.
   The design treats "computer" as a workspace layer over sandboxes and Workers,
   not as a new runtime.
2. **open-compute is the only complete single-machine Workers platform, but it
   is not HA.** That is why the design runs it behind the agent with a
   per-machine failure domain, rather than treating it as a cluster.
3. **amika's control plane is closed.** That removed the option of forking amika
   and made the "build the control plane" decision explicit rather than assumed.

Two findings did not change the design but are worth recording:

- Cloudflare's 648 ms sandbox start is a benchmark claim, not a guarantee. The
  design targets 1 to 3 seconds cold and 200 to 500 ms warm, and says so.
- MinIO's archive and AGPL licence make it unsuitable as a base, even though it
  is the most familiar self-hosted S3.

## 7. Open questions

These are unresolved and named in `docs/spec.md` section 19:

1. Does `cf1:network` reach a route on a tunnel created after the binding,
   without a Worker redeploy?
2. What is the real throughput and latency of Workers VPC `fetch()` to an agent?
3. Do WebSocket upgrades survive VPC `fetch()`?
4. What is Incus container start time on the target hardware?
5. What is Firecracker restore time on the target hardware?
6. What is `cloudflared` throughput and CPU cost per machine?
7. What is open-compute's idle memory and CPU per tenant?
8. What is D1 query latency from the control plane's region?
9. Can the audit hash chain be implemented with D1 triggers?
10. What is the restore time for a 1 TB bucket from R2?

None of these is answered by reading. Each needs a prototype and a measurement.

## 8. Threats to the validity of this research

- **Recency.** The ecosystem moves weekly. A star count or a beta status from
  2026-10-29 may be wrong next month. Re-check before depending on a project.
- **Documentation bias.** A README describes intent. It does not prove the code
  does it. The design labels these claims as relayed for that reason.
- **Selection bias.** The catalog favours projects that appeared in searches and
  in the awesome-cloudflare-selfhosted list. A better project with no SEO may be
  missing.
- **No benchmarks.** Performance claims in the design are targets, not results.
- **No legal review.** Licence classifications from the GitHub API are a starting
  point. NOASSERTION and mixed licences need a human.
- **Agent relay.** Fourteen research agents produced summaries that were not
  saved. The facts that survived are in the documents and the captures; the
  reasoning that produced them is not reproducible from this repository alone.

## 9. Reproducing the checks

Repository metadata:

```bash
gh api repos/OWNER/REPO \
  --jq '[.full_name, (.stargazers_count|tostring), (.license.spdx_id // "NONE"),
         .pushed_at, (.archived|tostring), (.language // "NONE"), .created_at,
         (.description // "" | gsub("[\t\r\n]+"; " ")),
         ((.topics // []) | join(","))] | @tsv'
```

A full refresh, with care for rate limits and the 120-second shell timeout:

```bash
xargs -a repos.txt -P 6 -I{} bash fetch.sh {} > repos.fresh.tsv
```

Cloudflare documentation:

```bash
curl -qfsSL https://developers.cloudflare.com/containers/platform/pricing/ \
  -o evidence/cf-docs/containers-pricing.md
```

Prices and limits change. Re-read before quoting them.
