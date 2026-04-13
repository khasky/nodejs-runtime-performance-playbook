# Node.js Runtime & Performance Playbook

Practical guide for Node.js APIs and workers: event-loop health, streaming and backpressure, memory, diagnostics, and production performance habits.

> *If I were running a serious Node.js API or worker today, what would I care about before I cared about micro-optimizing random lines of JavaScript?*

This playbook is about production realities:
- event-loop health;
- worker-pool pressure;
- streaming and backpressure;
- memory behavior;
- CPU hotspots;
- shutdown discipline;
- diagnostics you can actually use.

---

## Table of Contents

- [Node.js Runtime \& Performance Playbook](#nodejs-runtime--performance-playbook)
  - [Table of Contents](#table-of-contents)
  - [Companion playbooks](#companion-playbooks)
  - [Philosophy](#philosophy)
  - [The performance model that matters](#the-performance-model-that-matters)
  - [The defaults I'd reach for first](#the-defaults-id-reach-for-first)
  - [Event-loop rules](#event-loop-rules)
    - [Smells](#smells)
  - [Streams and backpressure](#streams-and-backpressure)
    - [My default view](#my-default-view)
    - [What usually goes wrong](#what-usually-goes-wrong)
  - [Memory and diagnostics](#memory-and-diagnostics)
    - [What I want available](#what-i-want-available)
    - [Signals I care about](#signals-i-care-about)
  - [Operational habits](#operational-habits)
    - [The boring production habits that matter](#the-boring-production-habits-that-matter)
    - [For jobs and workers](#for-jobs-and-workers)
  - [Common anti-patterns](#common-anti-patterns)
  - [Worth reading](#worth-reading)
  - [License](#license)

---

## Companion playbooks

These repositories form one playbook suite:

- [Auth & Identity Playbook](https://github.com/khasky/auth-identity-playbook) — sessions, tokens, OAuth, and identity boundaries across the stack
- [Backend Architecture Playbook](https://github.com/khasky/backend-architecture-playbook) — APIs, boundaries, OpenAPI, persistence, and errors
- [Best of JavaScript](https://github.com/khasky/best-of-javascript) — curated JS/TS tooling and stack defaults
- [Caching Playbook](https://github.com/khasky/caching-playbook) — HTTP, CDN, and application caches; consistency and invalidation
- [Code Review Playbook](https://github.com/khasky/code-review-playbook) — PR quality, ownership, and review culture
- [DevOps Delivery Playbook](https://github.com/khasky/devops-delivery-playbook) — CI/CD, environments, rollout safety, and observability
- [Engineering Lead Playbook](https://github.com/khasky/engineering-lead-playbook) — standards, RFCs, and technical leadership habits
- [Frontend Architecture Playbook](https://github.com/khasky/frontend-architecture-playbook) — React structure, performance, and consuming API contracts
- [Marketing and SEO Playbook](https://github.com/khasky/marketing-and-seo-playbook) — growth, SEO, experimentation, and marketing surfaces
- [Monorepo Architecture Playbook](https://github.com/khasky/monorepo-architecture-playbook) — workspaces, package boundaries, and shared code at scale
- **Node.js Runtime & Performance Playbook** — event loop, streams, memory, and production Node performance
- [Testing Strategy Playbook](https://github.com/khasky/testing-strategy-playbook) — unit, integration, contract, E2E, and CI-friendly test layers

---

## Philosophy

Most Node.js performance work is not about clever syntax tricks.

It is about:
- not blocking the event loop;
- not over-buffering work;
- not pretending memory issues are "just GC being weird";
- not reading huge payloads into RAM when a stream should exist;
- measuring before rewriting.

---

## The performance model that matters

Node.js gives you a powerful async runtime, but it is still very easy to create latency and throughput problems when CPU-heavy work, synchronous APIs, or uncontrolled concurrency pile up.

My baseline mental model:
- the **event loop** should stay free for short, fast coordination work;
- expensive or high-volume data movement should use **streaming** where appropriate;
- memory growth is a production signal, not an aesthetic concern;
- queue depth, latency, and error rates tell you more than "it worked locally".

---

## The defaults I'd reach for first

If I were building a production Node service today, I would usually start with:

- **Runtime:** current supported Node LTS policy for the repo
- **HTTP layer:** a framework that makes lifecycle hooks explicit
- **Observability:** logs + metrics + tracing from day one
- **Payload strategy:** streams for large files and long pipelines
- **Diagnostics:** repeatable profiling and memory investigation workflow
- **Shutdown:** explicit graceful termination path
- **Concurrency:** limits, pools, and backpressure-aware design

---

## Event-loop rules

These rules catch a surprising amount of pain:

1. **Never normalize blocking work as "fine because it's rare".**
2. **Treat synchronous filesystem / crypto / compression calls carefully.**
3. **Keep request handlers thin and short-lived.**
4. **Cap internal concurrency instead of trusting the universe.**
5. **Measure tail latency, not only averages.**

### Smells

- regexes or parsing work that spike CPU
- giant JSON serialization in hot paths
- retries without jitter or concurrency limits
- synchronous startup checks leaking into request paths
- one endpoint that allocates massive objects per request

---

## Streams and backpressure

Large payloads, exports, imports, uploads, ETL jobs, and proxying work should often be treated as **streaming problems**, not "read everything first" problems.

### My default view

- stream when data is large or unbounded;
- honor backpressure;
- avoid buffering entire files unless there is a clear business reason;
- prefer pipeline-style flows over hand-rolled event spaghetti.

### What usually goes wrong

- ignoring `false` from writable writes
- mixing flowing and paused stream assumptions blindly
- loading entire archives / CSVs / blobs into memory
- turning every stream into a `Buffer` "for convenience"

---

## Memory and diagnostics

Memory issues in Node are rarely solved by guessing.

### What I want available

- heap snapshots when needed
- CPU profiles for hot paths
- GC traces for suspected memory pressure
- request / job correlation to connect symptoms to workloads

### Signals I care about

- rising heap usage after steady-state traffic
- GC activity climbing with latency
- worker processes restarting due to memory pressure
- throughput collapsing during large batch jobs

---

## Operational habits

### The boring production habits that matter

- set timeouts intentionally;
- make outbound calls abortable;
- instrument queues, retries, and circuit-breaker style protections;
- cap body sizes and upload limits;
- separate request-serving concerns from background-job concerns;
- rehearse graceful shutdown and drain logic.

### For jobs and workers

- work should be idempotent where possible;
- retry rules should be explicit;
- poison messages should have a quarantine path;
- job payload size should stay bounded and documented.

---

## Common anti-patterns

- using Node as if it were a thread-per-request runtime;
- overusing in-process memory as a durable coordination mechanism;
- assuming "async" means "safe under unlimited concurrency";
- doing whole-file transforms in RAM when streams fit better;
- skipping diagnostics until the incident is already happening;
- treating performance as a one-time benchmark instead of an operational property.

---

## Worth reading

- [The Node.js event loop](https://nodejs.org/learn/asynchronous-work/event-loop-timers-and-nexttick)
- [Don't block the event loop](https://nodejs.org/learn/asynchronous-work/dont-block-the-event-loop)
- [Backpressuring in streams](https://nodejs.org/en/learn/modules/backpressuring-in-streams)
- [How to use streams](https://nodejs.org/learn/modules/how-to-use-streams)
- [Profiling Node.js applications](https://nodejs.org/learn/getting-started/profiling)
- [Tracing garbage collection](https://nodejs.org/learn/diagnostics/memory/using-gc-traces.html)

---

## License

MIT is a sensible default for a repository like this, but choose the license that fits how you want others to reuse the material.
