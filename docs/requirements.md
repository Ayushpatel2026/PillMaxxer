# PillMaxxer — Requirements Document

## 1. Overview

PillMaxxer is a medication adherence and care-coordination platform. It is a
multi-role system connecting **Patients**, **Providers**, and **Caregivers** around a
shared, permission-scoped view of a patient's medication schedule and adherence.

The system is a portfolio/learning project, not a production healthcare product. It is
designed *as if* it needed to respect patient data privacy (least-privilege access,
auditability, explicit consent for data sharing), but does not claim regulatory
compliance (e.g. HIPAA).

## 2. Actors / Roles

| Role | Description |
|---|---|
| **Patient** | Owns their medication schedule and adherence data. Has full control over who else can see it. |
| **Provider** | A self-declared role at registration (no license verification in this project). Prescribes medication to patients who've granted them access. |
| **Caregiver** | A person (e.g. family member) linked to one or more patients to monitor adherence. |
| **Admin** | Platform-level access; TODO - determine exact capabilities of the admin account |

A single user account has exactly one role, chosen at registration.

## 3. Functional Requirements

### 3.1 Account & Identity (all roles)
- FR-1: Users register with email + password.
- FR-2: Email verification is required before the account can be used.
- FR-3: Users authenticate via email/password and receive a session (JWT access + refresh token).
- FR-4: Users can log out (refresh token is revoked).
- FR-5: Password reset via an emailed, time-limited token.

### 3.2 Connections Between Roles
- FR-6: A patient can search for and send a connection request to a provider (by name/email).
- FR-7: A provider can search for and send a connection request to a patient (by name/email).
- FR-8: A patient must explicitly approve any provider connection request before that provider can see any of the patient's data or prescribe to them.
- FR-9: Provider access is **all-or-nothing**: once approved, the provider can see the patient's full prescription list and adherence data.
- FR-10: A patient can search for and send a connection request to a caregiver (by email).
- FR-11: A caregiver cannot search for patients in the app - they are only connected to patients that explictly send them an email.
- FR-12: Caregiver access is **granular**: the patient chooses one of two visibility levels when approving: *adherence stats only* (counts of taken/missed doses + medication name + no notifications of drug interactions), or *adherence stats + full prescription list + dosage schedule + notifications of drug interactions*.
- FR-13: Connection requests generate both an in-app notification and an email notification to the recipient (patient). Approval/denial action must happen in app after login.
- FR-14: A patient can revoke a provider's or caregiver's access at any time; revocation takes effect immediately (subsequent requests from that user are rejected at the authorization layer, not just hidden in the UI).
- FR-15: Any approval, denial or revocation of access generates an in-app notification and email for the caregiver/provider whose access permissions were modified. 
- FR-16: A patient can view the current list of everyone who has access to their data and at what level.

### 3.3 Prescriptions & Medication Schedule
- FR-17: A provider can create a prescription for a patient who has granted them access.
- FR-18: A patient can also self-report a prescription/medication they're taking that wasn't issued by a connected provider (e.g. OTC, supplement, or a med from a provider not on the platform).
- FR-19: When a prescription is created, individual scheduled dose instances are generated for its duration.
- FR-20: When a prescription is created, this generates a full dosage schedule (frequency, times, duration).
- FR-21: Prescriptions can be entered manually through a form.
- FR-22: Prescriptions can be entered through Image-scan (OCR).
- FR-23: When a prescription is created (by provider or patient), the system checks for drug interactions against the patient's other active medications, using RxNav (https://rxnav.nlm.nih.gov/REST).
- FR-24: Only the party entering the prescription (provider or patient) sees the full interaction check result at entry time. Caregivers never see full interaction-check detail.
- FR-25: If a critical interaction is detected, the connected provider(s) + caregiver(s) with the full access permissions for that patient and the patient receive an alert notification.

### 3.4 Dose Logging & Adherence (Patient)
- FR-26: Each scheduled dose instance has a status: `PENDING`, `TAKEN`, or `MISSED`.
- FR-27: A patient can log a dose as taken. They cannot log a dose before its scheduled time.
- FR-28: A patient can log a dose retroactively, up to 48 hours after its scheduled time.
- FR-29: If a dose is not logged as taken within a configurable grace period after its scheduled time (default: 2 hours), it is automatically transitioned to `MISSED` by a background process.
- FR-30: The patient has a progress tracker showing adherence over a selected window: today / this week / 30 days / all-time. 
- FR-31: The progress tracker will updated in real-time via WebSocket push
- FR-32: Patients earn points and rewards for adherence streaks (consecutive on-time doses).
- FR-33: Patients can view a points leaderboard of other patients using the app.

### 3.5 Notifications
- FR-34: Patients receive a notification when a dose is due, for connection requests from Caregivers and Providers and for critical drug interactions. 
- FR-35: Providers receive notifications only for: critical drug interactions, and periodic (weekly/monthly) adherence digest reports for their patients. Providers do **not** get per-missed-dose alerts.
- FR-36: Caregivers receive notifications when something is "wrong" with a linked patient's adherence. This is configurable by the caregiver - e.g. any single missed dose vs. a pattern like two consecutive misses 

