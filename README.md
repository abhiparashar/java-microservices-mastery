# Java Microservices Architecture Mastery

A phased, production-first roadmap that takes a mid-level Java developer to Staff/Principal-level
microservices architect — the person who is trusted to choose, not just to implement.

This is not a tutorial collection. There is no "hello world" service here. Every phase is built
around three things: **why a technique exists**, **how it fails in production**, and **how large
engineering organizations actually operate it**.

---

## Who this is for

| | |
|---|---|
| **You are** | 2–5 years of Java, comfortable with Spring Boot, have shipped a monolith |
| **You want** | Senior Backend Engineer → Microservices Architect → Staff → Principal |
| **You will get** | Design judgement, production instincts, interview evidence, a portfolio |
| **You will not get** | Certification prep, framework trivia, or a guarantee of a salary band |

The honest version: this roadmap builds capability and evidence. It does not substitute for
production experience at real scale, and good interviewers can tell the difference. The fix is in
Phase 13 — take the hardest work available where you are now, volunteer for on-call, drive a
migration.

---

## How this roadmap is different

- **Distributed systems first.** Phase 0 exists because most "microservices problems" are
  distributed-systems problems wearing a Spring costume. Skipping it produces architects who can
  draw boxes but cannot debug a cascading failure.
- **Hard concepts get the explanation ladder.** Every genuinely difficult idea — idempotency,
  write skew, backpressure, consensus, exactly-once, CPU throttling — is explained as:
  *Plain English → Analogy → How a product you use does it → Mechanics → What breaks.*
- **One running system.** Examples reuse the same marketplace, **ShopKart**
  (`catalog`, `search`, `cart`, `order`, `payment`, `inventory`, `shipping`, `pricing`,
  `notification`, `user`), so knowledge compounds instead of resetting every chapter.
- **"When NOT to use" is mandatory.** Every pattern states the condition under which it is the
  wrong choice. Most architecture damage comes from correct patterns applied to the wrong problem.
- **Big tech, with the caveat attached.** Netflix, Amazon, Uber, Stripe, Monzo, Shopify and others
  appear throughout — always split into *copy this* / *do not copy this at your scale*.
- **Projects are specified, not pre-built.** You get the spec, the acceptance criteria, and the
  measurement to produce. Writing the code is the learning; reading someone else's is not.
- **Current as of late 2026.** Spring Boot 4.x, Java 25 LTS, Kafka 4.x on KRaft, Kubernetes 1.37,
  Gateway API, ambient mesh, Micrometer Tracing. The dead things (Sleuth, Hystrix, Ribbon, Zuul 1)
  are covered only as migration targets, because you will meet them in legacy code.

---

## The 14 phases

> **Publication status.** This roadmap is written and published one phase at a time, so the
> content stays dense rather than padded. **Published:** Phases 0 through 12, the tech
> radar, the ADR template, this index, and the tracker. Links to unpublished phases below resolve
> once that phase lands — the schedule, dependency graph, and exit criteria for all 14 are already
> final.

| Phase | Weeks | Topic | Core output |
|---|---|---|---|
| [0](phases/phase-00-distributed-systems-foundations.md) | 1–4 | Distributed systems foundations | Failure-mode fluency, latency math |
| [1](phases/phase-01-service-design-and-ddd.md) | 5–10 | Service design, DDD, boundaries | Context map + decomposition dossier |
| [2](phases/phase-02-spring-boot-production-core.md) | 11–17 | Spring Boot production core | Golden-path service template |
| [3](phases/phase-03-communication-and-apis.md) | 18–23 | Communication, APIs, contracts | Versioned API + idempotency layer |
| [4](phases/phase-04-data-and-consistency.md) | 24–30 | Data ownership and consistency | Saga + outbox + reconciliation |
| [5](phases/phase-05-events-and-streaming.md) | 31–36 | Events, messaging, streaming | Event backbone with replay tooling |
| [6](phases/phase-06-resilience-engineering.md) | 37–41 | Resilience engineering | Resilience harness + game day |
| [7](phases/phase-07-security-and-compliance.md) | 42–47 | Security, identity, compliance | Zero-trust security baseline |
| [8](phases/phase-08-observability-and-operations.md) | 48–53 | Observability and operations | SLOs, burn-rate alerts, runbooks |
| [9](phases/phase-09-containers-kubernetes-cloud.md) | 54–60 | Containers, Kubernetes, cloud | Zero-downtime deploy proof |
| [10](phases/phase-10-delivery-and-platform-engineering.md) | 61–65 | Delivery and platform engineering | GitOps pipeline + canary analysis |
| [11](phases/phase-11-testing-strategy.md) | 66–70 | Testing strategy | Contract tests replacing E2E gates |
| [12](phases/phase-12-performance-scale-and-cost.md) | 71–77 | Performance, scale, cost | 10x throughput case study |
| [13](phases/phase-13-architecture-leadership-and-migration.md) | 78–90 | Architecture leadership, migration | Migration programme + ADR portfolio |

