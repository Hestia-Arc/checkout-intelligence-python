# Architecture Options and Decisions

**Status:** Proposed for review  
**Project:** Checkout Intelligence  
**Cost constraint:** Use free tools and services during development and portfolio demonstration. Avoid mandatory paid dependencies.  
**Purpose:** Compare architecture options against the documented business problem, functional requirements, non-functional requirements, and project constraints before implementing the application.

---

## 1. Architectural objectives

The architecture must support the following objectives:

1. Collect checkout journey events from a merchant's store through a defined integration interface.
2. Preserve event integrity and reconstruct checkout journeys reliably.
3. Calculate checkout metrics using deterministic, testable logic.
4. Identify potential checkout problems and present evidence for investigation.
5. Add AI-assisted explanations and recommendations without allowing AI to invent or replace authoritative metrics.
6. Allow different merchant applications to consume Checkout Intelligence without accessing its internal modules or database directly.
7. Provide a separate demo storefront, reference dashboard, and product landing page.
8. Run locally with free development tools and be suitable for deployment on free-tier services when practical.
9. Remain understandable and testable for a solo developer while allowing future extension.

These objectives are based on the proposed project requirements. They are not proof that real merchants have validated the product's business value.

## 2. System boundary

Checkout Intelligence is an independent, merchant-integratable system. It is not software that must be built inside a merchant's existing dashboard.

The merchant's store remains responsible for its own cart, checkout, payment processing, and authoritative order state. Checkout Intelligence analyzes the checkout information made available to it through supported integration interfaces.

The project will also have its own reference dashboard to demonstrate its analytical capabilities. This dashboard is a consumer of Checkout Intelligence, not the merchant's actual dashboard.

The product landing page introduces the product and explains its value. It is a separate concern from checkout analytics.

### Conceptual context

```text
                    Merchant's environment
             +--------------------------------+
             | Merchant Store                 |
             | Cart / Checkout / Orders       |
             +---------------+----------------+
                             |
                   Integration interface
                             |
                             v
+------------------------------------------------------+
|                 Checkout Intelligence                |
|                                                      |
| Event Intake -> Event Storage -> Journey Analysis    |
|                                  |                   |
|                                  v                   |
|                         Metrics and Findings         |
|                                  |                   |
|                                  v                   |
|                       AI-assisted Interpretation     |
|                                                      |
|                 Documented API                       |
+-------------------------------+----------------------+
                                |
                 +--------------+--------------+
                 |                             |
                 v                             v
       Merchant's existing              Project's reference
       application/dashboard             dashboard

       Separate demo storefront generates simulated
       checkout activity through the same interface.

       Separate landing page explains the product.
```

This diagram describes logical responsibilities, not a final deployment topology. The precise integration mechanism remains undecided.

## 3. Architecture option A: modular monolith versus microservices

### Option A1 — Modular monolith

One backend application, deployed as a single service, contains clearly separated modules with explicit responsibilities and interfaces.

Potential modules include:

- Event ingestion and validation
- Event persistence
- Checkout journey reconstruction
- Metric calculation
- Finding generation
- AI-assisted interpretation
- API delivery
- Configuration and operational concerns

These are candidate logical modules. Their final boundaries will be refined when the domain model and event contract are designed.

**Advantages**

- Lower hosting and operational complexity.
- Easier local development using Docker and a Python virtual environment.
- Simpler debugging and integration testing.
- Clear internal boundaries without requiring network calls between every module.
- Suitable for a solo developer and an initial portfolio system.

**Disadvantages**

- Modules share a deployment lifecycle.
- Poor internal boundaries can create tight coupling.
- Scaling one workload independently is less straightforward.

### Option A2 — Microservices

Separate services own different responsibilities and communicate through network interfaces or messaging.

**Advantages**

- Independent deployment and scaling.
- Stronger operational separation where different workloads have substantially different needs.
- Potentially useful for larger teams and mature systems.

**Disadvantages**

- More deployments, configuration, network failures, observability requirements, and integration testing.
- Additional infrastructure may be incompatible with our free-service constraint.
- Distributed processing would introduce complexity before the project's actual workload is known.

**Proposed decision:** Start with a modular monolith.

The application will have meaningful internal boundaries, but we will not introduce separate deployable services simply to make the architecture look enterprise-grade.

Revisit this decision only when a concrete requirement justifies independent deployment, scaling, ownership, or failure isolation.

## 4. Architecture option B: synchronous processing versus background processing

### Option B1 — Synchronous ingestion and processing

The API receives an event, validates it, persists it, and returns a response. More expensive analysis can happen when the merchant requests analytics.

**Advantages**

- Fewer infrastructure dependencies.
- Easier to implement and test.
- Straightforward local execution.
- No queue or worker needs to be operated initially.

**Disadvantages**

- A request may become slow if too much work is performed before returning.
- Large workloads may eventually exceed request timeouts.
- Retrying requests requires carefully designed event identity and idempotency rules.

### Option B2 — Queue and background worker

The API accepts an event and places work onto a queue. A worker processes the event asynchronously.

