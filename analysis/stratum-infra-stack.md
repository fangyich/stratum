# Stratum Platform — Infrastructure Stack

## Guiding Principles

- **Consistent stack across deployment modes** — the same components, container images, Helm charts, and runbooks cover both cloud-hosted and on-prem. Operational divergence is a maintenance liability.
- **Pay per use** — pipeline workers, training jobs, and ML inference scale to zero. Nothing idles.
- **S3 API as the universal storage interface** — S3/GCS in cloud-hosted, MinIO in on-prem. No code divergence between modes.
- **OSS throughout** — no proprietary services in the stack. Every component runs identically on Kubernetes in both deployment modes.

---

## Edge Tier

All components are OSS and deployment-mode invariant.

| Component | Choice | License | Rationale |
|---|---|---|---|
| Container runtime | containerd | Apache 2.0 | Lightweight; matches CI/CD output format |
| Local message bus | NATS (embedded) | Apache 2.0 | Zero-broker; low-latency in-process messaging between edge components |
| MQTT client | Eclipse Paho (or EMQX SDK) | Apache 2.0 | Publishes outbound sync data (state transitions, diagnostic data, live telemetry, ground surface data) to EMQX on the Orchestration Tier over the secondary uplink; QoS 1 ensures at-least-once delivery across intermittent connectivity |
| Local storage | SQLite | Public domain | Single file; no separate process; sufficient for task queue, event log, job state, and diagnostic outbound sync queue at edge scale |
| Live telemetry buffer | Bounded ring buffer (in-process) | — | Separate from the diagnostic sync queue per spec; drop-oldest semantics with independent flush interval to avoid blocking state sync |
| OTA agent | Mender client | Apache 2.0 | Handles A/B rootfs updates, signature verification, and USB standalone mode; pairs with Mender Server on the Orchestration Tier |

---

## Orchestration Tier

Same components in both deployment modes. Infrastructure hosting differs; the application layer does not.

| Component | Choice | License | Cloud-hosted | On-prem |
|---|---|---|---|---|
| Container orchestration | Kubernetes | Apache 2.0 | Managed (EKS / GKE / AKS) — free control plane on GKE | K3s or RKE2 — single-binary, no external DCS |
| Database | PostgreSQL + PgBouncer | PostgreSQL / ISC | Managed (RDS / Cloud SQL) | CloudNativePG operator — HA without etcd/Consul |
| Object storage | MinIO (S3-compatible) | AGPL 3.0 | S3 / GCS | MinIO Operator on Kubernetes |
| Event bus | NATS JetStream | Apache 2.0 | On-cluster | On-cluster |
| MQTT broker | EMQX Community | Apache 2.0 | On-cluster — eliminates managed IoT broker per-message fees | On-cluster |
| Identity | Keycloak | Apache 2.0 | On-cluster | On-cluster; integrates with customer IdP (LDAP / AD / SAML / OIDC) |
| TLS / PKI | cert-manager + Step CA | Apache 2.0 | Let's Encrypt or private CA | Private CA only; no external certificate dependency |
| Update service | Mender Server (OSS) | Apache 2.0 | On-cluster; uses existing object storage backend | On-cluster; packages imported out-of-band by admin |
| Ingress | Traefik | MIT | On-cluster | On-cluster |
| OCI registry | GHCR / Zarf-managed | Free / Apache 2.0 | GHCR — native GitHub Actions integration, currently free | Zarf init bootstraps internal registry; no outbound registry access |
| Monitoring | Prometheus + Grafana + Loki | Apache 2.0 | On-cluster | On-cluster |
| Tracing | Tempo + OpenTelemetry Collector | Apache 2.0 | On-cluster | On-cluster |
| Live telemetry | VictoriaMetrics | Apache 2.0 | On-cluster | On-cluster |
| Diagnostic / ground surface data | Object storage | — | S3/GCS under `diagnostics/<job-id>/` and `ground-scan/<job-id>/` | MinIO; same prefix structure |
| Training data | Parquet in object storage | — | S3/GCS under `telemetry/training/` | MinIO; exported out-of-band |
| ML serving | BentoML + KEDA | Apache 2.0 | On-cluster; KEDA scales to zero | On-cluster; KEDA scales to zero |

### Key Choices Explained

**Kubernetes hosting** — Managed control plane on cloud-hosted (GKE free tier offsets the control plane cost for a single zonal cluster). K3s/RKE2 on-prem: single binary, no external DCS needed, CNCF-conformant.

**PostgreSQL** — CloudNativePG chosen over Patroni for on-prem because it manages HA natively as a Kubernetes operator without requiring a separate etcd or Consul cluster, reducing the component count.

**NATS JetStream** — single component covers three concerns: pipeline job queue, diagnostic sync fan-out, and push event delivery. Replaces SQS/Pub-Sub (cloud) and avoids Kafka's operational overhead at Stratum's message volume.

**EMQX Community** — runs on-cluster in both modes; eliminates managed IoT broker per-message and per-connection fees. Identical operational model in cloud-hosted and on-prem.

**VictoriaMetrics** — chosen over InfluxDB for live telemetry because it handles high-cardinality fleet metrics more efficiently and supports both Prometheus and InfluxDB line protocol, so existing Prometheus metrics flow in without change. Stores live telemetry signals only — diagnostic data is not time-series and lives in object storage.

**Mender (OSS edition)** — used as an internal infrastructure component; the Orchestration Tier calls its REST API directly. Mender UI and API are not exposed to end users. RBAC is enforced at the Stratum application layer; identity federation is handled by Keycloak. This keeps the deployment within the free Apache 2.0 edition — no paid license required.

