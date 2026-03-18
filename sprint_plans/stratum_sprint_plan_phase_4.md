# Stratum Platform — Sprint Plan: Phase 4

## Overview

**Phase goal:** Enable multi-site fleet management, introduce AI/ML capabilities, complete the hardware abstraction layer, and establish a hardware-in-the-loop testing framework.
**Sprint length:** 2 weeks
**Sprints:** 1–9 (18 weeks)

Sprints in this document are numbered from 1. Cross-phase dependencies reference their source phase explicitly (e.g. "Phase 3 Sprint 8").

Sprint estimates assume a consistent team velocity and are intended as a planning baseline.


This document covers Phase 4 of the Stratum transition. Sprint length is **2 weeks**. Sprint numbers continue from Phase 3 (which ended at Sprint 28).

Phases 1–3 are covered in a separate document with 1-week sprints. Phase 4 begins once Sprint 28 (Phase 3 integration testing and security review) is complete.

Sprint estimates assume a consistent team velocity and are intended as a planning baseline. Dependencies between sprints are noted where a sprint cannot begin until a prior sprint's output is available.

---

## Phase 4 — Fleet Management and AI

**Phase goal:** Enable multi-site fleet management, introduce AI/ML capabilities, complete the hardware abstraction layer, and establish a hardware-in-the-loop testing framework.
**Sprints:** 1–9 (18 weeks)

---

### Sprint 1 — Fleet Management Dashboard

**Goal:** Implement the Fleet Management Dashboard providing real-time visibility across all registered units and all active sites.

**Deliverables:**
- Fleet Management Dashboard accessible to Fleet Managers via the cloud Application Tier
- All registered units visible: real-time status, current job, site location, any active alerts
- Last-known state displayed for offline units; Fleet Manager alerted on reconnection
- Multi-site grouping and filtering in the dashboard view

**Tasks:**
- Design the fleet status data model: unit identifier, site, current job, task state, connectivity state, last-seen timestamp, active alerts
- Implement fleet status aggregation service in the Orchestration Tier: collect and maintain current status for all registered units
- Implement real-time status streaming from Edge Tier to cloud: unit heartbeat, job state changes, alert events
- Implement offline detection: flag units that have not sent a heartbeat within a configurable threshold; alert Fleet Manager on reconnection
- Build Fleet Management Dashboard UI: unit cards with status, current job, and alert indicators; site grouping; multi-site filter; offline unit section with last-known state
- Build Fleet Manager alert centre: list of active alerts across the fleet with acknowledgement
- Write tests covering: status update propagation, offline detection, reconnection alert, multi-unit dashboard rendering

**Depends on:** Phase 3 Sprint 8

---

### Sprint 2 — Operational Telemetry Collection

**Goal:** Implement systematic telemetry collection across all units, streamed to cloud storage and available for analysis.

**Deliverables:**
- Print performance metrics, sensor readings, and system events collected from all units during print jobs
- Telemetry streamed to cloud Orchestration Tier and stored in a queryable time-series or analytical store
- Anomaly-windowed capture implemented for high-volume hardware telemetry: full-fidelity around fault events, sampled otherwise
- Configurable retention policies per telemetry data layer
- Fleet Manager can access historical telemetry for any unit via the Admin console

**Tasks:**
- Define telemetry schema: print performance metrics (layer completion times, speed actuals vs. commanded), sensor readings (ultrasonic distances, temperatures), system events (state transitions, fault codes)
- Implement telemetry capture in the Edge Tier: instrument machine control and sensor feedback services to emit structured telemetry events
- Implement anomaly-windowed capture: on fault event, expand capture to full fidelity for a configurable window before and after; revert to sampled rate otherwise
- Implement telemetry streaming: Edge Tier buffers and forwards telemetry to cloud via the sync protocol
- Provision cloud telemetry storage (time-series database or analytical store); implement ingestion pipeline
- Implement configurable per-layer retention policies in the cloud telemetry store
- Build telemetry history view in the Admin console: per-unit, per-job telemetry browsing with time range filter and fault event highlighting

**Depends on:** Sprint 1

---

### Sprint 3 — AI/ML Infrastructure

**Goal:** Establish the AI/ML model lifecycle infrastructure end-to-end: training pipeline, model serving, and deployment workflow. No models are trained in this sprint; the focus is entirely on building the foundation that all subsequent model sprints depend on.

**Deliverables:**
- ML training pipeline infrastructure in place: data extraction from telemetry store, feature engineering scaffolding, training job orchestration, model evaluation, and versioned artifact storage
- Model serving infrastructure in place on the Edge Tier: load model artifact, run inference on incoming telemetry, emit scored events
- Model deployment workflow in place in the Orchestration Tier: package model as a versioned deployment artifact, push to target units, monitor inference health
- End-to-end pipeline verified with a trivial baseline model: artifact produced, deployed to edge, inference confirmed running

