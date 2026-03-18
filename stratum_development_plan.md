# Stratum Platform — Transition Development Plan

## Executive Summary

This document identifies the major improvements required to transition the current D-Crafter system to the Stratum platform and proposes a phased development plan. The plan is structured around four phases, each with a clear goal, scope boundary, and set of workstreams. The phases are designed to be sequentially dependent: each phase stabilises or extends the system before the next expands its capabilities.

---

## Current System Assessment

Before defining the plan, the following table summarises the most critical gaps between the current system and the Stratum platform target.

| Area | Current State | Gap |
|---|---|---|
| Architecture | Monolithic, all logic on IPC | No tier separation; tightly coupled components |
| Database | SQLite, no encryption, no migrations | Not production-grade; data loss risk |
| Security | No authentication, no encryption | Any user can perform any action |
| Logging & Diagnostics | No structured logging or crash reporting | Failures are invisible |
| Software Updates | Manual, via email | No version tracking, no integrity verification |
| UI | Single interface, no role separation | All users see all controls |
| Deployment | Manual IPC configuration | Not reproducible; no provisioning baseline |
| Cloud / Fleet | None | Entire capability missing |
| AI/ML | None | Entire capability missing |
| Hardware Testing | No HIL framework | Features deployed without validation |

---

## Phase 1 — Stabilisation & Foundation

**Goal:** Make the existing system reliable, observable, and updateable. No new end-user features are added. The focus is entirely on improving reliability, refactoring the architecture toward the three-tier Stratum model, and establishing the operational foundation for all future phases.

**Deployment target:** On-premises, single-robot. Both the Orchestration Tier and Edge Tier reside on the robot's IPC.

**UI scope:** Site operator interface only. The existing operator workflows are preserved without behavioural change.

---

### 1.1 Architecture Refactoring

**Problem:** The current codebase, particularly `dcrafter_print_service`, conflates multiple responsibilities and does not map to the Stratum three-tier model (Orchestration, Edge, Application). This makes bugs hard to isolate, changes risky, and future tier separation very difficult.

**Work required:**

- Decompose `dcrafter_print_service` into clearly bounded services aligned with the Stratum tier model:
  - **Orchestration-tier services** (running on IPC for now, structured for later cloud migration): project management, toolpath pipeline orchestration, job lifecycle management
  - **Edge-tier services:** real-time machine control, motion execution, sensor feedback, local task queue
  - **Application-tier services:** API gateway and serving layer for the operator UI
- Define and enforce explicit interfaces between tiers using the existing ROS 2 interface packages (`cc_print_interfaces`, `dcrafter_control_interfaces`) as a model; extend as needed
- Ensure each service has a single, documented responsibility and that cross-tier communication passes through defined API boundaries
- Document the revised architecture so all future work is made against a known baseline

**Affected packages:** `dcrafter_print_service`, `dcrafter_control_service`, `cc_print_server`, `dcrafter_controller`, `dcrafter_bringup`

---

### 1.2 Database Upgrade

**Problem:** The current SQLite database has no encryption, does not support schema migrations, and is not suited to production-scale data management. Any schema change results in data loss.

**Work required:**

- Replace SQLite with a production-grade embedded or self-hosted relational database (e.g. PostgreSQL running as a local container on the IPC)
- Implement a schema migration framework (e.g. Alembic or Flyway) so schema changes can be applied safely and rolled back if needed
- Enable encrypted storage at rest
- Migrate all existing data models and queries to the new database
- Write and test migration scripts for upgrading existing deployments

---

### 1.3 Structured Logging and Diagnostic Data

**Problem:** The system has no structured logging, no crash reporting, and no diagnostic data collection. Failures are invisible and root-cause analysis relies on operator recall.

**Work required:**