Full schedule, pacing options, and exit criteria: **[ROADMAP.md](ROADMAP.md)**.

---

## Repository layout

```
README.md                     you are here
ROADMAP.md                    week-by-week schedule, cadences, milestones, exit criteria
phases/                       the 14 phase guides — the core of the roadmap
projects/
  small-projects.md           24 focused builds (1–5 days each), specs only
  large-projects.md           12 system-scale builds (2–8 weeks each), specs only
  project-rubric.md           how portfolio work is judged; README and demo templates
interview/
  system-design-playbook.md   a repeatable method + 25 worked problems
  microservices-question-bank.md  150+ Q&A with follow-ups and weak-answer tells
  coding-and-design-drills.md 30 implementation drills (rate limiter, saga, outbox, ...)
  career-leveling-and-negotiation.md  levels, loops, behavioural stories, negotiation
reference/
  big-tech-playbooks.md       15 company playbooks: copy this, not that
  anti-patterns-and-war-stories.md  35 anti-patterns + 12 incident narratives
  best-practices-checklists.md      20 review and launch checklists
  tech-radar-2026.md          adopt / trial / assess / hold, plus the dead list
  reading-and-resources.md    books, papers, blogs, and a 2-hour weekly ritual
  glossary.md                 ~150 terms in plain language
  adr/                        ADR template + 20 worked architecture decisions
progress/tracker.md           checkboxes for every phase, project, and milestone
```

---

## How to actually use this

1. **Do not read it front to back.** Read a phase, then build that phase's project, then return
   for the interview drilldown. Reading without building produces confident wrongness.
2. **One phase at a time, in order.** The dependencies are real: Phase 4 (consistency) is
   unlearnable without Phase 0, and Phase 12 (performance) is meaningless without Phase 8
   (observability).
3. **A phase is not done until its exit criteria are checked.** Each phase ends with an
   observable self-assessment. "I read it" is not a criterion.
4. **Measure everything you build.** A project without a number — p99, throughput, cost per 1k
   requests, failed-request count during a rolling deploy — is a tutorial, not evidence.
5. **Write the decision down.** Every large project ships with ADRs. A folder of real ADRs is the
   most credible artifact you can bring to a Staff interview.
6. **Track it.** Use [progress/tracker.md](progress/tracker.md).

### Fast paths

| If you need | Read this first |
|---|---|
| An interview in 6 weeks | [system-design-playbook](interview/system-design-playbook.md) → [question bank](interview/microservices-question-bank.md) → Phases 4, 6, 8 |
| To fix a system that is on fire | Phase 6 → Phase 8 → [anti-patterns](reference/anti-patterns-and-war-stories.md) |
| To split a monolith | Phase 1 → Phase 4 → Phase 13 migration playbook |
| To decide if you even need microservices | Phase 1, "when NOT to use" → [big-tech-playbooks](reference/big-tech-playbooks.md), the 2023–2026 correction |
| To level up to Staff | Phase 13 → [career leveling](interview/career-leveling-and-negotiation.md) → [ADR catalog](reference/adr/ADR-CATALOG.md) |

---

## Technical baseline

Java 25 LTS · Spring Boot 4.1.x · Spring Framework 7 · Spring Cloud 2025.1.x · Jakarta EE 11 ·
Kafka 4.x (KRaft) · Kubernetes 1.37 · Gateway API v1.6 · OpenTelemetry / Micrometer Tracing ·
PostgreSQL 18 · Resilience4j · Testcontainers · Argo CD

Verified 2026-09-12. Versions drift; principles do not. Re-check
[reference/tech-radar-2026.md](reference/tech-radar-2026.md) quarterly.

---

## A note on scope

Projects in this roadmap are **specified, never implemented here**. You get the problem, the
constraints, the architecture sketch, the acceptance criteria, and the measurement to produce.
That is deliberate: handing over a finished repository teaches nothing that survives an interview.
