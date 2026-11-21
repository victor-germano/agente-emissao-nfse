# NFS-e Autonomous Emission Agent

[![CI](https://github.com/victor-germano/agente-emissao-nfse/actions/workflows/ci.yml/badge.svg)](https://github.com/victor-germano/agente-emissao-nfse/actions/workflows/ci.yml)

> **Autonomous fiscal agent — receives a payment event, automatically matches service provider + client, issues the NFS-e (Brazilian service invoice) with the city hall, sends the PDF by email and notifies the app. Zero manual intervention on the happy path.**

---

## Architecture

```
POST /webhooks/stripe     POST /webhooks/gateway2 (Pix/installments)
        │                               │
        └───────────────┬───────────────┘
                        ▼
           normalizes to PaymentEvent
                        ▼
         InvoiceEmissionService.processPayment
                        │
       ┌────────────────┼──────────────────┐
       ▼                ▼                  ▼
  idempotency    matches provider    creates Invoice "processing"
 (by externalId)    + client
                        │
                        ▼
      NFSeClient.emit()  with exponential retry/backoff
                        │
             ┌──────────┴──────────┐
             ▼                     ▼
        success: "issued"     retries exhausted: "failed"
             │                (lastError + audit trail)
             ▼
    EmailSender + AppNotifier
```

**Architecture decision:** the app receives webhooks directly (no intermediary like n8n). Logic with atomic idempotency and stateful retry in a visual orchestrator becomes a testability bottleneck — versioned, tested code has a real advantage here.

---

## Resilience — What's Implemented

| Mechanism | What it does |
|---|---|
| **Atomic idempotency** | `findOrCreate` with in-memory mutex + `UNIQUE(payment_external_id)` in Postgres — duplicate webhook races are impossible |
| **Exponential backoff retry** | Configurable via `NFSE_MOCK_FAILURE_RATE` / attempts; non-retryable errors (e.g. invalid credentials) abort immediately |
| **Fast ack** | `POST /webhooks/stripe` responds `202` with invoice in `processing` immediately; emission continues in background |
| **Stuck invoice recovery** | `recoverStuckInvoices` scans invoices stuck in `processing` (crash mid-flight) and retries — runs periodically + `POST /debug/recover-stuck` |
| **Reconciliation** | `ReconciliationService` compares approved payments vs. existing invoices to catch missed webhooks |
| **Chargeback cancellation** | `charge.refunded` locates the invoice, cancels the issued NFS-e and notifies the client by email |
| **Graceful shutdown** | `SIGTERM`/`SIGINT` waits for in-flight emissions to finish (5s timeout) before exiting |
| **Real health check** | `GET /health` tests Supabase connectivity; responds `503` if DB unreachable |
| **Endpoint authentication** | `requireApiKey` protects `/invoices`, `/alerts`, `/metrics` and `/debug/*` via `Authorization: Bearer`; constant-time comparison |
| **Debug routes blocked in production** | `NODE_ENV=production` deregisters all `/debug/*` (404) |
| **Immutable audit trail** | Append-only log of each state transition per invoice — queryable via `GET /invoices/:id/audit` |
| **Structured logging** | JSON per event (`{level, message, context, timestamp}`) consumable by Datadog/ELK/CloudWatch |

---

## Stack

| Layer | Technology |
|---|---|
| Runtime | Node.js · TypeScript |
| API | Fastify |
| Persistence | Supabase (managed PostgreSQL) |
| Payments | Stripe webhooks (signature verified) |
| Metrics | `GET /metrics` — issued/failed/cancelled/recovered invoices + average duration |

---

## City Hall Integration

The official SOAP API (`marilia.sigiss.com.br/marilia/ws/sigiss_ws.php`) exists and has a native idempotency field (`id_sis_legado`) compatible with this design. The current implementation uses `PrefeituraMariliaMockClient` (same interface) — switching to `FocusNfeClient` or the direct API means implementing `NFSeClient` and injecting it in `src/index.ts`.

**Practical production recommendation:** a gateway like Focus NFe (only one with explicitly verified Marília support, free sandbox, no lock-in) rather than the direct API — absorbs the cost of tracking Brazil's national NFS-e transition (Lei Complementar 214/2025).

---

## Running locally

```bash
npm install
cp .env.example .env
npm run dev        # http://localhost:3000
```

Without `SUPABASE_URL` + `SUPABASE_SERVICE_ROLE_KEY` configured, the app uses in-memory implementations — works for local development without network dependencies.

**To enable real persistence:** fill in `SUPABASE_SERVICE_ROLE_KEY` in `.env` (from the Supabase dashboard, project already created with schema applied).

**Visual dashboard:** `http://localhost:3000` — simulate payments, force failures, monitor metrics and audit trail in real time.

---

## Tests

```bash
npm run typecheck
npm test
```

Cover: retry/backoff, idempotency (including concurrency via `Promise.all`), decoupled ack, stuck invoice recovery, chargeback/cancellation, reconciliation, Stripe signature (valid/tampered/expired), API key auth, health check, non-retryable errors.

Always run against in-memory implementations — never touch real Supabase.

---

## End-to-end Demo

```bash
npm run dev              # terminal 1
bash scripts/demo.sh      # terminal 2
```

Fires in sequence: Stripe payment, Pix, unstable city hall (retry), inconsistent data, stuck invoice + recovery, missed payment + reconciliation, chargeback. Prints final state of `/invoices`, `/alerts` and `/metrics`.

---

## License

MIT
