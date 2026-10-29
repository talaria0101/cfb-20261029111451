# Cost

All Cloudflare prices below are list prices read on 2026-10-29 from the pages
cited at the end. They exclude tax, committed-use discounts and enterprise
negotiation. The data-plane model is parametric because hardware, power and
bandwidth prices vary by country and contract; no hardware price is invented
here.

The point of this page is not to prove that self-hosting is cheaper. It is to
show where it wins, where it loses, and what has to be true for the answer to
change.

## 1. Control-plane list prices

| Item | Included | Overage | Source |
| --- | --- | --- | --- |
| Workers Paid | $5/month; 10M requests; 30M CPU-ms | $0.30/M requests; $0.02/M CPU-ms | Workers pricing |
| Static assets | Free requests, unlimited | None | Workers static assets |
| D1 | 5 GB storage; 25B rows read; 50M rows written | $0.75/GB-month; $0.001/M rows read; $1.00/M rows written | D1 pricing |
| Durable Objects | 1M requests; 400,000 GB-s; 5 GB SQLite storage | $0.15/M requests; $12.50/M GB-s; $0.20/GB-month | DO pricing |
| R2 Standard | 10 GB; 1M Class A; 10M Class B | $0.015/GB-month; $4.50/M Class A; $0.36/M Class B; egress free | R2 pricing |
| R2 Infrequent Access | None | $0.01/GB-month; $9.00/M Class A; $0.90/M Class B; $0.01/GB retrieval; 30-day minimum | R2 pricing |
| Queues | 1M operations | $0.40/M operations | Queues pricing |
| Workflows | 500,000 steps; 1 GB-month storage | $0.80/100,000 steps; $0.20/GB-month | Workflows pricing |
| Containers | 25 GiB-hours memory; 375 vCPU-min; 200 GB-hours disk | $0.0000025/GiB-s; $0.000020/vCPU-s; $0.00000007/GB-s | Containers pricing |
| Container egress, NA/EU | 1 TB | $0.025/GB | Containers pricing |
| Container egress, Oceania/Korea/Taiwan | 500 GB | $0.05/GB | Containers pricing |
| Container egress, elsewhere | 500 GB | $0.04/GB | Containers pricing |
| Tunnels | 1,000 per account | $4.50/tunnel/month | Tunnels for Platforms |
| Access | Free plan: teams under 50 users | $7/user/month, paid annually | Access pricing |
| Workers Builds | 6,000 minutes/month | $0.005/minute | Builds pricing |

D1's 10 GB per-database limit cannot be raised. The 1 TB per-account limit is a
separate cap.

## 2. Worked Cloudflare examples

These use the formulas above and no other assumptions except where stated. The
arithmetic was done with a calculator, not copied from a quote.

### 2.1 One always-on container, standard-2

Instance: 1 vCPU, 6 GiB memory, 12 GB disk. Running 730 hours per month.

Memory:

```
6 GiB * 730 h = 4,380 GiB-h
4,380 - 25 included = 4,355 GiB-h
4,355 * 3,600 = 15,678,000 GiB-s
15,678,000 * $0.0000025 = $39.20
```

Disk:

```
12 GB * 730 h = 8,760 GB-h
8,760 - 200 included = 8,560 GB-h
8,560 * 3,600 = 30,816,000 GB-s
30,816,000 * $0.00000007 = $2.16
```

CPU at 30% average utilization:

```
0.30 * 730 h * 3,600 = 788,400 vCPU-s
375 vCPU-min * 60 = 22,500 vCPU-s included
788,400 - 22,500 = 765,900 vCPU-s
765,900 * $0.000020 = $15.32
```

Total: **$56.67/month of Containers usage**, plus the $5/month Workers Paid
plan, Workers and DO usage, and egress.

At 100% CPU utilization:

```
730 * 3,600 = 2,628,000 vCPU-s
2,628,000 - 22,500 = 2,605,500
2,605,500 * $0.000020 = $52.11
```