### 3.6 Reporting
- FR-37: Providers and caregivers can view an adherence report for each patient they have access to, scoped to their permission level (§3.2).
- FR-38: Basic report content: count of taken doses, count of missed doses, over a selectable window.
- FR-39: The report will also contain trend analysis (e.g. adherence improving/declining over time).

### 3.7 Admin *(Post-MVP)* TODO
- FR-40: Admin can view all accounts.
- FR-41: Admin can deactivate an account.

## 4. Non-Functional Requirements

### 4.1 Security
- NFR-1: All authentication uses a custom-implemented JWT scheme (access token + rotating refresh token), not a third-party auth provider.
- NFR-2: Passwords are hashed (BCrypt or equivalent); never stored or logged in plaintext.
- NFR-3: Authorization is enforced server-side at the API layer for every request.
- NFR-4: Authorization for provider/caregiver access is *relationship-scoped*: a provider or caregiver can only access data for patients who have explicitly granted them access, and only at the granted permission level.
- NFR-5: All inter-service and client-service traffic assumed to run over HTTPS in any deployed environment.
- NFR-6: Sensitive actions (granting/revoking access, viewing patient data) are logged for audit purposes.

### 4.2 Reliability & Data Integrity
- NFR-7: The missed-dose detection job must be idempotent — re-running it must not double-process or corrupt already-resolved dose instances.
- NFR-8: Revoking a connection must take effect atomically from an authorization standpoint. There should be no window where a revoked user can still read data.

### 4.3 Observability
- NFR-9: Each service produces structured logs.
- NFR-10: A correlation/request ID is generated at the API Gateway and propagated through all downstream service calls for a given request, to support tracing a request across services.

### 4.4 Scalability / Maintainability
- NFR-11: The system is built as a modular monolith with clear bounded-context boundaries, organized so that if it makes sense for a module to be extracted into an independently deployable service with its own database in the future, it is easy to do so without any major problems. 

## 5. Technical Constraints / Decisions

- Backend: Java, Spring Boot, modular monolith to start, fronted by an API Gateway.
- Database: PostgreSQL, one database per service.
- Frontend: Angular, single web app with role-based views/dashboards.
- Containerization: Docker + Docker Compose for local/dev; deployment target is AWS.
- External integration: RxNav REST API for drug interaction data.
- Notifications (MVP): email (e.g. via an SMTP sandbox provider) + in-app; no message broker yet.
- Notifications (post-MVP): introduce a message broker (RabbitMQ or Kafka) for the various different kinds of notifications; WebSocket for real-time UI push.

## 6. MVP Scope

**Included in MVP:**
- All of §3.1 — FR-1 to FR-5 (accounts, auth, email verification)
- All of §3.2 — FR-6 to FR-16 (connections: request/approve/deny, all-or-nothing provider access, granular caregiver access, revocation, notifications on request/approval/revocation, access list view)
- §3.3 — FR-17, FR-18, FR-19, FR-20, FR-21, FR-23, FR-24, FR-25 (provider-issued and self-reported prescriptions, dose instance + schedule generation, manual entry, drug interaction checking and scoped visibility, critical interaction alerts)
  - Excluded: FR-22 (OCR entry)
- §3.4 — FR-26, FR-27, FR-28, FR-29 (dose status enum, logging taken doses, retroactive logging, automatic MISSED transition)
  - FR-30 (progress tracker) partial: a basic, non-adjustable "current state" progress view is in MVP; the today/week/30-day/all-time window selector is post-MVP
  - Excluded: FR-31 (WebSocket real-time push), FR-32 (streaks/rewards), FR-33 (leaderboard)
- §3.5 — FR-34, FR-35 (patient and provider notifications, full scope)
  - FR-36 partial: MVP supports a single fixed trigger ("any missed dose"); configurable/pattern-based triggers (e.g. two consecutive misses) are post-MVP
- §3.6 — FR-37, FR-38 (report access for providers/caregivers, basic taken/missed counts over a selectable window)
  - Excluded: FR-39 (trend analysis)
- All of §4 (non-functional requirements) — NFR-1 to NFR-11
- All of §5 (technical constraints), except the post-MVP notes already marked within it (message broker, WebSocket push)

**Explicitly excluded from MVP:**
- FR-22 — OCR prescription entry
- FR-30 (partial) — adjustable progress-tracker windows
- FR-31 — real-time WebSocket push
- FR-32 — streaks and rewards
- FR-33 — patient leaderboard
- FR-36 — configurable/pattern-based missed-dose alerting for caregivers
- FR-39 — adherence trend analysis
- FR-40, FR-41 — Admin role (§3.7)
- Message broker (RabbitMQ/Kafka) and WebSocket push — §5, post-MVP notification architecture
- Pill count tracking / refill reminders — not yet assigned an FR number; to be specified in the post-MVP requirements pass

*Note: requirements for all excluded items above are intentionally left at their current level of detail and will be fully fleshed out in a post-MVP requirements review.*

**Definition of Done for MVP:** a working frontend app and the core microservices deployed to AWS.