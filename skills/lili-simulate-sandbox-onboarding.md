---
name: lili-simulate-sandbox-onboarding
description: Drive a Lili sandbox application end-to-end — document verification, LLC documents, KYB vendor and compliance approval — so an integration can be tested against a real approved-account outcome without a human in the loop.
api: Lili Application API
operations:
  - registerAccount
generated: '2026-08-25'
method: generated
source: >-
  Authored by API Evangelist from https://dev.lili.co/guides/lili-connect-api ("Automating Sandbox
  Onboarding Testing"). The simulation endpoint is published by Lili and is not described in any
  OpenAPI document, so it has no operationId to reference.
---

# Simulate a full Lili onboarding in the sandbox

Lili's sandbox lets you stand in for the vendors and people whose real responses cannot be
reproduced in a test environment. Use it to prove your integration handles the whole state machine.

## Steps

1. **Register a sandbox webhook** first (see `lili-track-application-webhooks`, with the sandbox IP
   allowlist), so you can observe each transition rather than guessing at it.
2. **Create a lead** — `registerAccount`, `PUT https://sandbox.lili.co/lili/api/v1/lead`. Keep the
   `customerId` from the 201 response.
3. **Submit the onboarding form** by following the `location` URL returned by that call.
4. **Advance the application** by POSTing to the simulation endpoint once per stage, in this order:

   `POST https://sandbox.lili.co/lili/api/v1/sandbox/{customerId}/{activity}`

   | order | activity | simulates |
   | --- | --- | --- |
   | 1 | `docv_upload` | Customer document verification — passport plus selfie |
   | 2 | `llc_doc_upload` | LLC document upload; all required documents treated as uploaded |
   | 3 | `kyb_provider_upload` | The KYB vendor response, approving the business |
   | 4 | `compliance_approve` | Lili compliance team approval |

5. **Assert on the webhooks, not on the calls.** A correct run walks
   `submitApplication` → `idVerification*` → `firstRequestedDocs` → `firstUploadDocs` →
   `onboardingComplete`. Assert that your handler read `isTemp`, `expirationDate` and
   `missingDocuments` from `onboardingComplete`.

## Rules an agent must not break

- **Sandbox only.** This endpoint exists on `sandbox.lili.co`. Never point a simulation call at
  `prod.lili.co`.
- Lili publishes **no test SSNs, EINs, cards or bank accounts** — the sandbox works by simulating
  vendor DECISIONS, not by accepting magic input values. Do not invent "known good" test
  identifiers; supply plausible arbitrary data and drive the outcome with the activities above.
- Sandbox and production credentials are different key pairs with no distinguishing prefix. Assert
  on your configured base URL before every call.
