# Stratum Platform Specification

## Platform Overview

Stratum is a distributed platform for autonomous robotic construction. It orchestrates a fleet of SCARA-type concrete 3D printing robots that fabricate buildings layer by layer, and extends well beyond a single robot running locally — it is an **intelligent, connected construction platform** designed to scale from a single unit on one site to a coordinated fleet spanning multiple active sites.

The platform is organized across three tiers:

**Orchestration Tier** handles everything that does not need to be on-site: project design ingestion and toolpath pre-computation, fleet orchestration, OTA update distribution, centralized data storage and analytics, AI/ML model training and lifecycle management, and multi-tenant user management with role-based access control. The Orchestration Tier is deployable as a hosted service or as an on-premises installation on customer-managed infrastructure, including fully air-gapped environments.

**Edge Tier (on-robot IPC)** handles real-time, latency-sensitive operations: motion execution, sensor feedback, machine control, and local fallback when connectivity is lost. The edge layer syncs with the Orchestration Tier when available but operates independently when offline. It also accepts locally applied software and firmware updates when Orchestration Tier connectivity is unavailable.

**Application Tier** provides role-specific interfaces for architects, site operators, fleet managers, and developers:

- The **Site Operator interface** runs on-site, connecting directly to the robot's local WiFi access point for real-time machine control. It does not require connectivity to the Orchestration Tier and remains fully operational when that connectivity is unavailable.
- The **Architect, Fleet Manager, and Developer interfaces** connect to the Orchestration Tier — whether cloud-hosted or on-premises — and are intended for remote use. These roles have no operational need to be physically present on site.

---

## Deployment Modes

The platform supports two deployment modes. Both modes expose the same functional capabilities to all tiers; the deployment mode affects only where the Orchestration Tier is hosted and how it is provisioned.

**Cloud-hosted deployment** is the default mode. The Orchestration Tier is operated as a centrally managed hosted service. Robot units connect to it via their secondary uplink (cellular or site WiFi). OTA updates are distributed from the hosted service.

**On-premises deployment** allows the full Orchestration Tier to be installed on customer-managed infrastructure — bare-metal servers, private cloud, or a VM cluster — within the customer's own network. This mode supports organizations with data-residency requirements, restricted internet access, or air-gapped environments. In an on-prem deployment:

- All platform features operate against the local installation; no outbound internet connectivity is required.
- OTA update distribution is served from the on-prem update server rather than an external hosted service.
- Container images are pulled from a customer-managed registry or a local mirror.
- Authentication integrates with the customer's existing identity provider.
- Data replication occurs within the customer's own infrastructure.
- AI/ML model serving runs on-premises; model training may remain cloud-hosted or be explicitly scoped out per deployment configuration.

Both modes share a single CI/CD pipeline artifact, a single codebase, and a single update package format. Behavior is identical; only the hosting and provisioning target differs.

---

## System Setup

### Hardware

The SCARA robot — with its rover, servo drives, ultrasonic sensor, and CK3E motion controller — is treated as one node in a potentially large fleet. Each robot unit is equipped with:

- A hardened edge compute module with sufficient local storage to cache active print jobs and sustain fully offline operation
- Standardized hardware provisioning so that each unit can be reproduced exactly and consistently
- OTA-capable firmware on the motion controller and sensor module, eliminating the need for a dedicated IDE or physical presence to apply updates; locally applied updates via portable media (e.g. USB) are also supported for sites where Orchestration Tier connectivity is unavailable or restricted
- A plugin-style hardware abstraction layer so that new robot types, generations, and auxiliary equipment (sensors, tooling modules, material handling systems) can be integrated without requiring core architectural redesign

### Networking

Each robot unit provides a dedicated radio access point for reliable, low-latency local operator connectivity. A secondary uplink — cellular or site WiFi — handles Orchestration Tier and fleet communication independently. In on-prem deployments, the secondary uplink connects to the customer's internal network rather than an external hosted service.

### Deployment

The software stack is containerized and deployed via a CI/CD pipeline. Container images are pulled from a central registry in cloud-hosted deployments, or a customer-managed registry or local mirror in on-prem deployments. Versioned update packages are distributed OTA when Orchestration Tier connectivity is available, or transferred via portable media and applied locally when it is not. The Orchestration Tier is packaged as a standard deployment artifact (e.g. Helm chart or equivalent) for on-prem installation without internet access. Every unit's software version is tracked centrally regardless of delivery method or deployment mode.

