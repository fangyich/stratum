# Stratum Platform — GCP Cost Analysis

**Region:** us-central1 (Iowa)  
**Pricing basis:** GCP list prices as of April 2026. All prices in USD.  
**Scope:** Cloud-hosted operational cluster only. ML training cluster costs are per-run and shown separately.

---

## Assumptions

### Fleet

| Parameter | Value | Rationale |
|---|---|---|
| Active robot units | 20 | Mid-point of target fleet; scales sub-linearly |
| Concurrent active print jobs | 20 | All units may print simultaneously |
| Pipeline runs per day | 8 | Architect-triggered; bursty, short-lived |

### Telemetry

| Parameter | Value | Rationale |
|---|---|---|
| Job-level sync frequency | Once per 30 seconds per unit during active jobs | Higher cadence during active task execution; suspended when no task is running |
| Job-level payload per sync | ~2 KB (sampled baseline) | Task state, print/extrusion speed, extrusion output, motion controller values — compressed |
| Unit-level sync frequency | Once per 60 seconds per unit, continuously | Low fixed cadence regardless of job state |
| Unit-level payload per sync | ~1 KB | Servo states, temperatures, voltages, connectivity state, firmware version — compressed |
| Anomaly-windowed burst size | ~500 KB per fault event | Full-fidelity capture for 60 seconds around fault; per spec |
| Fault events per unit per month | 300 | Includes E-Stops, recoverable faults, and motion controller anomalies |
| Active job hours per unit per day | 8 | Single shift |
| Days per month | 30 | Working days |

**Telemetry volume derivation (per unit per month):**

```
Job-level signals (active jobs only):
  Syncs  = (8 hrs × 3600 s/hr) / 30 s × 30 days = 28,800 syncs
  Data   = 28,800 × 2 KB = 57,600 KB = ~56 MB

Unit-level signals (continuous, 24/7):
  Syncs  = (24 hrs × 3600 s/hr) / 60 s × 30 days = 43,200 syncs
  Data   = 43,200 × 1 KB = 43,200 KB = ~42 MB

Fault bursts = 300 events × 500 KB = 150,000 KB = ~146 MB

Total per unit   = ~244 MB/month
Total (20 units) = ~4,880 MB/month ≈ 4.88 GB/month
```

Note: fault burst data is part of the diagnostic hardware telemetry snapshot category and flows through the diagnostic sync queue, not the live telemetry buffer. It does not contribute to the Parquet training prefix. Only job-level live telemetry signals are written to Parquet.

### Storage

| Parameter | Value | Rationale |
|---|---|---|
| Pipeline stage outputs (IFC → motion path) per project | ~150 MB | Parsed geometry + slices + toolpaths + motion path |
| Active Print Projects per month | 10 | New projects; existing outputs accumulate |
| Job packages (assembled at dispatch) | ~80 MB each | Motion path + task manifest + metadata |
| Job packages dispatched per month | 600 | 1 job/unit/day × 20 units × 30 days |
| Mender OTA artifacts | ~200 MB per release, 1 release/month | Edge firmware + application update package |
| VictoriaMetrics retention | 21 days (3 weeks) | Full-fidelity operational telemetry |
| Parquet telemetry (training prefix) retention | 12 months active, then Nearline | Long-term training data accumulation |
| Diagnostic data (command logs, execution traces) | ~20 MB per job | Per spec; high-fidelity around faults |
| Ground surface data | ~8 MB per job | Spatial height map per completed ground scan |

### Network

| Parameter | Value | Rationale |
|---|---|---|
| Telemetry ingress (edge → Orchestration Tier) | 4.88 GB/month | Computed above; ingress is free on GCS |
| Artifact egress (OTA packages to robots) | 20 units × 200 MB = 4 GB/month | OTA delivery from GCS to robot secondary uplink |
| Artifact egress (job packages to robots) | 600 dispatches × 80 MB = 48 GB/month | Job package delivery at dispatch |
| API egress (Application Tier → Orchestration) | ~2 GB/month | Stage output previews, job packages, dashboard data; estimate |
| Total egress to internet | ~54 GB/month | Dominated by job package delivery at scale |

