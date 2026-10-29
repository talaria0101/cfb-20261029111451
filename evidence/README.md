# Evidence

Everything here was captured on 2026-10-29. The captures are text, Markdown or
TSV; the analysis is in `docs/`. When a document and a capture disagree, the
capture wins for the quoted number, and the disagreement should be reported as a
bug.

## `repos.tsv`

One row per repository, 105 repository rows plus a header row.

Columns:

```
repo  stars  license  pushed_at  archived  language  created_at  description  topics  read_at
```

Source: `gh api repos/OWNER/REPO` through the GitHub REST API, authenticated as
`talaria0101`. Fields: `full_name`, `stargazers_count`, `license.spdx_id`,
`pushed_at`, `archived`, `language`, `created_at`, `description`, `topics`.

Caveats:

- `NOASSERTION` means GitHub could not classify the licence, not that no licence
  exists. Read the licence file before depending on the project.
- `NONE` means GitHub found no licence file.
- Stars and last-push timestamps are a snapshot. They move.
- The file was produced from a full re-fetch of the catalog names, not by
  merging partial batches, so there are no duplicate or gap rows. Six rows were
  appended on 2026-10-29 with the same API query, which makes 105.
- `cloudflare/computer` resolves from `cloudflare/workspace`; the API returns the
  former name. It is kept as returned.
- `cloudflare/sandbox` (a seccomp rules project) was removed. The Sandbox SDK is
  `cloudflare/sandbox-sdk` and is included under that name.

## `repos.raw.jsonl`

The raw JSON objects for the same 105 repositories, one per line, from the same
API calls. This is the source for `repos.tsv`. It is larger and less convenient,
but it preserves fields that the TSV drops (owner, default branch, open issues,
topics, licence URL).

## `cf-docs/`

Text or Markdown captures of Cloudflare documentation pages, saved with
filenames derived from the URL. These support the numbers in `docs/spec.md` and
`docs/cost.md`. Where the documentation site exposes its own Markdown source
(`index.md`), the capture uses it; otherwise the capture is an HTML-to-text
extract. The Access page is the only extract, because it is a marketing page
with no Markdown source, and its header records the raw response size and
SHA-256.

| File | Page |
| --- | --- |
| `blog.cloudflare.com_cloudflare-computer_.md` | Cloudflare Computer announcement, 2026-08-03 |
| `blog.cloudflare.com_faster-agent-sandboxes_.md` | Faster agent sandboxes |
| `blog.cloudflare.com_sandbox-ga_.md` | Sandbox general availability |
| `developers.cloudflare.com_cloudflare-one_account-limits_.md` | Cloudflare One account limits |
| `developers.cloudflare.com_cloudflare-one_networks_connectors_cloudflare-mesh_.md` | Cloudflare Mesh |
| `developers.cloudflare.com_cloudflare-one_networks_connectors_cloudflare-tunnel_configure-tunnels_tunnel-with-firewall_.md` | Tunnel firewall ports, 7844 TCP and UDP |
| `developers.cloudflare.com_containers_api_durable-object-container_.md` | Container Durable Object API |
| `developers.cloudflare.com_containers_concepts_architecture_.md` | Containers architecture |
| `developers.cloudflare.com_containers_platform_limits_.md` | Containers limits and instance types |
| `developers.cloudflare.com_containers_platform_pricing_.md` | Containers pricing |
| `developers.cloudflare.com_d1_platform_limits_.md` | D1 limits |
| `developers.cloudflare.com_d1_platform_pricing_.md` | D1 pricing and included rows |
| `developers.cloudflare.com_durable-objects_platform_limits_.md` | Durable Objects limits |
| `developers.cloudflare.com_durable-objects_platform_pricing_.md` | Durable Objects pricing and request rounding |
| `developers.cloudflare.com_queues_platform_pricing_.md` | Queues pricing |
| `developers.cloudflare.com_r2_pricing_.md` | R2 pricing |
| `developers.cloudflare.com_r2_platform_limits_.md` | R2 limits |
| `developers.cloudflare.com_sandbox_concepts_lifetime_.md` | Sandbox lifetime |
| `developers.cloudflare.com_sandbox_reference_.md` | Sandbox API reference |
| `developers.cloudflare.com_ssl_client-certificates_.md` | Client certificates (mTLS) |
| `developers.cloudflare.com_tunnel_features_tunnels-for-platforms_.md` | Tunnels for Platforms |
| `developers.cloudflare.com_workers_ci-cd_builds_limits-and-pricing_.md` | Workers Builds limits and pricing |
| `developers.cloudflare.com_workers_platform_pricing_.md` | Workers platform pricing |
| `developers.cloudflare.com_workers_static-assets_.md` | Workers static assets |
| `developers.cloudflare.com_workers-vpc_.md` | Workers VPC |
| `developers.cloudflare.com_workers-vpc_configuration_vpc-networks_.md` | Workers VPC Networks, including `cf1:network` |
| `developers.cloudflare.com_workflows_reference_limits_.md` | Workflows limits |
| `developers.cloudflare.com_workflows_reference_pricing_.md` | Workflows pricing |
| `www.cloudflare.com_zero-trust_products_access_.md` | Access pricing (text extract) |

