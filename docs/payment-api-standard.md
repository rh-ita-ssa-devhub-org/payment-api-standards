# Payment API Architecture Standard

**Version:** 1.0.0
**Status:** Active
**Owner:** billing-payments-engineering
**Last updated:** 2026-09-24

---

## 1. Purpose & Scope

This document defines the mandatory design rules for all APIs operating within the **payment domain** of the Parasol Insurance platform. It is the authoritative standard against which payment APIs are reviewed, approved, and certified.

### 1.1 What this standard covers

Any API that initiates, executes, tracks, or manages a financial transaction on behalf of a customer or internal system is in scope. This includes:

- Premium payment initiation (card, direct debit, Open Banking account-to-account)
- Payment status retrieval and tracking
- Direct debit mandate setup and amendment
- Refund and chargeback processing

APIs governed by this standard are mapped to the following **BIAN Service Domains**:

| Capability | BIAN Service Domain |
|---|---|
| Premium payment initiation | Customer Billing |
| Payment execution (card capture) | Payment Order |
| Direct debit mandate management | Direct Debit Mandate |
| Payment status tracking | Payment Tracking |
| Refund processing | Customer Refunds |

### 1.2 Relationship to the BIAN Compliance Framework

This standard is additive to the [BIAN API Compliance Validation Guide](https://backstage-developer-hub-rhdh.apps.cluster-nrk99.dyn.redhatworkshops.io/docs/default/component/bian-architecture). All seven BIAN checks (CHK-001 through CHK-007) apply. This document adds **three payment-specific checks** (CHK-P01 through CHK-P03) described in Section 9.

### 1.3 What this standard does NOT cover

- Internal ML model or data platform APIs
- Identity and access management (IAM) platform APIs
- Claim or policy management APIs (see the Policy API Standard — forthcoming)

---

## 2. BIAN Domain Mapping for Payments

Every payment API must be mapped to **exactly one** BIAN Service Domain. The table below defines the authorised mappings, their Control Records (CR), allowed Action Terms, and corresponding HTTP methods.

| BIAN Service Domain | Control Record | Action Term | HTTP Method | Description |
|---|---|---|---|---|
| Customer Billing | Payment Facility | Initiate | POST | Create a new payment facility / initiate a payment |
| Customer Billing | Payment Facility | Retrieve | GET | Fetch payment facility state |
| Customer Billing | Payment Facility | Update | PUT / PATCH | Amend an existing payment facility |
| Payment Order | Payment Order | Initiate | POST | Submit a payment order for execution |
| Payment Order | Payment Order | Execute | POST | Trigger card capture or A2A transfer |
| Payment Order | Payment Order | Retrieve | GET | Query payment order status |
| Direct Debit Mandate | Direct Debit Mandate | Initiate | POST | Set up a new direct debit mandate |
| Direct Debit Mandate | Direct Debit Mandate | Update | PUT | Amend mandate details |
| Direct Debit Mandate | Direct Debit Mandate | Control | PUT | Suspend or cancel a mandate |
| Payment Tracking | Payment Log | Retrieve | GET | Query payment history and tracking events |
| Customer Refunds | Refund Transaction | Initiate | POST | Initiate a refund |
| Customer Refunds | Refund Transaction | Retrieve | GET | Retrieve refund status |

---

## 3. URI Design Rules (CHK-001, CHK-003)

### 3.1 Mandatory URI pattern

All payment API endpoints must follow the BIAN URI structure:

```
/{service-domain}/{control-record}/{cr-reference-id}[/{behavior-qualifier}/{bq-reference-id}]
```

- `service-domain`: kebab-case BIAN Service Domain name
- `control-record`: kebab-case Control Record name
- `cr-reference-id`: unique identifier of the Control Record instance (path parameter)
- `behavior-qualifier` / `bq-reference-id`: optional, for accessing a sub-component of the CR

### 3.2 Compliant URI examples

```http
# Initiate a new payment facility (Customer Billing domain)
POST /customer-billing/payment-facility

# Retrieve an existing payment facility
GET /customer-billing/payment-facility/{paymentFacilityId}

# Update the direct debit behavior qualifier on a payment facility
PUT /customer-billing/payment-facility/{paymentFacilityId}/direct-debit/{directDebitId}

# Submit a payment order for execution
POST /payment-order/payment-order

# Execute a card capture on an existing payment order
POST /payment-order/payment-order/{paymentOrderId}/card-capture

# Retrieve payment tracking log
GET /payment-tracking/payment-log/{paymentLogId}
```

### 3.3 Non-compliant URI examples

The following patterns are **forbidden**:

```http
# BAD: Generic verb in URI — verbs belong in HTTP methods, not the path
POST /payments/create
POST /payments/execute

# BAD: No Service Domain, no Control Record structure
GET  /api/v1/pay/{id}
POST /v2/process-payment

# BAD: Spanning two Service Domains in a single base path
POST /payment-and-billing/initiate

# BAD: Exposing implementation technology in the URI
GET  /payment-service/db/transactions/{id}
```

### 3.4 Query parameters

Query parameters are allowed only for **filtering, pagination, and field selection** on Retrieve actions. They must never encode state-mutating operations.

```http
# Compliant: filtering a retrieve
GET /payment-tracking/payment-log/{paymentLogId}?from=2026-01-01&status=settled

# Non-compliant: mutation via query parameter
POST /customer-billing/payment-facility?action=cancel
```

---

## 4. HTTP Method & Action Term Mapping (CHK-004)

All HTTP methods must map to a BIAN Action Term. No Action Term may be expressed as a URI verb.

| BIAN Action Term | HTTP Method | Idempotent | Notes |
|---|---|---|---|
| Initiate | POST | No | Creates a new Control Record instance. Must return `201 Created` with `Location` header. |
| Execute | POST | Yes (with `x-idempotency-key`) | Triggers a point-in-time action (card capture, A2A transfer). |
| Retrieve | GET | Yes | Read-only. No side effects. Must support `ETag` / conditional GET. |
| Update | PUT | Yes | Full replacement of a CR or BQ. |
| Request | POST | No | Submits a request for processing (e.g. refund request). |
| Control | PUT | Yes | Alters processing status (suspend, cancel, resume mandate). |

### 4.1 Payment-specific method examples

```http
# Initiate: POST returns 201 + Location
POST /customer-billing/payment-facility
→ 201 Created
   Location: /customer-billing/payment-facility/pf-8a3f2c1d

# Execute: POST with idempotency key
POST /payment-order/payment-order/po-7b4e1a9c/card-capture
x-idempotency-key: idem-cc-2026092401
→ 200 OK

# Retrieve: GET returns 200 + ETag
GET /customer-billing/payment-facility/pf-8a3f2c1d
→ 200 OK
   ETag: "a3f92c1"

# Control: PUT to suspend a mandate
PUT /direct-debit-mandate/direct-debit-mandate/ddm-5c2b8e7f/status
→ 200 OK
```

---

## 5. Request / Response & Data Model Standards (CHK-005)

### 5.1 Content negotiation

| Requirement | Value |
|---|---|
| Request `Content-Type` | `application/json` |
| Response `Content-Type` | `application/json` |
| Accept header | `application/vnd.parasol.payment.v1+json` (see Section 8) |
| Character encoding | UTF-8 |
| Date/time format | ISO 8601 with timezone: `2026-09-24T14:30:00Z` |
| Currency amounts | ISO 4217 code + decimal string, e.g. `{"amount": "125.00", "currency": "EUR"}` |

### 5.2 BIAN BOM alignment

All request and response payloads must derive their field names from the **BIAN Business Object Model (BOM)**. Where the BOM incorporates ISO 20022, ISO 20022 field names take precedence.

#### Compliant payment initiation request body

```json
POST /customer-billing/payment-facility
Content-Type: application/json
Authorization: Bearer <JWT>
x-correlation-id: corr-2026092401
x-idempotency-key: idem-pi-2026092401

{
  "paymentFacilityReference": null,
  "paymentServiceType": "PremiumPayment",
  "paymentTransaction": {
    "paymentTransactionInitiatorReference": "cust-00123456",
    "paymentAmount": {
      "amount": "450.00",
      "currency": "EUR"
    },
    "paymentMechanism": "DirectDebit",
    "valueDate": "2026-10-01"
  },
  "paymentFacilitySchedule": {
    "paymentScheduleType": "Monthly",
    "startDate": "2026-10-01"
  }
}
```

#### Compliant response body

```json
HTTP/1.1 201 Created
Content-Type: application/json
Location: /customer-billing/payment-facility/pf-8a3f2c1d

{
  "paymentFacilityReference": "pf-8a3f2c1d",
  "paymentServiceType": "PremiumPayment",
  "paymentFacilityStatus": "Active",
  "paymentTransaction": {
    "paymentTransactionReference": "pt-9e5c3d2b",
    "paymentTransactionInitiatorReference": "cust-00123456",
    "paymentAmount": {
      "amount": "450.00",
      "currency": "EUR"
    },
    "paymentMechanism": "DirectDebit",
    "valueDate": "2026-10-01",
    "paymentTransactionStatus": "Pending"
  }
}
```

### 5.3 Error responses (RFC 7807 Problem Details)

All error responses must use the RFC 7807 `application/problem+json` format:

```json
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/problem+json

{
  "type": "https://api.parasol.com/problems/payment/invalid-mandate",
  "title": "Direct Debit Mandate Invalid",
  "status": 422,
  "detail": "The provided IBAN failed Luhn validation.",
  "instance": "/customer-billing/payment-facility/pf-8a3f2c1d",
  "correlationId": "corr-2026092401"
}
```

#### Standard HTTP status codes for payment APIs

| Code | Usage |
|---|---|
| 200 OK | Successful Retrieve, Update, Control, Execute |
| 201 Created | Successful Initiate (must include `Location` header) |
| 202 Accepted | Asynchronous payment accepted for processing |
| 400 Bad Request | Malformed request (validation failure) |
| 401 Unauthorized | Missing or invalid token |
| 403 Forbidden | Valid token but insufficient scope |
| 404 Not Found | CR reference does not exist |
| 409 Conflict | Idempotency key reused with different payload |
| 422 Unprocessable Entity | Semantically invalid payload (e.g. invalid IBAN) |
| 429 Too Many Requests | Rate limit exceeded (include `Retry-After` header) |
| 500 Internal Server Error | Unexpected server failure |

---

## 6. Security Standards (CHK-007)

### 6.1 Authentication & authorisation

| Endpoint type | Required security mechanism |
|---|---|
| Public (customer-facing) | OAuth 2.0 Authorization Code + FAPI 1.0 Advanced profile |
| Public (Open Banking A2A) | OAuth 2.0 + JARM (JWT Secured Authorization Response Mode) |
| Internal (service-to-service) | mTLS + OAuth 2.0 Client Credentials |

All public endpoints must conform to the **FAPI 1.0 Advanced** profile:

- JWT access tokens signed with RS256 or PS256
- Token lifetime: max 900 seconds (15 minutes) for payment initiation
- `sub` claim must identify the authenticated customer
- `aud` claim must match the API's registered audience

### 6.2 PCI-DSS tokenisation (CHK-P01)

**No raw card data may appear in any API payload, log, or header.**

| Field | Policy |
|---|---|
| Primary Account Number (PAN) | Must be replaced with a PSP-issued token before entering the Parasol API layer |
| CVV / CVC | Must never be stored or forwarded; accepted only at point-of-entry and immediately discarded |
| Expiry date | May be passed only within a PCI-scoped encrypted token |
| Cardholder name | Allowed in plaintext only for display purposes; must not be in payment processing payloads |

```json
// COMPLIANT: tokenised card reference
{
  "paymentInstrumentReference": "tok-4a7e2f1c9b3d",
  "paymentInstrumentType": "DebitCard"
}

// NON-COMPLIANT: raw card data
{
  "cardNumber": "4111111111111111",
  "cvv": "123",
  "expiry": "12/28"
}
```

### 6.3 Required HTTP headers

Every request to a payment API must include the following headers:

| Header | Required | Description |
|---|---|---|
| `Authorization` | Yes | `Bearer <JWT>` for public; client certificate for mTLS |
| `x-correlation-id` | Yes | UUID v4 — propagated through all downstream calls for tracing |
| `x-idempotency-key` | Yes (mutating) | UUID v4 — required on all POST/PUT requests |
| `x-fapi-interaction-id` | Yes (FAPI) | Required for Open Banking and FAPI-compliant endpoints |
| `x-fapi-customer-ip-address` | Conditional | Required when the request originates from a customer device |
| `Content-Type` | Yes (body) | `application/json` |

### 6.4 Transport security

- TLS 1.2 minimum; TLS 1.3 recommended
- Certificate pinning required for mobile client integrations
- HSTS header mandatory on all public endpoints (`max-age=31536000; includeSubDomains`)

---

## 7. API Specification Requirements (CHK-006)

### 7.1 OpenAPI Specification

All payment APIs must publish an OpenAPI Specification (OAS) at version **3.0.3 or higher**.

#### Mandatory OAS fields

```yaml
openapi: "3.0.3"
info:
  title: Customer Billing API         # BIAN Service Domain name
  version: "1.0.0"                   # Semantic version
  description: |
    Manages payment facility lifecycle operations within the
    BIAN Customer Billing Service Domain.
  contact:
    name: billing-payments-engineering
    email: billing-api@parasol.com

servers:
  - url: https://api.parasol.com/customer-billing/v1
    description: Production

tags:
  - name: Payment Facility
    description: Control Record operations

security:
  - BearerAuth: []

components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
```

### 7.2 Developer Hub registration

Every payment API must be registered in Developer Hub with the following minimum `catalog-info.yaml`:

```yaml
apiVersion: backstage.io/v1alpha1
kind: API
metadata:
  name: <api-name>
  description: <description>
  annotations:
    backstage.io/techdocs-ref: dir:.
spec:
  type: openapi
  lifecycle: production
  owner: group:default/billing-payments-engineering
  definition:
    $text: ./openapi.yaml
```

---

## 8. Versioning & Lifecycle

### 8.1 Versioning strategy

Payment APIs use **content negotiation** for versioning. The URI path must **not** contain a version segment.

```http
# COMPLIANT: version in Accept header
GET /customer-billing/payment-facility/pf-8a3f2c1d
Accept: application/vnd.parasol.payment.v1+json

# NON-COMPLIANT: version in URI
GET /v1/customer-billing/payment-facility/pf-8a3f2c1d
GET /api/v2/payments/{id}
```

### 8.2 Semantic versioning

The API version in `info.version` follows semantic versioning (`MAJOR.MINOR.PATCH`):

| Change type | Version bump |
|---|---|
| Breaking change (field removal, type change, endpoint removal) | MAJOR |
| Additive change (new optional field, new endpoint) | MINOR |
| Bug fix, documentation correction | PATCH |

### 8.3 Deprecation policy

| Milestone | Lead time | Action required |
|---|---|---|
| Deprecation announcement | 90 days before removal | Notify consumers via Developer Hub changelog |
| Sunset header added | 60 days before removal | Add `Sunset: <date>` and `Deprecation: true` response headers |
| Endpoint removed | Day 0 | Returns `410 Gone` with `Link` header to successor |

### 8.4 Lifecycle states in Developer Hub

| Lifecycle | Meaning |
|---|---|
| `experimental` | Early design / proof of concept — not for production consumers |
| `production` | Stable, SLA-backed, covered by deprecation policy |
| `deprecated` | Sunset date announced, successor available |
| `retired` | Endpoint removed, returns 410 |

---

## 9. Compliance Checklist (BIAN + Payment Extensions)

Use this checklist during design review and before merging an OpenAPI specification.

### BIAN base checks

| Check ID | Category | Validation Criteria |
|---|---|---|
| CHK-001 | Architecture | API is mapped to exactly one BIAN Service Domain |
| CHK-002 | Architecture | API does not expose or mutate data from another Service Domain's Control Record |
| CHK-003 | Semantics | Endpoint URIs follow the `/{service-domain}/{control-record}/...` pattern |
| CHK-004 | Semantics | HTTP methods match BIAN Action Terms (POST=Initiate/Execute, GET=Retrieve, PUT=Update/Control) |
| CHK-005 | Data Model | Payload field names derive from the BIAN BOM / ISO 20022 |
| CHK-006 | Specification | API is documented in OAS 3.0.3+ and registered in Developer Hub |
| CHK-007 | Security | OAuth 2.0 + FAPI 1.0 Advanced for public endpoints; mTLS for internal |

### Payment-specific checks

| Check ID | Category | Validation Criteria |
|---|---|---|
| CHK-P01 | PCI-DSS | No raw PAN, CVV, or expiry date in any API payload, header, or log |
| CHK-P02 | Idempotency | All mutating endpoints (POST/PUT) require and validate `x-idempotency-key` |
| CHK-P03 | Observability | Payment status must be independently retrievable via a dedicated GET endpoint — no polling via a mutating endpoint |

---

## 10. Reference Implementation

The **`premium-payment-api`** registered in Developer Hub is the canonical reference implementation for this standard. It demonstrates:

- FAPI-compliant OAuth 2.0 flow
- Compliant BIAN URI structure under the Customer Billing domain
- PCI-DSS tokenisation via PSP integration
- Open Banking account-to-account payments (JARM)
- Idempotency key handling

Catalogue entry: [premium-payment-api](https://backstage-developer-hub-rhdh.apps.cluster-nrk99.dyn.redhatworkshops.io/catalog/default/api/premium-payment-api)

---

## Appendix A — Glossary

| Term | Definition |
|---|---|
| BIAN | Banking Industry Architecture Network — open standard for financial service interoperability |
| BOM | Business Object Model — BIAN's canonical data schema |
| Control Record (CR) | The core stateful entity managed by a BIAN Service Domain |
| Behavior Qualifier (BQ) | An optional sub-component of a Control Record |
| FAPI | Financial-grade API — OpenID Foundation security profile for high-value APIs |
| JARM | JWT Secured Authorization Response Mode — FAPI extension for Open Banking |
| PAN | Primary Account Number — the 16-digit card number |
| ISO 20022 | International standard for financial messaging |
| mTLS | Mutual TLS — both client and server present certificates |
| PCI-DSS | Payment Card Industry Data Security Standard |
