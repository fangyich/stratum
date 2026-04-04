# Stratum Platform Specification

## Platform Overview

Stratum is a distributed platform for autonomous robotic construction. It orchestrates a fleet of SCARA-type concrete 3D printing robots that fabricate buildings layer by layer, and extends well beyond a single robot running locally — it is an **intelligent, connected construction platform** designed to scale from a single unit on one site to a coordinated fleet spanning multiple active sites.

The platform is organized across three tiers:

**Orchestration Tier** handles everything that does not need to be on-site: project design ingestion and toolpath pre-computation, fleet orchestration, OTA update distribution (cloud-hosted deployments only), centralized data storage and analytics, AI/ML model training and lifecycle management, and multi-tenant user management with role-based access control. The Orchestration Tier is deployable as a hosted service or as an on-premises installation on customer-managed infrastructure, including fully air-gapped environments.

**Edge Tier (on-robot IPC)** handles real-time, latency-sensitive operations: motion execution, sensor feedback, machine control, and local fallback when connectivity is lost. The edge layer syncs with the Orchestration Tier when available but operates independently when offline. It also accepts locally applied software and firmware updates when Orchestration Tier connectivity is unavailable.

**Application Tier** provides role-specific interfaces for architects, site operators, fleet managers, and developers:

- The **Site Operator interface** runs on-site, connecting directly to the robot's local WiFi access point for real-time machine control. It does not require connectivity to the remote services and remains fully operational when that connectivity is unavailable.
- The **Architect, Fleet Manager, and Developer interfaces** connect to the Orchestration Tier — whether cloud-hosted or on-premises — and are intended for remote use. These roles have no operational need to be physically present on site.

---

## Deployment Modes

The platform supports two deployment modes. Both modes expose the same functional capabilities to all tiers and share a single codebase; the deployment mode affects only where the Orchestration Tier is hosted and how it is provisioned. The Orchestration Tier and the edge tier are built and packaged by separate CI/CD pipelines suited to their respective runtime environments, both owned and operated by the platform team, and both producing cryptographically signed outputs. The software stack is containerized; every unit's software version is tracked centrally regardless of delivery method or deployment mode.

**Cloud-hosted deployment** is the default mode. The Orchestration Tier is operated as a centrally managed hosted service. Robot units connect to it via their secondary uplink (cellular or site WiFi). Container images are pulled from a central registry. OTA updates are managed through a separately provisioned update service that the Orchestration Tier integrates with; versioned update packages are distributed OTA when connectivity is available, or transferred via portable media and applied locally when it is not.

**On-premises deployment** allows the full Orchestration Tier to be installed on customer-managed infrastructure — bare-metal servers, private cloud, or a VM cluster — within the customer's own network. This mode supports organizations with data-residency requirements, restricted internet access, or air-gapped environments. In an on-prem deployment:

- All platform features operate against the local installation; no outbound internet connectivity is required.
- The Orchestration Tier is delivered as a self-contained deployment package (e.g. Helm chart or equivalent) that includes all required container images and manifests. No external container registry is required at deploy time. The package is produced by the CI/CD pipeline, delivered to the customer out-of-band, and applied to the customer's infrastructure by the customer's administrator. Updates to the Orchestration Tier follow the same process — a new self-contained package is delivered out-of-band and applied by the administrator.
- The update service is provisioned and operated by the customer on their own infrastructure. Signed update packages produced by the CI/CD pipeline are delivered to the customer out-of-band as part of each release and imported into the update service by the customer's administrator.
- Authentication integrates with the customer's existing identity provider.
- Data replication occurs within the customer's own infrastructure.
- AI/ML model serving runs on-premises; model training may remain cloud-hosted or be explicitly scoped out per deployment configuration.

In both deployment modes, imported update packages are available to Fleet Managers for deployment to units through the Orchestration Tier's admin console. Units with no connectivity to the update service can still be updated via portable media using the same signed packages.

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

### Database

The database layer uses a production-grade system with proper schema migration support and encrypted storage. In cloud-hosted deployments, data is replicated to cloud storage to prevent loss from local hardware failure. In on-prem deployments, replication targets customer-managed storage within their own infrastructure; high-availability topology is the customer's responsibility and is documented in the reference architecture.

### Security

Role-based access control is enforced across the platform; administrators, operators, and observers hold distinct permissions. In cloud-hosted deployments, authentication is managed by the hosted service. In on-prem deployments, the platform integrates with customer-managed identity providers (LDAP, Active Directory, SAML 2.0, OIDC) and provides a local bootstrap admin account for initial setup.

All communication — on the local network and over the uplink to the Orchestration Tier — is encrypted. On-prem deployments support private certificate authorities and self-managed TLS chains, removing any dependency on externally issued certificates. Update packages are cryptographically signed and verified before installation regardless of delivery method.

---

## System Behaviour

### Orchestration Tier

#### Design Model

A **Design Model** is a named container created by the architect to represent a design project or site. A single Design Model may encompass one or more buildings — for example, a residential complex or a multi-structure development — as defined in the uploaded IFC. An architect maintains a collection of Design Models and each accumulates versions over time as the design evolves.

A **Design Model version** is created by uploading an IFC file to an existing Design Model. Each upload produces a new, immutable version rather than overwriting the previous one; the Design Model itself is never replaced. A Print Project always references a specific version, not the Design Model container, which allows multiple projects to reference different iterations of the same design independently.

When an IFC file is uploaded, the platform parses it and extracts the following content into the new version record:

*Building geometry:*
- One or more buildings identified within the IFC, each treated as a distinct printable structure
- Structural elements (walls, slabs, columns, beams) per building, with their geometry and spatial relationships
- Storey and level definitions per building, used to derive layer boundaries during slicing and to determine job scope boundaries when splitting work across units
- Openings (doors, windows) per building that define voids in printable elements

*Site information:*
- Site boundary and coordinate reference, used to establish the print coordinate frame
- Keep-out zones defined in the IFC model (e.g. as spaces or zones with a designated classification), representing areas the robot must never enter
- Known obstacles defined in the IFC model (e.g. as site elements, existing structures, or reserved spaces), representing physical obstructions to be avoided during toolpath planning