### Database

The database layer uses a production-grade system with proper schema migration support and encrypted storage. In cloud-hosted deployments, data is replicated to cloud storage to prevent loss from local hardware failure. In on-prem deployments, replication targets customer-managed storage within their own infrastructure; high-availability topology is the customer's responsibility and is documented in the reference architecture.

### Security

Role-based access control is enforced across the platform; administrators, operators, and observers hold distinct permissions. In cloud-hosted deployments, authentication is managed by the hosted service. In on-prem deployments, the platform integrates with customer-managed identity providers (LDAP, Active Directory, SAML 2.0, OIDC) and provides a local bootstrap admin account for initial setup.

All communication — on the local network and over the uplink to the Orchestration Tier — is encrypted. On-prem deployments support private certificate authorities and self-managed TLS chains, removing any dependency on externally issued certificates. Update packages are cryptographically signed and verified before installation regardless of delivery method.

---

## System Behaviour

### Tier Interaction Model

#### Overview

The three tiers interact through two distinct interfaces:

- The **Orchestration–Edge interface** carries job packages from the Orchestration Tier to the edge tier, and diagnostic and state data from the edge tier back to the Orchestration Tier. Communication occurs over the robot's secondary uplink when connectivity is available. When connectivity is unavailable, the edge tier operates from its locally cached job package and buffers outbound state for later sync.
- The **Edge–Application interface** is a local API exposed by the edge tier over the robot's dedicated WiFi access point. The Site Operator interface communicates exclusively through this API. It does not require, and is not affected by, Orchestration Tier connectivity.

These two interfaces are independent. The operator's ability to control the robot is determined solely by the Edge–Application interface and the robot's local network — not by whether the Orchestration Tier is reachable.

#### Job Package

A job package is the self-contained unit assembled by the Orchestration Tier at dispatch time and delivered to the edge tier over the secondary uplink. It contains everything the edge tier needs to execute the job and support the operator's pre-print and print workflows without any further communication with the Orchestration Tier.

A job package includes:

- **Job identity and metadata** — job identifier, Print Project reference, target robot type and generation, job scope (layer and segment range), and site-specific notes.
- **Motion path** — the approved, robot-specific motion path output from Stage 4 of the toolpath generation pipeline, covering the full layer and segment scope of the job.
- **Task manifest** — the ordered list of tasks the edge tier is required to execute for this job, including a ground scanning task in Queued state and a print task in a pre-Ready state pending ground scan completion. The task manifest is the authoritative definition of what the robot must do; the edge tier does not derive or construct tasks independently.
- **Operator instructions** — site-specific notes and any job-level guidance the Fleet Manager has attached, surfaced to the operator in the job review screen.
- **Capability assertion** — a record of the capability profile the job was validated against at dispatch time, retained on the edge tier for local reference.

The job package is cryptographically signed by the Orchestration Tier. The edge tier verifies the signature before accepting the package. A package that fails verification is rejected and the job is not staged.

Once delivered, the job package is cached on the robot's local storage. The edge tier operates entirely from this local cache for the duration of the job. No re-fetching of motion path data or task definitions from the Orchestration Tier is required or attempted during execution.

#### Task Queue

The edge tier maintains a local task queue populated from the task manifest in the job package. Tasks are ordered and gated: a task does not become executable until all preceding tasks in the manifest have reached a terminal state (Completed or Aborted).

For a standard print job, the task queue contains two tasks in order:

1. **Ground scanning task** — must reach Completed before the print task becomes executable.
2. **Print task** — becomes executable once the ground scanning task is Completed.

The task queue is managed entirely by the edge tier. The operator interface reads queue state and sends control commands (start, abort, resume, E-Stop) through the Edge–Application interface. The Orchestration Tier does not issue task commands directly; it communicates intent through the job package at dispatch time.

When a ground scanning task is Aborted, the edge tier automatically inserts a new ground scanning task in Queued state ahead of the print task, so the operator can retry without manual intervention and without Orchestration Tier involvement.

#### Edge–Application Interface

