# Stratum Platform — System Requirements

---

## 1. Introduction

This document identifies and categorizes the system requirements derived from the Stratum Platform Specification. Requirements are organized by domain and tagged with a unique identifier for traceability.

**Requirement ID format:** `[DOMAIN]-[NUMBER]`

---

## 2. Functional Requirements

### 2.1 Project Planning & Toolpath Generation

**PLAN-001** The platform shall accept IFC files as the input format for building designs.

**PLAN-002** The platform shall allow an architect or project owner to upload an IFC file and configure print parameters from a remote device, without requiring the robot to be powered on.

**PLAN-003** The platform shall execute a toolpath generation pipeline that includes model parsing, slicing, robot motion planning, and machine code generation.

**PLAN-004** The platform shall notify the user when toolpath generation is complete.

**PLAN-005** The platform shall provide a 3D interactive simulation of the generated toolpath for remote review prior to on-site mobilization.

**PLAN-006** The platform shall allow operators to configure keep-out zones and known obstacles for a specific site and automatically incorporate them into toolpath planning.

**PLAN-007** The platform shall validate that an assigned robot has the hardware capabilities required to execute a job before dispatch.

---

### 2.2 Print Execution & Machine Control

**EXEC-001** The platform shall support creating, starting, pausing, resuming, and aborting print tasks from the operator interface.

**EXEC-002** The platform shall support creating and executing a ground scanning task prior to printing to collect surface data for compensation.

**EXEC-003** The platform shall allow operators to override printing speed and extrusion speed in real time during an active print task.

**EXEC-004** The platform shall detect layers that require lintel placement and pause execution, requiring operator confirmation before printing resumes.

**EXEC-005** The platform shall provide real-time tracking of active print job progress, including completion status and estimated time remaining.

**EXEC-006** The platform shall support machine homing through a guided workflow that prevents operators from skipping required steps.

**EXEC-007** The platform shall support an E-Stop mechanism that halts robot motion immediately.

---

### 2.3 Fleet Management

**FLEET-001** The platform shall allow a Fleet Manager to manually register robot units and auxiliary equipment, specifying each unit's generation and capability profile.

**FLEET-002** The platform shall provide a centralized dashboard displaying the status, current project, and alerts for all active robots across one or more construction sites.

**FLEET-003** The platform shall display the last known status of a robot that has lost cloud connectivity and alert the Fleet Manager when it comes back online.

**FLEET-004** The platform shall allow a Fleet Manager to manually assign print jobs and tasks to specific registered robots or auxiliary equipment.

**FLEET-005** The platform shall support onboarding new robot types, generations, and auxiliary equipment without requiring changes to core platform logic.

---

### 2.4 Software & Firmware Updates

**UPDATE-001** The platform shall support over-the-air (OTA) distribution of software updates to one robot unit or the entire fleet from a central admin console, without requiring a site visit.

**UPDATE-002** The platform shall support OTA updates for robot firmware and sensor module firmware, eliminating the need for a dedicated IDE or physical presence.

**UPDATE-003** The platform shall centrally track the software version running on every robot unit.

**UPDATE-004** The platform shall support local (offline) application of software and firmware updates via portable media (e.g. USB) for units at sites where cloud connectivity is unavailable or restricted.

**UPDATE-005** The platform shall allow a Fleet Manager to download a signed, versioned update package from the admin console for manual transfer to a site operator.

**UPDATE-006** All update packages — regardless of delivery method (OTA or offline) — shall be cryptographically signed, and the edge tier shall verify the signature before applying any update.

**UPDATE-007** When a robot unit that received an offline update reconnects to the cloud, the platform shall automatically reconcile its reported software version with the central version registry.

**UPDATE-008** The CI/CD pipeline shall produce a single update package artifact that is valid for both OTA and offline delivery, without requiring separate release builds.

---

### 2.5 System Health & Diagnostics

**HEALTH-001** The platform shall provide automated system health checks that an operator can run on any robot unit to confirm hardware readiness before a print job begins.

**HEALTH-002** The platform shall provide structured error and crash reports with sufficient context to reproduce and diagnose issues.

**HEALTH-003** The platform shall support centralized logging and monitoring of system events across all units.

---

### 2.6 AI/ML Model Lifecycle

**ML-001** The platform shall provide a managed pipeline for deploying, monitoring, and updating AI/ML models (e.g., print quality assessment, anomaly detection).

**ML-002** The platform shall collect operational telemetry and print data to support ongoing model training and improvement.

---

### 2.7 Testing & Development

**DEV-001** The platform shall provide a structured hardware-in-the-loop testing framework enabling developers to validate new features against a simulated or real robot before deployment.

