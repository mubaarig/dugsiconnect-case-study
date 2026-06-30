# dugsiconnect-case-study
Here is the complete README.md content:


# DugsiConnect

> A school communication and classroom management platform built for schools in Somaliland — replacing fragmented WhatsApp groups and manual paperwork with a structured, role-based, real-time school system.

**Built with React Native · Expo · TypeScript · Firebase (Auth, Firestore, Cloud Functions, Security Rules)**

---

> ⚠️ **Note on source code**
> DugsiConnect is an **active commercial product**, and its production source code is **private**. This repository is a **public engineering case study**. It documents the product, system architecture, data and security model, and the engineering decisions behind it — **without exposing any private code, Firebase configuration, secrets, security rules, customer data, or internal business logic.** All diagrams and descriptions here are illustrative and intentionally generalized.

---

## Table of Contents

1. [Product Overview](#product-overview)
2. [Problem](#problem)
3. [Solution](#solution)
4. [Tech Stack](#tech-stack)
5. [Core Features](#core-features)
6. [Architecture](#architecture)
7. [Data & Access Model](#data--access-model)
8. [Real-Time Messaging Architecture](#real-time-messaging-architecture)
9. [Security & Permissions](#security--permissions)
10. [Offline & Reliability](#offline--reliability)
11. [Testing](#testing)
12. [Engineering Challenges Solved](#engineering-challenges-solved)
13. [My Role](#my-role)
14. [Status](#status)
15. [Screenshots](#screenshots)

---

## Product Overview

DugsiConnect is a mobile-first platform that structures communication and classroom workflows between **school admins, teachers, parents, and students**. Each role gets a tailored experience — admins manage their school, teachers run their classes, and parents stay connected to their own child's progress — all backed by a multi-tenant Firestore data model with enforced, role-aware access control.

The product handles the day-to-day operations a school actually runs on: messaging, announcements, attendance, homework, grades, events, and class management — in a single app instead of a dozen disconnected WhatsApp groups.

---

## Problem

Schools in the target market typically coordinate through **WhatsApp groups, phone calls, and paper records**. This breaks down quickly at scale:

- **No structure or access control** — every parent in a group sees every message; there's no concept of "this parent, this child, this class."
- **Information is lost** — announcements, homework, and grades disappear in chat scroll-back.
- **No source of truth** — attendance and marks live in notebooks or scattered spreadsheets.
- **No tenancy or privacy boundaries** — sensitive student data is mixed with casual chat, with no enforcement of who can see what.
- **No accountability** — there's no reliable record of what was communicated, to whom, and when.

The core challenge is not "build a chat app" — it's enforcing **the right data boundaries for the right role across many independent schools**, reliably, on mobile networks that are frequently slow or intermittent.

---

## Solution

DugsiConnect replaces that fragmented workflow with a **structured, multi-tenant school platform**:

- A **multi-role access model** (admin / teacher / parent / student) where the UI *and* the backend both enforce what each role can do.
- A **multi-tenant data structure** that isolates each school's data so one school can never read or write another's.
- **Real-time messaging** scoped to legitimate relationships (teacher ↔ parent/student within a class).
- **Structured academic workflows** — attendance, homework, grades, announcements, and events — stored as first-class data rather than chat messages.
- **Offline-tolerant behavior** so the app stays usable on unreliable connections, with writes queued and reconciled on reconnect.

Access decisions are enforced at the data layer through **Firestore Security Rules** and **Cloud Functions**, not just hidden in the UI — so the boundaries hold regardless of client.

---

## Tech Stack

| Layer | Technology |
|---|---|
| **Mobile client** | React Native, Expo, TypeScript |
| **Authentication** | Firebase Authentication |
| **Database** | Cloud Firestore (real-time listeners) |
| **Authorization** | Firestore Security Rules + custom claims / role documents |
| **Backend logic** | Firebase Cloud Functions (event-driven & callable) |
| **Notifications** | Push notifications (Expo / FCM) |
| **Offline** | Firestore offline persistence + app-level queued writes |
| **Testing** | Jest, React Native Testing Library, Firebase Emulator Suite |
| **Language** | TypeScript end-to-end (client + Cloud Functions) |

---

## Core Features

**Administration**
- School admin dashboard — manage classes, staff, and school-wide settings
- Staff invites and onboarding
- Class management with **class join codes**
- School activity overview

**Teaching**
- Teacher dashboard scoped to assigned classes
- Attendance tracking
- Homework assignment and tracking
- Grades / marks entry
- Announcements to a class or the whole school

**Communication**
- Real-time teacher ↔ parent/student messaging
- Announcements feed
- Events and calendar
- Push notifications for messages, announcements, and updates

**Parent / Student**
- Parent view scoped strictly to their **own child's** data
- Student access to their classes, homework, grades, and announcements

**Platform**
- Role-based routing and navigation
- Role-based UI rendering
- Reported content / moderation hooks for messaging safety

> Feature availability per role is governed by the access model below.

---

## Architecture

DugsiConnect is a **client-driven, serverless architecture**. The React Native client talks directly to Firestore through real-time listeners for reads and writes, while Cloud Functions handle privileged, multi-document, or fan-out operations that must not be trusted to the client. Security Rules sit between the client and the data as the enforcement boundary.

```mermaid
flowchart TD
    subgraph Client["📱 React Native + Expo Client"]
        UI["Role-based UI<br/>(Admin / Teacher / Parent / Student)"]
        Router["Role-based routing"]
        Cache["Offline cache +<br/>queued writes"]
    end

    subgraph Firebase["☁️ Firebase Backend"]
        Auth["Firebase Auth<br/>(identity + role claims)"]
        Rules["Firestore Security Rules<br/>(authorization boundary)"]
        FS[("Cloud Firestore<br/>multi-tenant data")]
        CF["Cloud Functions<br/>(fan-out, invites, cleanup)"]
        Push["Push Notifications"]
    end

    UI --> Router
    Router --> Cache
    Cache <-->|"real-time listeners"| Rules
    UI -->|"sign in"| Auth
    Auth -->|"role / school claims"| Rules
    Rules <--> FS
    FS -->|"onWrite triggers"| CF
    CF -->|"privileged writes"| FS
    CF --> Push
    Push -.->|"deliver"| Client
Key principles

The client never trusts itself. The UI hides what a role can't do, but every read/write is independently authorized by Security Rules.
Cloud Functions own privileged work. Anything that crosses tenant boundaries, fans out to many recipients, or must be idempotent runs server-side.
Real-time by default. Listeners keep dashboards, messages, and announcements live without manual refresh.
Data & Access Model
Multi-tenancy
Every record is scoped to a school (tenant). School data is isolated so that no user from one school can read or write another school's classes, students, messages, or grades. School scope is derived from the authenticated user's identity/claims — never from a client-supplied value that the user could tamper with.

Roles
Role	Scope of access
Admin	Manages only their own school — classes, staff, settings, school-wide announcements
Teacher	Reads/writes data only for classes they're assigned to — attendance, homework, grades, messaging
Parent	Reads only their own child's data — grades, homework, attendance, announcements, and messaging with that child's teachers
Student	Reads their own classes, homework, grades, and announcements
Conceptual data shape

flowchart LR
    School["School (tenant)"]
    School --> Classes["Classes"]
    School --> Staff["Staff / Teachers"]
    School --> Students["Students"]
    School --> Announcements["Announcements"]
    School --> Events["Events"]
    Classes --> Enrollment["Enrollment<br/>(student ↔ class)"]
    Classes --> Homework["Homework"]
    Classes --> Attendance["Attendance"]
    Classes --> Grades["Grades / Marks"]
    Students --> Guardians["Parent ↔ Student links"]
    Classes --> Threads["Message threads"]
The model is deliberately built around relationships (teacher↔class, parent↔student, student↔class), because those relationships are exactly what the security layer checks when authorizing a read or write.

Real-Time Messaging Architecture
Messaging is the most read-heavy and latency-sensitive part of the system, so it's designed for cheap, bounded reads rather than naive queries.

Design goals

A parent's inbox should cost O(1) reads to load — not "scan every message in the school."
Message delivery should be idempotent — retries and reconnects must never duplicate a message.
Reads should stay bounded and predictable as a school grows.
Approach

Thread-scoped messages. Messages live under their conversation thread, and each participant has a lightweight thread summary (last message, unread count). Loading an inbox reads the participant's own thread index — independent of total message volume.
Fan-out on write via Cloud Function. When a message is sent, an onWrite-triggered Cloud Function fans the update out to each participant's thread index and triggers push notifications. The client does one cheap write; the function handles distribution server-side where it can be authorized and rate-controlled.
Deterministic document IDs for idempotency. Messages and thread documents use deterministic IDs derived from their stable inputs (e.g., participants + message identity). A retried or replayed write targets the same document instead of creating a duplicate — which makes the offline queue and network retries safe.
Efficient real-time reads. Clients subscribe to bounded, indexed queries (e.g., latest N messages in a thread, the user's own thread index) so a live conversation doesn't re-read history.

sequenceDiagram
    participant T as Teacher (client)
    participant FS as Firestore
    participant CF as Cloud Function (fan-out)
    participant P as Parent (client)

    T->>FS: write message (deterministic ID)
    FS-->>CF: onWrite trigger
    CF->>FS: update each participant thread index
    CF->>P: push notification
    FS-->>P: real-time listener delivers message
    Note over FS,P: Parent inbox = O(1) read of own thread index
Security & Permissions
Authorization is enforced at the data layer, so the rules hold no matter what client connects.

Firestore Security Rules as the source of truth. Every document type has explicit read/write conditions checked against the requester's identity, role, and school. The UI's role gating is a usability layer — the Security Rules are the actual boundary.
Tenant isolation. Rules verify that the requester belongs to the same school as the document. Cross-tenant access is rejected at the database, not filtered in the app.
Relationship checks.
A parent can read a student's record only if a verified parent↔student link exists.
A teacher can read/write class data only for classes they're assigned to.
An admin can manage resources only within their own school.
Server-enforced privileged operations. Staff invites, role assignment, join-code redemption, and fan-out run in Cloud Functions, so a client can't grant itself a role or write across boundaries.
Moderation hooks. Reported-content workflows give the platform a path to review and act on messaging abuse.
Actual rule definitions and configuration are part of the private repository and are intentionally not published here.

Offline & Reliability
The app is built for intermittent, low-bandwidth mobile networks, which are the norm in the target market.

Offline persistence. Firestore's local cache keeps the app readable and interactive without a connection; cached data hydrates the UI immediately on launch.
Queued writes. Actions taken offline (sending a message, marking attendance) are queued locally and reconciled on reconnect.
Idempotent, deterministic writes. Because writes target deterministic document IDs, replaying a queued write after a flaky connection is safe — it converges to the same document instead of creating duplicates.
Network-recovery awareness. The app handles reconnection gracefully, re-establishing listeners and flushing the write queue without user intervention.
Cleanup & archival. Background routines archive or clean up stale conversations and aged school data to keep active reads small and predictable over time.
Testing
Testing focuses on the two things most likely to break a school platform: business logic and authorization.

Unit & component tests — Jest and React Native Testing Library cover core logic, hooks, and role-based UI rendering.
Security Rules tests — the Firebase Emulator Suite is used to assert authorization behavior directly: a parent cannot read another child's grades, a teacher cannot write to an unassigned class, a user from School A cannot touch School B's data. These tests treat the rules as code under test, not as configuration.
Cloud Function tests — fan-out and privileged operations are validated against the emulator, including idempotency (replayed writes don't duplicate).
The emulator-based approach means access-control guarantees are verified automatically, not just reasoned about.

Engineering Challenges Solved
Enforcing a four-role, multi-tenant access model end-to-end — keeping UI gating, routing, Security Rules, and Cloud Functions consistent so the boundaries can't be bypassed by a crafted client.
Keeping parent inbox reads O(1) — a per-participant thread index so loading messages doesn't scale with total message volume.
Idempotent message delivery — deterministic document IDs so retries, reconnects, and queued offline writes never produce duplicates.
Safe fan-out — moving message distribution and notification into a Cloud Function so it's authorized and controlled server-side rather than trusted to clients.
Offline-tolerant workflows — making attendance, messaging, and other actions usable without a stable connection and reconciling them cleanly on reconnect.
Tenant isolation under shared infrastructure — guaranteeing that many independent schools coexist in one Firestore project without ever leaking across each other.
Bounded growth — archival/cleanup of stale data so read costs and performance stay predictable as schools accumulate history.
My Role
I am the Senior Frontend / React Native Developer behind DugsiConnect, responsible for the product end-to-end:

Designed and built the React Native + Expo + TypeScript mobile client, including role-based navigation and role-specific UI.
Designed the multi-tenant Firestore data model and the relationship structure that the access model depends on.
Authored the Firestore Security Rules and the role-based authorization strategy, and validated them with emulator-based tests.
Built the Cloud Functions powering message fan-out, staff invites, join-code redemption, and cleanup/archival.
Designed the real-time messaging architecture for bounded reads and idempotent delivery.
Implemented offline-tolerant behavior and network-recovery handling.
Set up push notifications and the testing strategy across client, rules, and functions.
Status
Stage: Active, in production use as a commercial product.
Source: Production code is private; this repository is a public case study only.
Roadmap (representative): expanded analytics/school activity reporting, richer moderation tooling, and continued performance/archival work as schools scale.
Screenshots
Screenshots are illustrative of the product experience. Sensitive data is redacted/mocked.

Screen	Preview
Admin dashboard	[Add screenshot here]
Teacher dashboard	[Add screenshot here]
Real-time messaging	[Add screenshot here]
Announcements feed	[Add screenshot here]
Attendance	[Add screenshot here]
Homework & grades	[Add screenshot here]
Parent view (single child)	[Add screenshot here]
Class join code / staff invite	[Add screenshot here]
This README documents the engineering behind a private production application. It intentionally omits source code, configuration, credentials, and customer data.



A few notes on choices I made:

- **Kept everything you flagged as "if relevant"** (push notifications, moderation, analytics, fan-out) but framed them as designed-in capabilities rather than overclaiming live metrics.
- **No invented URLs, names, metrics, or store links** — screenshots are placeholders, and Status avoids fake numbers.
- **Two Mermaid diagrams plus a sequence diagram** for messaging, since the fan-out + O(1)-read story is your strongest senior-level signal and reads better visually.
- **Security section explicitly says rules are private** — this reinforces good judgment to a reviewer rather than looking like an omission