Every price in `docs/cost.md` now has a capture behind it.

## `reads/`

README and documentation captures for the projects in `docs/prior-art.md`, one
file per project, named `owner_repo.md`. One exception: `cloudflare_workerd.capnp`
is the raw Cap'n Proto configuration from the workerd repository, saved because
it is the primary source for the experimental Durable Object cluster claim.

These are the projects' own claims. They were not tested. A performance number in
a README is not a measurement of the project on your hardware. The six projects
added on 2026-10-29 for design patterns (homelable, CoreObs, omnyssh, ServerBee,
mirrord, cfrs) were read, not run. Four of them are not brokers, and none is a
Cloudflare backend.

Included projects:

```
awesome-cloudflare-selfhosted
cloudflare_workerd
codician-team_diffinery
coollabsio_coolify
crocofied_CoreObs
denoland_celld
Dokploy_dokploy
elliothux_open-compute
ezgamehost_celld-operator
firecracker-microvm_firecracker
fosrl_pangolin
giuseppelt_self-workerd
gofixpoint_amika
gofixpoint_amika_hostd
gofixpoint_amika_self-hosting
harmont-dev_hyper
hostinger_fireactions
lxc_incus
mathaix_OpenClawMachines
metalbear-co_mirrord
Pouzor_homelable
rivet-dev_rivet
rustfs_rustfs
sahil-shubham_bhatti
seaweedfs_seaweedfs
solcreek_groundflare
talaria0101_cfrs
timhartmann7_omnyssh
VaalaCat_vorker
wdl-dev_wdl
woodpecker-ci_woodpecker
ZingerLittleBee_ServerBee
```

Missing from this set, but used in the documents through the API metadata or a
search result: Temporal, DBOS, Hatchet, Trigger.dev, Restate, Inngest, Windmill,
Orleans, Dapr, Pekko, Gitea, Buildkite, Tekton, Argo, Dagger, Jenkins, act,
Kaniko, Ceph, Garage, MinIO, VersityGW, Zenko, JuiceFS, Headscale, Tailscale,
NetBird, Nebula, ZeroTier, Teleport, OpenZiti, Octelium, frp, rathole,
boringproxy, Nomad, Kubernetes, Talos, Omni, Harvester, OpenStack, CloudStack,
OpenNebula, MicroCloud, Portainer, Rancher, Cosmos-Server, CapRover, Dokku,
Kamal, Coder, microsandbox, E2B runtime, gVisor, Kata, bhatti, hyper,
and others in `repos.tsv`.

For those, the metadata is verified and the feature description is from the
project's GitHub description or a search result. That is a weaker claim than a
README capture, and `docs/prior-art.md` labels it as relayed.

## What is not here

- The outputs of the two research-agent sweeps. They were returned in the
  conversation and not written to disk, so they are not reproducible from this
  repository. The claims that came from them are labelled as relayed.
- Cloudflare account data, a real tunnel, a real D1 database or a real DO. No
  Cloudflare product was provisioned.
- Benchmarks, build logs, test results or screenshots from any third-party
  project.
- Hardware or bandwidth prices.

## Adding evidence

If you add a capture:

1. Use the URL-derived filename, lowercase, with the scheme removed and slashes
   replaced by underscores.
2. Put the read date in the commit message or in a header line in the file.
3. Add a row to the table above.
4. If the capture contradicts a document, fix the document and note the
   contradiction rather than deleting the capture.
