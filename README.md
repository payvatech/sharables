# Payva Merchant Onboarding API

## Overview

Payva provides an API and embeddable JavaScript widget that allows platforms to onboard their merchants. Platforms submit merchant data to Payva, receive a token, and embed a Payva hosted iframe where the merchant authenticates and links their account.

---

## How It Works

```
1. Platform calls POST /merchant/token with merchant data and x-api-key header
2. Payva returns a token 
3. Platform passes the token to the Payva JS widget
4. Widget opens an iframe where the merchant authenticates (phone OTP or email/password)
5. Payva creates or links the merchant account, transfers application data
6. Widget fires a callback to the platform with the result
7. If application data is incomplete, the merchant is prompted to finish onboarding in a new tab
```

### Step-by-step

**Platform Server → Payva API:**
```
POST /merchant/token
Header: x-api-key: <your_platform_api_key>
Body: { merchant data — see payload below }

Response:
{
  "token": "mtk_a1b2c3d4e5f6...",
  "application_url": "https://app.payva.com/merchant-application/mtk_a1b2c3d4e5f6...",
  "expires_at": "2026-03-26T12:00:00.000Z"
}
```

**Platform frontend → Payva JS widget:**
```js
const payva = new Payva();

payva.on("applicationComplete", (data) => {
  // Merchant successfully linked 
});

payva.on("applicationFailure", (data) => {
  // Something went wrong
});

payva.on("applicationClose", () => {
  // Merchant dismissed the modal without completing
});

payva.initiateApplication({
  token: "mtk_a1b2c3d4e5f6...",
  application_url: "https://app.payva.com/merchant-application/mtk_a1b2c3d4e5f6...",
  expires_at: "2026-03-26T12:00:00.000Z"
});
```

---

## Token Details

- Tokens are valid for **7 days** from creation
- Each token can only be used **once** — after the merchant completes the flow, the token is marked as consumed
- Platforms can check token status via `GET /merchant/token/:token/status`

**Status values:** `pending`, `consumed`, `expired`

---

## Payload Reference

### Applicant vs Owner — How Account Creation Works

The `applicant` is the person who will go through the iframe and create (or link) their Payva account. This might be the business owner, or it might be an administrator, bookkeeper, or developer acting on behalf of the business.

**How Payva resolves who to create the account for:**

1. If `applicant` is provided, Payva uses the applicant's `first_name`, `last_name`, and `email` for account creation.
2. If `applicant` is not provided but `owner[0]` exists, Payva falls back to the first owner's details for account creation.
3. If the applicant IS an owner, you can either:
   - Set `applicant.role` to `owner` and also include them in the `owner` array (with their KYC data like SSN, DOB, etc.)
   - Or just provide them in `owner[0]` and omit `applicant` entirely — Payva will use `owner[0]` as the applicant.

**Example — applicant is the owner (two equivalent approaches):**

*Approach A: Explicit applicant + owner*
```json
{
  "applicant": { "first_name": "John", "last_name": "Doe", "email": "john@acme.com", "role": "owner" },
  "owner": [{ "first_name": "John", "last_name": "Doe", "email": "john@acme.com", "ssn": "123-45-6789", ... }]
}
```

*Approach B: Owner only (applicant omitted, Payva uses owner[0])*
```json
{
  "owner": [{ "first_name": "John", "last_name": "Doe", "email": "john@acme.com", "ssn": "123-45-6789", ... }]
}
```

**Example — applicant is NOT the owner:**
```json
{
  "applicant": { "first_name": "Sarah", "last_name": "Admin", "email": "sarah@acme.com", "role": "administrator" },
  "owner": [{ "first_name": "John", "last_name": "Doe", "email": "john@acme.com", "ssn": "123-45-6789", ... }]
}
```
Sarah gets the Payva account with admin permissions. John's data is stored for KYC/underwriting but he doesn't need to go through the iframe.

---

### Required Fields

These fields must be present or the request will be rejected with a 400 error.