Total: **$93.46/month of Containers usage**, plus the $5/month Workers Paid
plan (about **$98.46** on a standalone account). The $5 is per account, so it is
not multiplied by the number of containers.

### 2.2 One hour of sandbox per day

Instance: `standard-1` (0.5 vCPU, 4 GiB memory, 8 GB disk), running 1 hour per
day (30 hours per month). Assume it uses its full 0.5 vCPU while running, which
is the same 0.5 vCPU average as the earlier draft. Cloudflare's custom instance
types require at least 1 vCPU and at least 3 GiB of memory per vCPU, so the
smaller 1 vCPU / 2 GiB shape used in an earlier draft was not deployable.

Memory: `4 * 30 = 120 GiB-h`; over 25 included by 95 GiB-h; `95 * 3,600 *
0.0000025 = $0.855`.

Disk: `8 * 30 = 240 GB-h`; over 200 included by 40 GB-h; `40 * 3,600 *
0.00000007 = $0.01`.

CPU: `0.5 * 30 * 3,600 = 54,000 vCPU-s`; over 22,500 by 31,500; `31,500 *
0.00002 = $0.63`.

Total: **about $1.50/month of Containers usage**. Containers require the
Workers Paid plan, which is $5/month, so a standalone account pays about
**$6.50/month** before Workers and DO requests. This is why billing only while
a sandbox runs is attractive for bursty use and why a dedicated machine is a
bad fit for the same workload.

### 2.3 R2 storage

1 TB stored for a month, Standard class, no operations:

```
1,000 GB - 10 GB free = 990 GB-month
990 * $0.015 = $14.85
```

10 TB stored:

```
10,000 GB - 10 = 9,990 GB-month
9,990 * $0.015 = $149.85
```

Cloudflare bills R2 storage in decimal GB; its own pricing example treats 1 TB
as 1,000 GB. Both examples above use that convention.

Infrequent Access is $0.01/GB-month but has a 30-day minimum and a $0.01/GB
retrieval fee. For data read once a month, the retrieval fee can exceed the
storage saving.

### 2.4 A small control plane

Assume 10 machines, 10 tenants, 100,000 API requests per month, 1 GB of metadata,
1,000 audit events per day, and one DO per machine with 15-second heartbeats.

- Workers: 100,000 requests is 0.1M, within the 10M included. CPU depends on the
  code; assume 5 ms per request = 500,000 CPU-ms, under 30M. $5.
- D1: 1 GB storage, within 5 GB. Rows read and written are far below 25B/50M.
  $0.
- DO: 10 machines * 172,800 heartbeats/month = 1,728,000 requests. Included is
  1M, so 728,000 over. The billing page rounds billable usage up to the next
  million requests, so 1M billable * $0.15/M = $0.15. Duration depends on
  hibernation; if each DO is active only during a request, duration is small. If
  each DO is active continuously at 128 MB: `10 * 0.128 GB * 730 h * 3,600 =
  3,363,840 GB-s`; over 400,000 by 2,963,840; rounded up to 3,000,000 GB-s; `3 *
  $12.50 = $37.50`. Hibernation matters.
- R2: 10 GB of logs and artifacts, within the free tier. $0.
- Queues: under 1M operations. $0.
- Tunnels: 10, within 1,000. $0.
- Access: under 50 users. $0.

Control-plane total: **$5.15 to about $43/month**, dominated by DO duration if the
per-machine objects never hibernate. Design the DO to hibernate and use alarms;
otherwise the cheap control plane is not cheap.

### 2.5 Relay WebSocket cost

The default transport is one WebSocket per machine, terminated by a Durable
Object. Billing is not one request per message. Cloudflare's DO pricing page
says: a request is needed to create the WebSocket connection; outgoing messages
and WebSocket protocol pings are free; incoming messages are billed at a 20:1
ratio for request billing. Twenty incoming messages count as one billable
request.

