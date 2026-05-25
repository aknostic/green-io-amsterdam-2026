# KEIT, One Year On — Green IO Amsterdam 2026

Resources from Flavia Paganelli's talk *"KEIT, One Year On: Measuring Kubernetes Carbon on a Sovereign Cloud"* at [Green IO Amsterdam 2026](https://greenio.tech/conference/25/amsterdam-2026-june), 10 June 2026.

This repository is the canonical landing page for everything the talk references: the methodology behind the numbers, the tools and dashboards built along the way, citations, and follow-up reading.

If you saw the talk and want to do this yourself: start with the [reproducibility checklist](#reproducibility-checklist) at the bottom.

---

## Talk artefacts

- **Slides** — *(will be added here as PDF after the talk)*
- **Four reusable Grafana dashboards** — [`aknostic/keit:grafana/dashboards/`](https://github.com/aknostic/keit/tree/main/grafana/dashboards). Plain JSON imports for any Grafana 9+, plus Grafana Operator CRDs for GitOps users.
- **`scaleway-footprint-exporter`** — Prometheus exporter for the Scaleway Environmental Footprint API. Code + Helm chart at [`aknostic/keit:scaleway-footprint-exporter/`](https://github.com/aknostic/keit/tree/main/scaleway-footprint-exporter). Public image at `ghcr.io/aknostic/scaleway-footprint-exporter:0.2.0`.
- **`grid-intensity` public image mirror** — `ghcr.io/aknostic/grid-intensity:v0.7.0`. Upstream doesn't publish a container image; this is a public mirror so anyone can install the chart.
- **Upstream contribution** — [PR #90 to `thegreenwebfoundation/grid-intensity-go`](https://github.com/thegreenwebfoundation/grid-intensity-go/pull/90), merged 2026-05-11. Fixed a stale `provider: ElectricityMap` → `ElectricityMaps` example that was crash-looping anyone following the chart docs.

---

## TL;DR — the talk in five bullets

1. KEIT is an open-source tool for per-workload Kubernetes carbon, composing Kepler (energy) + ElectricityMaps (grid intensity) + Boavizta (embodied) into the Green Software Foundation's SCI metric. Originally built for AWS EKS.
2. Eighteen months on, we extended KEIT to **Scaleway** — a European sovereign cloud with a native carbon API (rare in 2026). Built a small Go exporter, two new Grafana dashboards, and discovered some real practitioner findings.
3. The Kepler tool we'd built KEIT around **isn't measuring on Kapsule VMs — it's modelling.** Kepler's own logs say so: no RAPL, no ACPI, falls back to a linear regression with a single CPU-time feature and a hardcoded power profile. Worse, the v0.10+ rewrite dropped VM support entirely.
4. **Everything in this stack is an estimate** — vendor APIs, energy modelling, embodied lookups, PUE figures. None are ground truth. That's the state of the art in 2026, and it's still enough to take useful action.
5. The talk's headline evidence: dashboard surfaced a 5.6× object-storage emissions spike on 2026-05-12 → diagnosed within hours as two huge buckets with no lifecycle policy → ~25 lines of YAML fixed it → drop visible by 2026-05-21. Closed-loop, multi-hundred-kg/year, all from the same dashboard. No Kepler needed.

The full story is below.

---

## Methodology

What we built, what numbers we used, where they came from, what we decided and why. Designed to be reproducible from this document alone.

### 1. Context

KEIT was built for AWS EKS. The talk extends it to **Scaleway Kapsule**, comparing the SCI we compute *ourselves* with the carbon figure Scaleway publishes natively via its Environmental Footprint API.

**Why Scaleway.** Aknostic — my company — runs production on Kapsule (managed K8s, no RAPL on VMs, no bare metal). Most practitioners are in the same situation. Scaleway also exposes a real per-resource carbon API (rare in 2026), with both kgCO2eq and m³ water — water tracking that neither Kepler nor Boavizta provide.

**The core claim.** Two ways to compute the same thing — vendor-bundled, or composable from KEIT's parts — answer different questions and support different actions. Neither is "right." Both are estimates with different blind spots.

### 2. Cluster context

Aknostic's `groupware` cluster, a Scaleway Kapsule in fr-par:
- 3 worker nodes, one per zone (`fr-par-1`, `fr-par-2`, `fr-par-3`). The talk uses PRO2-S figures as the baseline (the cluster was resized from PRO2-S to PRO2-XS on 2026-05-16 after GitLab migration freed capacity).
- Workloads: Nextcloud, Mattermost, Plausible, internal tenants, and Heystaq's hosted observability infra.
- Observability: Grafana Alloy → Heystaq Mimir (`https://mimir.heystaq.com`) under tenant `groupware`. All metrics get an external `kubernetes_cluster=groupware` label.
- GitOps: Flux v2 + HelmRelease, SOPS-encrypted secrets via age.

### 3. Data sources

**Scaleway Environmental Footprint API** ([docs](https://www.scaleway.com/en/developers/api/environmental-footprint/)). Polled hourly by a custom Go exporter ([source](https://github.com/aknostic/keit/tree/main/scaleway-footprint-exporter), image at `ghcr.io/aknostic/scaleway-footprint-exporter:0.2.0`). Queries the last 14 UTC days as separate 1-day windows; emits per-day gauges labelled by `report_date`, `project_name`, `region`, `zone`, `sku`, `service_category`, `product_category`.

Three things you discover when you actually use this API and don't read about it:
- **Data lags 3–4 days.** The exporter exposes `keit_scaleway_data_lag_days` so you can see it on the dashboard rather than wondering why "yesterday" returns zeros.
- **SKU is a (service · machine type · zone) triple**, not a per-instance identifier. `/compute/pro2_s/run_fr-par-1` aggregates *all* PRO2-S in that zone in that project. Per-pod attribution from this API alone is impossible — you have to join with K8s metadata and pick an attribution rule.
- **August 2026 methodology refresh announced.** Scaleway's own numbers will shift under us. The ground keeps moving.

**Kepler** (chart `kepler@0.6.1`, binary `v0.8.0`). DaemonSet on every worker. On a Kapsule VM, Kepler can't measure — it estimates. The first 10 lines of its pod logs are damning:

```
Kepler running on version: v0.8.0
failed to open path /dev/cpu/0/msr: no such file or directory
Unable to obtain power, use estimate method
Could not find any ACPI power meter path. Is it a VM?
using none to obtain power
Using the Ratio Power Model to estimate PROCESS_TOTAL Power
Feature names: [bpf_cpu_time_ms]
Requesting for Machine Spec: amd_epyc_7543_32_core ...
```

That's a linear regression with one input feature (CPU time in milliseconds), against a hardcoded AMD EPYC 7543 power profile, on a VM that doesn't actually run on that CPU. Plausible numbers (~165W average per node), but not measurement.

In production on Kapsule, Kepler attributes ~70% of cluster energy to `container_namespace="kernel"`, ~17% to `"system"`, and only ~12% to actual pods. That 12% is what the right-sizing dashboard works with — useful for relative rankings between pods, not for cluster-total accounting.

**Kepler upstream landscape (2026-05):**
- [PR #2217](https://github.com/sustainable-computing-io/kepler/pull/2217) (merged 2025-07-11): "chore: announce kepler reboot" — 0.9.x officially frozen, no more updates.
- [Issue #2363](https://github.com/sustainable-computing-io/kepler/issues/2363) (open since 2025-12): "DRAM metrics not supported in 0.10.0 post rewrite? kepler_container_joules_total missing" — the very metric KEIT depends on is gone in 0.10+.
- v0.11.4 (2026-02-16, latest) still doesn't restore generic VM support.
- [PR #2419](https://github.com/sustainable-computing-io/kepler/pull/2419) adds KVM via libvirt, but only for Proxmox — not Kapsule, EKS, GKE, or AKS.

So on managed Kubernetes — where the majority of K8s users actually run — Kepler currently doesn't work as advertised. This is a real regression for the practitioner community.

**grid-intensity-go** (chart pinned to tag `v0.7.0`). Provider `ElectricityMaps`, location `FR`, deployed via Flux with a SOPS-encrypted API key. Image at `ghcr.io/aknostic/grid-intensity:v0.7.0` — public mirror Aknostic published since [the upstream Dockerfile expects a goreleaser-built binary that isn't in the source tree](https://github.com/thegreenwebfoundation/grid-intensity-go/blob/main/Dockerfile). The chart has no `podAnnotations` hook, so we use a Flux `postRenderers` Kustomize patch to inject `prometheus.io/scrape` annotations onto the Pod template. (Adding `podAnnotations` to the chart is a clean upstream contribution opportunity — I haven't filed it yet.)

PR #90 to upstream fixed a stale `provider: ElectricityMap` (singular) example in the chart values that was crash-looping anyone following the docs. Three lines of comment changes; one fewer trap for the next person. [Merged 2026-05-11 by Chris Adams](https://github.com/thegreenwebfoundation/grid-intensity-go/pull/90).

**Boavizta for embodied** (hardcoded). One lookup against the Boavizta API:

```bash
curl -s -X POST "https://api.boavizta.org/v1/cloud/instance?duration=8760" \
  -H "Content-Type: application/json" \
  -d '{"provider":"scaleway","instance_type":"pro2-s"}'
```

Returns **13.0 kgCO2eq/instance/year** (range 8.142–22.84). For our 3-node cluster: **195 kgCO2eq over 5 years** (matching the existing dashboard's amortization). Boavizta's range is wide — uncertainty is real. The point is *we know how to calculate it*, not that the number is precise.

### 4. PUE

Scaleway publishes 2024 PUE per DC at [scaleway.com/en/environmental-leadership](https://www.scaleway.com/en/environmental-leadership/):

| DC | Region | PUE | WUE |
|---|---|---|---|
| DC2 PAR1 | Paris | 1.45 | 0.009 |
| DC3 PAR1 | Paris | 1.39 | 0.00009 |
| DC4 | Paris | 1.44 | 0.00002 |
| DC5 PAR2 | Paris | 1.25 | 0.25 |
| AMS1 | Amsterdam | 1.38 | 1.64 |
| AMS2 | Amsterdam | 1.40 | — |
| AMS3 | Amsterdam | 1.20 | — |
| WAW1 | Warsaw | 1.50 | — |
| WAW2 | Warsaw | 1.24 | — |
| WAW3 | Warsaw | 1.50 | — |

Company-wide 2024 average: PUE 1.37.

**What we use: 1.38** — the average of the four Paris DCs (since the cluster is firmly in `fr-par`). Scaleway doesn't publish a zone↔DC mapping, so we can't pick the specific DC each of our three workers sits in — but we can scope to the region we know. 1.38 is tighter than the company-wide 1.37 (which averages in Amsterdam and Warsaw).

Talk-relevant variance: within Paris alone, the published PUE range is **1.25 → 1.45** — a 16% spread that's invisible to us as a customer. The granular data exists; the mapping that would let me use it doesn't.

### 5. SCI formula as applied

Per the [Green Software Foundation's SCI specification](https://sci-guide.greensoftware.foundation/):

```
SCI = ((E × I) + M) / R
```

- **E** = energy used (kWh, from Kepler)
- **I** = grid carbon intensity (gCO2eq/kWh, from grid-intensity-exporter)
- **M** = embodied emissions amortized over the time window (gCO2eq, from hardcoded Boavizta value)
- **R** = functional unit (cluster-window for the side-by-side; could be per-request, per-day, etc.)

In our dashboard's variables:

| Variable | Value | Source |
|---|---|---|
| `PUE` | **1.38** | Average of Scaleway's four Paris DCs (2024) |
| `Embodied` (gCO2e over query range) | `($__range_s / 157766400) × 195 × 1000` | Boavizta → 195 kg/cluster/5yr |
| `Energy` | from `kepler_container_joules_total` × kWh conversion | Kepler |
| `EnergyCarbonIntensity` | from `grid_intensity_carbon_average` | grid-intensity-exporter |
| `SCI` | `(Energy × CarbonIntensity × PUE) + Embodied` | composition |

### 6. The May 12 → May 21 closed-loop story

The talk's headline evidence that this stack delivers practitioner value:

1. **2026-05-12** — The Carbon Hotspots dashboard surfaces object-storage emissions jumping 5.6× across two Scaleway projects. Compute, network, containers all flat — clearly a storage event, not a workload event.
2. **Diagnosis** (~30 minutes in the Scaleway console) — two largest buckets in the entire org had **no lifecycle policy**:
   - `groupware-postgres-backups`: 2.57 TB, 188,605 objects (mostly historical GitLab Postgres backups)
   - `heystaq-mimir-blocks-v5`: 2.35 TB, 241,586 objects (Mimir long-term metric blocks)
3. **2026-05-18** — Colleague's commit in our internal infrastructure repo: dropped S3 versioning on TSDB buckets (which was doubling writes on append-only stores) and added a lifecycle policy to postgres-backups.
4. **2026-05-21** — Fix deployed via OpenTofu. Dashboard immediately shows storage emissions dropping; the storage time-series panel visibly steps down.

Full arc from anomaly detection to fix-verified: **9 days, ~25 lines of YAML, multi-hundred-kg/year carbon win.** The Kepler-side stack isn't needed for any step of this story — vendor data + dashboard discipline is enough.

After the fix landed, the new #1 hotspot in the Carbon Hotspots dashboard became **compute running in Warsaw**: same machine types as in Paris, but Poland's coal-heavy grid pushes emissions ~20× higher than the same machine in fr-par. Region choice ≫ tier choice for sustainability outcomes. **Fix one thing and the next hotspot reveals itself.**

### 7. Other findings worth flagging

These came out of the same dashboards over six weeks of work, none of them needing precise SCI numbers:

- **The observability stack is the heaviest emitter** in the entire Aknostic org — Heystaq alone is ~42% of total Scaleway emissions. *"The system I built to measure carbon emits the most carbon on my infra."*
- **Same machine, 20× more emissions in Warsaw vs Paris** — the regional grid effect. Visualised as a heatmap in the Carbon Hotspots dashboard.
- **A second Kubernetes cluster** in the Groupware Scaleway project that Kepler doesn't see — adds ~37% to the compute footprint that the KEIT-vs-Scaleway comparison can't account for.
- **ClickHouse storing 6,320 events** — `plausible-clickhouse-0` is the heaviest single pod on the cluster by Kepler's ranking. ClickHouse is designed for billions; we have 6,320 events in 66 days. Tool/data mismatch of five orders of magnitude.
- **Envoy Gateway wasting 4.5 GiB requested memory** — three replicas × 544 MiB requested, ~39 MiB actually used (7% utilisation).
- **13 pods with no CPU request set** — including the entire Cilium DaemonSet. Scheduling-unstable AND invisible to Kepler's ratio-based attribution.
- **Postgres standby drawing 1W while primary draws 14W** — Kepler is sensitive enough to spot roles within a stateful set. Useful, but a reminder that "low-utilisation pod" isn't always a right-sizing candidate (the standby needs that capacity when it gets promoted).

---

## Decisions log

Why specific choices were made, with dates, in case future-you wonders.

| Date | Decision | Why |
|---|---|---|
| 2026-04-24 | Pivot from "rebuild KEIT on bare-metal" to "extend KEIT to Scaleway via the Footprint API" | Scaleway's API is real and ADEME-backed; bare-metal rebuild was disproportionate work. Better practitioner story (managed K8s is the common case). |
| 2026-04-26 | Custom Go exporter, not a generic Prometheus adapter | The Footprint API's hierarchical response shape doesn't fit a static template; native code is simpler. ~190 LoC. |
| 2026-05-03 | Rolling 14-day window in the exporter, not "yesterday only" | Discovered Scaleway data lag of 3–4 days. A "yesterday only" exporter goes silent under lag. Per-day labels with `report_date` + a `keit_scaleway_data_lag_days` gauge surface the lag rather than hide it. |
| 2026-05-10 | Pin upstream `grid-intensity-go` GitRepository to **tag** `v0.7.0`, not `branch: main` | Upstream chart changes shouldn't auto-deploy to production. Bumps are deliberate. |
| 2026-05-10 | Build and publish our own image at `ghcr.io/aknostic/grid-intensity:v0.7.0` | Upstream doesn't publish one. Public mirror unblocks any KEIT user, not just Aknostic. |
| 2026-05-10 | Hardcode embodied at 195 kg / 5yr for the cluster (Boavizta) | Same pattern KEIT already uses elsewhere. Boavizta supports Scaleway natively. |
| 2026-05-10 | Use Paris-DC-average PUE = 1.38, not the company-wide 1.37 nor a per-DC value | Cluster is firmly in `fr-par`. Per-DC isn't possible since Scaleway doesn't publish zone↔DC mapping. Paris-DC average is the tightest defensible number. |
| 2026-05-18 / 2026-05-21 | Storage lifecycle fix | Resolution of the May 12 spike. The headline closed-loop story above. |

---

## AI in the methodology

This work was done with [Claude](https://claude.ai/) (Anthropic) as an active thinking partner — not a code generator. Recording where it helped and didn't, because how the work happens matters as much as what gets produced.

**Where Claude actively helped:**
- **Dashboard authoring** — most of the PromQL in the four KEIT-related Grafana dashboards was Claude-generated, then iterated on with real screenshots. Building a panel from scratch (correct field configs, transformations, gridPos layout) is ~5× faster than doing it by hand.
- **API response interpretation** — the structural observation that Scaleway's SKU is a `(service · machine_type · zone)` triple came from Claude reading the raw JSON and surfacing it. Easy to miss skimming docs; obvious once stated.
- **Cross-repo timeline correlation** — explaining the May 21 storage drop required correlating commits across two repos, the cluster's GitOps reconcile state, and Scaleway's emission curves. Claude pieced the timeline together.
- **Doc drafting** — this README itself was co-drafted with Claude. The structure is mine; most of the prose drafting and number-crunching is Claude's.
- **Investigation prompts** — when the May 12 spike appeared, Claude was the first to ask "which storage type — block or object?" That narrowed the search instantly.

**Where Claude was NOT the right tool:**
- **Judgment calls** — "is ClickHouse storing 6,320 events overkill?" needs domain knowledge about Plausible-as-a-product, not what ClickHouse can do. "Should the talk criticise the Kepler rewrite or stay neutral?" needs a sense of audience and upstream relationships. Both human calls.
- **Talk arc and framing** — the decision that "everything is an estimate" is the talk's central reframe. That came from sitting with the data, not from any single Claude exchange.
- **The pessimism filter** — Claude defaults to encouraging. The "but is this useful?" moment that forced a pivot from "comparison-as-headline" to "lifecycle-policies-as-headline" came from a deliberate human pushback against AI-cheerful drift.

**Honest reflections:**
- About 40% of the ~40 hours of work on this talk involved Claude in some role (and counting — slides still to build). Most of that was analysis (read this, find the pattern) or boilerplate (write this query, build this panel). Maybe 5% was genuine collaboration on *what to do next*.
- Watch out for: AI generating dashboard panels that *look right* but quietly use the wrong PromQL pattern (caught at least twice during this project, both visually). AI over-committing to a frame once stated. AI defaulting to "add more bullets" when the right work was almost always cuts.
- The irony of using a carbon-heavy AI tool to investigate carbon is not lost on me. The ratio of insight-per-watt felt favourable compared to doing it manually, but that's an assumption, not a measurement.

---

## Reproducibility checklist

To do this yourself on another Scaleway Kapsule cluster (or, with minor adjustments, on EKS/GKE/AKS):

1. **Install Kepler v0.8.0 / chart 0.6.1** (pre-rewrite). The 0.10+ rewrite dropped VM support and the `kepler_container_joules_total` metric KEIT uses — don't upgrade yet on managed K8s. ([Issue #2363](https://github.com/sustainable-computing-io/kepler/issues/2363).)
2. **Install grid-intensity-exporter**. Chart pinned to `v0.7.0`. Image at `ghcr.io/aknostic/grid-intensity:v0.7.0` (public). Provider `ElectricityMaps` (plural — *not* `ElectricityMap`). Get an API key from [electricitymaps.com](https://electricitymaps.com/).
3. **Provision a Scaleway IAM Application** with `ProjectReadOnly` + Environmental Footprint read perms. Install [`keit/scaleway-footprint-exporter`](https://github.com/aknostic/keit/tree/main/scaleway-footprint-exporter) with credentials via Secret.
4. **Look up embodied for your worker SKU(s)** via Boavizta API: `POST https://api.boavizta.org/v1/cloud/instance?duration=8760` with `{"provider":"<your-provider>","instance_type":"<sku>"}`. Multiply by node count × your assumed lifetime.
5. **Look up your zones' DC PUE** on your provider's sustainability page. Default to the region-DC-average for the region your cluster is in; use a specific DC's value only if you know which DC your zone maps to.
6. **Import the four dashboards** from [`aknostic/keit:grafana/dashboards/`](https://github.com/aknostic/keit/tree/main/grafana/dashboards). See that directory's README for required metrics and tweakable constants.
7. **Set the dashboard's `cluster` template variable** to your `kubernetes_cluster` external label value. Set `project` to your Scaleway project name. Set `PUE_scaleway` and `embodied_kg_5yr` to your values.

Whole flow takes a working day end-to-end if the cluster, observability stack, and registry creds are already in place.

---

## Contact

- **Aknostic** (consultancy): [aknostic.com](https://aknostic.com) — contact form there
- **KEIT**: [github.com/aknostic/keit](https://github.com/aknostic/keit)
- **CNCF TAG Environmental Sustainability**: [tag-env-sustainability.cncf.io](https://tag-env-sustainability.cncf.io/)

Talk feedback, corrections, questions, or "I tried this and here's what I found" — all very welcome. [Open an issue](https://github.com/aknostic/green-io-amsterdam-2026/issues) on this repo.

---

## License

The text of this document is [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). The code and dashboards it references inherit KEIT's license.