All extracted content is made available for architect review in the 3D model preview window. The preview renders all buildings and site information together — including keep-out zones and obstacles — so the architect can verify correct interpretation of the full site before proceeding.

#### Print Configuration

A Print Configuration is a named, immutable set of fabrication parameters:

- **Nozzle geometry** — the physical dimensions of the print nozzle, used to compute extrusion paths
- **Nozzle type** — the nozzle variant, which may affect extrusion behaviour and path offset calculations
- **Speed parameters** — print speed and extrusion speed, applied during toolpath generation

Print Configurations are managed independently of Design Models and Print Projects. A single Print Configuration can be referenced by multiple Print Projects. If the parameters of an existing configuration no longer meet requirements, the architect creates a new named configuration and selects it for the relevant Print Projects.

#### Print Project

A Print Project composes four inputs:

- A specific **Design Model version**
- A **layer height** — determines how the building geometry is sliced into printable layers
- A named **Print Configuration**
- A target **robot type and generation**

These inputs together fully define the pipeline run. The architect creates a Print Project by selecting each input explicitly. No pipeline execution occurs until the architect manually triggers it.

#### Hardware Capability Modeling

Hardware is modeled via a configurable capability profile. A robot type and generation — representing a capability class — is selected by the architect when creating a Print Project and used as the target for motion path generation. The platform validates that the specific registered unit assigned by the Fleet Manager at dispatch time satisfies the same capability profile before the job is dispatched.

#### Toolpath Generation Pipeline

Triggering a Print Project runs a four-stage pipeline. Each stage's output is stored as it is produced and associated with the specific input set that produced it. Outputs from a previous run are not overwritten when a new run begins; a new run produces a new set of outputs under a new run identifier, leaving prior outputs available for reference.

**Stage 1 — Parse**
Input: Design Model version (IFC-extracted geometry and site information)
Output: Platform-native geometry representation

**Stage 2 — Slice**
Input: Parsed geometry, layer height (from Print Project)
Output: Per-layer contour set with total layer count and per-layer index

**Stage 3 — Toolpath Generation**
Input: Sliced layers, nozzle geometry, nozzle type, speed parameters (from Print Configuration)
Output: Toolpath per layer — extrusion paths and segment breakdown

**Stage 4 — Motion Path Generation**
Input: Toolpath, robot type and generation
Output: Full robot motion path

#### Pipeline Re-entry

The pipeline runs to completion automatically once triggered. All four stage outputs are inspectable independently. Explicit architect approval is required only on the Stage 4 motion path output before the Print Project transitions to Approved state.

When any input to a Print Project is changed and the pipeline is manually re-triggered by the architect, the platform identifies the earliest stage affected by the change and re-runs only that stage and all subsequent stages. Previously computed outputs for unaffected stages are reused.

Re-entry points are determined as follows:

| Changed input | Re-entry stage | Stages recomputed |
|---|---|---|
| Design Model version | Stage 1 — Parse | All four stages |
| Layer height | Stage 2 — Slice | Stages 2, 3, 4 |
| Print Configuration | Stage 3 — Toolpath | Stages 3, 4 |
| Robot type or generation | Stage 4 — Motion Path | Stage 4 only |

The pipeline does not re-run automatically on input change. The architect must explicitly re-trigger execution. Any previously approved motion path is invalidated when the pipeline is re-triggered and requires re-approval once the new run completes.

#### Pipeline Run State

A pipeline run moves through a defined set of states during its execution:

- **Pending** — triggered, not yet started.
- **Running(stage N)** — currently executing the indicated stage.
- **Completed** — all stages finished; all outputs available.
- **Failed(stage N, reason)** — execution halted at the indicated stage with a structured error reason.

#### Print Project States

A Print Project moves through a defined set of states as it progresses from design to production:

- **Draft** — the project has been created and its pipeline has not yet completed, or the architect has not yet approved the motion path output. The architect may modify inputs and re-trigger the pipeline freely.
- **Approved** — the architect has approved the Stage 4 motion path output. The project is ready for a print request to be sent. The architect may delete the project in this state. If the architect re-triggers the pipeline, the project returns to Draft.
- **Requested** — the architect has sent a print request. The request is Open and awaiting acceptance.
- **In Production** — a Fleet Manager has accepted the request. Print jobs have been automatically created and the build is underway. The project remains in this state until all derived jobs reach a terminal state.

The print request lifecycle is defined in Print Request States below.

#### Print Request States

A print request is created when the architect sends a request from an Approved Print Project. It moves through the following states:

- **Open** — the request has been submitted and is sitting in the shared Fleet Manager pool. It is visible to all Fleet Managers, any of whom may inspect its details and accept it. The architect may withdraw the request at any time while it remains in this state.
- **Withdrawn** — the architect withdrew the request before any Fleet Manager accepted it. The parent Print Project returns to Approved state. Terminal.
- **Accepted** — a Fleet Manager accepted the request. The platform creates print jobs automatically and the parent Print Project transitions to In Production state. Once accepted, the request cannot be undone. Terminal.

#### Job States

Print jobs are created automatically by the platform when a Fleet Manager accepts a print request. The platform splits the full build scope into one or more jobs by building and storey, each in Pending Dispatch state. Each job moves independently through the following states:

- **Pending Dispatch** — the job has been created and is awaiting hardware assignment by the Fleet Manager.
- **Dispatched** — the job has been assigned to a robot unit by the Fleet Manager. A job package is assembled and delivered to the edge tier. A ground scanning task in Queued state is included in the task manifest. The job remains in this state until the ground scanning task reaches Completed, at which point the operator's pre-print workflow becomes available.
- **Active** — the job is being executed on the robot unit. This state covers the Ready and Running print task states and persists until the task reaches a terminal state.
- **Paused** — the print task has been deliberately suspended by the operator. The job can be resumed from the point of suspension.
- **Aborted** — the print task was terminated due to an E-Stop, a fault, or manual intervention. The platform records the execution point at which the abort occurred. An aborted job cannot be resumed; it must be restarted.
- **Restarting** — the job is in the pre-restart workflow: the operator has initiated a restart and is being guided through position confirmation before execution resumes.
- **Completed** — all layers and segments within the job's scope have been successfully executed.