- Introduce a structured logging framework across all services (e.g. JSON-formatted log output, consistent severity levels, correlation IDs tied to job identifiers)
- Capture and store the five diagnostic data categories defined in the Stratum spec for every job:
  - Command log (instructions issued, with timestamp and actor)
  - Execution trace (measured robot behaviour)
  - Controller log (PMAC internal decisions and fault codes)
  - Hardware telemetry snapshot (sensor readings, drive states, fault flags)
  - Operator event log (lintel confirmations, E-Stop events, etc.)
- Store all diagnostic data locally in the upgraded database under a shared job identifier
- Implement configurable per-layer retention policies so local storage does not grow without bound
- Add structured error and crash reporting: capture stack traces and system state at the time of any fault

**Affected packages:** All services; `dcrafter_controller`, `cc_print_server`, `dcrafter_print_service`

---

### 1.4 Diagnostic Report Export

**Problem:** There is no mechanism for operators or engineers to extract diagnostic data from the system.

**Work required:**

- Add a report export capability to the operator UI allowing download of:
  - Per-job diagnostic reports (command log, execution trace, operator event log, hardware telemetry summary)
  - System health summaries
- Exported reports should be in a portable format (PDF or CSV) suitable for offline review and support escalation
- No cloud upload is required in this phase; local export is sufficient

---

### 1.5 Guided and Enforced Homing Workflow

**Problem:** The current homing workflow does not enforce step sequence; operators can skip required joint positioning steps, leading to incorrect joint references and potential collisions.

**Work required:**

- Refactor the homing workflow in both the backend and UI to enforce a strict step sequence
- The system must block progression to the next step until the current step is confirmed
- If a step is skipped or attempted out of order, display a clear, actionable error message

**Affected packages:** `dcrafter_controller`, `dcrafter_control_service`, `dcrafter_gui`

---

### 1.6 Software Packaging — Mender-Compatible Format

**Problem:** Update packages are prepared manually with no structured format and no integrity guarantees.

**Work required:**

- Package the D-Crafter software stack in a format compatible with the Mender OTA update framework, producing signed `.mender` artifact files with proper versioning and metadata
- Automate the packaging process in the CI/CD pipeline so each release produces a Mender-compatible artifact without manual steps
- Plan and implement A/B partition layout on the IPC disk at OS provisioning time (via `platform_wizard`), as this is required for Mender's full-image atomic update and rollback and cannot be retrofitted later
- The update delivery mechanism remains manual (portable media / USB) in this phase; the packaging format is being standardised now to enable OTA delivery in Phase 3

**Affected packages:** `dcrafter_packaging`, `platform_wizard`, CI/CD pipeline

---

### 1.7 Mender-Assisted Offline Update Tooling

**Problem:** There is no structured, auditable process for applying updates to an IPC in the field. Manual processes are error-prone and unverified.

**Work required:**

- Provision the Mender client on the IPC with the trusted public key baked in during platform setup
- The Mender client verifies the cryptographic signature of each update artifact before applying it, even in fully offline mode, and handles automatic rollback if the update fails
- Provide operators with a clear, documented procedure for applying updates via USB using the Mender client (or a wrapper script that automates the mount and install)
- Every update application is logged locally with version, timestamp, and outcome
- This establishes the Mender infrastructure (client, artifact format, A/B partitioning, verification) required for OTA activation in Phase 3

---

### 1.8 Test Coverage Improvement

**Problem:** Test coverage is low and focused only on normal use cases. Error paths, boundary conditions, and hardware interaction paths are untested.

**Work required:**

- Expand unit and integration test coverage for all refactored services, with particular attention to error paths and boundary conditions (e.g. unsupported IFC features, invalid print configurations, unexpected hardware responses)
- Establish a minimum coverage threshold enforced in CI
- Begin documenting a hardware-in-the-loop (HIL) testing strategy for later implementation in Phase 4

---

### Phase 1 — Summary of Deliverables

