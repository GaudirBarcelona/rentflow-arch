# 5. SDD Workflow: Specify Before Implementation

## Intent

SDD is used when the specification must be an approved control artifact. Implementation is constrained by the specification, and deviations return to the owning requirement or architecture stage.

## Workflow

```text
grill-with-docs
    ↓
EGM Brainstorm → Draft → Review → Resolve
    ↓
Domain model + ADR set + required approvals
    ↓
OpenSpec Propose
    ↓
Specification review and approval
    ↓
investigate-first + caveman-discover
    ↓
Feasibility checkpoint
    ↓
Approved task plan
    ↓
Superpowers implementation with behavior-first BDD
    ↓
JUnit 5 + integration/contract tests + JaCoCo + PIT
    ↓
caveman-review
    ↓
caveman-evidence-review
    ↓
shadcn-improve
    ↓
OpenSpec Verify
    ↓
Release approval
    ↓
OpenSpec Archive
```

### SDD Prompt 1: Full Domain Discovery

#### Objective

Establish the business and operational model before architecture is selected.

#### Example Prompt

```text
/grill-with-docs
Analyze the global supplier event replay capability for SDD delivery.

Cover:
- authoritative business terminology
- domain and data ownership
- producer and consumer responsibilities
- permissions and separation of duties
- ordering, deduplication, idempotency, retries, and poison events
- retention and replay time windows
- EU, US, and CN differences
- privacy and regulated-data constraints
- partial failures and recovery procedures
- auditability and evidence retention
- unresolved policy decisions and ADR candidates

Read existing glossary entries, ADRs, event contracts, security policies,
and operational runbooks first.
Separate business rules from technical proposals.
Do not design the solution.
```

#### Deliverables

- Approved terminology
- Business and operational rules
- Regional constraints
- Edge-case catalogue
- Decision owners and open questions

### SDD Prompt 2: Architecture Brainstorm

#### Objective

Explore viable architectures against explicit quality attributes.

#### Example Prompt

```text
/egm-brainstorm
Design the global supplier event replay capability.

Evaluate at least two viable options across:
- Kafka topology and replay mechanism
- Microsoft Fabric ingestion and downstream consistency
- EU, US, and CN deployment constraints
- data residency and authorization
- event ordering, deduplication, and idempotency
- observability, capacity, and operational ownership
- disaster recovery and regional failure
- migration and rollback
- backward and forward compatibility
- test seams and verification strategy

For each option provide assumptions, trade-offs, failure modes,
reversibility, operational cost, and disqualifying conditions.
Use approved domain artifacts. Do not create implementation tasks.
```

#### Deliverables

- Architecture options
- Quality-attribute comparison
- Risks and assumptions
- Preferred option with rationale

### SDD Prompt 3: Draft, Review, and Resolve Architecture

#### Objective

Create an independently challenged and resolved design.

#### Example Prompt

```text
/egm-draft
Create path/to/supplier-event-replay-design.md from the approved brainstorm.
Include context, quality attributes, architecture, data flows,
security, operability, failure modes, regional constraints,
migration, rollback, test strategy, non-goals, and ADR references.
Make every assumption explicit.
```

```text
/egm-review path/to/supplier-event-replay-design.md
Review independently as a skeptical architecture board.
Challenge correctness, hidden coupling, event-contract evolution,
data residency, authorization, operational ownership, capacity assumptions,
recovery, rollback, and testability.
Classify findings as blocking, major, or advisory.
Do not rewrite the design.
```

```text
/egm-resolve
Resolve every blocking and major finding from the architecture review.
For each finding record the resolution, artifact changed,
remaining risk, owner, and approval needed.
Do not silently accept risk.
```

#### Deliverables

- Approved desi