Heartbeats, at one incoming message per machine every 15 seconds:

```
10 machines * 172,800 = 1,728,000 incoming messages/month
1,728,000 / 20 = 86,400 billable requests
86,400 is under the 1M included; $0
```

Ingress and logs add incoming messages. A site serving 1M HTTP requests/month
with two incoming relay messages per request:

```
2,000,000 / 20 = 100,000 billable requests
100,000 * $0.15/M = $0.015
```

A workload that streams 1 GB of logs in 64 KiB frames sends 16,384 messages,
billed as 819 requests, about $0.0001. At 1 KB frames the same gigabyte sends
1,048,576 messages, billed as 52,429 requests, about $0.008. Frame size still
matters, but the ratio makes the relay cheap for control traffic.

Duration is billed while the DO is active. With the hibernation API, an idle
socket does not hold the object in memory; a message wakes it. A DO that never
hibernates costs `0.128 GB * 730 h * 3,600 = 336,384 GB-s`; at $12.50/M GB-s that
is `0.336384 * $12.50 = $4.20/month` per machine at 128 MB. Hibernation is
required, not optional.

The relay still adds a hop and a per-machine failure domain. Native Cloudflare
Tunnel removes the DO from the data path and is better for sustained throughput.
Use the relay where the network forbids UDP, and the native tunnel or direct S3
where throughput dominates.

### 2.6 Workers CI at the limit

6,000 minutes included. A heavy project using 10,000 minutes:

```
10,000 - 6,000 = 4,000 minutes
4,000 * $0.005 = $20.00
```

$20 is not the problem. The 20-minute job timeout, the 6-concurrent-build limit
and the 4 vCPU / 8 GB / 20 GB build machine are the problem. Self-hosting CI is
about capability and queue time before it is about price.

## 3. Data-plane model

For one machine-month:

```
C_machine = H + E + N + S + R + O + F
```

| Term | Meaning | How to get it |
| --- | --- | --- |
| H | Hardware amortization | Purchase price / useful life in months |
| E | Electricity | Watts * 730 h / 1,000 * price per kWh |
| N | Network | Port cost + transit or bandwidth overage |
| S | Storage media | Disk purchase / life, plus replacement |
| R | Redundancy | Replicas, erasure coding overhead, spare capacity |
| O | Operator time | Hours per month * loaded hourly cost |
| F | Facilities | Rack space, UPS, cooling, IP addresses |

Per unit of capacity:

```
C_unit = C_machine / usable_units
```

Usable units must account for the 20% headroom, the failure reserve, and the
difference between provisioned and active utilization. A machine with 64 GiB is
not 64 GiB of sellable memory.

Electricity example, with the price as an input:

```
150 W * 730 h / 1,000 * $0.15/kWh = $16.43/month
```

This is arithmetic, not a claim about your power price.

### 3.1 A scenario, not a quote

Assume a machine with 16 vCPU, 64 GiB memory and 2 TB NVMe. Assume it can host 10
"standard-2 equivalent" containers (1 vCPU, 6 GiB each) by memory, and that 80%
utilization means 8 running. Exclude storage and operator time for the first
table.

| Hardware H per month | Power | Cost per running container-month at 80% |
| --- | --- | --- |
| $50 | $16.43 | $8.30 |
| $100 | $16.43 | $14.55 |
| $200 | $16.43 | $27.05 |
| $400 | $16.43 | $52.05 |
| $600 | $16.43 | $77.05 |

Cloudflare's standard-2 at 30% CPU is $56.67/month. On these assumptions, the
machine wins below roughly $437/month of hardware cost at 80% utilization,
before operator time and storage.

Now add $500/month of operator time:

| Hardware H per month | Cost per running container-month at 80% |
| --- | --- |
| $50 | $70.80 |
| $100 | $77.05 |
| $200 | $89.55 |
| $400 | $114.55 |
| $600 | $139.55 |