#### Job Abort State

When a print job is aborted, the platform automatically records the layer index and segment offset of the last confirmed execution point as structured fields on the job record. This abort position is machine-readable and available to the restart workflow without relying on operator recall or log parsing.

#### Job Scope Adjustment

The Fleet Manager may adjust the layer and segment range of a job before dispatch. This allows a partially completed job to be assigned to a replacement unit covering only the remaining work. The platform validates that the replacement unit satisfies the same capability profile as the original before dispatch is permitted.

#### Restart Audit Trail

A restarted job — whether on the original unit or a replacement — is recorded under the same job identifier as the original. The abort event, any scope adjustment, any unit substitution, and the restart point are all captured as structured entries in the job's diagnostic record, so the full execution history is reconstructable from a single record.

#### Diagnostic Data Correlation and Retention

The platform maintains a structured, correlated diagnostic record for every print job. Five categories of data are captured and stored together under a shared job identifier and monotonic timestamp reference:

- **Command log** — every instruction issued to the robot, including motion commands and task transitions, with the timestamp and identity of the issuing actor (operator or system process)
- **Execution trace** — the robot's actual behaviour in response to commands: measured motion, encoder feedback, extrusion output, and task state transitions as they occurred
- **Controller log** — the motion controller's internal decisions, compensation values applied (e.g. ground scan offsets), fault codes raised, and internal state at the time of any deviation or anomaly
- **Hardware telemetry snapshot** — sensor readings, servo drive states, temperatures, voltages, and hardware fault flags, captured at sufficient frequency to reconstruct hardware conditions at the time of any event
- **Operator event log** — structured records of human-in-the-loop actions that are not robot commands, including: job acknowledgement, lintel placement confirmations, and restart position confirmations. Each entry captures the event type, timestamp, layer and segment index at the time of the event, and the identity of the acting operator.

Within each job record, all log data is indexed by layer and segment, so any entry can be located and retrieved in the context of the specific point in the build at which it was produced.

Retention is managed at the job level through independently configurable policies applied uniformly to all jobs; the same policy governs how long each data category is kept before it is purged or transitioned to compressed long-term storage. At the edge tier, a local buffer cap limits on-device storage; data is synced to the Orchestration Tier and pruned locally once the cap is reached or the retention window expires. Hardware telemetry — the highest-volume category — defaults to anomaly-windowed capture, retaining full-fidelity data around fault events and sampled data otherwise. All defaults are configurable by administrators. Unbounded log growth is not acceptable in any deployment mode.

---

### Edge Tier

#### Task Queue

The edge tier maintains a local task queue populated from the task manifest in the job package. Tasks are ordered and gated: a task does not become executable until all preceding tasks in the manifest have reached a terminal state (Completed or Aborted).

For a standard print job, the task queue contains two tasks in order:

1. **Ground scanning task** — must reach Completed before the print task becomes executable.
2. **Print task** — becomes executable once the ground scanning task is Completed.

The task queue is managed entirely by the edge tier. The operator interface reads queue state and sends control commands through the Edge–Application interface. The Orchestration Tier does not issue task commands directly; it communicates intent through the job package at dispatch time.

When a ground scanning task is Aborted, the edge tier automatically inserts a new ground scanning task in Queued state ahead of the print task, so the operator can retry without manual intervention and without remote service connectivity.

#### Ground Scanning Task States

A ground scanning task moves through a defined set of states during its lifecycle:

- **Queued** — the task has been delivered to the edge tier as part of the job package and is available for operator execution regardless of remote service connectivity.
- **Active** — the task is currently being executed by the robot unit.
- **Completed** — the task finished successfully. The resulting ground surface data is stored and registered against the associated print job at the edge tier, and synced when connectivity is available.
- **Aborted** — the task was stopped by the operator or terminated due to a fault before completion. No ground surface data is registered. The edge tier automatically inserts a new ground scanning task in Queued state ahead of the print task so the operator can retry without additional steps. Task state is synced when connectivity is restored.

The associated print job remains in Dispatched state while a ground scanning task is Queued or Active. The job may only be advanced to the operator's pre-print workflow once a ground scanning task has reached Completed state.

#### Print Task States

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

#### Restart Position Validation

For a restarted job, the platform uses the recorded abort position to define the valid restart boundary. When the operator confirms their jogged position, the platform compares it against this boundary and only permits the restart to proceed if the position falls within the aborted segment. The operator is blocked from initiating motion until this check passes.

---

### Tier Interaction Model

#### Overview

The three tiers interact through three distinct interfaces:

- The **Orchestration–Edge interface** carries job packages from the Orchestration Tier to the edge tier at dispatch, and state and diagnostic data from the edge tier back to the Orchestration Tier via an outbound sync queue. Communication occurs over the robot's secondary uplink when connectivity is available.
- The **Edge–Application interface** is a local API exposed by the edge tier over the robot's dedicated WiFi access point. The Site Operator interface communicates exclusively through this API. It does not require, and is not affected by, remote service connectivity.
- The **Orchestration–Application interface** is the API through which the Architect, Fleet Manager, and Developer interfaces connect to the Orchestration Tier. It carries both request–response interactions and server-initiated push events.

These interfaces are independent. The operator's ability to control the robot is determined solely by the Edge–Application interface and the robot's local network — not by whether the remote services are reachable.

#### Offline Operation

The system degrades gracefully based on connectivity state. When connected to the remote services, full platform features are available. When connectivity is lost, core print operations continue fully offline; the edge tier operates independently and syncs once connectivity is restored.

---

#### Orchestration–Edge Interface

##### Overview

The Orchestration–Edge interface is a two-directional channel over the robot's secondary uplink. Downstream, the Orchestration Tier delivers signed job packages to the edge tier at dispatch time. Upstream, the edge tier maintains an outbound sync queue and flushes accumulated state and diagnostic data to the Orchestration Tier whenever connectivity is available. The interface operates independently of job execution; OTA update delivery is handled by the separately provisioned update service over the same uplink and is not part of this interface.

When connectivity is unavailable, the edge tier operates entirely from its locally cached job package and buffers outbound state until connectivity is restored. No mid-job re-fetching of motion path data or task definitions is required or attempted.

