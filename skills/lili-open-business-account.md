---
name: lili-open-business-account
description: Create a pre-filled Lili business bank account application for a prospect and hand them a ready-to-finish onboarding URL, then track the application to approval or rejection.
api: Lili Application API
operations:
  - registerAccount
  - updateLead
  - uploadAdditionalDoc
generated: '2026-08-25'
method: generated
source: >-
  Authored by API Evangelist from openapi/lili-application-api-openapi.yml and
  https://dev.lili.co/guides/lili-connect-api. Every operationId below appears verbatim in the
  published contract.
---

# Open a Lili business bank account for a prospect

Use this when a partner platform already holds a small-business customer's details and wants to
open a Lili business checking account for them without re-typing anything.

## Before you start

- You need a partner credential. It is a PAIR: `Authorization: Lili <accessKey>:<secretKey>`.
  Lili issues a separate pair per environment and the key itself does not tell you which
  environment it belongs to — bind the key to the environment in your own config.
- Sandbox is `https://sandbox.lili.co`; production is `https://prod.lili.co`. The published
  contract lists only the sandbox host in `servers[]`.
- Register a webhook FIRST (see `lili-track-application-webhooks`). Without it you have no way to
  learn the outcome, because there is no polling operation for application status.

## Steps

1. **Create the application** — `registerAccount`, `PUT /lili/api/v1/lead`.
   - The only truly required field is `email`. It is the customer identifier: the applicant may
     override every other value during onboarding but not this one. Choose it carefully.
   - Send everything you already know. `firstName`, `lastName`, `ssn`, `birthDate`, address fields,
     `ein`, `businessName`, `businessType`, `incorporationState`, `naicsCode`, `duns`,
     `ownershipPercent` and `uboList` all pre-fill the flow and shorten it.
   - Set `uniqueId` to your own record ID and `metadata` to anything you need back later — both
     round-trip to you in `LeadResponse` (`partnerSuppliedId`, `metadata`).
   - Expect **201**. The response carries `customerId`, `personId`, `location` (the onboarding URL)
     and `token` (a JWT for the embedded flow).
2. **Hand off the applicant.** Either redirect to `location`, open it in a WebView for a native
   app, or embed it with the loader described in `components/lili-components.yml`.
3. **Amend if something changes** — `updateLead`, `POST /lili/api/v1/lead/{customerId}`, same
   `LeadRequest` body.
4. **Supply documents when asked** — `uploadAdditionalDoc`,
   `PUT /lili/api/v1/lead/{customerId}/upload`, with a `documentDataList` of
   `{content, contentType, fileName, llcFileType}`. Drive this off the `firstRequestedDocs`
   webhook, whose `missing_docs` parameter names exactly which documents Lili wants.

## Rules an agent must not break

- **There is no undo.** No operation deletes, cancels, withdraws or voids a lead once created, and
  no operation removes a document already uploaded. Do not call `registerAccount` speculatively.
  Confirm the applicant's identity and intent before the first call.
- **There is no idempotency key.** If `registerAccount` times out, do not blind-retry — the same
  `email` converges, but any other field you changed between attempts will have been applied.
- **Errors are thin.** A failure is `400` with a `JsonErrorResponse`; log `codeName`, which is the
  only stable machine discriminator Lili exposes. Lili publishes no enumeration of its values, so
  build your mapping from observation. Note that the contract binds the error schema to
  `application/xml` and leaves `application/json` empty — that is a spec defect; read the body you
  actually get.
- **Never log the response `token` or the applicant's `ssn`/`passportNumber`.**