At $500/month of operator time, Cloudflare is cheaper for this shape unless the
machine is busier, the operator time is shared across more workloads, or the
workload needs something Cloudflare will not sell (long jobs, custom hardware,
private network access, no egress bill).

That is the honest result: **self-hosting wins on utilization and egress, not on
small idle fleets.**

### 3.2 Where self-hosting wins

- **Egress-heavy workloads.** 10 TB/month from Containers in NA/EU is
  `(10,000 - 1,000) * $0.025 = $225.00` in overage. An owned machine with a
  committed transit link often pays a fraction of that, or nothing marginal.
- **High sustained CPU.** A container at 100% CPU costs $93.46/month on
  Cloudflare. An owned machine at high utilization amortizes much lower.
- **Long-running jobs.** Workers and Containers have time limits. A build that
  runs for hours needs a machine.
- **Custom hardware.** GPUs, large local NVMe, special accelerators. Cloudflare's
  instance types do not cover them.
- **Data gravity.** A 50 TB dataset read locally does not pay egress, and does not
  wait on a WAN.
- **Private network access.** A CI job that must reach an internal database is
  simpler on the same LAN.

### 3.3 Where self-hosting loses

- **Low utilization.** An idle machine still draws power and depreciates.
- **Small teams.** Operator time dominates. One hour a week at a loaded rate can
  exceed the Cloudflare bill for a small workload.
- **Bursty workloads.** Cloudflare scales to zero; an owned machine does not.
- **Global latency.** One site cannot serve users on three continents well.
- **Durability.** R2's durability is not free to reproduce. Three replicas plus
  offsite plus restore drills is real cost.
- **Compliance.** If you need a certification, you inherit the audit burden.

## 4. Break-even formulas

### 4.1 Compute

Let `U` be the fraction of a machine's capacity that is sold or used, `H` the
monthly hardware cost, `E` power, `O` operator time, and `P_cf` the Cloudflare
price for equivalent capacity. Self-hosting is cheaper when:

```
(H + E + O) / (units * U) < P_cf
```

Solve for the utilization that makes it break even:

```
U > (H + E + O) / (units * P_cf)
```

Using the scenario: `H=200`, `E=16.43`, `O=0`, `units=10`, `P_cf=56.67`:

```
U > 216.43 / 566.7 = 0.382
```

So the machine needs to be about 38% utilized to match Cloudflare on this shape,
before storage and redundancy.

Add `O=500` and the numerator becomes 716.43:

```
U > 716.43 / 566.7 = 1.264
```

The threshold is above 100%, so at `H=200` this shape never breaks even against
Cloudflare once operator time is included. Even at `H=50` the threshold is
`566.43 / 566.7 = 0.9995`, which means the machine must be fully utilized. That
is the strongest argument on this page against self-hosting small fleets.

### 4.2 Storage

Let `C_gb` be the owned cost per GB-month (media amortization plus redundancy
plus power plus operator share) and `P_r2` the R2 Standard price.

```
C_gb < $0.015  ->  owned storage is cheaper per GB-month
```

A 2 TB NVMe at $200 with a 5-year life is `$200 / 60 / 2,000 = $0.00167/GB-month`
for one copy. Four copies (three replicas plus one spare drive) is about
`$0.0067/GB-month`, before power and operator time. At that point owned storage
is cheaper than R2 Standard on media alone, but not necessarily after redundancy,
replacement and the cost of running a storage cluster.

R2 Infrequent Access at $0.01/GB-month plus $0.01/GB retrieval is often the
cheapest place for cold backups, because you are not paying to keep a second
store warm.

### 4.3 CI

Cloudflare Workers Builds: `$0.005/minute` after 6,000 minutes. An owned runner
costs `(H + E + O) / minutes_used`. At `$200/month` all-in and 10,000 minutes
used, that is `$0.02/minute`, four times the Cloudflare overage. At 100,000
minutes, it is `$0.002/minute`, cheaper. Heavy CI wins; light CI does not.