##### Job Package

A job package is the self-contained unit assembled by the Orchestration Tier at dispatch time and delivered to the edge tier over the secondary uplink. It contains everything the edge tier needs to execute the job and support the operator's pre-print and print workflows without any further communication with the Orchestration Tier.

A job package includes:

- **Job identity and metadata** — job identifier, Print Project reference, target robot type and generation, job scope (layer and segment range), and site-specific notes.
- **Motion path** — the approved, robot-specific motion path output from Stage 4 of the toolpath generation pipeline, covering the full layer and segment scope of the job.
- **Task manifest** — the ordered list of tasks the edge tier is required to execute for this job, including a ground scanning task in Queued state and a print task in a pre-Ready state pending ground scan completion. The task manifest is the authoritative definition of what the robot must do; the edge tier does not derive or construct tasks independently.
- **Operator instructions** — site-specific notes and any job-level guidance the Fleet Manager has attached, surfaced to the operator in the job review screen.
- **Capability assertion** — a record of the capability profile the job was validated against at dispatch time, retained on the edge tier for local reference.

The job package is cryptographically signed by the Orchestration Tier. The edge tier verifies the signature before accepting the package. A package that fails verification is rejected and the job is not staged. Once delivered, the job package is cached on the robot's local storage for the duration of the job.

##### State Synchronisation

The edge tier maintains an outbound sync queue of state changes to be reported to the Orchestration Tier. Sync occurs over the secondary uplink whenever connectivity is available. If connectivity is unavailable, changes accumulate in the sync queue and are flushed in order when connectivity is restored.

State changes that are synced include:

- **Task state transitions** — every change in ground scanning and print task state, with timestamp.
- **Job state transitions** — derived from task state transitions per the state propagation rules defined in Print Task States.
- **Operator events** — job acknowledgement, lintel placement confirmations, restart position confirmations, and speed overrides, each with timestamp and operator identity.
- **Diagnostic data** — command log entries, execution trace records, controller log entries, hardware telemetry snapshots, and operator event log entries, subject to the retention and telemetry capture policies defined in Diagnostic Data Correlation and Retention.
- **Ground surface data** — the output of a completed ground scanning task, registered against the job identifier.

Sync is append-only from the edge tier's perspective. The Orchestration Tier treats synced state as authoritative for the purposes of job tracking, fleet visibility, and diagnostic record construction. Where the Orchestration Tier has no connectivity to a robot, it retains the last synced state and flags the unit as connectivity-impaired, consistent with FM-004.

---

#### Edge–Application Interface

##### Overview

The Edge–Application interface is a local API exposed by the edge tier over the robot's dedicated WiFi access point. It is the sole communication channel between the Site Operator interface and the robot. All operator interactions — job review, task control, speed overrides, E-Stop, restart confirmation — are mediated through this interface. It is available as long as the robot's WiFi access point is operational, regardless of secondary uplink or remote service connectivity.

##### Capabilities

- **Job state and task queue reads** — the operator interface retrieves the current job state, task queue contents, and task states to render the guided workflow and control surfaces.
- **Task control commands** — start, abort, pause, resume. Commands are accepted only when the task queue state permits them; the edge tier enforces sequencing rules and rejects commands that would violate them.
- **Operator event submission** — structured records for human-in-the-loop confirmations (job acknowledgement, lintel placement, restart position confirmation). These are written to the edge tier's operator event log and included in the diagnostic record.
- **Speed override** — print speed and extrusion speed can be adjusted by the operator within bounds defined in the job's Print Configuration. Overrides take effect immediately and are recorded in the command log.
- **E-Stop** — a dedicated control that immediately halts all robot motion regardless of task state. Triggers the same abort path as a hardware E-Stop button event.
- **Telemetry and status reads** — live hardware telemetry, servo state, and sensor readings for display in the operator interface.

---

#### Orchestration–Application Interface

##### Overview

The Orchestration–Application interface is the primary API through which the Architect, Fleet Manager, and Developer roles interact with the platform. It connects remote clients to the Orchestration Tier, whether cloud-hosted or on-premises, and carries both request–response interactions and server-initiated push events. The Site Operator interface does not use this interface; all operator interactions with the robot are mediated exclusively through the Edge–Application interface.

All capabilities are available in both cloud-hosted and on-premises deployments. In on-premises deployments the interface endpoint is resolved against the customer's own infrastructure; no outbound internet connectivity is required.

The interface provides five categories of capability:

- **Resource management** — create, read, version, and delete the artefacts that define a build: Design Models, Print Configurations, Print Projects, and robot capability profiles.
- **Pipeline control** — trigger and inspect the toolpath generation pipeline; approve the final motion path output.
- **Fleet and job operations** — register units, review and accept print requests, assign jobs, adjust job scope, dispatch, and monitor execution across the fleet.
- **Update management** — push OTA updates, download signed packages for offline delivery, and track fleet-wide version state.
- **Observability** — retrieve diagnostic records, telemetry, error reports, ML model state, and historical usage data.

##### Resource Management

###### Design Model

**Design Model (container)**

| Operation | Initiating role | Behaviour |
|---|---|---|
| Create Design Model | Architect | Creates a new named Design Model container. Does not upload or process any IFC content; the container is empty until a version is added. Returns the Design Model identifier. |
| Read Design Model | Architect | Returns the Design Model name, identifier, and the list of its versions in creation order. |
| List Design Models | Architect | Returns all Design Models owned by or visible to the requesting architect. |
| Delete Design Model | Architect | Permanently deletes the Design Model container and all its versions. Rejected if any version is referenced by a Print Project; the architect must delete or reassign those projects first. |

**Design Model version**

| Operation | Initiating role | Behaviour |
|---|---|---|
| Upload IFC file | Architect | Adds a new, immutable version to an existing Design Model. Triggers IFC parsing and geometry extraction. Returns the new version identifier. |
| Read Design Model version | Architect | Returns extracted building geometry, site information (keep-out zones, known obstacles), and any extraction warnings or unrecognised elements. |
| Delete Design Model version | Architect | Permanently deletes a specific version. Rejected if the version is referenced by a Print Project; the architect must delete or reassign those projects first. The Design Model container and all other versions are unaffected. |

