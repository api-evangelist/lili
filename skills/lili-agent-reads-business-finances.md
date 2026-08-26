---
name: lili-agent-reads-business-finances
description: Connect an AI assistant to the Lili MCP server over OAuth and answer a business owner's or accountant's questions about balances, transactions, invoices, bills, taxes and statements.
api: Lili MCP Server
operations:
  - lili_get_account_summary
  - lili_search_transactions
  - lili_list_transaction_categories
  - lili_get_financial_summary
  - lili_list_invoices
  - lili_get_invoice
  - lili_list_bills
  - lili_get_tax_bucket
  - lili_estimate_taxes
  - lili_list_statements
  - lili_download_statement
  - list_customers
  - select_customer
generated: '2026-08-25'
method: generated
source: >-
  Authored by API Evangelist from https://dev.lili.co/mcp and
  https://dev.lili.co/guides/lili-mcp-connect. Every tool name above is transcribed verbatim from
  Lili's published tool reference; the full 44-tool list is in mcp/lili-mcp.yml.
---

# Read a Lili business's finances as an agent

The Lili MCP server is remote and OAuth-protected: `https://mcp.lili.co/mcp`, Streamable HTTP, MCP
spec 2025-03-26. Point any MCP client at that URL — there is no Lili-published package to install.

## Connect

1. Configure the server URL. The OAuth flow is authorization-code with PKCE (S256) and dynamic
   client registration, discoverable at
   `https://mcp.lili.co/.well-known/oauth-authorization-server`.
2. The user logs in with their Lili credentials and grants access. Refresh tokens are issued, so
   this is normally once per device.
3. **Understand what consent means here.** The authorization server declares NO scopes. One
   approval grants the agent all 44 tools — full transaction history, statements, tax filings and
   beneficial owners — and for an accountant, every connected client. Say so plainly before asking
   the user to approve, and revoke via `https://mcp.lili.co/oauth/revoke` when done.

## Establish who you are acting for

- **Business user** — the account owner. Omit `businessUserId` on every call; you are already
  scoped to their own account.
- **Accountant** — call `list_customers` to enumerate clients who granted access, then
  `select_customer` with the chosen `customerExternalId`. Most tools then need `businessUserId`;
  the invoice and financial-summary tools want `customerExternalId` instead. Getting the wrong
  identifier on the wrong tool is the most common failure.

## Common flows

- **"What's my balance?"** — `lili_get_account_summary`. Returns routing number, masked account
  number, balances in cents and USD, and sub-accounts (Tax Bucket, BusinessBuild collateral).
- **"Show me software spend last quarter."** — `lili_list_transaction_categories` first to get the
  machine `code`, then `lili_search_transactions` with `startDate`, `endDate`, `category` and
  optionally `minAmountUsd`. Paginate with `page` (1-based) and `recordCnt` (max 100).
- **"How was Q1?"** — `lili_get_financial_summary` with `chartType` INCOME | EXPENSE | PROFIT and
  `granularity` MONTH | QUARTER | YEAR. `isBusiness` does not apply to PROFIT.
- **"What am I owed?"** — `lili_list_invoices` with a status filter, then `lili_get_invoice` for
  line items and the payment link.
- **"What do I owe?"** — `lili_list_bills` (`status` PAID | PENDING | CANCELED | ALL, sorted by
  `DUE_DATE` or `CREATE_DATE`), then `lili_get_bill` and `lili_get_bill_payment_methods`.
- **"How much tax should I set aside?"** — `lili_get_tax_bucket` for what is already reserved,
  `lili_estimate_taxes` for the projection.
- **"Send me last month's statement."** — `lili_list_statements`, then `lili_download_statement`;
  the PDF comes back base64-encoded with a suggested filename.

## Rules an agent must not break

- **Everything here is a read.** There is no MCP tool that moves money, pays a bill, sends an
  invoice or opens an account. If a user asks you to pay something, say you cannot and point them
  at the Lili app. Do not reach for the partner REST API — that is a different product with
  different credentials.
- **Pagination is not uniform.** Accountant and invoice tools use `page`/`size` from page 0;
  transaction and tax-bucket tools use `page`/`recordCnt` from page 1; bill tools use
  `pageNumber`/`pageSize` from page 0. Special-case per family; do not assume.
- **Handle the error codes**: `AUTHENTICATION_REQUIRED` (re-authorize), `ACCESS_DENIED` (check
  `lili_get_client_access_status`), `INVALID_PARAMETER` (dates must be `YYYY-MM-DD`, enums exact),
  `NOT_FOUND` (re-resolve the ID from its listing tool), `RATE_LIMITED` (back off — Lili publishes
  no limit, no window and no `Retry-After`, so use your own exponential backoff),
  `INTERNAL_ERROR` (retry, then contact dev@lili.co).
- **Never echo account numbers, routing numbers, DUNS, SSNs or beneficial-owner details** into a
  transcript, a log, or a downstream tool call. Card PAN/CVV/expiry are never returned at all.