**DEV-002** The platform shall support integration of new robot types or auxiliary equipment through a well-defined interface and capability profile registration, without modifying core platform logic.

---

## 3. Non-Functional Requirements

### 3.1 Availability & Offline Operation

**AVAIL-001** All safety-critical and print-control workflows — including motion execution, start/pause/resume, and local operator control — shall function fully offline without cloud connectivity.

**AVAIL-002** The edge tier shall cache active print jobs locally and sustain fully independent operation when the cloud uplink is unavailable.

**AVAIL-003** Cloud-dependent features (remote monitoring, fleet dispatch, OTA updates) shall degrade gracefully when connectivity is unavailable.

**AVAIL-004** The offline update workflow shall be operable entirely through the existing on-site operator interface without requiring cloud access at any step of the installation process.

---

### 3.2 Performance

**PERF-001** The local operator interface shall be responsive enough for real-time print control over the robot's dedicated radio access point.

**PERF-002** The toolpath generation pipeline shall support concurrent processing of multiple requests to avoid single-request bottlenecks.

---

### 3.3 Security

**SEC-001** All communication on the local network and over the cloud uplink shall be encrypted and protected against unauthorized access or interference.

**SEC-002** Role-based access control shall be enforced across the platform, with distinct permission sets for administrators, operators, and observers.

**SEC-003** Update packages applied via portable media shall be subject to the same cryptographic signature verification as OTA-delivered packages; the edge tier shall reject any package that fails verification.

---

### 3.4 Scalability & Extensibility

**SCALE-001** The platform architecture shall support fleet-level orchestration for multiple robots operating concurrently across one or more construction sites.

**SCALE-002** The hardware abstraction layer shall use a plugin-style design so that new robot types, generations, and auxiliary equipment can be integrated without core architectural redesign.

**SCALE-003** The platform shall model hardware using a configurable capability profile to support different generations of the same robot or equipment type.

---

### 3.5 Data Durability & Integrity

**DATA-001** The platform shall use a production-grade database with schema migration support and encrypted storage.

**DATA-002** Data stored at the edge shall be replicated to the cloud to prevent loss from local hardware failure.

---

### 3.6 Deployability & Provisioning

**DEPLOY-001** The software stack shall be containerized and deployed via a CI/CD pipeline, with container images pulled from a central registry.

**DEPLOY-002** Hardware provisioning shall be standardized so that each robot unit can be reproduced exactly and consistently, without manual configuration.

---

## 4. Interface Requirements

**IF-001** The operator interface shall be a responsive web or tablet application accessible both on-site via the robot's local WiFi and remotely via the cloud.

**IF-002** The platform shall provide role-specific interfaces tailored to the needs of architects, site operators, fleet managers, and developers.

**IF-003** The platform shall provide clear, actionable error messages to operators when something goes wrong, enabling issue resolution without escalation.

**IF-004** Each robot unit shall maintain a dedicated radio access point for reliable, low-latency local operator connectivity, with a secondary uplink for cloud and fleet communication.

**IF-005** The admin console shall provide a download mechanism for Fleet Managers to retrieve signed, versioned update packages for offline distribution to site operators.

---

## 5. Constraint Requirements

**CON-001** Robot registration into the platform shall be performed manually by a Fleet Manager. Automated discovery is out of scope for the initial release.

**CON-002** Job-to-hardware assignment shall be manual. Automated job dispatch is out of scope for the initial release.

**CON-003** The edge tier's local networking and real-time control path shall be independent of the cloud uplink to ensure low-latency, reliable operation.

**CON-004** Both OTA and offline update delivery paths shall produce an identical installed software state on the robot unit. Divergent behavior between delivery methods is not permitted.

---

## 6. Requirements Traceability Summary