The edge tier exposes a local API over the robot's dedicated WiFi access point. This is the sole communication channel between the Site Operator interface and the robot. All operator interactions — job review, task control, speed overrides, E-Stop, restart confirmation — are mediated through this interface.

The interface provides:

- **Job state and task queue reads** — the operator interface retrieves the current job state, task queue contents, and task states to render the guided workflow and control surfaces.
- **Task control commands** — start, abort, pause, resume. Commands are accepted only when the task queue state permits them; the edge tier enforces sequencing rules and rejects commands that would violate them.
- **Operator event submission** — structured records for human-in-the-loop confirmations (job acknowledgement, lintel placement, restart position confirmation). These are written to the edge tier's operator event log and included in the diagnostic record.
- **Speed override** — print speed and extrusion speed can be adjusted by the operator within bounds defined in the job's Print Configuration. Overrides take effect immediately and are recorded in the command log.
- **E-Stop** — a dedicated control that immediately halts all robot motion regardless of task state. Triggers the same abort path as a hardware E-Stop button event.
- **Telemetry and status reads** — live hardware telemetry, servo state, and sensor readings for display in the operator interface.

The Edge–Application interface is available as long as the robot's WiFi access point is operational, regardless of secondary uplink or Orchestration Tier connectivity.

#### State Synchronisation

The edge tier maintains an outbound sync queue of state changes to be reported to the Orchestration Tier. Sync occurs over the secondary uplink whenever connectivity is available. If connectivity is unavailable, changes accumulate in the sync queue and are flushed in order when connectivity is restored.

State changes that are synced to the Orchestration Tier include:

- **Task state transitions** — every change in ground scanning and print task state, with timestamp.
- **Job state transitions** — derived from task state transitions per the state propagation rules defined in Print Task States.
- **Operator events** — job acknowledgement, lintel placement confirmations, restart position confirmations, and speed overrides, each with timestamp and operator identity.
- **Diagnostic data** — command log entries, execution trace records, controller log entries, hardware telemetry snapshots, and operator event log entries, subject to the retention and telemetry capture policies defined in Diagnostic Data Correlation and Retention.
- **Ground surface data** — the output of a completed ground scanning task, registered against the job identifier.

Sync is append-only from the edge tier's perspective. The edge tier does not receive updated task definitions or motion path data from the Orchestration Tier mid-job; the job package delivered at dispatch is the sole source of execution inputs. The only post-dispatch communication from the Orchestration Tier to the edge tier is OTA update delivery, which is handled independently of job execution.

The Orchestration Tier treats synced state as authoritative for the purposes of job tracking, fleet visibility, and diagnostic record construction. Where the Orchestration Tier has no connectivity to a robot, it retains the last synced state and flags the unit as connectivity-impaired, consistent with FM-004.

---

### Building Model Versioning

A Building Model is created by uploading an IFC file. The platform parses the IFC and extracts the following:

**Building geometry:**
- Structural elements (walls, slabs, columns, beams) with their geometry and spatial relationships
- Storey and level definitions, used to derive layer boundaries during slicing
- Openings (doors, windows) that define voids in printable elements

**Site information:**
- Site boundary and coordinate reference, used to establish the print coordinate frame
- Keep-out zones defined in the IFC model (e.g. as spaces or zones with a designated classification), representing areas the robot must never enter
- Known obstacles defined in the IFC model (e.g. as site elements, existing structures, or reserved spaces), representing physical obstructions to be avoided during toolpath planning

All extracted content is made available for architect review in the 3D model preview window. The preview renders the building geometry and site information together — including keep-out zones and obstacles — so the architect can verify correct interpretation before proceeding. Extraction errors or unrecognised elements are surfaced as warnings so the architect can resolve them in the originating BIM software and re-upload before proceeding.

Each IFC upload creates a new, immutable version of the Building Model rather than overwriting the previous one. Versions are retained and can be referenced by multiple Print Projects. Once confirmed by the architect, a Building Model version cannot be modified.

### Print Configuration

A Print Configuration is a named, versioned set of fabrication parameters:

- **Layer height** — determines how the building geometry is sliced into printable layers
- **Nozzle geometry** — the physical dimensions of the print nozzle, used to compute extrusion paths
- **Nozzle type** — the nozzle variant, which may affect extrusion behaviour and path offset calculations
- **Speed parameters** — print speed and extrusion speed, applied during toolpath generation