| Deliverable | Description |
|---|---|
| Refactored service architecture | Services decomposed and aligned with the three-tier Stratum model |
| Production database | PostgreSQL with encrypted storage and schema migration support |
| Structured logging | JSON-structured logs across all services with job-correlated IDs |
| Diagnostic data capture | Five-category diagnostic record stored per job, with retention policies |
| Report export | Operator-accessible export of per-job diagnostic reports |
| Enforced homing workflow | Step-locked homing sequence in backend and UI |
| Mender-compatible packaging | CI/CD pipeline producing signed `.mender` artifacts; A/B partition layout on IPC |
| Mender offline update tooling | Mender client on IPC supporting verified local update via USB |
| Improved test coverage | Expanded tests with enforced minimum coverage in CI |

---

## Phase 2 — Full Stratum Feature Set (On-Premises)

**Goal:** Deliver the complete Stratum feature set — including the full project planning and toolpath generation pipeline, role-based workflows, the full job lifecycle, and security hardening — with the Orchestration Tier still running locally on the IPC. All four user roles are fully supported. Because the Orchestration Tier was structured for tier separation in Phase 1, Phase 3 can lift it to the cloud without re-implementing any of these features.

**Deployment target:** On-premises, single-robot. Orchestration Tier and Edge Tier both reside on the IPC.

**UI scope:** All four role-specific interfaces: Site Operator, Architect / Project Owner, Fleet Manager, and Admin.

---

### 2.1 Authentication and Role-Based Access Control (RBAC)

**Problem:** Any user who can reach the GUI can perform any action. There are no roles, no authentication, and no permission boundaries.

**Work required:**

- Implement an authentication layer across the platform (username/password at minimum; LDAP/OIDC integration prepared for on-prem deployments in Phase 3)
- Define distinct roles and their permission sets:
  - **Site Operator** — machine control, task queue management, speed overrides, guided homing and scanning workflows, report export
  - **Architect / Project Owner** — IFC upload, print configuration management, Print Project creation, pipeline triggering, simulation review and approval, job status monitoring
  - **Fleet Manager** — job dispatch, hardware assignment, scope adjustment, fleet status visibility, update distribution
  - **Admin** — all permissions plus user management, system configuration, version registry
- Enforce permissions at the API layer, not only in the UI
- Provide a local bootstrap admin account for initial setup on a fresh IPC

**Affected packages:** `dcrafter_gui`, orchestration-tier services, `dcrafter_control_service`

---

### 2.2 Encrypted Communication

**Problem:** Communication between the operator interface and the robot is not encrypted.

**Work required:**

- Enable TLS for all communication on the local robot network (operator tablet to Application Tier API)
- Support self-signed certificates and a private certificate authority for on-premises deployments
- Ensure all inter-service communication within the IPC is authenticated and encrypted where it crosses a process boundary

---

### 2.3 Role-Specific User Interfaces

**Problem:** A single interface exposes all controls to all users regardless of training or role.

**Work required:**

- Deliver four role-specific interfaces in `dcrafter_gui`, each presenting only the tools and information relevant to that role:
  - **Site Operator interface** — machine control, task queue, real-time print monitoring, speed overrides, guided homing and scanning workflows, diagnostic report export
  - **Architect / Project Owner interface** — IFC upload, Building Model version management, Print Configuration management, Print Project creation, pipeline status monitoring, per-stage output inspection (parsed model, slice viewer, toolpath viewer, motion path simulation), motion path approval, job status tracking
  - **Fleet Manager interface** — job dispatch queue, hardware assignment, capability validation, job scope adjustment, fleet status for the local unit, update package management, version registry
  - **Admin interface** — user management, RBAC configuration, system configuration, diagnostic log access, update package upload and application via Mender tooling

---

### 2.4 Full Project Planning and Toolpath Generation Pipeline

**Problem:** The current system has a minimal, single-threaded project creation flow with no versioning, no stage-by-stage inspection, and no approval workflow.

**Work required:**

