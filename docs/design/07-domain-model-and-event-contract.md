# Domain Model and Checkout Event Contract

**Status:** Proposed for review  
**Project:** Checkout Intelligence  
**Related documents:** Problem Discovery, Workflows and Problem Analysis, Capabilities and Success Criteria, Use Cases and Functional Requirements, Non-Functional Requirements, Architecture Options and Decisions.

## 1. Purpose

This document defines the initial business concepts, their relationships, and the rules that Checkout Intelligence must preserve when collecting and analyzing checkout activity.

It is a conceptual design, not a database schema or a final API specification.

The goal is to establish what the system means by an event, checkout journey, funnel stage, finding, and confirmed order outcome before choosing how to store or process them.

The Lecadence Decor scenario is hypothetical. The proposed event names, checkout stages, and business rules below are design proposals that must be validated against the intended demo behavior.

## 2. Domain boundaries

Checkout Intelligence observes and analyzes checkout activity made available through supported integration interfaces.

The merchant's store remains responsible for:
- Product catalog and pricing.
- Cart and checkout execution.
- Delivery options and applicable charges.
- Payment processing.
- Creating and maintaining authoritative order records.

Checkout Intelligence is responsible for:
- Accepting and validating supported events.
- Persisting accepted event data.
- Associating events with merchant and journey identities.
- Calculating metrics using explicit definitions.
- Identifying patterns supported by available evidence.
- Presenting findings and, where appropriate, AI-assisted interpretations.

Checkout Intelligence must not claim that an order or payment succeeded solely because a browser reported a checkout interaction.

## 3. Core domain concepts

### 3.1 Merchant

A merchant is the business whose checkout activity is being analyzed.

A merchant provides the ownership boundary for events, journeys, findings, and analytics.

Proposed rules:
- Every accepted event must be associated with a merchant.
- Data belonging to one merchant must not be exposed to another merchant.
- The demo may use one configured hypothetical merchant, but the internal design should not assume that all future usage will be single-merchant.

Whether merchant onboarding, authentication, and multiple production tenants belong in the MVP remains an open decision.

### 3.2 Checkout event

A checkout event is a recorded occurrence of a defined type at a particular time.

Examples:
- `product_added_to_cart`
- `checkout_started`
- `delivery_details_submitted`
- `payment_attempted`
- `checkout_completed`

These are proposed event names, not a final event taxonomy.

An event records what the integration reported. Its meaning depends on the event definition and the trustworthiness of its source.

Proposed event properties:

| Property | Purpose |
|---|---|
| `event_id` | Stable identifier used to recognize repeated delivery of the same event |
| `merchant_id` | Identifies the merchant that owns the event |
| `event_type` | Identifies the defined occurrence |
| `event_version` | Identifies the event contract version |
| `journey_id` | Associates the event with a checkout journey when available |
| `occurred_at` | Time the source says the event occurred |
| `received_at` | Time Checkout Intelligence received the event |
| `source` | Identifies the integration or event source |
| `properties` | Validated event-specific information |

The final contract may use different field names or representations. The purpose of these properties is to establish the required semantics.

### 3.3 Checkout journey

A checkout journey is a grouping of related events representing activity associated with one checkout attempt or shopping journey.

A journey can be incomplete. It does not have to end in an order.

Proposed rules:
- Multiple events may belong to one journey.
- A journey may have no payment attempt or confirmed order.
- Events may arrive late or out of order.
- A journey may contain repeated events, but repeated delivery of the same event must not automatically count as a new occurrence.
- The system must distinguish a missing event from proof that an action never happened.

A `journey_id` is not necessarily the same as a shopper identity. A shopper may begin multiple journeys, and a shopper identity may be unavailable.

The exact rules for starting, resuming, expiring, or merging journeys remain open until the demo workflow and integration contract are specified.

### 3.4 Funnel stage

A funnel stage represents a defined step used to measure progress through a checkout process.

