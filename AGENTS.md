# AGENTS.md

Sellora — single-vendor digital-goods store. Go 1.26 + Chi + `html/template` + HTMX + PostgreSQL (`pgx/v5`). Modular monolith; no SPA, no new infra (brokers, microservices) without a concrete requirement.

## Commands

Setup order matters — migrate, then seed, then serve:

```bash
cp .env.example .env
go run cmd/migrate/main.go   # apply SQL migrations (embedded via migrations/migrations.go)
go run cmd/seed/main.go       # idempotent; override via SEED_ADMIN_EMAIL / SEED_ADMIN_PASSWORD / SEED_ADMIN_NAME
go run cmd/server/main.go
curl http://localhost:8080/healthz
```

Verify (mirrors CI in `.github/workflows/ci.yml`):

```bash
go test ./...          # unit tests need no DB; all packages pass
go test -race ./...    # CI runs with -race
go vet ./...
CGO_ENABLED=0 go build ./cmd/server ./cmd/migrate ./cmd/seed
```

Docker: `docker compose up --build` (db → one-shot `migrate` → `app`). `Dockerfile` builds all three binaries.

## Layout

- `cmd/{server,migrate,seed}/main.go` — entrypoints only, no business logic.
- `internal/http/router.go` — all routes + middleware stack. `internal/http/middleware.go` — CSRF, security headers, logging, panic recovery.
- `internal/{auth,product,order,customer}/` — handler (thin) + service (rules) + repository (SQL). Services never render HTML; handlers never run SQL.
- `internal/payment/` — `provider.go` factory + `midtrans.go`, `lynk.go`, `mock.go` + `service.go` (webhook flow) + `handler.go`. `internal/domain/` — models + `PaymentProvider` interface + order statuses. Never import Chi/HTTP into `internal/domain`.
- `internal/{config,database,storage,email,view}/` — config, pgx pool + `MigrateUp`, file storage, mail sender, template renderer.
- `migrations/*.sql` — append a new numbered file for schema changes; never edit an applied migration.
- `web/templates/` (embedded via `templates.go`: `layouts/` + `pages/`) and `web/static/` (served at `/static/*`). No frontend build step.

## Payment / orders (highest-risk area)

- `PAYMENT_PROVIDER` (`config.PaymentProvider`, default `mock`) selects `midtrans | lynk | mock` via `payment.NewProvider`. Business logic must depend only on the `domain.PaymentProvider` interface; provider specifics stay in `internal/payment/`.
- Prod guards in `config.Load()` fail startup: `mock` provider banned in production, `SESSION_SECRET` required, `MIDTRANS_SERVER_KEY` required for midtrans. `/mock-checkout*` routes exist only when `!IsProduction() && provider == "mock"`.
- Only `POST /webhooks/{provider}` (verified server-to-server event) may mark an order paid. Never trust the client return page (`/orders/{reference}/success` is display-only). `ProcessWebhook` enforces: provider-name match → signature verify → `payment_events` idempotency check → amount-equals-total check on paid → atomic `ProcessPaymentResult` (status update + event insert in one tx).
- Order FSM (`order/service.go:156`): `pending → paid|failed|cancelled`; `paid → refunded` only; `failed|cancelled|refunded` are terminal. Same-state update is a no-op. Keep status mutations in the order/payment path, not scattered handlers.
- Webhook retry idempotency on side-effects (Anti-Double-Count): Payment providers routinely retry webhooks. While database status updates are idempotent no-ops when already paid, all payment side-effects (`handleOrderPaid`, receipt email dispatch, coupon `used_count` increment) MUST verify that the order was NOT ALREADY in `StatusPaid` before executing (`wasAlreadyPaid := order.Status == domain.StatusPaid; ... if status == domain.StatusPaid && !wasAlreadyPaid`). Never trigger side-effects on same-state duplicate events across any status mutation entry points (`ProcessPaymentResult`, `UpdateOrderStatus`, `SyncPaymentStatus`).
- Defense-in-depth validation & calculations: Never rely solely on frontend/form sanitization. Service layers must enforce business constraints (`MaxDiscountAmount >= 0`, `MinPurchaseAmount >= 0`, `Percent >= 0 && Percent <= 100`). Discount calculation (`domain.CalculateDiscount`) must defensively clamp to `[0, subtotal]` and never produce negative discounts (which inflate order totals). For providers requiring itemized line totals (Midtrans Snap), discounts MUST be passed as an item detail with negative price (`ID: "DISCOUNT", Price: -order.DiscountAmount, Quantity: 1`) so that the sum of `item_details` strictly matches `gross_amount`.
- Order items snapshot price at purchase; product price edits must not rewrite history.

## Security / HTTP gotchas

- `/healthz` returns `503` with `"status":"degraded"` when DB ping fails (orchestrator relies on it). Server `log.Fatalf` on DB failure in production but starts degraded in dev — don't "fix" dev startup to hard-fail.
- CSRF is double-submit cookie (`sellora_csrf`); POST/PUT/PATCH/DELETE need `X-CSRF-Token` or `HX-CSRFToken` header or `csrf_token`/`_csrf` field. `/webhooks/*` is exempt (HMAC-verified) — keep it that way.
- Downloads (`GET /downloads/{fileID}`) require auth + paid-order ownership (`HasAccessToProduct`); files live under `STORAGE_PATH` with random internal names, never a user-supplied path. `DeleteProduct` deletes the DB row first (FK blocks deleting purchased products) and only then removes files — keep that order.
- Uploads: validate size/MIME/extension, never use the client filename as a path. SQL: parameterized `$1` queries only, no string concatenation. Never log passwords, session tokens, or provider secrets.
- Admin = `RequireAuth` + `RequireRole(admin)` in the router; hiding links is not authorization. Sessions are opaque DB tokens with an hourly cleanup worker; logout must invalidate.
- Handler error handling: Never swallow service/database errors with blank identifiers (`_ = service.Delete(...)`). Always log errors and return appropriate HTTP error status codes to the client.

## Conventions

- Language separation: Storefront (customer-facing UI, messages, checkout errors) is strictly in **Bahasa Indonesia**. Admin dashboard (navigation, tables, forms, buttons) is strictly in **English**.
- Template pointer handling: Go `html/template` does not support the asterisk pointer dereference operator (`*`). Never write `{{ * .PointerField }}` in templates. Use helper functions from `view.FuncMap()` (e.g. `derefInt`, `derefInt64`, `derefTime`, `formatDateTimeInput`).
- HTMX endpoints return HTML fragments, not JSON. Keep logic out of templates.
- HTMX form state synchronization: When partial swaps (e.g. `hx-target="#checkout-summary"` with `hx-swap="outerHTML"`) update dynamic components, all form fields that must be submitted by the main form (such as coupon codes or discount state) must remain synchronized. Put the input field inside the swapped component using HTML5 `form="<form-id>"` (e.g. `<input type="hidden" name="coupon_code" form="checkout-form">`), or use `hx-swap-oob`. Never let a partial swap leave the submitting form with stale or empty state (e.g. UI showing discount applied but form submitting full price).
- SMTP unset → `LogSender` (logs mail to console); set `SMTP_HOST` for real delivery. Receipts fire on transition to paid (`notifyOrderPaid`).
- Env comes from `.env` via godotenv with in-code defaults; never commit `.env` or secrets. `PRD.md` is the feature source of truth; `TODO.md` tracks deferred work (coupons, reviews, S3/R2).
