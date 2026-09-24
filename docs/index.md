# Payment API Standards

This site is the authoritative source for API architecture standards governing all payment-domain APIs at Parasol Insurance.

## What you will find here

| Document | Purpose |
|---|---|
| [Payment API Architecture Standard](payment-api-standard.md) | Mandatory design rules, URI patterns, data model requirements, security controls, and compliance checklist for all payment APIs |

## Who this is for

- **API designers and developers** — consult this standard before creating or modifying any payment API
- **Architects** — use the compliance checklist (Section 9) during design reviews
- **Security reviewers** — Section 6 covers PCI-DSS, FAPI, and mTLS requirements
- **Product owners** — Section 8 defines the versioning and deprecation policy

## Quick reference

### BIAN Service Domains in scope

- Customer Billing
- Payment Order
- Direct Debit Mandate
- Payment Tracking
- Customer Refunds

### Key rules at a glance

1. Every payment API maps to **exactly one** BIAN Service Domain
2. URI pattern: `/{service-domain}/{control-record}/{id}[/{bq}/{bq-id}]`
3. No raw card data (PAN, CVV) in any payload — tokenise at the PSP boundary
4. All mutating endpoints require `x-idempotency-key`
5. Public endpoints must comply with **FAPI 1.0 Advanced**
6. OpenAPI Specification 3.0.3+ required; register in Developer Hub

## Related resources

- [BIAN API Compliance Validation Guide](https://backstage-developer-hub-rhdh.apps.cluster-nrk99.dyn.redhatworkshops.io/docs/default/component/bian-architecture/BIAN_API_Compliance_Validation_Guide/) — base BIAN compliance checklist
- [premium-payment-api](https://backstage-developer-hub-rhdh.apps.cluster-nrk99.dyn.redhatworkshops.io/catalog/default/api/premium-payment-api) — reference implementation in Developer Hub
