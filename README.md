# Agente de Emissão NFS-e

[![CI](https://github.com/victor-germano/agente-emissao-nfse/actions/workflows/ci.yml/badge.svg)](https://github.com/victor-germano/agente-emissao-nfse/actions/workflows/ci.yml)

> **Agente autônomo de emissão fiscal — recebe evento de pagamento, associa automaticamente prestador + tomador, emite NFS-e junto à Prefeitura, envia PDF por e-mail e notifica o App. Zero intervenção manual no fluxo feliz.**

---

## Arquitetura

```
POST /webhooks/stripe     POST /webhooks/gateway2 (Pix/parcelado)
        │                               │
        └───────────────┬───────────────┘
                        ▼
           normaliza para PaymentEvent
                        ▼
         InvoiceEmissionService.processPayment
                        │
       ┌────────────────┼──────────────────┐
       ▼                ▼                  ▼
  idempotência   associa prestador   cria Invoice "processing"
 (por externalId)   + tomador
                        │
                        ▼
      NFSeClient.emit()  com retry/backoff exponencial
                        │
             ┌──────────┴──────────┐
             ▼                     ▼
        sucesso: "issued"     falhas esgotadas: "failed"
             │                (lastError + auditoria)
             ▼
    EmailSender + AppNotifier
```

**Decisão de arquitetura:** o app recebe webhooks diretamente (sem intermediário como n8n). Lógica com idempotência atômica e retry com estado em um orquestrador visual vira um gargalo de testabilidade — código versionado e testado tem vantagem real aqui.

---

## Resiliência — o que está implementado

| Mecanismo | O que faz |
|---|---|
| **Idempotência atômica** | `findOrCreate` com mutex em memória + `UNIQUE(payment_external_id)` no Postgres — corrida de webhooks duplicados é impossível |
| **Retry com backoff exponencial** | Configurable em `NFSE_MOCK_FAILURE_RATE` / tentativas; erros não-retryable (ex: credencial inválida) interrompem imediatamente |
| **Ack rápido** | `POST /webhooks/stripe` responde `202` com invoice em `processing` imediatamente; emissão continua em background |
| **Recuperação de travadas** | `recoverStuckInvoices` varre invoices presas em `processing` (crash mid-flight) e retenta — roda periódicamente + `POST /debug/recover-stuck` |
| **Reconciliação** | `ReconciliationService` compara pagamentos aprovados vs. invoices existentes para pegar webhooks perdidos |
| **Cancelamento por estorno** | `charge.refunded` localiza a invoice, cancela NFS-e emitida e notifica o tomador por e-mail |
| **Graceful shutdown** | `SIGTERM`/`SIGINT` espera emissões em voo terminarem (timeout 5s) antes de encerrar |
| **Health check real** | `GET /health` testa conectividade com Supabase; responde `503` se banco inacessível |
| **Autenticação nos endpoints** | `requireApiKey` protege `/invoices`, `/alerts`, `/metrics` e `/debug/*` via `Authorization: Bearer`; comparação em tempo constante |
| **Rotas de debug bloqueadas em produção** | `NODE_ENV=production` desregistra todos os `/debug/*` (404) |
| **Auditoria imutável** | Log append-only de cada transição de estado por invoice — consultável via `GET /invoices/:id/audit` |
| **Logging estruturado** | JSON por evento (`{level, message, context, timestamp}`) consumível por Datadog/ELK/CloudWatch |

---

## Stack

| Camada | Tecnologia |
|---|---|
| Runtime | Node.js · TypeScript |
| API | Fastify |
| Persistência | Supabase (PostgreSQL gerenciado) |
| Pagamentos | Stripe webhooks (assinatura verificada) |
| Métricas | `GET /metrics` — notas emitidas/falhas/canceladas/recuperadas + duração média |

---

## Integração com Prefeitura

A API SOAP oficial de Marília (`marilia.sigiss.com.br/marilia/ws/sigiss_ws.php`) existe e tem campo nativo de idempotência (`id_sis_legado`) compatível com este design. A implementação atual usa `PrefeituraMariliaMockClient` (mesma interface) — trocar por `FocusNfeClient` ou API direta é implementar `NFSeClient` e injetar em `src/index.ts`.

**Recomendação prática para produção:** gateway como Focus NFe (único com suporte a Marília verificado explicitamente, sandbox gratuito, sem trava de contrato) em vez de API direta — absorve o custo de acompanhar a transição nacional da NFS-e (Lei Complementar 214/2025).

---

## Rodando localmente

```bash
npm install
cp .env.example .env
npm run dev        # http://localhost:3000
```

Sem `SUPABASE_URL` + `SUPABASE_SERVICE_ROLE_KEY` configuradas, o app usa implementações in-memory — funciona para desenvolvimento local sem depender de rede.

**Para ativar persistência real:** preencha `SUPABASE_SERVICE_ROLE_KEY` no `.env` (obtida no dashboard do Supabase, projeto já criado e com schema aplicado).

**Painel visual:** `http://localhost:3000` — simula pagamentos, força falhas, monitora métricas e auditoria em tempo real.

---

## Testes

```bash
npm run typecheck
npm test
```

Cobrem: retry/backoff, idempotência (incluindo concorrência via `Promise.all`), ack desacoplado, recuperação de travadas, estorno/cancelamento, reconciliação, assinatura Stripe (válida/adulterada/expirada), autenticação por API key, health check, erros não-retryable.

Rodam sempre contra implementações in-memory — não tocam o Supabase real.

---

## Demo end-to-end

```bash
npm run dev              # terminal 1
bash scripts/demo.sh      # terminal 2
```

Dispara em sequência: pagamento Stripe, Pix, prefeitura instável (retry), dado inconsistente, nota travada + recuperação, pagamento perdido + reconciliação, estorno. Imprime estado final de `/invoices`, `/alerts` e `/metrics`.

---

## Licença

MIT