**Diagnostic and live telemetry separation** — the spec defines these as distinct streams with different storage backends, retention models, and consumers. Diagnostic data (five per-job categories, layer/segment indexed) goes to object storage. Live telemetry (periodic sampled signals) goes to VictoriaMetrics and the Parquet training prefix. Conflating them would lose per-category retention configurability.

**Training data — job-level signals only in Parquet** — only job-level live telemetry signals populate the `telemetry/training/` prefix. Unit-level signals are not included. Operator identity is excluded by the write filter at ingest time.

---

## Deployment Mode Differences

Everything not listed is identical between deployment modes.

| Concern | Cloud-hosted | On-premises |
|---|---|---|
| Kubernetes control plane | Managed (EKS / GKE / AKS) | K3s or RKE2 |
| PostgreSQL HA | Managed multi-AZ (RDS / Cloud SQL) | CloudNativePG operator |
| Object storage backend | S3 / GCS | MinIO Operator on Kubernetes |
| OCI registry | GHCR | Zarf-managed internal registry |
| TLS issuance | Let's Encrypt or private CA | Private CA only (Step CA) |
| Identity federation | Keycloak standalone or federated | Keycloak + customer IdP |
| Data replication | Cloud provider replication | Customer-managed |
| OTA package delivery | Mender + S3/GCS signed URLs | Packages delivered out-of-band; admin imports into Mender |
| Live telemetry retention | Unit-level: shorter default; job-level: longer default; both configurable | Same defaults |
| Diagnostic data | S3/GCS under `diagnostics/<job-id>/` | MinIO; same structure |
| Ground surface data | S3/GCS under `ground-scan/<job-id>/` | MinIO; same structure |
| Training data | Parquet in S3/GCS; read by training cluster | Parquet in MinIO; exported out-of-band |
| ML training | Separate ephemeral cluster | Not applicable |

---

## Toolpath Generation Pipeline

Each of the four pipeline stages (parse, slice, toolpath, motion path) runs as an ephemeral Kubernetes Job. NATS JetStream queues the trigger; the Job reads inputs from object storage, writes outputs, and exits. Scale-to-zero: no idle pipeline capacity in either deployment mode.

---

## CI/CD Pipeline

GitHub Actions for cloud-hosted. For air-gapped on-prem customers: Gitea + Woodpecker CI, included in the Zarf deployment package.

Two pipelines: one for the Orchestration Tier, one for the Edge Tier. Both produce cryptographically signed outputs (Sigstore/cosign for images; GPG for update packages). A single pipeline run produces both OTA-ready and portable media packages (DEV-003).

---

## On-Premises Packaging

Zarf is the packaging and delivery tool. On the connected side, `zarf package create` bundles all container images (pulled from GHCR) and Helm charts into a signed, checksummed tarball with an auto-generated SBOM. On the air-gapped side, `zarf package deploy` bootstraps an internal OCI registry, loads all images, and rewrites image references in Helm charts transparently. Same Helm charts as cloud-hosted; environment-specific overrides applied at deploy time.

---

## ML Training Cluster (Cloud-Hosted Only)

Ephemeral — spun up on demand, torn down after each run. No always-on cost.

| Component | Choice | Rationale |
|---|---|---|
| Cluster | GKE / EKS (ephemeral) | Provisioned programmatically; torn down after run |
| Experiment tracking | MLflow | OSS; backs `Read model state` and `Deploy model update` API operations |
| Training jobs | Kubernetes Jobs on spot GPU nodes | Scale to zero; cost only while running |
| Training data query | BigQuery (GCP) / Athena (AWS) | Serverless SQL over Parquet; no infra; first 1 TB/month free on GCP |
| Storage | Shared S3 / GCS bucket | Reads `telemetry/training/`; writes `models/<name>/<version>/` |
| PostgreSQL | Separate managed instance | MLflow metadata; isolated from operational DB |

**Integration with operational cluster** — object storage is the only coupling point. `Deploy model update` updates the `models/<name>/promoted/` pointer and triggers a BentoML rolling restart on the operational cluster. No VPC peering or shared control plane required.

---

## Key Cost-Efficiency Decisions

1. **Scale-to-zero pipeline workers** — pipeline stages are bursty; ephemeral Kubernetes Jobs avoid idle capacity.
2. **Ephemeral ML training cluster** — zero cost between runs; no impact on operational cluster.
3. **Spot nodes for the general pool** — stateless and quickly recoverable workloads tolerate preemption.
4. **EMQX on-cluster** — eliminates managed IoT broker per-message fees; material saving at 50+ robots.
5. **NATS as shared event backbone** — one component replaces SQS/Pub-Sub, a message queue, and a WebSocket fan-out broker.
6. **GKE free tier** — offsets the control plane fee for one zonal cluster; $73/month saving vs EKS.
7. **Single stack, two environments** — one set of Helm charts, runbooks, and upgrade procedures for both deployment modes.

---

## What to Avoid

| Anti-pattern | Why |
|---|---|
| Managed-only services in cloud-hosted deployment | Breaks stack parity with on-prem |
| Always-on pipeline workers | Pipeline is bursty; idle capacity is waste |
| Kafka for internal event bus | Operational overhead exceeds benefit at Stratum's message volume |
| Separate CI/CD builds per deployment mode | Spec requires a single artifact (DEV-003) |
| Storing blobs in PostgreSQL | Degrades performance; use object storage |
| Per-robot cloud resources | Fleet-shared resources; per-robot provisioning doesn't scale |