**Advantages**

- Separates event acceptance from downstream processing.
- Supports retries and independent worker scaling.
- Useful when processing is expensive or event volume is high.

**Disadvantages**

- Requires queue infrastructure and worker lifecycle management.
- Introduces eventual consistency, retry policies, duplicate delivery, dead-letter handling, and monitoring requirements.
- Free hosted infrastructure may constrain the available choices.

**Proposed decision:** Start with synchronous event validation and persistence. Keep analytics separate from event intake.

We will not add a queue until measured latency, throughput, reliability requirements, or a demonstrated workflow justifies it. The architecture must not assume that future asynchronous processing will be unnecessary.

Even with synchronous ingestion, event identity, duplicate handling, and failure behavior must be designed explicitly.

## 5. Architecture option C: one relational database versus a separate analytics store

### Option C1 — PostgreSQL as the initial source of truth

Store accepted checkout events and the initial operational records in PostgreSQL. Calculate the initial funnel metrics from the persisted data.

**Advantages**

- Open-source and available through Docker.
- Supports transactions, constraints, indexes, and relational queries.
- One database is easier to develop, test, back up, and explain.
- Suitable for a controlled portfolio dataset.

**Disadvantages**

- Complex analytics may eventually compete with ingestion for database resources.
- Large event volumes may require partitioning, pre-aggregation, or a specialized analytical store.

### Option C2 — Separate operational and analytical databases

Use one database for operational records and another optimized for analytics.

**Advantages**

- Can isolate workloads and optimize large analytical queries.
- May suit higher-volume or more demanding reporting workloads.

**Disadvantages**

- Additional cost and operational complexity.
- Data synchronization and consistency become more complicated.
- The project's current scale has not demonstrated a need for this arrangement.

**Proposed decision:** Use PostgreSQL as the initial source of truth and analytical store.

Start with correct event semantics, queries, indexes, and tests. Consider precomputed aggregates or a separate analytical store only when actual workload measurements or requirements justify them.

## 6. Architecture option D: merchant integration strategy

Potential integration approaches include:

### D1 — Documented HTTP API

A merchant integration sends events and requests analytics through documented HTTP endpoints.

Advantages include a clear system boundary, language-independent integration, and a straightforward demo client.

Limitations include the need to design authentication, request validation, event identity, retries, and access control.

### D2 — Platform-specific connector

A connector maps a particular ecommerce platform's events and data model into Checkout Intelligence's internal event model.

This can simplify integration for merchants using that platform, but couples the connector to platform-specific behavior and APIs.

### D3 — Webhooks and asynchronous event delivery

A store or platform sends events to a registered endpoint. Delivery may be repeated or delayed, so the receiving system must handle retries, duplicates, and ordering limitations.

Webhooks are an integration mechanism, not a guarantee of reliable event delivery.

### D4 — Store-side event collector

A small client-side integration captures selected checkout interactions and sends events to Checkout Intelligence.

This can observe browser interactions, but browser events alone cannot authoritatively confirm payment or order completion. Those outcomes may require a trusted server-side source.

**Proposed decision:** Define a platform-independent event contract and documented HTTP API first. Use the demo storefront as the initial integration client.

The first implementation will demonstrate the integration boundary without claiming to support a real ecommerce platform. A platform-specific connector, webhook integration, or browser collector will be selected later based on the requirements of a clearly scoped integration.

The merchant's existing dashboard must not need direct database access or access to internal Python modules.

## 7. Architecture option E: deterministic analytics versus AI-led analysis

### E1 — Deterministic analytics first

Calculate funnel counts, conversion rates, stage progression, and other agreed metrics using explicit rules and verified event data.

Generate findings from defined rules and evidence.

**Advantages**

- Reproducible results.
- Straightforward unit testing and debugging.
- Clear distinction between observed facts and possible explanations.
- No AI API dependency for core analytics.

**Limitations**

- Rule-based findings may not explain every complex pattern.
- Rules require maintenance and do not automatically generalize to every situation.

### E2 — AI-led analytics

Send checkout data to an AI model and ask it to determine performance, diagnose problems, and recommend actions.

**Advantages**

- Flexible interpretation of structured evidence.
- Potentially useful for natural-language summaries and investigation assistance.

**Disadvantages**

- Outputs can be incorrect, inconsistent, or unsupported by the data.
- Model availability, latency, privacy terms, and usage limits become dependencies.
- Unconstrained outputs are difficult to evaluate reliably.

**Proposed decision:** Deterministic analytics first; AI-assisted interpretation second.

AI may explain verified findings, summarize relevant evidence, and suggest hypotheses or next investigative steps.

AI must not be the authority for numeric metrics. It must not turn a correlation into a proven cause or present a plausible explanation as an observed fact.

The AI component will sit behind a defined internal interface so that the core application can be tested without calling an external model. A free-tier model may be used for demonstrations, with explicit handling for unavailable models, rate limits, invalid output, and insufficient evidence.

## 8. Architecture option F: frontend and product presentation

The project needs three distinct presentation concerns:

1. **Product landing page:** Explains the problem, product capabilities, evidence, and demo.
2. **Demo storefront:** Simulates a merchant shopping journey and generates clearly labelled demonstration events.
3. **Reference dashboard:** Allows a reviewer to inspect checkout metrics and findings returned by Checkout Intelligence.

These are separate clients or presentation surfaces. None should bypass the backend's documented interface to access internal modules or its database directly.

**Proposed decision:** Keep the landing page, demo storefront, and reference dashboard as distinct presentation concerns. Final frontend framework, deployment arrangement, and repository layout will be decided during implementation planning.

The landing page can use free static hosting if its final implementation is compatible with that hosting model. Backend functionality will remain independent of the marketing site.

## 9. Initial technology candidates

| Concern | Candidate | Rationale |
|---|---|---|
| Language | Python | Supports backend engineering and AI integration |
| HTTP API | FastAPI | Open-source framework with API schema support |
| Relational database | PostgreSQL | Reliable relational storage and SQL analytics |
| Database access | SQLAlchemy | Explicit database interaction and persistence mapping |
| Schema migrations | Alembic | Versioned database schema changes |
| Testing | pytest | Automated unit and integration testing |
| Local isolation | Python `venv` | Reproducible, project-specific Python environment |
| Local infrastructure | Docker Compose | Repeatable database and service setup |
| AI | Provider-independent adapter | Avoids coupling domain logic to one AI vendor |
| Source control | Git | Incremental history and reviewable changes |

These are technology candidates, not permission to begin implementing every dependency immediately. We should finalize the domain model and event contract before creating application modules.

## 10. Free-service and cost constraints

The architecture should work without requiring a paid hosted database, paid AI API, paid message broker, or paid observability service.

Initial approach:

- Develop and test locally.
- Run PostgreSQL in Docker.
- Keep core metrics functional without an AI provider.
- Use synthetic checkout data for the demonstration.
- Avoid services that require paid upgrades for essential local development.
- Keep configuration and secrets outside committed source code.
- Select hosted services only after checking current free-tier limits, inactivity behavior, persistence, account requirements, and potential billing risks.
- Keep the application portable enough to move to another host later.

A free tier is not necessarily permanent, production-grade, or free of usage limits. Hosting decisions will therefore be documented separately from the core application architecture.

## 11. Proposed architecture decisions

| ID | Decision | Status |
|---|---|---|
| ADR-001 | Use a modular monolith initially | Proposed |
| ADR-002 | Use PostgreSQL as the initial source of truth | Proposed |
| ADR-003 | Separate deterministic metrics from AI interpretation | Proposed |
| ADR-004 | Expose a documented integration API | Proposed |
| ADR-005 | Begin with synchronous event persistence | Proposed |
| ADR-006 | Keep merchant integration independent from the reference dashboard | Proposed |
| ADR-007 | Prioritize free local development and portable deployment | Proposed |

These decisions will be accepted or revised after review. Each decision should be revisited when new evidence or a concrete requirement changes the trade-off.

## 12. Deferred architectural decisions

Do not decide these prematurely:

- First real ecommerce platform to support.
- Final authentication and merchant onboarding design.
- Tenant and authorization model details.
- Exact event schema, event identity, and journey reconstruction rules.
- Trusted source for payment and order completion.
- Event retention, deletion, and privacy policy.
- Whether and when to add a queue or background worker.
- Whether to introduce precomputed aggregates.
- Production deployment topology and capacity targets.
- AI provider, model selection, evaluation thresholds, and cost controls.
- Final frontend framework and repository structure.

These decisions require additional domain modeling, requirements clarification, or implementation evidence.

## 13. Risks and mitigations

**Risk: The system measures the wrong thing.**  
Mitigation: Define event semantics, funnel rules, and acceptance tests before trusting the metrics.

**Risk: AI generates convincing but unsupported explanations.**  
Mitigation: Ground interpretation in verified evidence, represent uncertainty, and test model outputs against a repeatable evaluation set.

**Risk: Free hosting loses data or becomes unavailable.**  
Mitigation: Keep local development independent of hosted services, label demo data, and document backup/export options before relying on hosted persistence.

**Risk: Premature infrastructure complexity slows learning.**  
Mitigation: Add components only when a requirement, test, or measured limitation justifies them.

**Risk: The demo appears to support integrations that have not been implemented.**  
Mitigation: Clearly distinguish simulated integration from a tested, supported platform connector.

**Risk: The portfolio overstates business validation.**  
Mitigation: Label the Lecadence Decor scenario and all unverified merchant assumptions as hypothetical. Present technical results as evidence of engineering competence, not proof of commercial impact.

## 14. Exit criteria before application scaffolding

Before creating the main application structure, confirm that:

- The system boundary and integration direction are clear.
- The initial architecture options have been reviewed.
- The domain model and event contract are documented.
- The first vertical slice has a defined business outcome and acceptance criteria.
- The required local tools and free-service constraints are understood.

Once these are satisfied, create only the initial project structure needed for the first vertical slice. Do not scaffold every anticipated module before its responsibilities are understood.