Possible demo stages:

1. Cart activity.
2. Checkout started.
3. Delivery details submitted.
4. Payment attempted.
5. Checkout completed or order confirmed, depending on the chosen metric.

These stages are provisional. We must not assume every merchant uses the same checkout sequence.

Each stage needs an explicit definition describing:
- Which event or authoritative state qualifies.
- Which journeys are eligible for the calculation.
- The time window used.
- How repeated events are handled.
- How incomplete or late-arriving data affects the result.

### 3.5 Metric

A metric is a reproducible calculation over a defined set of eligible events, journeys, or outcomes.

Examples:
- Number of journeys that started checkout.
- Number of eligible journeys that reached a specified stage.
- Proportion of eligible journeys reaching the next stage.
- Number of payment attempts with a known outcome.

A metric must have a documented definition, not just a display label.

For example, a stage progression rate might be defined as:

`eligible journeys reaching stage B / eligible journeys reaching stage A`

This formula is valid only if the eligibility rules, stage definitions, and measurement window are explicit. It must not be applied blindly to unrelated event counts.

The system must not silently treat missing data as a failed checkout.

### 3.6 Finding

A finding is a statement about a potentially important pattern in checkout activity.

Examples:
- Fewer observed journeys reached the payment-attempt stage during a selected period.
- A significant portion of recorded journeys have no subsequent event.
- The available data is insufficient to determine what happened after checkout started.

A finding should distinguish:
- **Observation:** what the data directly shows.
- **Interpretation:** a possible explanation of the observation.
- **Hypothesis:** an explanation that still needs investigation.
- **Recommended next step:** an action or investigation supported by the evidence.

The system must not label a possible cause as a confirmed cause without sufficient evidence.

### 3.7 Order outcome

An order outcome represents an outcome reported by an authoritative source, such as the merchant's order system or a verified server-side integration.

Potential outcomes may include order created, order cancelled, or payment confirmed, depending on the available source and the agreed definitions.

An attempted payment is not proof of successful payment. A checkout completion event from a browser is not automatically proof that an order exists.

The authoritative source for order and payment outcomes has not yet been selected.

### 3.8 Intervention

An intervention is a change a merchant makes in response to an identified issue.

Examples could include changing delivery-cost presentation or fixing a checkout validation problem. These are hypothetical examples, not known Lecadence Decor problems.

If intervention tracking is included in the MVP, it should record the change, its relevant time, and the metric or finding it was intended to address.

A later improvement in a metric does not by itself prove the intervention caused the improvement. Other factors may have changed.

Intervention tracking remains a scope decision.

## 4. Relationships between concepts

The initial conceptual relationships are:

- A merchant owns many checkout events.
- A merchant has many checkout journeys.
- A checkout journey may contain many events.
- An event may be associated with a journey when sufficient identity information exists.
- Metrics are calculated from eligible events, journeys, or authoritative outcomes.
- Findings reference the metrics and evidence that support them.
- AI-assisted interpretations consume selected findings and evidence; they do not replace the underlying evidence.
- An intervention, if supported, may reference a finding and a subsequent evaluation period.

These are conceptual relationships. They do not yet prescribe relational tables, ORM models, or foreign-key arrangements.

## 5. Initial event contract principles

### 5.1 Event identity and duplicate delivery

Each event must have a stable identity within the agreed merchant and integration boundary.

If the same event is delivered repeatedly, the system must recognize the repeat and avoid counting it as a separate occurrence.

The exact uniqueness constraint and behavior for conflicting payloads with the same identity must be specified in the API and persistence design.

### 5.2 Event time and receipt time

The system should preserve both:
- The source-reported occurrence time.
- The time Checkout Intelligence received the event.

These timestamps answer different questions. Receipt time must not silently replace occurrence time during analytics.

The system must validate timestamps and define how it handles missing, invalid, or implausibly future-dated values.

### 5.3 Validation and versioning

The system must validate the required envelope fields and event-specific properties.