Print Configurations are managed independently of Building Models and Print Projects. A single Print Configuration can be referenced by multiple Print Projects, and a Building Model can be combined with different Print Configurations to produce different toolpath outputs without re-uploading the IFC.

### Print Project Composition

A Print Project composes three inputs:

- A specific **Building Model version**
- A specific **Print Configuration version**
- A target **robot type and generation**

These inputs together fully define the pipeline run. The architect creates a Print Project by selecting each input explicitly. No pipeline execution occurs until the architect manually triggers it.

### Toolpath Generation Pipeline

Triggering a Print Project runs a four-stage pipeline. Each stage's output is stored and associated with the specific input versions that produced it.

**Stage 1 — Parse**
Input: Building Model version (IFC-extracted geometry and site information)
Output: Platform-native geometry representation, available for inspection in the 3D model preview

**Stage 2 — Slice**
Input: Parsed geometry, layer height (from Print Configuration)
Output: Per-layer contour set, available for inspection in the slice viewer — showing each layer's contour, total layer count, and layer index navigation

**Stage 3 — Toolpath Generation**
Input: Sliced layers, nozzle geometry, nozzle type, speed parameters (from Print Configuration)
Output: Toolpath per layer — extrusion paths and segment breakdown — available for inspection in the toolpath viewer

**Stage 4 — Motion Path Generation**
Input: Toolpath, robot type and generation
Output: Full robot motion path, available for review in the motion path simulation viewer — including robot motion playback, progress slider, and jump-to-layer controls

### Pipeline Re-entry

The pipeline runs to completion automatically once triggered. All four stage outputs are inspectable independently. Explicit architect approval is required only on the final motion path output before the Print Project is marked ready for dispatch.

When any input to a Print Project is changed and the pipeline is manually re-triggered by the architect, the platform identifies the earliest stage affected by the change and re-runs only that stage and all subsequent stages. Previously computed outputs for unaffected stages are reused.

Re-entry points are determined as follows:

| Changed input | Re-entry stage | Stages recomputed |
|---|---|---|
| Building Model version (IFC change) | Stage 1 — Parse | All four stages |
| Layer height | Stage 2 — Slice | Stages 2, 3, 4 |
| Nozzle geometry, nozzle type, or speed parameters | Stage 3 — Toolpath | Stages 3, 4 |
| Robot type or generation | Stage 4 — Motion Path | Stage 4 only |

The pipeline does not re-run automatically on input change. The architect must explicitly re-trigger execution. Any previously approved motion path is invalidated when the pipeline is re-triggered and requires re-approval once the new run completes.

### Offline Operation

The system degrades gracefully based on connectivity state. When connected to the Orchestration Tier, full platform features are available. When connectivity is lost, core print operations continue fully offline; the edge tier operates independently and syncs with the Orchestration Tier once connectivity is restored.

### Diagnostic Data Correlation and Retention

The platform maintains a structured, correlated diagnostic record for every print job. Five categories of data are captured and stored together under a shared job identifier and monotonic timestamp reference:

- **Command log** — every instruction issued to the robot, including motion commands and task transitions, with the timestamp and identity of the issuing actor (operator or system process)
- **Execution trace** — the robot's actual behaviour in response to commands: measured motion, encoder feedback, extrusion output, and task state transitions as they occurred
- **Controller log** — the motion controller's internal decisions, compensation values applied (e.g. ground scan offsets), fault codes raised, and internal state at the time of any deviation or anomaly
- **Hardware telemetry snapshot** — sensor readings, servo drive states, temperatures, voltages, and hardware fault flags, captured at sufficient frequency to reconstruct hardware conditions at the time of any event
- **Operator event log** — structured records of human-in-the-loop actions that are not robot commands, including: job acknowledgement, lintel placement confirmations, and restart position confirmations. Each entry captures the event type, timestamp, layer and segment index at the time of the event, and the identity of the acting operator.

Retention is managed per layer through independently configurable policies; older data is purged or transitioned to compressed long-term storage accordingly. At the edge tier, a local buffer cap limits on-device storage; data is synced to the Orchestration Tier and pruned locally once the cap is reached or the retention window expires. Hardware telemetry — the highest-volume layer — defaults to anomaly-windowed capture, retaining full-fidelity data around fault events and sampled data otherwise. All defaults are configurable by administrators. Per-layer retention policies are required; unbounded log growth is not acceptable in any deployment mode.

