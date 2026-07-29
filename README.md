<div align="center">

# System Design Interview Prep

**Production-minded high-level design case studies for senior backend, platform, cloud, and AI infrastructure interviews.**

[![Status](https://img.shields.io/badge/status-active-success)](#case-studies)
[![Focus](https://img.shields.io/badge/focus-HLD%20%7C%20Distributed%20Systems-blue)](#what-each-case-study-covers)
[![Diagrams](https://img.shields.io/badge/diagrams-Excalidraw%20%7C%20draw.io-orange)](#using-the-diagrams)
[![License](https://img.shields.io/badge/license-MIT-lightgrey)](LICENSE)

</div>

---

## About this repository

This repository contains structured system-design case studies built for interview preparation and engineering discussion.

Each design starts with requirements and scale, then progressively covers:

- APIs and data models
- High-level architecture
- Critical consistency boundaries
- Failure recovery
- Multi-tenancy and fairness
- Scaling and backpressure
- Observability
- Security
- Trade-offs
- Future evolution

The goal is not to present one universally correct architecture. The goal is to make assumptions explicit, identify the most important correctness boundaries, and explain why each design decision was made.

> [!NOTE]
> These are independent educational designs. They do not represent the internal architecture of any company.

---

## Case studies

| Status | Case study | Primary concepts | Artifacts |
|---|---|---|---|
| ✅ Complete | [ML Job Execution Platform](./ml-job-execution-platform/) | Scheduling, worker pools, transactional outbox, leases, retries, tenant fairness, GPU-aware autoscaling | README, Excalidraw, draw.io, SVG |
| 🟡 Planned | Visit-Once Link Service | Atomic consumption, concurrency, expiry, replay protection, multi-region consistency | — |
| 🟡 Planned | Secure Code Execution Sandbox | Isolation, microVMs, egress control, secrets, warm pools | — |
| 🟡 Planned | TinyURL with Analytics | Key generation, caching, hot keys, expiry, analytics | — |
| 🟡 Planned | Multi-Tenant Inference API | Streaming, dynamic batching, model routing, GPU utilisation | — |
| 🟡 Planned | Telemetry Platform | High-volume ingestion, indexing, retention, cardinality | — |

---

## Repository map

```text
system-design-interview-prep/
├── README.md
├── LICENSE
├── .gitignore
├── CONTRIBUTING.md
└── ml-job-execution-platform/
    ├── README.md
    ├── diagrams/
    │   ├── architecture.svg
    │   ├── reliability-flows.svg
    │   ├── ml-job-execution-hld.excalidraw
    │   └── ml-job-execution-hld.drawio
    └── docs/
        ├── api-contracts.md
        ├── data-model.md
        ├── failure-scenarios.md
        └── interview-walkthrough.md
```

---

## What each case study covers

<details>
<summary><strong>1. Requirements and assumptions</strong></summary>

- Functional requirements
- Non-functional requirements
- Capacity and latency assumptions
- Explicit scope boundaries
- Initial single-region design versus future evolution

</details>

<details>
<summary><strong>2. Architecture and correctness</strong></summary>

- Control plane versus data plane
- Source-of-truth decisions
- Transaction boundaries
- Idempotency
- At-least-once versus exactly-once semantics
- Concurrency control
- Reconciliation and failure recovery

</details>

<details>
<summary><strong>3. Scale and operations</strong></summary>

- Horizontal scaling
- Resource-aware autoscaling
- Backpressure
- Load shedding
- Observability
- Alerting
- Cost and utilisation trade-offs

</details>

---

## Using the diagrams

The case studies include multiple diagram formats:

- **SVG** — quick preview directly on GitHub
- **Excalidraw** — editable, presentation-friendly diagrams
- **draw.io** — detailed editable architecture diagrams

To edit:

1. Open `.excalidraw` files in Excalidraw.
2. Open `.drawio` files in diagrams.net.
3. Export updated previews as SVG.
4. Commit both the editable source and the rendered preview.

---

## Suggested study workflow

```mermaid
flowchart LR
    A[Read the prompt] --> B[Clarify requirements]
    B --> C[Estimate scale]
    C --> D[Define APIs and data]
    D --> E[Draw high-level architecture]
    E --> F[Deep-dive into correctness]
    F --> G[Cover failures and scaling]
    G --> H[Summarise trade-offs]
    H --> I[Repeat as a timed mock]
```

---

## Design review checklist

- [ ] Did I confirm the requirements before designing?
- [ ] Did I identify the authoritative data store?
- [ ] Did I define the acknowledgment boundary?
- [ ] Did I address idempotency and duplicate delivery?
- [ ] Did I explain concurrent updates?
- [ ] Did I cover partial failures?
- [ ] Did I address backpressure and overload?
- [ ] Did I define useful operational metrics?
- [ ] Did I state the main trade-offs?
- [ ] Can I summarise the design in two minutes?

---

## Planned additions

- [ ] Visit-once link service
- [ ] Secure multi-tenant code-execution sandbox
- [ ] TinyURL with analytics
- [ ] Multi-tenant inference service
- [ ] Telemetry and metrics platform
- [ ] Cross-case-study pattern index

---

## Contributing

Suggestions and design-review comments are welcome. See [CONTRIBUTING.md](CONTRIBUTING.md).

## Licence

Released under the [MIT Licence](LICENSE).