Extraction errors and unrecognised elements are surfaced as structured warnings on the version record, not as free-form log text, so the Architect interface can present them as actionable items for resolution in the originating BIM software before re-uploading (APO-001).

###### Print Configuration

| Operation | Initiating role | Behaviour |
|---|---|---|
| Create Print Configuration | Architect | Creates a new named, immutable configuration record with the supplied nozzle geometry, nozzle type, and speed parameters. |
| Read Print Configuration | Architect | Returns the full parameter set for the named configuration. |
| List Print Configurations | Architect | Returns all named configurations. |
| Delete Print Configuration | Architect | Permanently deletes a named configuration. Rejected if any Print Project references it; the architect must update those projects to reference a different configuration first. |

A Print Configuration is immutable once created. If existing configurations do not meet requirements, the architect creates a new one (APO-002).

###### Print Project

| Operation | Initiating role | Behaviour |
|---|---|---|
| Create Print Project | Architect | Creates a Print Project record in Draft state composed of a Design Model version, a layer height, a named Print Configuration, and a target robot type and generation. Does not trigger pipeline execution. |
| Read Print Project | Architect, Fleet Manager | Returns the project record, its composed inputs, current state, pipeline run state, stage outputs available for inspection, and all associated print jobs. |
| List Print Projects | Architect, Fleet Manager | Returns all Print Projects visible to the requesting role, including their state and pipeline state. |
| Trigger pipeline | Architect | Manually initiates a pipeline run. The platform determines the re-entry stage from the current input set and runs from that stage forward. A previously approved motion path is invalidated immediately; the project returns to Draft. Only permitted when the project is in Draft or Approved state. |
| Approve motion path | Architect | Marks the Stage 4 output as approved, transitioning the Print Project to Approved state. Only accepted when the pipeline run has completed and no invalidation is pending. |
| Delete Print Project | Architect | Permanently deletes the Print Project and all associated pipeline outputs. Only permitted when the project is in Draft or Approved state. |

**Print request**

| Operation | Initiating role | Behaviour |
|---|---|---|
| Send print request | Architect | Sends a print request to the shared Fleet Manager pool, transitioning the project to Requested state. Only permitted when the project is in Approved state. A project may only have one active request at a time. |
| Withdraw print request | Architect | Withdraws the request from the pool, returning the project to Approved state. Only permitted while the project is in Requested state. |
| List print requests | Fleet Manager | Returns all requests currently in the pool, each with the associated Print Project summary, building count, and estimated job breakdown by building and storey. |
| Read print request | Fleet Manager | Returns the full details of a specific request: Print Project record, composed inputs, motion path summary, and proposed job breakdown. |
| Accept print request | Fleet Manager | Accepts a request from the pool. The platform removes the request from the pool, automatically creates one or more print jobs split by building and storey each in Pending Dispatch state, and transitions the project to In Production state. Once accepted, the request cannot be undone. |

###### Robot Capability Profile

| Operation | Initiating role | Behaviour |
|---|---|---|
| Register robot type / generation | Fleet Manager, Developer | Creates or updates a capability profile for a robot type and generation, specifying the capability class used for Print Project targeting and dispatch validation. |
| Register robot unit | Fleet Manager | Registers a specific physical unit against a type and generation. Associates the unit with the capability profile of its class. |
| Read capability profile | Fleet Manager, Developer | Returns the capability profile for a given type and generation. |
| List registered units | Fleet Manager | Returns all registered units with their type, generation, connectivity state, current job assignment, and last-seen timestamp. |

New robot types, generations, and auxiliary equipment are registered through this operation without requiring changes to core platform logic (FM-001, FM-002, DEV-001).

##### Pipeline Control

###### Stage Output Inspection

All four stage outputs are inspectable independently; the Architect interface does not need to wait for the full pipeline to complete before displaying earlier stage outputs (APO-005).

| Output | Available to | Content |
|---|---|---|
| Stage 1 — Parsed geometry | Architect | Platform-native geometry representation, renderable in the 3D model preview. |
| Stage 2 — Slice | Architect | Per-layer contour set with total layer count and per-layer index, renderable in the slice viewer. |
| Stage 3 — Toolpath | Architect | Per-layer extrusion paths and segment breakdown, renderable in the toolpath viewer. |
| Stage 4 — Motion path | Architect | Full robot motion path with playback data, progress slider input, and layer-jump index, renderable in the motion path simulation viewer. |

###### Pipeline Run State

The interface exposes the current pipeline run state so the Architect interface can reflect progress without polling. The pipeline run state model is defined in the Orchestration Tier › Pipeline Run State section above.

##### Push Notifications

The interface delivers server-initiated events to connected clients. Clients subscribe on connection; events are delivered for the duration of the session. Clients that are not connected at the time an event is generated receive it upon reconnection.

| Event | Delivered to | Trigger condition |
|---|---|---|
| `pipeline.completed` | Architect | A pipeline run reaches Completed state. Payload includes the Print Project identifier and run identifier (APO-004). |
| `pipeline.failed` | Architect | A pipeline run fails. Payload includes the stage, structured error reason, and Print Project identifier. |
| `print_request.submitted` | Fleet Manager | A new print request has been added to the pool. Payload includes the request identifier and Print Project summary. |
| `print_request.withdrawn` | Fleet Manager | A request has been withdrawn from the pool by the architect. Payload includes the request identifier. |
| `print_request.accepted` | Architect | A Fleet Manager has accepted the request. Payload includes the request identifier and the list of created job identifiers. |
| `job.state_changed` | Architect, Fleet Manager | A print job transitions to a new state. Payload includes the job identifier, previous state, new state, and timestamp (APO-008, APO-009, FM-008). |
| `job.progress_updated` | Architect, Fleet Manager | A running job's completion percentage or estimated time remaining changes. Payload includes the job identifier, percentage complete, and ETA. Delivered at a cadence sufficient for real-time tracking without flooding clients (APO-008). |
| `robot.connectivity_lost` | Fleet Manager | A robot unit's secondary uplink becomes unreachable. Payload includes the unit identifier, last-known state, and timestamp (FM-004). |
| `robot.connectivity_restored` | Fleet Manager | A robot unit's secondary uplink reconnects. Payload includes the unit identifier and timestamp (FM-004). |
| `robot.version_reconciled` | Fleet Manager | A locally updated unit has reconnected and its software version has been reconciled with the central version registry. Payload includes the unit identifier and the reconciled version record (FM-010). |