### Hardware Capability Modeling

Hardware is modeled via a configurable capability profile. A robot type and generation — representing a capability class — is selected by the architect when creating a Print Project and used as the target for motion path generation. The platform validates that the specific registered unit assigned by the Fleet Manager at dispatch time satisfies the same capability profile before the job is dispatched.

### Ground Scanning Task States

A ground scanning task moves through a defined set of states during its lifecycle:

- **Queued** — the task has been automatically created by the Orchestration Tier at the point the print job is dispatched, and delivered to the edge tier as part of the job package. It is available for operator execution via the robot's local network regardless of subsequent Orchestration Tier connectivity.
- **Active** — the task is currently being executed by the robot unit.
- **Completed** — the task finished successfully. The resulting ground surface data is stored and registered against the associated print job at the edge tier, and synced to the Orchestration Tier when connectivity is available.
- **Aborted** — the task was stopped by the operator or terminated due to a fault before completion. No ground surface data is registered. The edge tier automatically inserts a new ground scanning task in **Queued** state ahead of the print task so the operator can retry without additional steps and without requiring Orchestration Tier connectivity. Task state is synced to the Orchestration Tier when connectivity is restored.

The associated print job remains in **Dispatched** state while a ground scanning task is Queued or Active. The job may only be advanced to the operator's pre-print workflow once a ground scanning task for it has reached Completed state.

### Print Task States

A print task is the on-robot execution of a dispatched print job. It moves through a defined set of states at the edge tier:

- **Ready** — the task has been received and staged on the robot unit, and the operator has acknowledged the job. The robot is not yet in motion.
- **Running** — the robot is actively executing the toolpath.
- **Paused** — execution has been deliberately suspended by the operator. The robot holds position and can be resumed from the point of suspension.
- **Aborted** — the task was terminated due to an E-Stop (triggered via either the software E-Stop control or the hardware E-Stop button), a fault, or manual intervention. All robot motion is immediately halted. The abort position is recorded. An aborted task cannot be restarted directly; the operator must proceed through the restart workflow.
- **Completed** — the task finished successfully. All layers and segments within the job's scope have been executed.

Print task state changes propagate to the parent print job state as follows:

| Print task state | Resulting print job state |
|---|---|
| Ready | Active |
| Running | Active |
| Paused | Paused |
| Aborted | Aborted |
| Completed | Completed |

### Job States

A print job moves through a defined set of states during its lifecycle:

- **Pending Dispatch** — the job has been created from an approved project and is awaiting hardware assignment by the Fleet Manager.
- **Dispatched** — the job has been assigned to a robot unit by the Fleet Manager. The Orchestration Tier assembles a job package and delivers it to the edge tier over the secondary uplink. A ground scanning task in **Queued** state is included in the job package task manifest. The job remains in this state until the ground scanning task reaches **Completed**, at which point the operator's pre-print workflow (homing, job acknowledgement) becomes available. All ground scanning interactions from this point are handled at the edge tier and do not require Orchestration Tier connectivity.
- **Active** — the job has been dispatched to a robot unit and is being executed. This state covers the Ready and Running print task states and persists until the task reaches a terminal state.
- **Paused** — the print task has been deliberately suspended by the operator. The job can be resumed from the point of suspension.
- **Aborted** — the print task was terminated due to an E-Stop, a fault, or manual intervention. The platform records the execution point at which the abort occurred. An aborted job cannot be resumed; it must be restarted.
- **Restarting** — the job is in the pre-restart workflow: the operator has initiated a restart and is being guided through position confirmation before execution resumes.
- **Completed** — all layers and segments within the job's scope have been successfully executed.

### Job Abort State

When a print job is aborted, the platform automatically records the layer index and segment offset of the last confirmed execution point as structured fields on the job record. This abort position is machine-readable and available to the restart workflow without relying on operator recall or log parsing.

### Restart Position Validation

For a restarted job, the platform uses the recorded abort position to define the valid restart boundary. When the operator confirms their jogged position, the platform compares it against this boundary and only permits the restart to proceed if the position falls within the aborted segment. The operator is blocked from initiating motion until this check passes.

### Job Scope Adjustment