---

## Operational Cluster — Cost Breakdown

### 1. GKE Control Plane

| Item | Calculation | Monthly cost |
|---|---|---|
| Control plane fee | $0.10/hr × 730 hrs = $73.00 | **$0.00** |
| GKE free tier credit | −$74.40/billing account for one zonal cluster | −$73.00 (fully offset) |

**Control plane: $0.00/month** — the free tier credit fully covers one zonal Standard cluster.

---

### 2. Kubernetes Nodes — General Pool

Runs: API services, NATS JetStream, EMQX, Keycloak, Mender Server, VictoriaMetrics, Traefik, Prometheus, Grafana, Loki, Tempo, BentoML.

**Instance choice:** `e2-standard-4` (4 vCPU, 16 GB RAM) — good balance for mixed stateful/stateless workloads.  
**Node count:** 3 nodes running continuously.  
**Pricing model:** Spot VMs — all workloads on this pool are either stateless (API, NATS, EMQX) or quickly recoverable (VictoriaMetrics with PVC persistence). Spot price for `e2-standard-4` in us-central1 is approximately $0.0617/hr.

| Item | Calculation | Monthly cost |
|---|---|---|
| 3 × e2-standard-4 spot | 3 nodes × $0.0617/hr × 730 hrs | **$135.22** |

> **Spot risk note:** If all 3 nodes are preempted simultaneously (rare with Spot VMs — GCP reclaims incrementally), stateful services (VictoriaMetrics, PostgreSQL connections via PgBouncer) require a brief restart. Mitigate by running 2 of 3 nodes on-demand as a floor: 1 on-demand ($0.134/hr) + 2 spot ($0.0617/hr) = $97.82 + $90.08 = $187.90/month. The analysis uses full-spot as the base case.

---

### 3. Kubernetes Nodes — Pipeline Pool

Runs: Toolpath generation pipeline Jobs (parse, slice, toolpath, motion path). Scales to zero between pipeline runs.

**Instance choice:** `c2-standard-4` (4 vCPU, 16 GB RAM, compute-optimised) — CPU-intensive pipeline stages benefit from the higher clock speed vs e2.  
**Spot price for `c2-standard-4`** in us-central1: approximately $0.050/hr (spot discount ~76% off $0.210/hr on-demand).  
**Usage model:** 8 pipeline runs/day × 4 stages × average 8 minutes/stage = ~25.6 node-minutes/run × 8 runs = ~205 node-minutes/day = ~3.4 node-hours/day.

| Item | Calculation | Monthly cost |
|---|---|---|
| c2-standard-4 spot (pipeline burst) | 3.4 hrs/day × 30 days × $0.050/hr | **$5.10** |

Pipeline workers scale to zero; cost is proportional to pipeline run frequency. At 8 runs/day this is negligible.

---

### 4. Managed PostgreSQL — Cloud SQL Enterprise

Stratum's write load is modest (state transitions, sync flushes). A 2-vCPU, 8 GB instance handles hundreds of concurrent robot units.

