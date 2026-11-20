# 4. Contract as a dedicated aggregate

- Status: Accepted
- Date: 2026-11-19

## Context

A Contract is surfaced in two places — Real Estate details (contract history) and Tenant details (contract list) — and its lifecycle (drafting, e-signing, renewal, termination) spans more than either a property or a tenant. Both Real Estate and Tenant Management would be plausible owners.

## Decision

Own Contract in a dedicated Contract/Lease context, not under Tenant or Real Estate. A Contract links one or more Tenants to one Real Estate property; a property may have multiple active Contracts (shared lease or room-by-room), a Contract may have multiple Tenants (co-tenancy), and a Tenant may hold multiple Contracts across properties. Invoice and Income live under the Contract context.

## Consequences

- Contract lifecycle and invariants are centralized in one place.
- Tenant and Real Estate expose contracts as read-only history/lists.
- Renewal is modeled as a new Contract linked to the prior one (no implicit mutation of a signed Contract).
- Requires a clear cross-context reference between Contract, Tenant, and Real Estate.