| Requirement ID | Description | Source |
|----------------|-------------|--------|
| PLAN-001 | Accept IFC files as building design input | APO-001 |
| PLAN-002 | Remote IFC upload and parameter configuration without robot powered on | APO-001 |
| PLAN-003 | End-to-end toolpath generation pipeline | APO-001, APO-002 |
| PLAN-004 | Notify user when toolpath generation is complete | APO-002 |
| PLAN-005 | 3D simulation for remote review and approval | APO-002 |
| PLAN-006 | Keep-out zone and obstacle configuration incorporated into toolpath planning | APO-004 |
| PLAN-007 | Validate hardware capability before job dispatch | FM-004 |
| EXEC-001 | Create, start, pause, resume, and abort print tasks | SOP-001 |
| EXEC-002 | Ground scanning as a distinct task with its own queue lifecycle | SOP-003 |
| EXEC-003 | Real-time override of printing and extrusion speed | SOP-001 |
| EXEC-004 | Lintel layer detection with operator confirmation before resuming | SOP-002 |
| EXEC-005 | Real-time print progress tracking with estimated time remaining | APO-003 |
| EXEC-006 | Guided homing and pre-print workflow preventing skipped steps | SOP-004 |
| EXEC-007 | Emergency Stop to immediately halt all robot motion | SOP-009 |
| FLEET-001 | Manual robot and equipment registration with capability profile | FM-001 |
| FLEET-002 | Centralized fleet dashboard with status, project, and alerts | FM-002 |
| FLEET-003 | Last known status display and reconnection alert for offline robots | FM-003 |
| FLEET-004 | Manual job-to-hardware assignment | FM-004 |
| FLEET-005 | Onboard new robot types and equipment without core platform changes | FM-007 |
| UPDATE-001 | OTA software update distribution to one unit or entire fleet | FM-005 |
| UPDATE-002 | OTA firmware updates for motion controller and sensor module | FM-005 |
| UPDATE-003 | Centralized software version tracking across all units | FM-005, FM-009 |
| UPDATE-004 | Local offline update application via portable media | SOP-010, Architectural Assumption 7 |
| UPDATE-005 | Fleet Manager download of signed update package for offline distribution | FM-008 |
| UPDATE-006 | Cryptographic signature verification for all update packages | Architectural Assumption 7 |
| UPDATE-007 | Automatic version reconciliation after offline-updated unit reconnects | FM-009, Architectural Assumption 7 |
| UPDATE-008 | Single CI/CD artifact valid for both OTA and offline delivery | DEV-005 |
| HEALTH-001 | Automated pre-job system health checks | SOP-006 |
| HEALTH-002 | Structured error and crash reports for diagnosis | DEV-003 |
| HEALTH-003 | Centralized logging and monitoring of system events | FM-006, DEV-003 |
| ML-001 | Managed AI/ML model deployment, monitoring, and update pipeline | DEV-001 |
| ML-002 | Operational telemetry collection for model training | DEV-001 |
| DEV-001 | Hardware-in-the-loop testing framework | DEV-002 |
| DEV-002 | New hardware integration via well-defined interface and capability profile | DEV-004 |
| AVAIL-001 | Safety-critical and print-control workflows fully functional offline | Architectural Assumption 2 |
| AVAIL-002 | Edge tier caches jobs locally and operates independently offline | Architectural Assumption 2 |
| AVAIL-003 | Cloud-dependent features degrade gracefully when offline | Architectural Assumption 2 |
| AVAIL-004 | Offline update workflow operable without cloud access | SOP-010, Architectural Assumption 7 |
| PERF-001 | Operator interface responsive enough for real-time print control | SOP-008, Architectural Assumption 1 |
| PERF-002 | Toolpath generation supports concurrent requests | APO-001 |
| SEC-001 | All communication encrypted on local network and cloud uplink | SOP-008, Architectural Assumption 3 |
| SEC-002 | Role-based access control with distinct permissions per role | System Setup (Security) |
| SEC-003 | Portable-media update packages subject to same signature verification as OTA | Architectural Assumption 7 |
| SCALE-001 | Fleet-level orchestration for multiple concurrent robots | FM-002, FM-004 |
| SCALE-002 | Plugin-style hardware abstraction layer for extensibility | SOP-007, DEV-004 |
| SCALE-003 | Configurable hardware capability profiles for different generations | Architectural Assumption 6 |
| DATA-001 | Production-grade database with schema migration and encrypted storage | System Setup (Database) |
| DATA-002 | Edge data replicated to cloud to prevent loss from hardware failure | System Setup (Database) |
| DEPLOY-001 | Containerized software stack deployed via CI/CD pipeline | System Setup (Software Stack) |
| DEPLOY-002 | Standardized hardware provisioning for consistent, reproducible units | System Setup (Hardware) |
| IF-001 | Responsive web/tablet interface accessible on-site and remotely | SOP-001, APO-003 |
| IF-002 | Role-specific interfaces for each user type | System Setup (Application Tier) |
| IF-003 | Clear, actionable error messages for operators | SOP-005 |
| IF-004 | Dedicated radio access point with independent secondary cloud uplink | Architectural Assumption 1 |
| IF-005 | Admin console download mechanism for offline update package distribution | FM-008 |
| CON-001 | Robot registration is manual; automated discovery out of scope | Architectural Assumption 4 |
| CON-002 | Job dispatch is manual; automated assignment out of scope | Architectural Assumption 5 |
| CON-003 | Local control path independent of cloud uplink | Architectural Assumption 1 |
| CON-004 | OTA and offline delivery paths must produce identical installed state | Architectural Assumption 7 |