Events are scoped to the data visible to the receiving role. A Fleet Manager receives connectivity and job events for all units and jobs within their managed scope. An Architect receives pipeline and job events for projects and jobs they own or have been granted access to.

##### Fleet and Job Operations

###### Job Lifecycle

Jobs are created automatically by the platform when a Fleet Manager accepts a print request. The Fleet Manager's role in the job lifecycle begins at dispatch.

| Operation | Initiating role | Behaviour |
|---|---|---|
| List dispatchable jobs | Fleet Manager | Returns all jobs in Pending Dispatch state awaiting hardware assignment, across all projects in In Production state within the Fleet Manager's scope (FM-005). |
| Adjust job scope | Fleet Manager | Updates the layer and segment range of a job in Pending Dispatch state. Only permitted before dispatch. The platform validates that the scope adjustment is consistent with the approved motion path. |
| Dispatch job | Fleet Manager | Assigns the job to a specific registered unit. The platform validates that the unit's capability profile satisfies the job's capability assertion before accepting the dispatch. On acceptance, assembles the job package and initiates delivery to the edge tier (FM-006, FM-007). |
| Read job | Architect, Fleet Manager, Developer | Returns the job record including current state, assigned unit, scope, abort position if applicable, and operator event entries as they have synced from the edge tier. |
| List jobs | Architect, Fleet Manager | Returns jobs filtered by project, unit, or state as specified by the caller. Architects see jobs created from their projects; Fleet Managers see all jobs within their managed scope. |

###### Fleet Dashboard

| Operation | Initiating role | Behaviour |
|---|---|---|
| Read fleet summary | Fleet Manager | Returns all registered units with their current state, connectivity status, active job identifier (if any), and any active alerts. Connectivity-impaired units are flagged with their last-known state and the timestamp of last contact (FM-003, FM-004). |

##### Update Management

| Operation | Initiating role | Behaviour |
|---|---|---|
| Push OTA update | Fleet Manager | Makes a specified update available to one or more targeted units with no job currently assigned, within the Fleet Manager's managed scope. The Orchestration Tier does not initiate a connection to the unit; each targeted unit discovers and downloads the update autonomously when it next checks in with the update service. Update state is tracked per unit (FM-009). |
| Download signed update package | Fleet Manager | Returns a signed, versioned update package via the update service, for portable media transfer to a site operator and local installation on a unit with no connectivity to the update service. The package is identical to those distributed via the update service; no separate build is required (FM-009). |
| Read unit version state | Fleet Manager | Returns the currently reported software and firmware versions for a specific unit, the delivery state of any pending OTA update (cloud-hosted deployments), and whether the version record has been reconciled following a local update (FM-010). |
| List fleet version state | Fleet Manager | Returns version and update delivery state for all units in the managed scope (FM-010). |

The Orchestration Tier integrates with the separately provisioned update service to expose update management operations to the Fleet Manager. All update packages are cryptographically signed by the CI/CD pipeline; the Orchestration Tier does not produce or distribute packages directly and does not accept upload of externally produced packages.

##### Observability

###### Diagnostic Records

| Operation | Initiating role | Behaviour |
|---|---|---|
| Read diagnostic record | Developer, Fleet Manager | Returns the correlated diagnostic record for a specified job: command log, execution trace, controller log, hardware telemetry snapshot, and operator event log, all under the shared job identifier and monotonic timestamp reference. Supports filtering by time range and data category (DEV-007). |
| Read error and crash reports | Developer | Returns structured error and crash reports for a specified unit or time range, with sufficient context (stack trace, state at time of fault, job identifier) to reproduce and fix issues (DEV-006). |

Diagnostic record retrieval returns data that has been synced to the Orchestration Tier at the time of the request. Data still buffered at the edge tier that has not yet synced is not included; the record reflects the last-synced state.

###### Historical Usage and Telemetry

| Operation | Initiating role | Behaviour |
|---|---|---|
| Read historical usage | Fleet Manager | Returns aggregated usage data for a specified unit or fleet scope over a specified time range: job counts, completion rates, fault frequency, and uptime (FM-011). |
| Read print telemetry history | Fleet Manager, Developer | Returns historical hardware telemetry for a specified job or unit, subject to the retention policies configured for the telemetry data layer (FM-011). |

###### ML Model Lifecycle

| Operation | Initiating role | Behaviour |
|---|---|---|
| Read model state | Developer | Returns the current deployment state of a managed ML model: version, serving status, last evaluation metrics, and any pending update (DEV-005). |
| Deploy model update | Developer | Initiates deployment of a new model version through the managed lifecycle pipeline. In on-premises deployments, model serving runs locally; model training scope is determined per deployment configuration (DEV-005). |

##### Access Control Summary

The table below summarises which roles may initiate each category of operation. Roles not listed for a given category have no access.

| Category | Architect | Fleet Manager | Developer |
|---|---|---|---|
| Design Model — create, read, upload, delete | ✓ | | |
| Print Configuration — create, read, delete | ✓ | | |
| Print Project — create, trigger, approve, delete, read | ✓ | Read only | |
| Pipeline stage output inspection | ✓ | | |
| Print request — send, withdraw | ✓ | | |
| Print request — list, read, accept | | ✓ | |
| Robot capability profile — register, read | | ✓ | ✓ |
| Robot unit — register, read, list | | ✓ | |
| Job — adjust scope, dispatch, read | | ✓ | Read only |
| Job — read own-project jobs | ✓ | | |
| Fleet dashboard | | ✓ | |
| Update — push OTA, download package, read version state | | ✓ | |
| Diagnostic records | | Read only | ✓ |
| Error and crash reports | | | ✓ |
| Historical usage and telemetry | | ✓ | ✓ |
| ML model lifecycle | | | ✓ |

Push notification delivery follows the same access boundaries: a role receives only events for resources within its access scope.