- Implement the full Stratum project planning model:
  - **Building Model versioning** — each IFC upload creates a new immutable version; versions are retained and referenceable; extraction errors and unrecognised elements are surfaced as warnings
  - **Print Configuration management** — named, versioned fabrication parameter sets managed independently of Building Models and reusable across multiple Print Projects
  - **Print Project composition** — a Print Project composes a Building Model version, a Print Configuration version, and a target robot type and generation; no pipeline execution occurs until manually triggered by the architect
- Implement the four-stage toolpath generation pipeline:
  - Stage 1 — Parse: IFC geometry and site information extraction, output inspectable in the 3D model preview
  - Stage 2 — Slice: per-layer contour generation, output inspectable in the slice viewer
  - Stage 3 — Toolpath Generation: extrusion paths per layer, output inspectable in the toolpath viewer
  - Stage 4 — Motion Path Generation: full robot motion path, output inspectable in the motion path simulation viewer with playback, progress slider, and layer-jump controls
- Implement pipeline re-entry: when an input changes and the pipeline is re-triggered, only the affected stage and all downstream stages are recomputed; previously computed outputs for unaffected stages are reused
- Keep-out zones and known obstacles defined in the IFC are incorporated into toolpath planning at Stage 3
- Explicit architect approval is required on the final motion path output before the Print Project is marked ready for dispatch; any previously approved motion path is invalidated when the pipeline is re-triggered
- Send a notification to the architect when a triggered pipeline run completes

**Affected packages:** `cc_print_server`, `cc_print_core`, `model_slicers`, `toolpath_planners`, `robot_path_planners`, `ifc_parser`, `printer_program_generator`

---

### 2.5 Hardware Capability Modeling and Validation

**Problem:** There is no mechanism to model robot capabilities or validate that a robot can execute a given job before dispatch.

**Work required:**

- Implement a configurable capability profile per robot type and generation
- The architect selects a target robot type and generation when creating a Print Project; the motion path is generated for that capability class
- The Fleet Manager assigns a specific registered unit at dispatch time; the platform validates that the assigned unit satisfies the required capability profile before dispatch is permitted
- If validation fails, the Fleet Manager is shown a clear explanation of the mismatch

---

### 2.6 Full Job Lifecycle

**Problem:** The current system has a minimal task queue with no structured job state machine, no abort position recording, and no restart workflow.

**Work required:**

- Implement the full job and print task state machines defined in the Stratum spec:
  - Job states: Pending Dispatch → Active → Paused / Aborted → Restarting → Completed
  - Print task states: Ready → Running → Paused / Aborted → Completed
  - Print task state changes propagate to the parent job state as defined in the spec
- When a job is aborted, automatically record the layer index and segment offset of the last confirmed execution point as structured fields on the job record
- Implement the restart workflow: the operator is guided to manually jog the printhead to the intended restart point; the platform validates the jogged position against the recorded abort boundary before permitting motion to resume
- Job scope adjustment: the Fleet Manager can adjust the layer and segment range of a job before dispatch
- Maintain a full audit trail under a single job identifier: abort events, scope adjustments, unit substitutions, and restart points are all captured as structured entries in the job's diagnostic record

---

### 2.7 Ground Scanning Task Lifecycle

**Problem:** The current ground scanning workflow lacks a defined state machine and formal integration with the job lifecycle.

**Work required:**

- Implement the ground scanning task state machine defined in the Stratum spec: Queued → Active → Completed / Aborted
- The associated print job remains in Pending Dispatch state while a ground scanning task is Queued or Active; the job may only be advanced once a ground scanning task has reached Completed state
- Ground surface data from a completed scan is stored and registered against the associated print job

---

### 2.8 Automated Pre-Job Health Checks

**Problem:** There is no mechanism to verify hardware readiness before a print job begins. Faults discovered during execution cause wasted material and potential damage.

**Work required:**

- Implement a structured health check workflow that the site operator can run on the robot unit before a print job begins
- Health checks cover: servo drive engagement, sensor connectivity, motion controller communication, and any other hardware readiness signals available from the existing control stack
- Results are displayed clearly with pass/fail status per check and actionable guidance for any failure

---

