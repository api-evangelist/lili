---
name: lili-track-application-webhooks
description: Register a Lili webhook listener and consume the eleven onboarding lifecycle events, from application submission through document requests to account approval and first funding.
api: Lili Webhooks API
operations:
  - addWebhook
  - getAll
  - getWebhook
  - updateWebhook
  - deleteWebhook
generated: '2026-08-25'
method: generated
source: >-
  Authored by API Evangelist from openapi/lili-webhooks-api-openapi.yml and
  https://dev.lili.co/guides/lili-webhooks. Event names and parameters are transcribed from the
  provider's guide; the full catalog is in asyncapi/lili-webhooks.yml.
---

# Track a Lili application with webhooks

Webhooks are the ONLY way to learn what happened to an application. There is no status endpoint.

## Steps

1. **Register the listener** — `addWebhook`, `PUT /lili/api/v1/webhooks`, body
   `{serverUrl, status, version, type}`.
   - Set `version` to `V2_0`. `V1_0` is labelled deprecated by Lili: it delivers events as an HTTP
     **GET** with per-action query parameters and authenticates with a `lili-secret` header. `V2_0`
     delivers a JSON body over **POST** with `Authorization: Bearer`.
   - **Store the `token` from the response immediately.** It is returned once, at creation, and it
     is the only thing that lets you verify a delivery came from Lili.
2. **Allowlist the source IPs** before going live — sandbox `18.213.104.31, 3.208.116.48,
   34.202.116.80`; production `3.209.35.162, 52.202.86.146, 54.208.152.74`.
3. **Verify every delivery.** Check the bearer token matches the one you stored AND the source IP
   is on the list. The token is a shared secret, not an HMAC over the payload — there is no
   timestamp or nonce, so a captured delivery can be replayed. If replay matters to you, dedupe on
   `personId` + `action` yourself.
4. **Return HTTP 200 fast.** Anything else is treated as a failure and retried 4 times at
   60s, 180s, 600s and 1800s. Acknowledge first, process asynchronously.
5. **Handle the events.** Every delivery carries `personId`, `action` and `token`.
   - `submitApplication`, `idVerificationStart`, `idVerification`, `firstUploadDocs`,
     `idVerificationAdditionalUbos`, `resubmitDocs` — progress; usually just update your record.
   - `firstRequestedDocs` — act on it. `missing_docs` is a CSV of the document types Lili needs;
     collect them and call `uploadAdditionalDoc`.
   - `applicationRejected` — terminal. Do not retry the application.
   - `onboardingComplete` — approved. Carries `bankAccountNumber` and `routingNumber`. **Check
     `isTemp`**: a temporary approval also carries `expirationDate` and `missingDocuments`, and it
     lapses if you ignore them. Treating temp as final is the classic mistake here.
   - `firstMoneyInOf1` — the account was funded for the first time.
   - `payment_reconciled` — carries `paymentId`; reconcile it against your payment records.
6. **Pause rather than delete.** To stop deliveries, prefer `updateWebhook`
   (`POST /lili/api/v1/webhooks/{webhookId}`) toggling `status` — Lili documents this as the normal
   way to disable and later reactivate. `deleteWebhook` is a hard delete and re-registering mints a
   NEW token, invalidating every verification path you have deployed.

## Rules an agent must not break

- Never store `bankAccountNumber` or `routingNumber` outside an encrypted store, and never echo
  them into a log line or an assistant response.
- `getAll` returns every registration with no pagination and no filter. Do not assume the list is
  small or ordered.