| Field | Type | Description |
|-------|------|-------------|
| `external_merchant_id` | string | Your platform's unique identifier for this merchant. Used for webhook callbacks and status lookups. |
| `applicant` | object | The person going through the iframe. **Required unless `owner` is provided** — if omitted, Payva uses `owner[0]`. |
| `applicant.first_name` | string | Applicant's first name. |
| `applicant.last_name` | string | Applicant's last name. |
| `applicant.email` | string | Applicant's email address. Used for account creation. |
| `business` | object | Business information (see Business below). |
| `business.legal_name` | string | Registered legal name of the business as it appears on tax documents. |

> **Note:** Either `applicant` or `owner` (with at least `first_name`, `last_name`, and `email` on `owner[0]`) must be provided. If both are provided, `applicant` takes priority for account creation.

### Optional Fields — Applicant

| Field | Type | Description |
|-------|------|-------------|
| `applicant.phone_number` | string | Applicant's phone number in E.164 format. |
| `applicant.role` | enum | Applicant's role at the business. Values: `owner`, `administrator`, `bookkeeper`, `developer`, `other`. Determines their permissions on the Payva account. Defaults to `owner` if not specified. |

### Optional Fields — Top Level

| Field | Type | Description |
|-------|------|-------------|
| `redirect_url` | string | URL to redirect the merchant to after completion. Optional if using the iframe widget with postMessage callbacks. |
| `webhook_url` | string | URL to receive async notifications for application status changes. |
| `mode` | enum | UI theme for the iframe. Values: `light`, `dark`. |
| `ip_address` | string | IP address of the merchant at time of submission. Used for fraud/compliance logging. |
| `beneficial_owners_complete` | boolean | Confirms all individuals with 25%+ ownership are listed (FinCEN requirement). |

### Optional Fields — Consent

| Field | Type | Description |
|-------|------|-------------|
| `consent.terms_accepted_at` | string | ISO 8601 timestamp of when the merchant accepted Payva Terms of Service. |
| `consent.ofac_consent` | boolean | Whether the merchant consents to OFAC/sanctions screening. |

> **Note:** Additional consents (credit check authorization, E-Sign agreement, Privacy Policy, ACH debit authorization) are collected by Payva directly during the onboarding process. These require Payva's specific legal language and cannot be collected by platforms on Payva's behalf.

### Optional Fields — Owner

The `owner` array is optional. Each entry represents a beneficial owner with 25%+ ownership stake. If the applicant is also an owner, include them here as well with their KYC data (SSN, DOB, etc.). All fields per owner are optional.

| Field | Type | Description |
|-------|------|-------------|
| `first_name` | string | Owner's first name. |
| `last_name` | string | Owner's last name. |
| `email` | string | Owner's email address (separate from the applicant email). |
| `phone_number` | string | E.164 format (e.g. `+15551234567`). |
| `citizenship` | string | US citizen, permanent resident, or non-resident alien. |
| `address` | object | Residential address (see Address below). |
| `dob` | string | Date of birth in `YYYY-MM-DD` format. |
| `ssn` | string | Social Security Number in `XXX-XX-XXXX` format. |
| `title` | string | Role in the business (e.g. CEO, Partner, Member). |
| `is_control_prong` | boolean | Whether this person has significant management responsibility (FinCEN). |
| `is_politically_exposed` | boolean | PEP screening — government official, family member, or close associate. |
| `ownership_percent` | number | Whole number (e.g. `50` for 50%). |
| `identity_document` | object | Government-issued ID (see Identity Document below). |

### Optional Fields — Business