### 2.9 Centralised Version Tracking (Local)

**Problem:** There is no way to know which software or firmware version is running on the system.

**Work required:**

- Implement a local version registry that records the current software and firmware version on the IPC
- Display the current version in the admin interface
- Version information is included in every diagnostic report export
- This local registry will be extended to the centralised fleet-wide registry in Phase 3

---

### Phase 2 — Summary of Deliverables

| Deliverable | Description |
|---|---|
| Authentication and RBAC | Login system with four roles and permission enforcement at the API layer |
| Encrypted communication | TLS for all local network communication |
| Role-specific interfaces | Separate interfaces for Site Operator, Architect, Fleet Manager, and Admin |
| Building Model versioning | Immutable IFC versions with extraction warning surfacing |
| Print Configuration management | Named, versioned fabrication parameter sets |
| Print Project composition and pipeline | Four-stage pipeline with per-stage inspection, re-entry, and architect approval |
| Keep-out zone integration | Site obstacles and keep-out zones incorporated into toolpath planning |
| Pipeline completion notifications | Automated notifications to architect on pipeline completion |
| Hardware capability modeling | Capability profiles per robot type; dispatch validation |
| Full job lifecycle | Complete job and print task state machines with abort recording and restart validation |
| Ground scanning lifecycle | Formal state machine and integration with job lifecycle |
| Automated pre-job health checks | Hardware readiness verification before job execution |
| Local version registry | Current software and firmware version tracked and displayed |

---

## Phase 3 — Cloud-Hosted Orchestration and OTA Updates

**Goal:** Move the Orchestration Tier to the cloud, enabling remote access for all roles, migrating data to cloud storage, and activating OTA software and firmware update delivery via the Mender infrastructure established in Phase 1.

**Deployment target:** Cloud-hosted by default. The Orchestration Tier is deployed to centrally managed cloud infrastructure; the Edge Tier remains on the robot's IPC. For organisations with data-residency or air-gapped requirements, the Orchestration Tier can alternatively be deployed to customer-managed on-premises infrastructure using the same deployment artifact. Both modes are functionally equivalent.

---

### 3.1 Cloud Orchestration Tier Deployment

**Problem:** All processing and storage runs on the IPC. There is no remote infrastructure for off-site workflows or centralised management.

**Work required:**

- Deploy the Orchestration Tier services to cloud infrastructure
- The IPC Edge Tier connects to the cloud Orchestration Tier via a secondary uplink (cellular or site WiFi)
- The Edge Tier continues to operate fully offline when connectivity is lost, syncing state to the cloud when restored
- Define and implement the cloud sync protocol: job state, diagnostic data, and version registry updates
- The Application Tier is served from the cloud and accessible remotely from any device; the local robot network continues to serve the site operator UI for low-latency on-site control
- On-premises deployment (Orchestration Tier on customer-managed infrastructure) is supported using the same deployment artifact

---

### 3.2 Remote Access for All Roles

**Problem:** All role-specific interfaces are currently only accessible on-site via the robot's local network.

**Work required:**

- Architect / Project Owner interface: IFC upload, Print Project creation, pipeline triggering, simulation review, and job monitoring are now accessible from any remote device via the cloud
- Fleet Manager interface: job dispatch, fleet status, and update distribution are accessible remotely
- Toolpath generation pipeline runs on cloud infrastructure, enabling concurrent processing of multiple requests and eliminating the single-request bottleneck that exists when running on the IPC
- Remote print progress tracking: job status (completion percentage, estimated time remaining, current layer/segment) is accessible remotely and updates in near real time

---

### 3.3 Data Migration to Cloud

**Problem:** All user data is stored locally on the IPC with no backup or replication. A hardware failure results in permanent data loss.

**Work required:**

- Migrate the production database from local IPC storage to cloud-hosted storage
- Implement replication so the local edge database acts as a cache and operational fallback; the cloud is the system of record
- Historical diagnostic data and telemetry are retained in cloud storage per configured retention policies
- The Edge Tier continues to buffer and operate on local data during connectivity loss, syncing to the cloud on reconnection