The Fleet Manager may adjust the layer and segment range of a job before dispatch. This allows a partially completed job to be assigned to a replacement unit covering only the remaining work. The platform validates that the replacement unit satisfies the same capability profile as the original before dispatch is permitted.

### Restart Audit Trail

A restarted job — whether on the original unit or a replacement — is recorded under the same job identifier as the original. The abort event, any scope adjustment, any unit substitution, and the restart point are all captured as structured entries in the job's diagnostic record, so the full execution history is reconstructable from a single record.

---

## Architectural Assumptions

The following assumptions form the basis for all system design decisions.

| # | Topic | Assumption |
|---|-------|------------|
| 1 | Robot registration | Robots and auxiliary equipment are manually registered by a Fleet Manager before being assigned work. Automated discovery is out of scope for the initial release. |
| 2 | Job-to-hardware dispatch | Job assignment is manual. The platform surfaces jobs with an approved toolpath awaiting hardware assignment; the Fleet Manager selects and dispatches to a specific registered unit. Automated assignment is out of scope for the initial release. |
| 3 | Identity and authentication | On-prem deployments integrate with customer-managed identity providers and must provide a local bootstrap admin mechanism independent of external authentication services. |

---

## User Stories

### Architect / Project Owner

- **APO-001:** I can upload an IFC file to create a new Building Model version and review the extracted building geometry and site information — including keep-out zones and known obstacles — in the 3D model preview window, so I can verify that the platform has correctly interpreted the design and site constraints before proceeding.
- **APO-002:** I can create and save a named Print Configuration — specifying layer height, nozzle geometry, nozzle type, and speed parameters — so I can reuse it across multiple Print Projects and update it independently when fabrication parameters change.
- **APO-003:** I can create a Print Project by selecting a Building Model version, a Print Configuration version, and a target robot type and generation, and manually trigger the pipeline to generate the motion path, so I can produce executable output for any combination of design and fabrication parameters without re-uploading the IFC.
- **APO-004:** I can receive a notification when a triggered pipeline run completes, so I know the Print Project is ready for my review without having to poll for status.
- **APO-005:** I can inspect the output of each pipeline stage — the parsed model in the 3D preview, per-layer contours in the slice viewer, extrusion paths in the toolpath viewer, and robot motion in the motion path simulation — so I can identify and address issues at the earliest possible stage.
- **APO-006:** I can review the final motion path in the simulation viewer — using a progress slider and layer-jump controls to observe robot motion and toolpath trajectory across the full build — and either approve it to mark the Print Project ready for dispatch, or go back and adjust inputs such as nozzle type, speed parameters, or layer height and re-trigger the pipeline, so I can iteratively refine the output until I am satisfied.
- **APO-007:** I can track the real-time progress of an active print job from anywhere, including overall completion status and estimated time remaining, so I can coordinate logistics and planning without being physically present.
- **APO-008:** I can see the current status of every print job created from my Print Projects — whether it is awaiting dispatch, assigned to a robot unit, acknowledged by the site operator, active, or completed — so I always have an accurate picture of where each build stands.

### Site Operator

- **SOP-001:** I am notified when a print job has been dispatched to my robot unit — immediately if connected, or as soon as connectivity is restored — and can review the full job details, including project name, toolpath summary, 3D simulation, target robot type and generation, and site-specific notes, before beginning setup.
- **SOP-002:** My local connection to the robot is stable and responsive enough for real-time print control, and all communication between my device and the robot is secure.
- **SOP-003:** I can run automated system health checks on any robot unit to confirm hardware readiness before a print job begins, reducing troubleshooting time.
- **SOP-004:** When a robot unit has no connectivity to the Orchestration Tier, I can apply a software or firmware update locally using portable media so the unit can be kept current without waiting for connectivity to be restored.
- **SOP-005:** I can complete the pre-print sequence — running the ground scan, homing the machine, and acknowledging the job — and then execute a print job through a clear, guided workflow that prevents me from skipping required steps.
- **SOP-006:** I can start or abort the ground scanning task from the control station within the guided workflow using my local connection to the robot, without requiring connectivity to the Orchestration Tier. If the scan is aborted, I can retry immediately without additional steps, so that ground surface data is always collected and registered against the job before printing begins.
- **SOP-007:** I can use a tablet connected to the robot's local network to control the machine, start or abort tasks, and override print and extrusion speed, even when there is no connectivity to the Orchestration Tier.
- **SOP-008:** I am clearly notified when a layer requires lintel placement and must confirm before printing resumes, so structural requirements are never skipped.
- **SOP-009:** I can trigger an Emergency Stop at any time — using either the software E-Stop control or the hardware E-Stop button on the robot — to immediately halt all robot motion, so I can respond instantly to unsafe conditions on site.
- **SOP-010:** I receive clear, actionable error messages when something goes wrong, rather than ambiguous failures, so I can resolve issues quickly without escalating for support.
- **SOP-011:** When restarting an aborted job, I am prompted to manually jog the printhead to the intended restart point and confirm its position, so the platform can verify placement before allowing motion to begin.