| Field | Type | Description |
|-------|------|-------------|
| `entity_type` | enum | `sole_proprietorship`, `llc`, `corporation`, `partnership`, `nonprofit`. |
| `tax_id` | string | EIN in `XX-XXXXXXX` format, or SSN for sole proprietors. |
| `dba_name` | string | Doing Business As name if different from legal name. |
| `state_of_incorporation` | string | US state where the entity was formed. |
| `start_year` | number | Year established (e.g. `2019`). |
| `number_of_employees` | number | Total employee count. |
| `registered_agent` | string | Registered agent for service of process. |
| `phone_number` | string | Business phone in E.164 format. |
| `address` | object | Principal place of business (see Address below). |
| `email` | string | Business email address. |
| `industry` | string | General category (e.g. `retail`, `healthcare`). |
| `mcc_code` | string | 4-digit Merchant Category Code. |
| `business_model` | enum | `ecommerce`, `in_store`, `both`. |
| `product_type` | enum | `physical_goods`, `digital_goods`, `services`. |
| `offer_description` | string | Description of products or services offered. |
| `website` | string | Business website URL. |
| `annual_revenue` | number | Estimated annual revenue in USD. |
| `average_ticket_size` | number | Average transaction amount in USD. |
| `monthly_volume` | number | Expected monthly transaction volume in USD. |
| `refund_policy` | string | Return/refund policy. |
| `fulfillment_timeline` | string | Average days between purchase and delivery. |
| `recurring_billing` | boolean | Whether the merchant offers subscriptions. |
| `documents` | object | Supporting documents — EIN letter, formation docs, business license (file references or URLs). |

### Optional Fields — Primary Contact

A separate contact person managing the integration (may or may not be an owner).

| Field | Type | Description |
|-------|------|-------------|
| `primary_contact.first_name` | string | Contact first name. |
| `primary_contact.last_name` | string | Contact last name. |
| `primary_contact.email` | string | Contact email. |
| `primary_contact.phone_number` | string | Contact phone in E.164 format. |
| `primary_contact.title` | string | Job title or role. |

### Optional Fields — Processing History

| Field | Type | Description |
|-------|------|-------------|
| `processing_history.current_processor` | string | Current payment processor name. |
| `processing_history.has_been_terminated` | boolean | Whether the merchant has ever had an account terminated or placed on MATCH/TMF. |
| `processing_history.chargeback_history` | string | Chargeback ratio or summary. |

### Optional Fields — Bank Account

If your platform already has the merchant's banking information, you can pass it here. Otherwise, the merchant will complete Plaid Link during Payva onboarding.

| Field | Type | Description |
|-------|------|-------------|
| `bank_account.plaid_access_token` | string | If provided, Payva pulls account info, identity, and bank data directly from Plaid. This is the preferred method. |
| `bank_account.account_holder_name` | string | Name on the bank account. Optional if `plaid_access_token` is provided. |
| `bank_account.routing_number` | string | 9-digit ABA routing number. Optional if `plaid_access_token` is provided. |
| `bank_account.account_number` | string | Bank account number. Optional if `plaid_access_token` is provided. |
| `bank_account.account_type` | enum | `checking` or `savings`. Optional if `plaid_access_token` is provided. |

### Address Object (used in owner.address and business.address)

| Field | Type | Description |
|-------|------|-------------|
| `address_line_1` | string | Street address. |
| `address_line_2` | string | Suite, unit, etc. |
| `city` | string | City name. |
| `state` | string | 2-letter state abbreviation (US). |
| `zip` | string | ZIP or postal code. |
| `country` | string | ISO 3166-1 alpha-2 (e.g. `US`). |

### Identity Document Object (used in owner.identity_document)

| Field | Type | Description |
|-------|------|-------------|
| `id_type` | enum | `drivers_license`, `passport`, `state_id`. |
| `id_number` | string | Document number. |
| `id_issuing_state` | string | Issuing US state (for domestic IDs). |
| `id_issuing_country` | string | Issuing country (ISO 3166-1 alpha-2). |
| `id_expiration_date` | string | Expiration date in `YYYY-MM-DD` format. |

---

## Minimum Viable Request

```json
{
  "external_merchant_id": "merchant_123",
  "applicant": {
    "first_name": "Sarah",
    "last_name": "Admin",
    "email": "sarah@acmecorp.com"
  },
  "business": {
    "legal_name": "Acme Corp LLC"
  }
}
```

This creates a token and starts the flow. The applicant (Sarah) will go through the iframe to create her Payva account. Owner details, banking, and everything else can be filled in during Payva's onboarding process.

## Full Request Example

