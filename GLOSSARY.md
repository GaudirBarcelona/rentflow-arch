# Glossary

> Initial domain glossary for **Rentflow** — GitHub Issue #1 (GaudirBarcelona/rentflow-arch).
>
> **Canonical language is English.** English terms are the single source of truth for both code and documentation; Spanish domain terms (Propietario, Inmueble, Inquilino, Contrato) are not used as keys. See ADR-0002.

## How to read

- **Owner Module** is the module that is the *source of truth* for that record. Other modules may hold read-only references to it, but must not mutate it (ADR-0003).
- Modules are logical bounded contexts within one Quarkus backend and one PostgreSQL database; **Dashboard** is read-only.

## Terms

| Canonical Term | Definition | Owner Module |
|---|---|---|
| Owner | A natural person or legal entity (company) that owns one or more rental properties and on whose behalf the agency manages them. | Owners Management |
| Co-ownership | A property owned by more than one Owner. | Owners Management |
| Ownership Share | The percentage stake a specific Owner holds in a Real Estate property (0 when unknown). | Owners Management |
| Real Estate | A rental unit or listing managed by the agency: address, photos, expenses, contract history, incident history, and Idealista export. Synonyms: Property, Listing. | Real Estate Management |
| Expense | A property-level cost recorded with a tax base and a VAT rate, categorized by a Spending Type. | Real Estate Management |
| Incident | A maintenance or repair issue reported against a Real Estate property. | Real Estate Management |
| Incident State | Lifecycle of an Incident: Reported → In progress → Resolved. | Real Estate Management |
| Idealista Export | A formatted text message plus a photo list used to publish or update a listing on the Idealista portal. | Real Estate Management |
| Tenant | A renter holding one or more Contracts, identified by name, phone, and email. | Tenant Management |
| Co-tenancy | Multiple Tenants sharing a single Contract (co-signers). | Tenant Management |
| Candidate | An applicant for a specific Real Estate property (ID document, target property, email, phone) who uploads documents and is promoted to Tenant on contract signature. References the target property directly; there is no separate Application entity. | Candidates Management |
| Contract | The rental agreement (lease) linking one or more Tenants to a Real Estate property; generated from a Contract Template and carrying rent and dates. | Contract context |
| Contract State | Lifecycle of a Contract: Draft → Awaiting signature → Active → Renewed / Terminated / Expired. | Contract context |
| Invoice | A rent payment obligation under a Contract with status Paid, Pending, or Overdue. Payment is marked as a paid flag plus a date; no separate payment event is recorded. | Contract context |
| Income | Monthly rent due, derived per active Contract. | Contract context |
| Electronic Contract Signing | Signing of a Contract through a third-party e-signature provider. | Contract context |
| Document | A stored PDF (identity, payroll, consent, or the signed Contract) linked to an owning record (Tenant, Candidate, or Contract). | Document Store |
| ID Document | Identity document (DNI/NIE or Passport). | Document Store |
| Passport | Travel identity document. | Document Store |
| Payroll | Proof-of-income document. | Document Store |
| Data Protection Consent | Signed LOPD/GDPR data-processing consent (privacy) form. | Document Store |
| Document Type | Configurable category of documents. Built-in types: ID Document, Passport, Payroll, Contract, Data Protection Consent. | Settings |
| Contract Template | A merge-field document (tenant name, property address, rent, dates, owner name) maintained per language (English, Spanish). | Settings |
| Spending Type | An expense category; a predefined set plus user-defined additions. | Settings |
| Dashboard | A read-only view aggregating monthly income/expenses, pending incidents, and annual rent reviews. Owns no data. | — (read-only) |
| Pending Incident | An Incident in the Reported or In progress state. | Dashboard (derived) |
| Annual Rent Review | A manual reminder of an upcoming rent review on a Contract's anniversary. No automatic calculation is performed. | Dashboard (derived) |

## Module ownership summary

| Module | Owns (source of truth) | Reads (read-only references) |
|---|---|---|
| Owners Management | Owner, Ownership Share | Real Estate (of an owner) |
| Real Estate Management | Real Estate, Expense, Incident | Owner, Contract, Tenant, Document |
| Tenant Management | Tenant | Contract, Document |
| Candidates Management | Candidate | Real Estate (target property), Document |
| Contract context | Contract, Invoice, Income | Tenant, Real Estate, Contract Template, Document |
| Document Store | Document | Tenant, Candidate, Contract |
| Settings | Contract Template, Spending Type, Document Type | — |
| Dashboard | — (read-only) | Contract, Expense, Incident |

## Lifecycle states

- **Contract**: Draft → Awaiting signature → Active → (Renewed | Terminated | Expired). Renewal is a manual new Contract linked to the prior one.
- **Incident**: Reported → In progress → Resolved.
- **Invoice**: Paid | Pending | Overdue.

## Out of scope / dropped terms

- **Factura** and **Suministro** (from the earlier ARCHITECTURE.md) are dropped — superseded by Invoice and Incident/Expense respectively.
- **Application** is not an entity; a Candidate references its target Real Estate directly.

## Assumptions & open questions

- Currency is EUR only. Monetary amounts store a tax base plus a VAT rate; the exact storage precision (e.g., integer cents) is an open question for a future ADR.
- Ownership Share is recorded under Owners Management; if a future screen makes property-side ownership editing primary, this boundary can be revisited.
- E-signature provider is third-party but not yet selected.
- Document retention: consent is required before storage; documents are retained per the legal minimum. The specific retention period is still to be defined.