---

### 3.4 OTA Software Update via Mender

**Problem:** Software updates are distributed manually via USB. The Mender packaging and client infrastructure established in Phase 1 is ready; this phase activates cloud-delivered OTA.

**Work required:**

- Deploy a Mender Server as part of the cloud Orchestration Tier
- Integrate the CI/CD pipeline to publish signed `.mender` artifacts to the Mender Server on each release
- Administrators can push software updates to a specific robot unit or all units from the admin console without visiting each site
- The Mender client on each IPC pulls and applies updates from the cloud update server, with automatic rollback on failure
- Offline update via USB (established in Phase 1) continues to be supported for sites without connectivity
- Update state is reported to the central version registry in the cloud

---

### 3.5 OTA Firmware Update

**Problem:** Motion controller and sensor module firmware can only be updated using a dedicated IDE with physical presence.

**Work required:**

- Implement OTA firmware update delivery using the same Mender artifact pipeline
- The Mender client on the IPC handles firmware update packages for the PMAC motion controller and sensor module, eliminating the need for a dedicated IDE or site visit
- Firmware version is reported to the central version registry alongside software version

---

### 3.6 Centralised Version Registry

**Problem:** There is no fleet-wide view of which software or firmware version is running on each unit.

**Work required:**

- Extend the local version registry (introduced in Phase 2) to the cloud, tracking software and firmware versions across all registered units
- When a unit that was updated offline via USB reconnects to the Orchestration Tier, its reported version is automatically reconciled with the central registry
- The Fleet Manager can see the update state of every unit from the admin console at all times

---

### Phase 3 — Summary of Deliverables

| Deliverable | Description |
|---|---|
| Cloud Orchestration Tier | Orchestration services deployed to cloud; Edge Tier syncs when connected |
| Remote access for all roles | All role-specific interfaces accessible remotely via the cloud |
| Concurrent toolpath processing | Pipeline handles multiple requests in parallel on cloud infrastructure |
| Cloud data storage | Production database replicated to cloud; local edge database is fallback |
| OTA software updates | Mender-based OTA delivery from cloud update server to registered units |
| OTA firmware updates | Firmware updates for PMAC and sensor module delivered OTA |
| Centralised version registry | Software and firmware versions tracked fleet-wide; offline updates reconciled on reconnect |

---

## Phase 4 — Fleet Management and AI

**Goal:** Enable multi-site fleet management, introduce AI/ML capabilities, complete the hardware abstraction layer, and establish a hardware-in-the-loop testing framework.

---

### 4.1 Fleet Management Dashboard

**Problem:** The system is designed for a single robot. There is no multi-site visibility, no cross-unit coordination, and no centralised orchestration.

**Work required:**

- Implement a Fleet Management Dashboard accessible to Fleet Managers via the cloud Application Tier
- The dashboard shows all registered robot units across all active sites: real-time status, current job, any alerts, and last-known state for offline units
- Fleet Managers can register new robot units and define their capability profiles
- Fleet Managers are alerted when an offline unit reconnects

---

### 4.2 Operational Telemetry Collection

**Problem:** The system collects no operational telemetry. There is no data to support product improvement or AI/ML capabilities.

**Work required:**

- Implement systematic collection of print performance metrics, sensor readings, and system events across all units
- Telemetry is streamed to the cloud Orchestration Tier and stored for analysis
- Default to anomaly-windowed capture for high-volume hardware telemetry (full-fidelity data around fault events; sampled data otherwise), as defined in the Stratum spec
- Configurable retention policies per data layer

---

### 4.3 AI/ML Model Lifecycle Management

**Problem:** There is no AI or ML functionality and no infrastructure to support it.

**Work required:**

- Establish a managed pipeline for training, deploying, monitoring, and updating AI/ML models using real operational telemetry as the training source
- Initial model candidates:
  - Print quality assessment (detecting deposition anomalies from sensor data)
  - Anomaly detection (identifying unusual machine behaviour before it becomes a fault)
  - Predictive maintenance (flagging components likely to require service)