```json
{
  "external_merchant_id": "merchant_12345",
  "mode": "light",
  "ip_address": "192.168.1.1",
  "beneficial_owners_complete": true,
  "consent": {
    "terms_accepted_at": "2026-03-18T12:00:00.000Z",
    "ofac_consent": true
  },
  "applicant": {
    "first_name": "Sarah",
    "last_name": "Admin",
    "email": "sarah@acmecorp.com",
    "phone_number": "+15559998888",
    "role": "administrator"
  },
  "primary_contact": {
    "first_name": "John",
    "last_name": "Doe",
    "email": "john@acmecorp.com",
    "phone_number": "+15551234567",
    "title": "CEO"
  },
  "owner": [
    {
      "first_name": "John",
      "last_name": "Doe",
      "email": "john@acmecorp.com",
      "phone_number": "+15551234567",
      "citizenship": "US",
      "address": {
        "address_line_1": "123 Main St",
        "address_line_2": "Suite 100",
        "city": "Austin",
        "state": "TX",
        "zip": "78701",
        "country": "US"
      },
      "dob": "1985-06-15",
      "ssn": "123-45-6789",
      "title": "CEO",
      "is_control_prong": true,
      "is_politically_exposed": false,
      "ownership_percent": 100,
      "identity_document": {
        "id_type": "drivers_license",
        "id_number": "DL123456789",
        "id_issuing_state": "TX",
        "id_issuing_country": "US",
        "id_expiration_date": "2028-06-15"
      }
    }
  ],
  "business": {
    "entity_type": "llc",
    "tax_id": "12-3456789",
    "legal_name": "Acme Corp LLC",
    "dba_name": "Acme",
    "state_of_incorporation": "TX",
    "start_year": 2019,
    "number_of_employees": 25,
    "phone_number": "+15559876543",
    "address": {
      "address_line_1": "456 Business Ave",
      "city": "Austin",
      "state": "TX",
      "zip": "78702",
      "country": "US"
    },
    "email": "info@acmecorp.com",
    "industry": "retail",
    "mcc_code": "5411",
    "business_model": "ecommerce",
    "product_type": "physical_goods",
    "offer_description": "Premium outdoor gear and camping equipment",
    "website": "https://acmecorp.com",
    "annual_revenue": 2500000,
    "average_ticket_size": 150,
    "monthly_volume": 200000,
    "refund_policy": "30-day full refund",
    "fulfillment_timeline": "3-5 business days",
    "recurring_billing": false
  },
  "processing_history": {
    "current_processor": "Stripe",
    "has_been_terminated": false,
    "chargeback_history": "< 0.5%"
  },
  "bank_account": {
    "plaid_access_token": null,
    "account_holder_name": "Acme Corp LLC",
    "routing_number": "021000021",
    "account_number": "123456789012",
    "account_type": "checking"
  }
}
```

---

## What Happens With the Data

- **Data the platform provides** is stored temporarily and pre-fills the merchant's Payva onboarding application
- **The more data provided**, the less the merchant has to fill in manually — a complete payload means the merchant can skip directly to review/automated approval.
- **Banking data** can be provided via a Plaid access token (preferred) or via manual account/routing numbers which will require manual review by our team and likely have to communicate with the merchant
- **If data is incomplete**, the merchant is shown a prompt(hyperlink) after authentication to finish their application in Payva's onboarding form
- **Sensitive data** (SSN, bank account numbers, etc.) all application data is proxied directly to our vault provider.
- **Consent data** is stored alongside the application. If the platform collected consent (terms, credit check, e-sign, privacy policy, ACH authorization), it carries through to the merchant's record. Missing consents are collected during onboarding.

---

## Authentication

All API requests require an `x-api-key` header. This key is issued to each platform and identifies which platform is making the request. Contact Payva to receive your API key.

```
x-api-key: your-platform-api-key-here
```

---

## Notes

- **`primary_contact` vs `applicant`:** The `applicant` is the person going through the iframe to create their account. The `primary_contact` is an optional secondary contact for the integration (e.g. if the CTO handles the technical setup but a different person manages the account day-to-day). If you only have one contact person, use `applicant` — you don't need both.
- **Date formats:** Dates should be in `YYYY-MM-DD` format for DOB and ID expiration fields. Timestamps (like `consent.terms_accepted_at`) should be ISO 8601 (e.g. `2026-03-18T12:00:00.000Z`).
