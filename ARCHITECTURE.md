# Rentflow Architecture

## System Context
Rentflow is a Rental Management SaaS built using the Quarkus (Java 25) REST API backend hosted on Render.com with Supabase PostgreSQL as the database. The frontend is a React 19 PWA (Progressive Web App) using Vite and TypeScript, hosted on Cloudflare Pages.

## Subsystems
- `aap-backend`: REST API backend
- `aap-frontend`: React PWA frontend
- `aap-design`: Design and UI components
- `OpenSpec`: Documentation and API specifications

## Key Domain Entities
- **Propietario**: Owner of the property
- **Inmueble**: Property or rental unit
- **Inquilino**: Tenant or renter
- **Contrato**: Rental contract or lease agreement
- **Factura**: Invoice or payment record
- **Suministro**: Supplies or maintenance service

## Event Flow for Contract Creation
1. **Inquilino submits a request**: The tenant submits a request for a rental unit.
2. **Propietario reviews request**: The owner reviews the request and approves or rejects it.
3. **Contrato creation**: If approved, the contract is created and signed by both parties.
4. **Factura generation**: An invoice is generated and sent to the tenant for payment.
5. **Suministro setup**: If required, setup for maintenance or supplies is initiated.