**Instance:** 2 vCPU, 8 GB RAM, Cloud SQL Enterprise edition, single zone (HA is optional — CloudNativePG handles HA for on-prem; cloud-hosted uses Cloud SQL's built-in single-zone reliability + automated backups for cost efficiency).  
**Storage:** 50 GB SSD (covers relational state, job records, audit trail, diagnostic metadata).

Cloud SQL Enterprise pricing in us-central1: $0.0413/vCPU-hr, $0.0070/GB-hr.

**What the 50 GB SSD covers:**

| Data category | Estimated size (3-year accumulation) | Notes |
|---|---|---|
| Design Model geometry records | ~180 MB | Extracted building geometry, storeys, openings, keep-out zones per version; ~10,000 rows/month at 500 bytes/row |
| Print Projects, pipeline runs, requests | ~18 MB | ~50 projects/month × 5 runs × 2 KB/row |
| Jobs, operator events, restart audit trail | ~227 MB | 600 jobs/month × 30 events × 200 bytes; job records ~5 KB each; 3-year accumulation |
| Fleet registry (units, capability profiles) | ~1.5 MB | 20 units × 2 KB; slow growth |
| RBAC and user accounts (Keycloak schema) | ~2 MB | ~50 users × 1 KB; near-static |
| Mender device state and deployment records | ~18 MB | 20 units × sync events × 500 bytes |
| MLflow run metadata (cloud-hosted training) | ~5 MB | Experiment runs, parameters, metrics; modest at this scale |
| PostgreSQL index overhead (~30% of data) | ~73 MB | Approximate |
| **Total user + operational data** | **~525 MB** | |
| **Provisioned SSD** | **50,000 MB** | ~98.9% headroom at 3 years |

The 50 GB SSD is deliberately over-provisioned relative to current data volume. Cloud SQL charges for provisioned storage (not used storage), but the provision ensures consistent I/O performance as the fleet scales. At 200 robots with proportionally higher job throughput, the allocation is still well within 50 GB.

| Item | Calculation | Monthly cost |
|---|---|---|
| Compute: 2 vCPU | $0.0413/hr × 2 × 730 hrs | $60.30 |
| Memory: 8 GB | $0.0070/hr × 8 × 730 hrs | $40.88 |
| SSD storage: 50 GB | $0.222/GB/month × 50 GB | $11.10 |
| Automated backup storage (7-day retention, ~5 GB) | $0.105/GB/month × 5 GB | $0.53 |
| **PostgreSQL subtotal** | | **$112.81** |

---

### 5. GCS Object Storage — Operational Artifacts

Covers pipeline stage outputs, job packages, Mender OTA packages, IFC source files, and general operational blobs. Does not include the Parquet telemetry training prefix (separate line below).

**Accumulated storage estimate:**

| Object type | Volume | Storage class | Notes |
|---|---|---|---|
| IFC source files (10 active projects × ~50 MB) | 0.5 GB | Standard | Uploaded by Architect; retained for re-upload / re-processing |
| IFC historical versions (prior months, Nearline after 60 days) | 1.0 GB | Nearline | Older versions retained for reference; lifecycle transition after 60 days |
| Active pipeline outputs (10 projects × 150 MB) | 1.5 GB | Standard | Accessed by Architect frequently |
| Historical pipeline outputs (prior months, 3-month rolling) | 4.5 GB | Nearline | Transitioned after project moves to In Production |
| Job packages (600/month × 80 MB, 1-month TTL) | 48.0 GB | Standard | Deleted after edge tier confirms receipt |
| Mender OTA artifacts (3 months × 200 MB) | 0.6 GB | Standard | Rarely accessed after fleet update |
| Diagnostic data (600 jobs × 20 MB, 3-month rolling) | 36.0 GB | Standard → Nearline after 30 days | Command logs, execution traces, controller logs |
| Ground surface data (600 jobs × 8 MB, 3-month rolling) | 14.4 GB | Standard → Nearline after 30 days | Per completed ground scanning task; registered against job |

**Monthly storage cost:**

| Tier | Volume | Rate | Monthly cost |
|---|---|---|---|
| Standard | ~67.4 GB | $0.020/GB/month | $1.35 |
| Nearline | ~39.1 GB | $0.010/GB/month | $0.39 |
| **Operational storage subtotal** | | | **$1.74** |

Storage cost is very low at this fleet scale. The dominant cost driver is compute, not storage.

---

### 6. GCS Object Storage — Parquet Telemetry Training Prefix

Telemetry written in Parquet, partitioned by `robot_unit/date/job_outcome`.

**Volume derivation:**

```
Raw live telemetry per month   = 1.16 GB (computed in Assumptions)
Parquet compression ratio      = ~4× (columnar + Snappy compression)
Parquet volume per month       = 1.16 GB / 4 = ~290 MB = ~0.29 GB/month
Accumulated over 12 months     = ~3.5 GB (with lifecycle: Standard for 3 months, Nearline after)
Standard (0–3 months)          = 0.29 GB × 3 = 0.87 GB
Nearline (3–12 months)         = 0.29 GB × 9 = 2.61 GB
```

| Tier | Volume | Rate | Monthly cost |
|---|---|---|---|
| Standard (recent 3 months) | 0.87 GB | $0.020/GB/month | $0.02 |
| Nearline (3–12 months old) | 2.61 GB | $0.010/GB/month | $0.03 |
| **Parquet telemetry subtotal** | | | **$0.05** |

At 20 robots with anomaly-windowed capture, Parquet storage cost is negligible. It grows linearly with fleet size and reaches ~$2.00/month at 200 robots.

---

### 7. VictoriaMetrics Persistent Storage (PVC)

VictoriaMetrics runs on the general pool nodes. Its PVC is backed by a GCP Persistent Disk (Standard balanced PD, `pd-balanced`).

**Volume derivation:**

```
Raw telemetry rate             = 4.88 GB/month = ~163 MB/day
VictoriaMetrics compression    = ~8–10× (TSDB compression on time-series)
On-disk per day                = ~18 MB/day
Retention = 21 days            = ~378 MB active storage
```

PD Balanced storage in us-central1: $0.100/GB/month.

| Item | Calculation | Monthly cost |
|---|---|---|
| VictoriaMetrics PVC | 20 GB provisioned (headroom for spikes) × $0.100/GB/month | **$2.00** |

A 20 GB PVC provides substantial headroom above the ~63 MB active footprint at 20 robots. It accommodates fault-windowed bursts and metrics from Prometheus/Grafana running on the same cluster.

---

### 8. Telemetry Egress (Edge Units → GCS)

Telemetry is written by the sync consumer from EMQX to GCS. Data moves from the robot's secondary uplink to the Orchestration Tier over the internet, then from the Orchestration Tier to GCS within the same GCP region.

- **Ingress (robot → GCP):** Free (GCS charges no ingress fee). Volume: 4.88 GB/month.
- **Intra-GCP (Orchestration Tier → GCS, same region):** Free (same-region GCP-to-GCS transfer is free).
- **Net telemetry ingress cost: $0.00/month.**

---

### 9. Artifact Egress (GCS → Robot Units)

Two types of artifact are pushed from GCS to robot units over their secondary uplink (internet egress from GCP):

- **OTA packages** — delivered via Mender at each software release; downloaded by each unit once per release.
- **Job packages** — assembled at dispatch time and delivered to the assigned unit; contain the approved motion path, task manifest, operator instructions, and capability assertion.

**Volume derivation:**

```
OTA packages      = 20 units × 200 MB × 1 release/month        =  4,000 MB =  4.00 GB/month
Job packages      = 600 dispatches/month × 80 MB/package        = 48,000 MB = 48.00 GB/month
Total egress      = 52.00 GB/month
```

**Rate:** $0.12/GB (Premium Tier, internet egress from us-central1 to robot secondary uplinks)

| Item | Calculation | Monthly cost |
|---|---|---|
| OTA package egress | 4.00 GB × $0.12/GB | $0.48 |
| Job package egress | 48.00 GB × $0.12/GB | $5.76 |
| **Artifact egress subtotal** | 52.00 GB × $0.12/GB | **$6.24** |

---

### 10. API Egress (Application Tier clients)

Stage output previews (3D geometry, slice viewer, motion path simulation), job package previews, and dashboard data served to Architect, Fleet Manager, and Developer clients.

**Estimated volume:** ~2 GB/month (low; internal tooling users, not public-facing traffic).  
**Rate:** $0.12/GB.

| Item | Calculation | Monthly cost |
|---|---|---|
| API egress | 2 GB × $0.12/GB | **$0.24** |

---

### 11. BigQuery (Training Data Queries)

Training data queries read from the Parquet prefix in GCS via BigQuery external tables.

**Query pattern:** Feature engineering and dataset preparation runs; estimated 3 training dataset preparation queries per month, each scanning the full telemetry prefix for the training window (e.g., 6 months of data for 20 robots).

```
Parquet volume scanned per query = 6 months × 0.22 GB/month = 1.32 GB = ~0.0013 TiB
Cost per query (after 1 TiB free tier)  = $0.00 — well within the free 1 TiB/month
```

At 20 robots, **all BigQuery queries fall within the free 1 TiB/month tier**. BigQuery query costs only become material at 100+ robots or with very broad historical scans.

| Item | Calculation | Monthly cost |
|---|---|---|
| BigQuery on-demand queries | ~0.004 TiB/month scanned; within 1 TiB free tier | **$0.00** |

---

### 12. ML Inference — BentoML + KEDA

BentoML serves deployed anomaly detection and print quality models. KEDA scales inference pods to zero between jobs. Inference runs per-job (triggered when a print job completes for quality assessment, or during active execution for anomaly detection).

**Usage:** 40 inference calls/month (one per dispatched job), each running for ~30 seconds on a small CPU pod.  
**Instance:** Inference runs on the existing general pool nodes — no dedicated inference nodes at this scale. Cost is absorbed into the general pool node cost.

| Item | Calculation | Monthly cost |
|---|---|---|
| ML inference compute | Absorbed into general pool node cost | **$0.00** |

---

## Operational Cluster — Monthly Cost Summary

| Line item | Monthly cost (USD) |
|---|---|
| GKE control plane (free tier offset) | $0.00 |
| Kubernetes general pool (3 × e2-standard-4 spot) | $135.22 |
| Kubernetes pipeline pool (c2-standard-4 spot, scale-to-zero) | $5.10 |
| Cloud SQL PostgreSQL (2 vCPU, 8 GB, 50 GB SSD) | $112.81 |
| GCS — operational artifacts (incl. IFC, diagnostic, ground surface data) | $1.74 |
| GCS — Parquet telemetry training prefix | $0.05 |
| VictoriaMetrics PVC (20 GB pd-balanced) | $2.00 |
| Telemetry ingress (edge → GCS) | $0.00 |
| Artifact egress — OTA packages + job packages (GCS → robots) | $6.24 |
| API egress (Application Tier clients) | $0.24 |
| BigQuery training queries | $0.00 |
| ML inference (absorbed into general pool) | $0.00 |
| **Total** | **$263.40 / month** |

---

## Sensitivity Analysis

How the total changes with key assumptions.

### Fleet size

| Fleet size | Total telemetry volume | Job dispatches/month | Total (est.) |
|---|---|---|---|
| 10 robots | ~2.44 GB/month | 300 | ~$148/month |
| **20 robots (base case)** | **~4.88 GB/month** | **600** | **~$263/month** |
| 50 robots | ~12.20 GB/month | 1,500 | ~$530/month |
| 100 robots | ~24.40 GB/month | 3,000 | ~$970/month |

Job package egress ($0.0096/dispatch) and diagnostic/ground surface storage both scale linearly with fleet size, making artifact egress and GCS storage the dominant growth drivers above 20 robots — overtaking compute cost at ~50 robots.

### Telemetry sync frequency

| Job-level sync interval | Total telemetry volume (20 robots) | Parquet volume (job-level only, 12 months) | VictoriaMetrics PVC |
|---|---|---|---|
| Every 60 s | ~4.72 GB/month | ~1.7 GB | 20 GB |
| **Every 30 s (base case)** | **~4.88 GB/month** | **~3.5 GB** | **20 GB** |
| Every 10 s | ~6.20 GB/month | ~10.4 GB | 20 GB |

Unit-level signals (60 s cadence, continuous) and fault bursts contribute ~3.72 GB/month regardless of job-level interval. VictoriaMetrics PVC is sized at 20 GB in all cases; the active footprint (~378 MB) remains well below the cap.

At every-10-second sync, VictoriaMetrics PVC cost rises to ~$6/month and Parquet storage to ~$0.10/month — still not material. Ingress cost remains $0 regardless (free).

### Pipeline run frequency

| Runs/day | Monthly pipeline compute | Notes |
|---|---|---|
| 4 | $2.55 | Low activity |
| **8 (base case)** | **$5.10** | |
| 20 | $12.75 | Active fleet with frequent design iterations |
| 50 | $31.88 | Intensive design phase |

Pipeline pool cost is negligible at all realistic run frequencies.

---

## ML Training Cluster — Per-Run Cost

The training cluster is ephemeral — it does not exist between runs. Costs below are per training run, not per month.

### Assumptions

| Parameter | Value |
|---|---|
| Training data window | 6 months of telemetry from 20 robots |
| Training data volume (Parquet) | ~1.32 GB |
| Feature engineering job duration | ~20 minutes (reads Parquet, computes windowed features, writes feature table) |
| Training job duration | ~2 hours (anomaly detection model; small dataset) |
| GPU instance | `n1-standard-4` + NVIDIA T4 GPU (g2-standard-4 equivalent); spot pricing |
| Spot GPU instance price | ~$0.15–$0.25/hr for T4 spot in us-central1 |
| MLflow PostgreSQL | `db-custom-1-3840` (1 vCPU, 3.75 GB); provisioned for run duration only (~3 hours) |
| MLflow PostgreSQL price | $0.0413/vCPU/hr + $0.0070/GB/hr = ~$0.067/hr |

### Per-Run Cost Breakdown

| Item | Calculation | Cost per run |
|---|---|---|
| Feature engineering (CPU job, 20 min) | c2-standard-4 spot × (20/60) hr × $0.050/hr | $0.02 |
| Training job (T4 GPU spot, 2 hrs) | ~$0.20/hr × 2 hrs | $0.40 |
| BigQuery feature dataset query | ~0.002 TiB scanned; within 1 TiB free tier | $0.00 |
| MLflow PostgreSQL (3 hrs) | $0.067/hr × 3 hrs | $0.20 |
| GCS model artifact storage (model weights, ~500 MB) | $0.020/GB/month × 0.5 GB × (1/30 days prorated) | $0.00 |
| **Total per training run** | | **~$0.62** |

At 1 training run per month, the training cluster adds less than $1/month to total platform cost. Even at weekly retraining (4×/month), total training cost is ~$2.50/month.

---

## Total Platform Cost (Cloud-Hosted, 20 Robots)

| Cluster | Cost |
|---|---|
| Operational cluster | $263.40 / month |
| ML training cluster | ~$0.62 / training run |
| **Monthly total (1 training run/month)** | **~$264.02 / month** |

---

## Notes and Caveats

1. **Spot VM interruption:** The general pool cost assumes 100% spot availability. If GCP reclaims spot nodes (low probability for e2 family in us-central1 due to ample supply), a brief failover adds a small on-demand premium. Budgeting a 10% on-demand buffer adds ~$13/month.

2. **Cloud SQL HA:** The analysis uses a single-zone Cloud SQL instance for cost efficiency. Enabling HA (standby replica in a second zone) doubles the Cloud SQL compute cost to ~$202/month. Recommended for production deployments where database availability SLA is critical.

3. **GCS free tier:** GCS in us-central1 benefits from a free tier for some usage aggregated across us-west1, us-central1, and us-east1. At Stratum's storage volumes, this may offset the first ~5 GB of standard storage per month — effectively $0.10 savings. Not counted above.

4. **BigQuery external tables vs native tables:** The analysis uses BigQuery external tables pointing at GCS Parquet (no data loading cost, no BigQuery storage cost). If native BigQuery tables are preferred for query performance, active storage costs ~$0.02/GB/month on top — at 2.6 GB of training data, this is ~$0.05/month additional.

5. **Prices are list prices.** GCP committed use discounts (1-year: ~28–31% off compute, ~25% off Cloud SQL) reduce costs significantly once the platform is in stable production. At 1-year commitment, Cloud SQL compute drops from ~$101 to ~$75/month, and GKE node compute drops proportionally.

6. **Region:** All prices are for us-central1 (Iowa), which is typically GCP's lowest-cost US region. Europe and Asia-Pacific regions run 10–20% higher.