In on-premises deployments, role assignment is managed through the customer's identity provider. In cloud-hosted deployments, role assignment is managed by the hosted service. In both cases, an administrator role governs user and role management and is not listed above as it falls outside the scope of the application-facing interface.

---

### Live Telemetry and Alerting

#### Overview

Live telemetry is a periodic stream of operational signals sampled by each robot unit during active operation and synced to the Orchestration Tier at a configurable interval. It is distinct from the diagnostic telemetry snapshot captured per job for post-hoc forensic analysis: live telemetry is intended for real-time fleet visibility, alert generation, and accumulation as a training corpus for AI/ML models. Unit-level signals feed Fleet Manager monitoring at the Orchestration Tier; job-level signals feed operator alerts at the edge tier.

#### Telemetry Signal Categories

Two categories of signal are captured:

**Unit-level signals** reflect the hardware and connectivity state of the robot independent of any active job:

- Servo drive states and fault codes
- Hardware health indicators: temperatures, voltages, and sensor module status
- Connectivity state: secondary uplink signal quality and last-contact timestamp
- Firmware and software version identifiers

**Job-level signals** reflect the execution state of an active print job:

- Current task state and active layer and segment index
- Print and extrusion speed (commanded and measured)
- Extrusion output and material flow readings
- Motion controller compensation values in effect (e.g. ground scan offsets)
- E-Stop events and fault codes raised during execution
- Job completion percentage and estimated time remaining

#### Capture and Buffering at the Edge Tier

The edge tier samples live telemetry continuously during active operation. Unit-level signals are sampled at a low fixed cadence regardless of job state. Job-level signals are sampled at a higher cadence during active task execution and suspended when no task is running.

Sampled signals are written to a local telemetry buffer on the edge tier, separate from the diagnostic outbound sync queue. The buffer is bounded; when the cap is reached, the oldest samples are dropped to make room for new ones. The buffer cadence and cap are configurable by administrators.

#### Job-Level Alerting at the Edge Tier

Job-level alerts are generated and surfaced at the edge tier, and are available to the operator through the Edge–Application interface. They do not require connectivity to the Orchestration Tier.

Job-level alerts are raised when:

- An E-Stop event occurs during active task execution
- A motion controller fault is raised during a running print task
- Extrusion output or material flow deviates from expected range for a sustained period
- A print task transitions to Aborted state

Alerts are surfaced in the operator interface alongside the active task state and control surfaces. Alerts clear automatically when the condition that triggered them is resolved.

#### Transmission to the Orchestration Tier

The edge tier periodically syncs buffered telemetry samples to the Orchestration Tier over the secondary uplink. Both unit-level and job-level signals are transmitted; job-level signals inform AI/ML model training and historical analysis at the Orchestration Tier even though alert generation for those signals occurs at the edge. Telemetry sync operates independently of the diagnostic sync queue; telemetry delivery does not block or delay state synchronisation. The sync interval is configurable by administrators.

When connectivity is unavailable, buffered samples accumulate locally and are flushed in order once the uplink is restored. Samples dropped due to buffer overflow during a connectivity gap are not recovered; the Orchestration Tier records the gap as a period of missing data rather than treating the absence as a fault.

#### Storage and Availability at the Orchestration Tier

The Orchestration Tier ingests live telemetry into a time-series store keyed by unit identifier and timestamp. All received signals are retained subject to configurable retention policies; the default retention window is longer for job-level signals than for unit-level signals, reflecting their higher analytical value. Retention defaults are configurable by administrators. Unbounded telemetry growth is not acceptable in any deployment mode.

Stored telemetry is accessible to Fleet Managers and Developers through the existing historical telemetry operations defined in the Observability section. It is also made available as a structured dataset for AI/ML model training through the managed model lifecycle pipeline, allowing models — for example, for print quality assessment or anomaly detection — to be trained on real operational data accumulated across the fleet.

#### Unit-Level Alerting at the Orchestration Tier

Unit-level alerts are generated by the Orchestration Tier from incoming live telemetry and are surfaced to Fleet Managers through the Fleet Dashboard and push notifications.

Unit-level alerts are raised when:

- A hardware fault code is received from a servo drive or sensor module
- A hardware health indicator (temperature, voltage) exceeds a configured threshold
- The secondary uplink becomes unreachable (also reflected in the `robot.connectivity_lost` push event)

Active alerts are attached to the unit that generated them and surfaced on the Fleet Dashboard alongside the unit's current state. Alerts clear automatically when the condition that triggered them is resolved — for example, a connectivity alert clears when the uplink is restored, and a fault alert clears when the fault code is no longer active. Push notifications for alert-generating conditions are delivered to Fleet Managers through the existing `robot.connectivity_lost` and `robot.connectivity_restored` events; no separate alert notification event is defined.

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

- **APO-001:** I can upload an IFC file to create a new Design Model version and review the extracted building geometry and site information — including keep-out zones and known obstacles — in the 3D model preview window, so I can verify that the platform has correctly interpreted the design and site constraints before proceeding.
- **APO-002:** I can create and save a named Print Configuration — specifying layer height, nozzle geometry, nozzle type, and speed parameters — so I can reuse it across multiple Print Projects and update it independently when fabrication parameters change.
- **APO-003:** I can create a Print Project by selecting a Design Model version, a Print Configuration version, and a target robot type and generation, and manually trigger the pipeline to generate the motion path, so I can produce executable output for any combination of design and fabrication parameters without re-uploading the IFC.
- **APO-004:** I can receive a notification when a triggered pipeline run completes, so I know the Print Project is ready for my review without having to poll for status.
- **APO-005:** I can inspect the output of each pipeline stage — the parsed model in the 3D preview, per-layer contours in the slice viewer, extrusion paths in the toolpath viewer, and robot motion in the motion path simulation — so I can identify and address issues at the earliest possible stage.
- **APO-006:** I can review the final motion path in the simulation viewer — using a progress slider and layer-jump controls to observe robot motion and toolpath trajectory across the full build — and either approve it to mark the Print Project ready for dispatch, or go back and adjust inputs such as nozzle type, speed parameters, or layer height and re-trigger the pipeline, so I can iteratively refine the output until I am satisfied.
- **APO-007:** I can submit and withdraw a print request for an approved Print Project so that I can put the build forward for production when ready and pull it back if plans change, and I am notified when it is accepted so I never have to follow up manually to know it is underway.
- **APO-008:** I can track the real-time progress of an active print job from anywhere, including overall completion status and estimated time remaining, so I can coordinate logistics and planning without being physically present.
- **APO-009:** I can see the current status of every print job created from my Print Projects — whether it is awaiting dispatch, assigned to a robot unit, acknowledged by the site operator, active, or completed — so I always have an accurate picture of where each build stands.

