# PillMaxxer — System Design Document
 
**Status:** Draft v0.1 — living document, will expand as design decisions are made.
**Scope:** Full target system, not MVP-only. Sections will be flagged where MVP scope differs from the long-term design.

## 1. Overview

PillMaxxer is a medication adherence and care-coordination platform. It is a
multi-role system connecting **Patients**, **Providers**, and **Caregivers** around a
shared, permission-scoped view of a patient's medication schedule and adherence.

The system is a portfolio/learning project, not a production healthcare product. It is
designed *as if* it needed to respect patient data privacy (least-privilege access,
auditability, explicit consent for data sharing), but does not claim regulatory
compliance (e.g. HIPAA).

Full functional and non-functional requirements live in the companion [**Requirements Document**](requirements.md); this document covers architecture and design.

## 2. Context
 
4 actor types interact with the system:
 
- **Patient** — owns their medication schedule and adherence data, and controls all access to it.
- **Provider** — self-declared at registration (no license verification in this project); prescribes to patients who've granted access.
- **Caregiver** — linked to one or more patients to monitor adherence, at a patient-chosen visibility level.
- **Admin** - Platform-level access; TODO - determine exact capabilities of the admin account - NOT implemented in MVP

The system's core workflows (registering, connecting patients to providers/caregivers, prescribing medication, logging doses, checking drug interactions, and notifying the right people) all revolve around one invariant: **every piece of patient data is reachable only through an explicit, revocable, permission-scoped relationship to that patient.**

---

## 3. Architectural Approach: Monolith vs. Microservices
 
### 3.1 The tradeoff
 
Microservices decompose a system into independently deployable services, each owning its own data store, communicating over well-defined APIs and/or events. Compared to a monolith, this buys:
 
- Independent scaling of features with different performance requirements (e.g. scaling Notifications separately from the core domain)
- Independent deployability (ship one service without redeploying everything)
- Fault isolation (one service degrading doesn't necessarily take down the rest)

It costs:
 
- **Distributed transactions.** A workflow that spans services (e.g. creating a prescription, generating a dose schedule, checking interactions, and firing alerts) is no longer one ACID transaction — it needs extra distributed-systems engineering to keep data consistent
- **Operational overhead.** Multiple databases, multiple deployables, distributed tracing, more complex local dev
- **Premature boundaries.** Splitting services before we understand where the real seams are tends to produce either (a) chatty, tightly-coupled "distributed monoliths" that share data models across network calls, or (b) boundaries that have to be redrawn later anyway.


### 3.2 Decision: modular monolith
 
PillMaxxer is built as a **modular monolith**:
 
- A single Spring Boot deployable.
- A single PostgreSQL database.
- The codebase is organized into modules along clear bounded contexts:
  - **Identity & Access** — accounts, credentials, JWT issuance/verification
  - **Connections** — connection requests between patients/providers/caregivers, approvals, revocation, permission levels
  - **Prescriptions** — prescription creation (manual/OCR), drug interaction checking via RxNav
  - **Doses & Scheduling** — generating dose instances from a prescription's schedule, dose logging, the missed-dose background job
  - **Adherence & Gamification** — progress tracking, streaks, points, leaderboard
  - **Reporting** — adherence reports and trend analysis for providers/caregivers, scoped by their permission level
  - **Notifications** — in-app and email notifications across all the triggers in the system
- Each module owns its own entities and repositories. A module never reaches into another module's repository or entities directly — if Prescriptions needs to know something about a user, it calls Identity & Access's service interface (e.g. `IdentityService.getUserById(...)`), not `UserRepository` directly.

## 3.3 Cross-module communication: service layer only, never controllers
 
Each module follows the standard layering: Controller (HTTP concerns — request/response mapping, status codes) -> Service (business logic; this is the module's actual public API) -> Repository (persistence, package-private to the module).
 
**A module only calls another module's service layer, never its controllers or repositories.** Controllers exist to adapt HTTP to the service layer for external clients. Routing an internal call through a controller would mean coupling one module's business logic to another module's request/response DTOs and HTTP-specific concerns for no reason. The service layer is the only thing meant to be depended on from outside the module.
 
Concretely, each module exposes its service as an interface (e.g. `IdentityService`), and other modules depend on that interface via dependency injection rather than the concrete implementation, and it keeps each module trivially mockable in tests for the modules that depend on it.

**Entities don't cross the boundary either.** A service method should never return its own `@Entity` type to a caller in another module, this would leak database specifics. Services return small, purpose-built DTOs/records representing exactly what the caller needs.