- Models are deployed to the Edge Tier for inference; the Orchestration Tier manages the model lifecycle (versioning, rollout, monitoring)

---

### 4.4 Plugin-Style Hardware Abstraction Layer

**Problem:** Integrating new robot types, generations, or auxiliary equipment requires changes to core platform logic.

**Work required:**

- Formalise the hardware abstraction layer (currently partially implemented via `dcrafter_control_adapters`) into a well-defined plugin interface
- New robot types, sensor modules, and tooling variants are integrated by implementing the plugin interface and registering a capability profile — no changes to core orchestration or planning logic are required
- Existing hardware variants are migrated to the plugin model
- The capability profile registration system is integrated with the Fleet Manager's dispatch validation

**Affected packages:** `dcrafter_control_adapters`, `robot_path_planners`, `scara_moveit_config`, `scara_description`

---

### 4.5 Hardware-in-the-Loop Testing Framework

**Problem:** There is no structured framework for testing new features against a simulated or real robot. Features involving hardware are deployed without thorough validation.

**Work required:**

- Implement a hardware-in-the-loop (HIL) testing environment that allows developers to validate new features against a simulated robot (using the existing simulation adapters in `dcrafter_control_adapters`) and against real hardware
- The framework supports automated test execution as part of the CI pipeline for simulation-based tests
- Real-hardware test runs are gated and documented separately
- New features involving hardware or motion must pass HIL tests before deployment to production units

---

### Phase 4 — Summary of Deliverables

| Deliverable | Description |
|---|---|
| Fleet Management Dashboard | Real-time status of all units across all sites; offline unit tracking |
| Operational telemetry | Systematic collection of print performance and sensor data across the fleet |
| AI/ML lifecycle pipeline | Training, deployment, monitoring, and updating of AI/ML models |
| Hardware abstraction layer | Plugin-style interface for new robot types and auxiliary equipment |
| HIL testing framework | Structured testing environment for hardware-dependent features |

---

## Cross-Cutting Concerns

The following concerns apply across all phases and should be treated as ongoing workstreams rather than one-time deliverables.

**Documentation:** Architecture documentation must be updated at the conclusion of each phase to reflect the current state of the system. All inter-tier interfaces must be documented before implementation begins.

**Security:** Each phase introduces new attack surface. A security review should be conducted before each phase is closed: Phase 1 (local service boundaries), Phase 2 (authentication and TLS), Phase 3 (cloud exposure and OTA integrity), Phase 4 (multi-tenant data isolation).

**Backwards compatibility:** Phase 1 refactoring must not break existing operator workflows. All changes in Phase 1 are internal; the operator-facing behaviour must be preserved without regression.

**Mender integration:** The Mender client and artifact format are introduced in Phase 1 and extended in Phases 2 and 3. Decisions made in Phase 1 about artifact structure, signing, and A/B partition layout must be forward-compatible with the OTA server integration planned for Phase 3.

---

## Phase Summary

| Phase | Goal | Key Deliverables |
|---|---|---|
| **Phase 1** | Reliable, observable, updateable system | Architecture refactor, production DB, structured logging, diagnostic data capture, report export, enforced homing, Mender packaging and offline update tooling |
| **Phase 2** | Full Stratum feature set on-premises | RBAC, authentication, TLS, role-specific interfaces, full toolpath pipeline, Building Model versioning, job lifecycle, ground scanning lifecycle, health checks |
| **Phase 3** | Cloud-hosted orchestration and OTA updates | Cloud Orchestration Tier, remote access for all roles, concurrent toolpath processing, cloud data storage, OTA software and firmware updates, centralised version registry |
| **Phase 4** | Fleet management and AI | Fleet dashboard, operational telemetry, AI/ML lifecycle pipeline, hardware abstraction layer, HIL testing framework |
