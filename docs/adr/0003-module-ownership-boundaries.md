# 3. Module ownership boundaries

- Status: Accepted
- Date: 2026-11-19

## Context

The application consists of logical modules (Owners, Real Estate, Tenant, Candidates, Contract, Settings, plus a shared Document Store and a read-only Dashboard) running in a single Quarkus backend over one PostgreSQL database. Without explicit ownership, entities would be written from multiple places, causing invariants to drift.

## Decision

Each entity has exactly one owning module — the source of truth. Every other module holds read-only references (foreign keys) and must not mutate an entity it does not own.

| Module | Owns (source of truth) |
|---|---|
| Owners Management | Owner, Ownership Share |
| Real Estate Management | Real Estate, Expense, Incident |
| Tenant Management | Tenant |
| Candidates Management | Candidate |
| Contract context | Contract, Invoice, Income |
| Document Store | Document |
| Settings | Contract Template, Spending Type, Document Type |
| Dashboard | — (read-only, owns nothing) |

Cross-module access is read-only: Real Estate lists its owners and contract history but does not mutate Owners or Contracts; Tenant lists its contracts and documents but does not mutate them.

## Consequences

- A single unambiguous source of truth for every entity.
- Invariants (e.g., a Contract's state) are enforced only in the owning module.
- Read-only references keep views (Dashboard, history lists) decoupled from writers.
- Ownership Share is initially anchored under Owners Management; revisit if property-side ownership editing becomes primary.
