# 6. Multi-tenant SaaS isolation

- Status: Accepted
- Date: 2026-11-19

## Context

The primary user is a property-management agency, and the product is a SaaS (multi-tenant). Each customer account must see only its own owners, properties, tenants, candidates, contracts, and documents.

## Decision

Adopt tenant-per-account isolation: every domain record is scoped to a customer account, and all queries are account-scoped at the data-access layer. The account is the top-level ownership boundary above the modules.

## Consequences

- Customer data cannot leak across accounts.
- Account scoping must be applied uniformly to every query and to the Document Store.
- Adds an account dimension to every aggregate; migrations and indexes must include it.
