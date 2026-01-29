---
author: Santanu Mukherjee
pubDatetime: 2026-01-20T15:20:35Z
title: Designing a Production-Grade Email System - From Architecture to Scale
modDatetime: 2025-06-09T07:42:54.791Z
featured: false
draft: false
tags:
  - backend
  - FastAPI
description: A deep dive into planning, building, deploying, and scaling a Gmail-like email system — including constraints, trade-offs, and real-world decisions.
---

Email systems look deceptively simple.

At a surface level, it feels like:

- receive emails
- store them
- display them in a UI

In reality, email platforms sit at the intersection of **distributed systems, data modeling, reliability, search, and business workflows**. Small architectural mistakes compound quickly once volume, concurrency, and product requirements increase.

This article walks through how I approached designing a **production-grade email system**, from planning to execution, while balancing real-world constraints.

---

## Problem Definition

The goal was to build an email platform that supports:

- inbound and outbound email
- conversation threading
- fast inbox rendering
- search and filtering
- spam handling
- push notifications
- future features like analytics and marketing automation

All while remaining:

- cost-aware
- scalable
- maintainable

This was **not** a toy project. It needed to behave like a real product.

---

## High-Level Architecture

At a high level, the system was divided into **three distinct layers**:

1. **Email Transport & Storage**
2. **Metadata & Indexing**
3. **Product & Business Pipelines**

Separating these early avoided tight coupling and allowed each layer to evolve independently.

---

## Planning the Data Model (The Most Important Step)

### Raw Emails vs Metadata

A critical early decision was to **separate raw email storage from structured metadata**.

#### Raw email storage:

- Original `.eml` files
- Immutable
- Stored cheaply
- Rarely accessed directly

#### Metadata:

- sender, recipients
- subject, timestamps
- thread relationships
- labels, flags, status
- search keywords

Trying to store everything in a single system creates unnecessary cost and complexity.

---

## Storage Strategy

### Raw Email Storage

Raw emails were stored as `.eml` files in object storage (e.g. S3).

Why object storage:

- extremely cheap
- durable
- scales automatically
- perfect for large binary blobs

Each email is written once and rarely modified.

**Immutability was a feature**, not a limitation.

---

### Metadata Storage

Metadata required:

- strong consistency
- complex querying
- transactional updates
- relational integrity

This ruled out document-only approaches early.

A relational database (PostgreSQL) was used for:

- users
- threads
- messages
- participants
- labels
- delivery status

This made threading, search, and filtering predictable and debuggable.

---

## Threading & Conversation Management

Threading is where many systems break.

Relying only on headers like `In-Reply-To` is not sufficient. Real-world email clients:

- drop headers
- modify subjects
- introduce new participants mid-thread

The solution combined:

- message headers
- normalized subject comparison
- participant overlap
- fallback thread creation

Threads became **first-class entities**, not just inferred relationships.

This made:

- inbox rendering faster
- conversation updates simpler
- search more accurate

---

## Inbound Email Flow

1. Email received via SMTP (SES)
2. Raw `.eml` stored in object storage
3. Lambda triggered asynchronously
4. Email parsed into structured metadata
5. Thread resolved or created
6. Metadata persisted transactionally
7. Notifications dispatched

Each step was:

- idempotent
- independently retryable
- observable via logs and metrics

Failures never blocked email ingestion.

---

## Outbound Email Flow

Outbound email followed a **mirrored but separate pipeline**.

This separation mattered.

Outbound emails:

- originate from users
- need delivery tracking
- require retries and bounce handling

Sent emails were:

- stored in the same raw format
- indexed similarly
- linked back into threads when appropriate

Inbound and outbound logic shared schemas — **not execution paths**.

---

## Search & Indexing

Search is often underestimated.

The system avoided full-text indexing initially and focused on:

- precomputed search keywords
- normalized fields
- indexed columns

This covered:

- subject search
- sender search
- participant search
- thread-level summaries

Advanced search engines can always be added later.  
Correct data modeling cannot.

---

## Performance Constraints

Early constraints shaped the design:

- mobile clients with limited memory
- large inboxes
- burst traffic
- cold starts

Optimizations included:

- paginated inbox queries
- thread-level summaries
- lazy loading email bodies
- push-based updates instead of polling

Inbox rendering became a **metadata problem**, not a content problem.

---

## Deployment Strategy

The system was deployed using:

- serverless functions for event-driven tasks
- managed database for transactional integrity
- object storage for raw data
- CDN for client delivery

Why serverless:

- predictable cost
- auto-scaling
- minimal ops overhead

Cold starts were mitigated by:

- small function size
- async-heavy design
- caching where appropriate

---

## Scaling Strategy

Scaling was planned in layers:

### Storage Scaling

- object storage scales automatically
- database scaled vertically first, horizontally later

### Read Scaling

- read-heavy endpoints optimized first
- caching at the query level
- pagination everywhere

### Write Scaling

- ingestion pipelines decoupled from user-facing APIs
- retries and queues handled spikes

The system scaled **by design**, not by reaction.

---

## Observability & Reliability

Every pipeline emitted:

- structured logs
- correlation IDs
- failure metrics

Failures were expected and handled:

- retries with backoff
- dead-letter queues
- alerts only for actionable issues

Silent failures are worse than loud ones.

---

## Marketing & Business Pipelines (Often Ignored)

Email platforms are not just communication tools — they are **business engines**.

The architecture allowed:

- tagging emails by source
- tracking campaign-related messages
- triggering workflows on specific events
- exporting metadata for analytics

This enabled:

- onboarding flows
- transactional email tracking
- marketing segmentation
- future CRM integrations

Building this later would have been painful.  
Building hooks early was cheap.

---

## Trade-Offs & Lessons Learned

No architecture is perfect.

Trade-offs made:

- more upfront planning
- higher schema discipline
- slightly slower initial iteration

What was gained:

- predictable scaling
- debuggability
- lower long-term cost
- confidence in production

If I had to rebuild it:

- I’d keep the same separation
- refine search earlier
- invest sooner in observability

---

## Final Thoughts

Email systems punish shortcuts.

They reward:

- clear boundaries
- boring reliability
- deliberate data modeling

This architecture wasn’t designed to impress —  
it was designed to **survive production**.

That, in the end, is the real benchmark.

---

If you’re building systems that need to last,  
design for clarity first — scale will follow.