### Site Operator

- **SOP-001:** I am notified when a print job has been dispatched to my robot unit — immediately if connected, or as soon as connectivity is restored — and can review the full job details, including project name, toolpath summary, 3D simulation, target robot type and generation, and site-specific notes, before beginning setup.
- **SOP-002:** My local connection to the robot is stable and responsive enough for real-time print control, and all communication between my device and the robot is secure.
- **SOP-003:** I can run automated system health checks on any robot unit to confirm hardware readiness before a print job begins, reducing troubleshooting time.
- **SOP-004:** When a robot unit has no connectivity to the remote services, I can apply a software or firmware update locally using portable media so the unit can be kept current without waiting for connectivity to be restored.
- **SOP-005:** I can complete the pre-print sequence — running the ground scan, homing the machine, and acknowledging the job — and then execute a print job through a clear, guided workflow that prevents me from skipping required steps.
- **SOP-006:** I can start or abort the ground scanning task from the control station within the guided workflow using my local connection to the robot, without requiring connectivity to the remote services. If the scan is aborted, I can retry immediately without additional steps, so that ground surface data is always collected and registered against the job before printing begins.
- **SOP-007:** I can use a tablet connected to the robot's local network to control the machine, start or abort tasks, and override print and extrusion speed, even when there is no connectivity to the remote services.
- **SOP-008:** I am clearly notified when a layer requires lintel placement and must confirm before printing resumes, so structural requirements are never skipped.
- **SOP-009:** I can trigger an Emergency Stop at any time — using either the software E-Stop control or the hardware E-Stop button on the robot — to immediately halt all robot motion, so I can respond instantly to unsafe conditions on site.
- **SOP-010:** I receive clear, actionable error messages when something goes wrong, rather than ambiguous failures, so I can resolve issues quickly without escalating for support.
- **SOP-011:** When restarting an aborted job, I am prompted to manually jog the printhead to the intended restart point and confirm its position, so the platform can verify placement before allowing motion to begin.

### Fleet Manager

- **FM-001:** I can manually register a new robot unit or piece of auxiliary equipment into the platform, defining its generation and capability profile, so it is ready to be assigned work.
- **FM-002:** I can onboard new robot types, generations, or auxiliary equipment into the fleet without requiring core platform changes, so the system scales naturally as hardware evolves.
- **FM-003:** I can view all active robots across one or more construction sites on a single dashboard, seeing their status, current project, and any alerts, so I can coordinate logistics and respond to issues proactively.
- **FM-004:** When a robot loses connectivity to the remote services, I can see its last known status and am alerted when it comes back online, so I never have an unknown or stale view of the fleet.
- **FM-005:** I can view incoming print requests with sufficient project detail to make an informed decision, accept one to take ownership of the build, and see the resulting jobs awaiting hardware assignment so I can begin dispatching them to available units.
- **FM-006:** I can assign a print job to a specific registered robot unit, with the platform confirming the assigned unit satisfies the job's required capability profile before dispatch is permitted.
- **FM-007:** I can adjust the scope of a print job — specifying the layer and segment range to be executed — before dispatching it to a robot unit, so that when a job needs to be restarted on a replacement unit, I can assign only the remaining work rather than the full job.
- **FM-008:** Once I dispatch a job to a robot unit, I can see when the site operator has opened and acknowledged it, so I have confirmation the assignment has landed and the job is progressing toward execution.
- **FM-009:** I can make a software or firmware update available to one or more specific robot units with no job currently assigned, so they stay on a consistent, known-good version without requiring a site visit; and for units with no connectivity to the update service, I can download a signed update package and hand it off to a site operator for local installation, so connectivity-impaired units can still receive updates in a controlled, auditable way.
- **FM-010:** Once a locally updated unit reconnects to the remote services, its reported software version is automatically reconciled with the central version registry, so I always have an accurate view of the entire fleet's update state.
- **FM-011:** I can review historical usage data, error logs, and print telemetry for any robot, so I can identify recurring issues and optimize operations over time.

### Developer / ML Engineer

- **DEV-001:** I can integrate support for a new robot type, generation, or piece of auxiliary equipment by implementing a well-defined interface and registering its capability profile, without modifying core platform logic.
- **DEV-002:** I can deploy the platform's Orchestration Tier to a customer's own infrastructure using a standard, self-contained installation package — including fully air-gapped environments — so the platform can be operated without depending on an external hosted service or requiring outbound internet access.
- **DEV-003:** I can build and publish a single signed update package from the CI/CD pipeline that is valid for all distribution paths — OTA delivery via the update service in cloud-hosted deployments, import into a customer-managed update service in on-premises deployments, and portable media transfer in either mode — so I don't maintain separate release processes for different deployment modes or connectivity scenarios.
- **DEV-004:** I can run hardware-in-the-loop tests against a simulated or real robot through a structured testing framework, so new features are validated before deployment and don't break existing behavior.
- **DEV-005:** I can deploy, monitor, and update AI/ML models — for example, for print quality assessment or anomaly detection — through a managed lifecycle pipeline, so models can improve over time with real operational data.
- **DEV-006:** I can view structured error and crash reports with sufficient context to reproduce and fix issues, so the feedback loop from field failures to software fixes is short.
- **DEV-007:** When troubleshooting any issue — hardware, motion, or operational — I can retrieve a correlated diagnostic record for any job that shows what was commanded, what the robot actually did, what decisions the motion controller made, what hardware state existed at the time, and what actions the operator took — so I can reconstruct the full sequence of events and diagnose root causes without needing physical access to the unit.