## 5. Hidden costs

These are the costs that make a "free" self-hosted stack expensive:

- **Operator time.** Patching, upgrades, incidents, backups, restore drills,
  certificate rotation, capacity planning, security review. This is usually the
  largest line item for a small team.
- **Redundancy.** Three replicas or erasure coding means 1.3x to 3x the raw
  capacity. A spare machine is idle until it is not.
- **Replacement.** Disks fail. Keep a spare and a replacement budget.
- **Power and cooling.** A home lab in a warm room costs more than the nameplate
  wattage suggests.
- **Rack space and IPs.** Colocation, cross-connects, IPv4 addresses, UPS.
- **Bandwidth commitments.** A cheap transit link has a commit and a burst fee.
- **Monitoring and logging.** Storage for logs grows faster than you expect.
- **Backup egress.** Restoring from R2 is free for egress, but restoring from a
  remote site over a WAN is not.
- **Licence review.** AGPL and BUSL components can force a legal review and a
  re-architecture if the product is distributed.
- **Security patches.** A hypervisor and a kernel need a maintenance window. That
  window is a cost.
- **Idle capacity.** A machine sized for peak is idle most of the month.

## 6. Cost guardrails in the design

The architecture should make cost visible and bounded:

- Every workload has a quota: CPU, memory, disk, egress, and a TTL for sandboxes.
- A sandbox without a TTL is a bug; the broker enforces a maximum.
- Idle containers can scale to zero on a schedule, not only on request.
- Storage lifecycle: hot -> warm -> R2 IA -> delete, with a policy per bucket.
- Snapshot retention is explicit; a snapshot without an expiry is rejected.
- The dashboard shows cost per tenant, machine and workload, using the same
  formulas as this page.
- Alerts fire at 70% and 90% of a quota, not after the bill.
- The control plane has a hard budget alarm; a runaway DO or log stream is a
  design failure, not a surprise.

## 7. What is not included

- Hardware prices. They vary too much to state without a source and a date.
- Power prices. The $0.15/kWh above is an example input, not a quote.
- Bandwidth prices. Same.
- Operator salaries or loaded rates. The $500/month is a scenario input.
- Taxes, currency, financing, and depreciation policy.
- Enterprise discounts, committed-use discounts, and private pricing.
- The cost of a security incident, a data loss, or downtime.
- The cost of building the control plane and the agent. That is the largest
  cost of this project and it is not modelled here.

## 8. Sources

Read 2026-10-29:

- Workers pricing: https://developers.cloudflare.com/workers/platform/pricing/
- Static assets: https://developers.cloudflare.com/workers/static-assets/
- D1 pricing: https://developers.cloudflare.com/d1/platform/pricing/
- D1 limits: https://developers.cloudflare.com/d1/platform/limits/
- Durable Objects pricing: https://developers.cloudflare.com/durable-objects/platform/pricing/
- R2 pricing: https://developers.cloudflare.com/r2/pricing/
- R2 limits: https://developers.cloudflare.com/r2/platform/limits/
- Queues pricing: https://developers.cloudflare.com/queues/platform/pricing/
- Workflows pricing: https://developers.cloudflare.com/workflows/reference/pricing/
- Containers pricing: https://developers.cloudflare.com/containers/platform/pricing/
- Containers limits: https://developers.cloudflare.com/containers/platform/limits/
- Tunnels for Platforms: https://developers.cloudflare.com/tunnel/features/tunnels-for-platforms/
- Access pricing: https://www.cloudflare.com/zero-trust/products/access/
- Workers Builds limits and pricing: https://developers.cloudflare.com/workers/ci-cd/builds/limits-and-pricing/

Captured copies of these pages are in `evidence/cf-docs/`. Every price in this
file has a capture there.
