# loyalty-core

Core API for a multi-tenant rewards and loyalty platform (Spring Boot 3, Java 21, PostgreSQL).
Merchants run points programs for their customers: earn points on purchases, redeem them for rewards.

## Design

- **Multi-tenant.** Every tenant-owned table has `merchant_id`. A merchant authenticates with `X-API-Key`; the
  filter binds that merchant to the request, and every query is scoped to it.
- **Append-only ledger.** Balances are never stored or edited. A customer's balance is the sum of
  `ledger_entries.points`; earn, redeem, adjust and expire are all new rows.
- **Safe under concurrency and retries.** Earn and redeem lock the customer row first, so a customer cannot
  overspend with parallel requests. Each write carries an idempotency key (the invoice reference for purchases,
  an `Idempotency-Key` header for redemptions), so a retried request returns the original entry.
- **Rules as data.** The earn rule (`points_per_100`) is a column on the merchant, not code.
- **Closed-loop points.** No cash value, no transfers between customers, no payments.

## Run

```bash
docker compose up -d                       # PostgreSQL
export LOYALTY_ADMIN_KEY=change-me         # enables the admin API
mvn spring-boot:run
mvn test                                   # unit tests (no database needed)
```

Flyway creates the schema from `src/main/resources/db/migration/V1__init.sql` on startup.

## Try it

```bash
# 1. Create a merchant (admin). The API key is shown once.
curl -s -X POST localhost:8080/admin/merchants -H 'X-Admin-Key: change-me' \
  -H 'Content-Type: application/json' -d '{"name":"Food House","pointsPer100":10}'

KEY=lk_...   # from the response

# 2. Add a reward and enrol a customer
curl -s -X POST localhost:8080/v1/rewards -H "X-API-Key: $KEY" -H 'Content-Type: application/json' \
  -d '{"name":"Free meal","pointsCost":200}'
curl -s -X POST localhost:8080/v1/customers -H "X-API-Key: $KEY" -H 'Content-Type: application/json' \
  -d '{"phone":"+201001234567"}'

# 3. Record a purchase (500 EGP earns 50 points) and redeem
curl -s -X POST localhost:8080/v1/purchases -H "X-API-Key: $KEY" -H 'Content-Type: application/json' \
  -d '{"customerId":"<customer id>","amount":500,"invoiceRef":"INV-1001"}'
curl -s -X POST localhost:8080/v1/customers/<customer id>/redemptions -H "X-API-Key: $KEY" \
  -H 'Idempotency-Key: redeem-1' -H 'Content-Type: application/json' -d '{"rewardId":"<reward id>"}'

# 4. Balance and history
curl -s localhost:8080/v1/customers/<customer id>/balance -H "X-API-Key: $KEY"
curl -s localhost:8080/v1/customers/<customer id>/history -H "X-API-Key: $KEY"
```

## Branches

A merchant can have several branches. Once a merchant has at least one branch, every purchase and redemption must
carry a `branchId` (the cashier page shows a branch selector). A merchant with no branches works as before.
`GET /v1/reports/branches` returns, per branch: purchases, distinct customers, repeat customers (two or more
purchases at that branch in the period) and points issued and redeemed. Days are calendar days in Africa/Cairo.
Customers are shared across a merchant's branches, so points earned at one branch can be redeemed at another.

## Web pages (Arabic, mobile-first)

Served by the same app, no build step:

- `/cashier.html`: for the cashier. Enter the merchant API key once (kept in the browser), look up or enrol a
  customer by phone, record a bill, redeem a reward, and add rewards. Shows a QR code and a WhatsApp button that
  send the customer their link.
- `/c.html?t=<token>`: the customer's "my points" page: balance, progress to each reward, recent activity.
  Opened from the QR or link; the token is unguessable, so no login is needed.

The QR code library loads from a CDN; if it is blocked, the WhatsApp link still works.

## Endpoints

| Method | Path | Auth | Purpose |
| --- | --- | --- | --- |
| POST | `/admin/merchants` | `X-Admin-Key` | Create a merchant, returns its API key once |
| POST / GET | `/v1/branches` | `X-API-Key` | Create / list the merchant's branches |
| GET | `/v1/reports/branches?from=&to=` | `X-API-Key` | Per-branch purchases, customers, repeat customers, points (default: last 30 days) |
| GET | `/public/customers/{token}` | none (token) | Customer's own balance, rewards and history |
| POST | `/v1/customers` | `X-API-Key` | Enrol by phone (idempotent) |
| GET | `/v1/customers/{id}/balance` | `X-API-Key` | Current points |
| GET | `/v1/customers/{id}/history` | `X-API-Key` | Ledger entries, newest first |
| POST | `/v1/purchases` | `X-API-Key` | Record a purchase and earn points |
| POST | `/v1/customers/{id}/redemptions` | `X-API-Key` + `Idempotency-Key` | Redeem a reward |
| POST / GET | `/v1/rewards` | `X-API-Key` | Create / list rewards |

## Not yet built (see the product plan roadmap)

Points expiry job, referral rewards, tiers, dashboard queries, webhooks and POS connectors, integration tests
against a real PostgreSQL (Testcontainers), rate limiting, and an audit log. Before real use, add HTTPS-only
deployment and key rotation for merchant API keys.