**Tasks:**
- Design the ML model lifecycle: training data extraction, feature engineering, training pipeline, model evaluation, versioned artifact storage, deployment to edge, inference, monitoring
- Provision training infrastructure: managed compute for training jobs, model artifact registry
- Implement data extraction pipeline: query telemetry store, produce structured training datasets
- Implement model serving on the Edge Tier: load model artifact, run inference on incoming telemetry, emit scored output events
- Implement model deployment workflow in the Orchestration Tier: package model as a versioned deployment artifact, push to target units, monitor inference health
- Verify the end-to-end pipeline with a trivial baseline model: confirm artifact flows from training through deployment to edge inference without error
- Write tests covering: training pipeline execution, model versioning, deployment to edge, inference output format

**Depends on:** Sprint 2

---

### Sprint 4 — Anomaly Detection Model

**Goal:** Train and deploy the anomaly detection model using the infrastructure established in Sprint 31.

**Deliverables:**
- Anomaly detection model trained on real operational telemetry and deployed to at least one unit
- Model inference results surfaced in the Fleet Manager dashboard as alerts
- Model monitoring established: inference latency, score distribution, and alert rate tracked over time

**Tasks:**
- Implement data extraction for anomaly detection: query telemetry store for labelled fault events and surrounding windows; produce training datasets
- Train anomaly detection model on available telemetry data; evaluate against held-out data
- Deploy model to the Edge Tier using the deployment workflow from Sprint 3
- Surface anomaly detection alerts in the Fleet Manager dashboard: alert on anomaly score exceeding threshold, with link to relevant telemetry window
- Establish model monitoring for inference latency, score distribution, and alert rate; alert on significant distribution shift

**Depends on:** Sprint 3

---

### Sprint 5 — Print Quality Assessment Model

**Goal:** Train and deploy the print quality assessment model using the telemetry and infrastructure established in Sprints 31 and 32.

**Deliverables:**
- Print quality assessment model trained and deployed: detects deposition anomalies from sensor data during print execution
- Print quality alert view in the Fleet Manager dashboard: per-layer quality score during active print, alert on threshold breach
- Model monitoring extended to cover the print quality model

**Tasks:**
- Define features and labels for print quality assessment: identify sensor signals and execution trace patterns correlated with poor deposition quality; extract labelled training data from telemetry store
- Train print quality assessment model; evaluate against held-out data
- Deploy model to the Edge Tier using the deployment workflow from Sprint 3
- Build print quality alert view in the Fleet Manager dashboard: per-layer quality score during active print, alert on threshold breach
- Extend model monitoring to cover print quality model inference latency, score distribution, and alert rate

**Depends on:** Sprint 4

---

### Sprint 6 — Plugin-Style Hardware Abstraction Layer

**Goal:** Formalise the hardware abstraction layer into a well-defined plugin interface, migrate existing hardware to the plugin model, and enable new hardware integration without core platform changes.

**Deliverables:**
- Plugin interface defined and documented: contract for integrating new robot types, sensor modules, and tooling variants
- Capability profile registration system integrated with the Fleet Manager dispatch validation
- Existing SCARA robot hardware migrated to the plugin model
- Existing simulation adapters migrated to the plugin model
- Developer documentation for integrating a new hardware variant

**Tasks:**
- Design the plugin interface: define the capability profile schema, the required interface methods (machine control, motion execution, sensor feedback), and the registration mechanism
- Implement the plugin registry: at startup, discover and register all available hardware plugins
- Migrate `dcrafter_control_adapters` real hardware adapter to the plugin interface
- Migrate `dcrafter_control_adapters` simulation adapter to the plugin interface
- Migrate `scara_moveit_config` and `scara_description` to be referenced by the SCARA plugin rather than core platform logic
- Integrate capability profile registration with Fleet Manager dispatch validation: dispatch validation reads the assigned unit's registered capability profile from the plugin registry
- Write developer documentation: step-by-step guide for implementing a new hardware plugin and registering a capability profile
- Write tests covering: plugin registration, correct capability profile lookup, dispatch validation using plugin-registered profiles, fallback on missing plugin

**Depends on:** Phase 3 Sprint 8

---

### Sprint 7 — Hardware-in-the-Loop Testing Framework

**Goal:** Implement the HIL testing framework, enabling developers to validate new features against a simulated robot in CI and against real hardware in gated test runs.

**Deliverables:**
- HIL testing framework implemented using the simulation adapter from the plugin model
- Automated HIL test suite running in CI against the simulated robot on every pull request
- Real-hardware test run procedure documented and gated
- Existing machine control and motion execution features covered by HIL tests