### Fleet Manager

- **FM-001:** I can manually register a new robot unit or piece of auxiliary equipment into the platform, defining its generation and capability profile, so it is ready to be assigned work.
- **FM-002:** I can onboard new robot types, generations, or auxiliary equipment into the fleet without requiring core platform changes, so the system scales naturally as hardware evolves.
- **FM-003:** I can view all active robots across one or more construction sites on a single dashboard, seeing their status, current project, and any alerts, so I can coordinate logistics and respond to issues proactively.
- **FM-004:** When a robot loses connectivity to the Orchestration Tier, I can see its last known status and am alerted when it comes back online, so I never have an unknown or stale view of the fleet.
- **FM-005:** I can see all print jobs that have an approved toolpath and are awaiting hardware assignment, so I know what work is ready to dispatch and can assign each job to an available, capable robot unit.
- **FM-006:** I can assign a print job to a specific registered robot unit, with the platform confirming the assigned unit satisfies the job's required capability profile before dispatch is permitted.
- **FM-007:** I can adjust the scope of a print job — specifying the layer and segment range to be executed — before dispatching it to a robot unit, so that when a job needs to be restarted on a replacement unit, I can assign only the remaining work rather than the full job.
- **FM-008:** Once I dispatch a job to a robot unit, I can see when the site operator has opened and acknowledged it, so I have confirmation the assignment has landed and the job is progressing toward execution.
- **FM-009:** I can push a software or firmware update to one robot or the entire fleet from a central admin console, without visiting each site, so all units stay on a consistent, known-good version.
- **FM-010:** I can download a signed, versioned update package from the admin console and hand it off to a site operator for local installation, so units at sites without connectivity can still receive updates in a controlled, auditable way.
- **FM-011:** Once a locally updated unit reconnects to the Orchestration Tier, its reported software version is automatically reconciled with the central version registry, so I always have an accurate view of the entire fleet's update state.
- **FM-012:** I can review historical usage data, error logs, and print telemetry for any robot, so I can identify recurring issues and optimize operations over time.

### Developer / ML Engineer

- **DEV-001:** I can integrate support for a new robot type, generation, or piece of auxiliary equipment by implementing a well-defined interface and registering its capability profile, without modifying core platform logic.
- **DEV-002:** I can deploy the platform's Orchestration Tier to a customer's own infrastructure using a standard, self-contained installation package — including fully air-gapped environments — so the platform can be operated without depending on an external hosted service or requiring outbound internet access.
- **DEV-003:** I can build and publish update packages that are valid for OTA delivery, on-prem update server distribution, and offline portable media transfer from a single CI/CD pipeline artifact, so I don't maintain separate release processes for different deployment modes or connectivity scenarios.
- **DEV-004:** I can run hardware-in-the-loop tests against a simulated or real robot through a structured testing framework, so new features are validated before deployment and don't break existing behavior.
- **DEV-005:** I can deploy, monitor, and update AI/ML models — for example, for print quality assessment or anomaly detection — through a managed lifecycle pipeline, so models can improve over time with real operational data.
- **DEV-006:** I can view structured error and crash reports with sufficient context to reproduce and fix issues, so the feedback loop from field failures to software fixes is short.
- **DEV-007:** When troubleshooting any issue — hardware, motion, or operational — I can retrieve a correlated diagnostic record for any job that shows what was commanded, what the robot actually did, what decisions the motion controller made, what hardware state existed at the time, and what actions the operator took — so I can reconstruct the full sequence of events and diagnose root causes without needing physical access to the unit.
