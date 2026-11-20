# 5. Shared document store

- Status: Accepted
- Date: 2026-11-19

## Context

PDF documents (ID Document, Passport, Payroll, Data Protection Consent, and the signed Contract) appear under both Tenant and Candidate, and Candidate documents must become Tenant documents upon signature. Duplicating storage per module would risk inconsistency and complicate retention and consent.

## Decision

Use a single shared Document Store for all documents. Each Document references its owning record (Tenant, Candidate, or Contract). Document types are configured centrally in Settings. When a Candidate is promoted to Tenant on contract signature, its documents transfer ownership to the resulting Tenant. Candidates upload documents through a tokenized, no-login, expiring link.

## Consequences

- One place to enforce retention, consent, and access rules (GDPR/LOPD).
- No document duplication across Tenant and Candidate.
- Transfer on conversion is a single ownership update, preserving upload history.
- Documents are coupled to their owner record via reference, not embedded.