**Tasks:**
- Implement HIL test harness: test runner that spins up the full Edge Tier stack against the simulation plugin, executes test scenarios, and asserts on outcomes
- Integrate HIL test harness into CI pipeline: run simulation-based HIL tests on every pull request; fail the build on test failure
- Write HIL test cases for existing machine control features: enable system, homing sequence, jogging, E-Stop, task queue operations
- Write HIL test cases for print execution: start task, pause, resume, abort, restart
- Write HIL test cases for ground scanning: start scan, abort, complete scan and verify data registration
- Document the real-hardware test run procedure: how to configure the harness for a real robot, what tests require real hardware, how results are recorded and reviewed
- Define the policy for new features: simulation HIL test required before merge; real-hardware test required before production deployment for any feature touching motion or machine control

**Depends on:** Sprint 6

---

### Sprint 8 — Fleet Management Completion

**Goal:** Complete all remaining fleet management capabilities: multi-unit job dispatch at scale, historical usage and telemetry review, and fleet-wide update management.

**Deliverables:**
- Fleet Manager can view all active jobs across all sites with drill-down to per-job detail
- Historical usage data, error logs, and print telemetry accessible per unit and per job via the Fleet Manager interface
- Fleet-wide software and firmware update management: push to individual units or all units, view rollout progress, download USB packages for offline units

**Tasks:**
- Build fleet-wide job overview in the Fleet Manager interface: all active jobs across all sites, status, progress, and assigned unit
- Build job drill-down view: full job detail including print task state, current layer, estimated completion, and link to diagnostic record
- Build historical usage view: per-unit job history with completion status, duration, and layer counts
- Build error log view: per-unit structured error and crash log with search and filter
- Build telemetry history view in the Fleet Manager interface (complementing the Admin console view from Sprint 30)
- Complete fleet-wide update management UI: software and firmware update deployment to selected units or all units, rollout progress tracking per unit, USB package download for offline units
- Write tests covering: multi-unit job overview accuracy, historical data retrieval, fleet update deployment tracking

**Depends on:** Sprint 7

---

### Sprint 9 — Phase 4 Integration Testing, Security Review, and Documentation

**Goal:** Conduct full Phase 4 integration testing, complete the multi-tenant security review, and bring all documentation up to date.

**Deliverables:**
- Full Phase 4 integration test suite passing
- Multi-tenant security review completed: data isolation between tenants, AI/ML model access controls, fleet management permission boundaries
- All identified security issues resolved or formally accepted
- Architecture documentation updated to reflect the completed Stratum platform
- Developer onboarding documentation complete

**Tasks:**
- Execute end-to-end integration tests across all Phase 4 features: fleet dashboard, telemetry collection, AI/ML model inference, hardware abstraction plugin, HIL test execution
- Run HIL test suite against real hardware for all machine control, print execution, and ground scanning test cases
- Conduct multi-tenant security review: verify data isolation (one Fleet Manager cannot access another's units or jobs), AI/ML model access controls (models deployed only to authorised units), fleet management permission boundaries
- Penetration test the fleet management and telemetry APIs
- Resolve all high and medium severity findings; document accepted findings
- Update architecture documentation: final three-tier architecture, hardware plugin interface, ML model lifecycle, fleet management data flows
- Write developer onboarding documentation: how to set up a local development environment, how to run the HIL test suite, how to implement a new hardware plugin, how to contribute a new AI/ML model

**Depends on:** Sprint 8

---

## Sprint Summary

| Sprint | Phase | Weeks | Title | Key Deliverables |
|---|---|---|---|---|
| 1 | 4 | 1–2 | Fleet Management Dashboard | Real-time fleet status, multi-site visibility, offline alerts |
| 2 | 4 | 3–4 | Operational Telemetry Collection | Telemetry capture, cloud storage, anomaly-windowed capture |
| 3 | 4 | 5–6 | AI/ML Infrastructure | ML lifecycle pipeline, model serving, deployment workflow, end-to-end verified |
| 4 | 4 | 7–8 | Anomaly Detection Model | Anomaly detection model trained, deployed, alerts in Fleet Manager dashboard |
| 5 | 4 | 9–10 | Print Quality Assessment Model | Print quality model trained, deployed, per-layer quality alerts in dashboard |
| 6 | 4 | 11–12 | Plugin-Style Hardware Abstraction Layer | Plugin interface, existing hardware migrated, developer docs |
| 7 | 4 | 13–14 | Hardware-in-the-Loop Testing Framework | HIL harness in CI, existing features covered by HIL tests |
| 8 | 4 | 15–16 | Fleet Management Completion | Fleet-wide job overview, historical data, fleet update management |
| 9 | 4 | 17–18 | Phase 4 Integration Testing, Security Review, and Documentation | Full integration tests, security review, final documentation |