Unknown event types, invalid values, unsupported versions, and malformed requests must have documented behavior.

The contract should be versioned so future changes do not silently alter the meaning of previously accepted events.

### 5.4 Event ordering

The system must not assume that events always arrive in chronological order.

Analytics rules should use the defined event-time and journey semantics. Where ordering cannot be established reliably, the system should preserve that limitation rather than invent a sequence.

### 5.5 Source and trust

The event source must be identifiable enough to evaluate what the event can prove.

A browser-generated event may establish that a client-side interaction was reported. It cannot independently establish a successful payment or an authoritative order outcome.

### 5.6 Data minimization

The initial demo should use synthetic data.

Do not collect names, email addresses, full payment details, or other personal information unless a documented requirement justifies it. Sensitive payment data is not needed to demonstrate the initial checkout analytics workflow.

Event properties should be explicitly defined and validated instead of accepting arbitrary data without limits.

### 5.7 Merchant isolation

Merchant identity must be established through a trusted integration context, not blindly trusted from an arbitrary client-supplied field.

The implementation must prevent one merchant from querying or modifying another merchant's records. The authentication mechanism and tenant authorization rules will be defined in a later design step.

## 6. Illustrative checkout sequence

The following is a hypothetical demonstration journey, not a claim about a real merchant's checkout:

1. A shopper adds a decor product to the cart.
2. The demo storefront reports checkout started.
3. The shopper submits delivery details.
4. The shopper attempts payment.
5. The demo system records a simulated order outcome from a designated trusted demo source, if that capability is included.

The reference dashboard then displays the metrics derived from the recorded events and outcomes.

The demonstration must clearly identify simulated events and simulated order outcomes. It must not imply that a real payment provider processed a transaction.

## 7. Initial invariants

The following invariants should hold regardless of the eventual database or framework:

1. An accepted event has a valid identity and defined meaning.
2. Events are associated with a merchant through a trusted context.
3. Repeated delivery of the same event does not automatically inflate metrics.
4. Event occurrence time and receipt time remain distinguishable.
5. A checkout journey may remain incomplete.
6. Missing evidence is not equivalent to a confirmed failure.
7. Metrics are computed using explicit, testable definitions.
8. AI-generated interpretation cannot override verified metric values.
9. A reported checkout or payment interaction is not automatically an authoritative order outcome.
10. Findings must distinguish observed evidence from unverified explanations.
11. Demo data and simulated outcomes are labelled as such.
12. The merchant's existing dashboard does not need direct access to Checkout Intelligence's database or internal modules.

These invariants will become candidates for automated tests as the corresponding features are implemented.

## 8. Open domain and contract decisions

Resolve these before finalizing the API schema and database model:

- What precisely begins a checkout journey?
- Can a journey continue across browser sessions or devices in the MVP?
- What event types are required for the first demo?
- Which events represent funnel stages?
- What counts as checkout completion for each metric?
- Which source can authoritatively confirm an order outcome?
- How are duplicate event IDs with conflicting payloads handled?
- What happens when an event lacks a journey ID?
- What is the policy for late-arriving events?
- How is merchant identity established and authorized?
- What event properties are allowed, required, or prohibited?
- Is intervention tracking included in the initial MVP?

Do not answer these questions by assuming every ecommerce platform behaves the same way.

## 9. Readiness criteria

This document is ready to support the next design step when:

- The core concepts and their distinctions are understood.
- The first demo journey is specified.
- The minimum event types and required properties are agreed.
- Journey identity and duplicate-handling rules are defined for the demo.
- Funnel stages and metric eligibility rules are explicit.
- The distinction between client-reported events and authoritative order outcomes is preserved.
- Remaining unknowns are recorded instead of silently treated as facts.

**Next step:** Turn the agreed concepts into a concrete first-slice specification and event contract. Then define the corresponding API request and response behavior and persistence requirements before implementing the first vertical slice.
