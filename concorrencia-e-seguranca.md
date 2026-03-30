# Plano de Robustez, Segurança e Concorrência — Oraculo

> Perspectiva de engenharia sênior em casa de apostas. Análise baseada no estado atual do
> código (2026-03-23). Cada item tem severidade, probabilidade e ação concreta.

---

## 1. Estado Atual — O que já funciona bem

Antes de listar os problemas, é importante reconhecer o que o sistema já faz certo. Isso
evita retrabalho e ajuda a priorizar o que falta.

| Mecanismo | Onde | O que protege |
|---|---|---|
| Lock condicional atômico no balance | `user-account.repository.ts` | Previne over-locking em ORDER |
| `FOR UPDATE SKIP LOCKED` | `settlement.service.ts` | Previne double-processing de trades em multi-instância |
| CAS lock no payout 3xChange | `execute-payout-3x.use-case.ts` | Previne double-spend no saque |
| Live verification antes do envio | `execute-payout-3x.use-case.ts` | Valida endereço on-chain antes de mover USDC |
| Distributed lock via Redis | `withDistributedLock()` | Serializa settlement batch entre instâncias |
| Ledger com idempotencyKey único | `LedgerEntry` | Previne entradas duplicadas no razão |
| Stale claim recovery (5 min) | `settlement.service.ts` | Libera locks órfãos após crash |
| Rate limiting (Redis + fallback) | Middleware | Limita abuso na camada de API |
| Circuit breaker para RPC 429 | `settlement.service.ts` | Pausa settlement em rate limit do Solana |
| Status machine no Payout | `PayoutStatus` | Garante transições unidirecionais |
| Idempotência via `externalId` | Webhooks de depósito | Previne crédito duplicado de PIX |

---

## 2. Matriz de Riscos

**Legenda**: S = Severidade (1–5) | P = Probabilidade (1–5) | R = Risco = S × P

| # | Risco | S | P | R | Categoria |
|---|---|---|---|---|---|
| R01 | Duplo saque: duas sessões de saque simultâneas do mesmo usuário | 5 | 4 | 20 | **CRÍTICO** |
| R02 | Webhook forjado credita saldo falso (sem verificação de assinatura) | 5 | 3 | 15 | **CRÍTICO** |
| R03 | Float em campos monetários causa erros de arredondamento acumulado | 4 | 5 | 20 | **CRÍTICO** |
| R04 | Idempotência de settlement em Redis puro (dados voláteis, TTL 7 dias) | 4 | 3 | 12 | **ALTO** |
| R05 | Race condition: cancelamento de order durante settlement | 4 | 3 | 12 | **ALTO** |
| R06 | Saldo DB não decrementado na abertura do fluxo de saque | 4 | 4 | 16 | **CRÍTICO** |
| R07 | Nenhum limite diário de saque por usuário | 4 | 4 | 16 | **CRÍTICO** |
| R08 | Exhaustion de conexões PostgreSQL sob carga (sem PgBouncer) | 4 | 4 | 16 | **ALTO** |
| R09 | Settlement idempotency key baseado em buyOrderId+sellOrderId pode colidir | 3 | 2 | 6 | MÉDIO |
| R10 | Sem alerta para itens FAILED na fila de settlement | 3 | 5 | 15 | **ALTO** |
| R11 | Chave PIX não vinculada ao CPF do usuário (permite fraude de saída) | 4 | 3 | 12 | **ALTO** |
| R12 | Order expiry sem cleanup automático (USDC locked para sempre) | 3 | 4 | 12 | **ALTO** |
| R13 | Ausência de reconciliação financeira automatizada diária | 3 | 5 | 15 | **ALTO** |
| R14 | Sem detecção de lavagem (depósito → saque imediato) | 4 | 3 | 12 | **ALTO** |
| R15 | Logs de erros financeiros expõem estado interno (leakage) | 2 | 4 | 8 | MÉDIO |
| R16 | Solana RPC single-point-of-failure sem fallback automático | 3 | 3 | 9 | MÉDIO |
| R17 | ReconcilePayouts3xService não está agendado (apenas manual) | 3 | 5 | 15 | **ALTO** |
| R18 | Sem freeze/bloqueio de conta para usuário suspeito | 4 | 3 | 12 | **ALTO** |
| R19 | CPF/dados pessoais armazenados em plaintext no DB | 3 | 4 | 12 | MÉDIO |
| R20 | Sem proteção contra wash trading no order book | 3 | 3 | 9 | MÉDIO |

---

## 3. Vulnerabilidades Críticas — Fix Imediato

### R01 — Duplo Saque Simultâneo

**O problema:**

O fluxo de saque tem 3 steps: `/quote` → `/authorize` → `/execute`. O saldo do usuário
**não é reservado** ao criar o quote. Se o mesmo usuário abrir duas sessões de saque
simultaneamente para valores que individualmente cabem no saldo mas juntos não:

```
Saldo: 100 USDC
Session A: /quote (80 USDC) → status: payout_pending
Session B: /quote (80 USDC) → status: payout_pending  ← ambos criados com sucesso

Session A: /authorize → payout_order_placed, depositAddress_A
Session B: /authorize → payout_order_placed, depositAddress_B  ← ambos autorizados

Session A: /execute → CAS lock A passa (status=payout_order_placed, lock=NULL) → envia 80 USDC
Session B: /execute → CAS lock B passa (status=payout_order_placed, lock=NULL) → envia 80 USDC
→ TOTAL ENVIADO: 160 USDC. Saldo on-chain: 0. DB ainda mostra 100.
```

**Causa raiz:** O CAS lock em `/execute` protege contra dupla execução do **mesmo payout**,
mas não protege contra dois payouts distintos do mesmo usuário em paralelo.

**Fix:**

```sql
-- Ao criar o quote (CREATE PAYOUT QUOTE), reservar o saldo atomicamente:
UPDATE user_accounts
SET "lockedBalance" = "lockedBalance" + ${amount},
    "lastUpdate" = ${timestamp}
WHERE "userWallet" = ${wallet}
  AND "balance" - "lockedBalance" >= ${amount}
-- Se 0 rows → saldo insuficiente → rejeitar o quote

-- Ao executar (/execute), liberar o locked e debitar o balance:
UPDATE user_accounts
SET "balance" = "balance" - ${amount},
    "lockedBalance" = GREATEST(0, "lockedBalance" - ${amount}),
    "lastUpdate" = ${timestamp}
WHERE "userWallet" = ${wallet}

-- Ao cancelar ou falhar o payout, liberar o locked sem debitar:
UPDATE user_accounts
SET "lockedBalance" = GREATEST(0, "lockedBalance" - ${amount}),
    "lastUpdate" = ${timestamp}
WHERE "userWallet" = ${wallet}
```

Adicionalmente, adicionar constraint única no DB para impedir dois payouts ativos
do mesmo usuário ao mesmo tempo:

```sql
-- Index parcial: um único payout ativo por usuário
CREATE UNIQUE INDEX idx_payouts_one_active_per_user
ON payouts ("userId")
WHERE status NOT IN ('payout_completed', 'payout_failed', 'payout_refund_pending');
```

---

### R02 — Webhook sem Verificação de Assinatura

**O problema:**

Qualquer um com a URL do webhook pode enviar um payload falso e creditar USDC infinito
para uma conta. Atualmente os webhooks são processados sem verificar HMAC/assinatura
da origem.

**Fix por provider:**

```typescript
// middleware/webhook-signature.middleware.ts

// BlindPay — HMAC-SHA256 do body com chave secreta
function verifyBlindpaySignature(req: Request, secret: string): boolean {
  const signature = req.headers.get("x-blindpay-signature");
  const body = await req.text();
  const expected = createHmac("sha256", secret).update(body).digest("hex");
  return timingSafeEqual(Buffer.from(signature ?? ""), Buffer.from(expected));
}

// Bitso — IP allowlist (Bitso assina via IP fixo; verificar documentação)
const BITSO_ALLOWED_IPS = ["34.x.x.x", "52.x.x.x"]; // obter com Bitso

// 3xChange — HMAC-SHA256 (já implementado em webhooks.routes.ts — MANTER)
// Verificar que WEBHOOK_SECRET está sempre configurado em produção

// Princípio geral:
// 1. Verificar assinatura ANTES de parsear o payload
// 2. Retornar 200 OK mesmo em caso de falha (evitar retry storm)
// 3. Logar falhas de assinatura como alerta de segurança
```

Criar tabela de auditoria de webhooks rejeitados:

```prisma
model WebhookAuditLog {
  id           Int      @id @default(autoincrement())
  provider     String
  eventType    String
  externalId   String?
  rawPayload   String   @db.Text
  sourceIp     String?
  signatureOk  Boolean
  rejectedAt   DateTime @default(now())
  reason       String?

  @@index([provider, signatureOk, rejectedAt(sort: Desc)])
}
```

---

### R03 — Float em Campos Monetários

**O problema:**

```prisma
model PixUsdcPayment {
  amount:     Float   // ← ERRADO: 10.1 + 0.2 = 10.299999999999999 em Float
  usdcAmount: Float
  rate:       Float
  usdcAmountSent: Float
  withdrawalFee:  Float
}
```

Float (IEEE 754 64-bit) não pode representar exatamente valores decimais. Em volume alto,
os erros se acumulam e causam divergências na reconciliação financeira.

**Fix:**

```prisma
model PixUsdcPayment {
  amount:         Decimal @db.Decimal(18, 6)
  usdcAmount:     Decimal @db.Decimal(18, 6)
  rate:           Decimal @db.Decimal(18, 8)  // taxa de câmbio precisa de mais casas
  usdcAmountSent: Decimal @db.Decimal(18, 6)
  withdrawalFee:  Decimal @db.Decimal(18, 6)
}
```

Mesmo para `Payout`:
```prisma
model Payout {
  requestAmount:  Decimal @db.Decimal(18, 6)  // era Float
  senderAmount:   Decimal @db.Decimal(18, 6)
  receiverAmount: Decimal @db.Decimal(18, 6)
  netAmountBrl:   Decimal @db.Decimal(18, 6)
}
```

---

### R06 — Saldo não Reservado na Abertura do Saque

Já detalhado em R01. Resumo da ação:

1. `create-payout-quote-3x.use-case.ts` → chamar `lockBalance(wallet, amount)` atomicamente
2. `authorize-payout-3x.use-case.ts` → verificar que payout ainda tem lock ativo
3. `execute-payout-3x.use-case.ts` → ao enviar USDC, liberar locked + debitar balance
4. Qualquer falha/cancelamento → `unlockBalance(wallet, amount)`

---

### R07 — Sem Limite Diário de Saque

Uma conta comprometida pode sacar todo o saldo de uma vez, ou ao longo de poucas
transações no mesmo dia.

**Fix:**

```prisma
model UserWithdrawalLimits {
  id              Int     @id @default(autoincrement())
  userId          Int     @unique
  dailyLimit      Decimal @db.Decimal(18, 6) @default(1000)  // 1000 USDC/dia padrão
  weeklyLimit     Decimal @db.Decimal(18, 6) @default(5000)
  perTxLimit      Decimal @db.Decimal(18, 6) @default(500)
  kycVerified     Boolean @default(false)
  cooldownHours   Int     @default(24)  // após primeiro depósito
  updatedAt       DateTime @updatedAt
}
```

```typescript
// No create-payout-quote use-case, ANTES de criar o payout:
async function validateWithdrawalLimits(userId: number, amount: number): Promise<void> {
  const limits = await getUserLimits(userId);

  // Verificar limite por transação
  if (amount > limits.perTxLimit) {
    throw new AppError(`Limite por transação: ${limits.perTxLimit} USDC`);
  }

  // Verificar total sacado nas últimas 24h
  const dailyTotal = await getWithdrawnLast24h(userId);
  if (dailyTotal + amount > limits.dailyLimit) {
    throw new AppError(`Limite diário atingido: ${limits.dailyLimit} USDC/dia`);
  }

  // Período de carência após primeiro depósito
  const firstDeposit = await getFirstDepositDate(userId);
  if (firstDeposit && hoursSince(firstDeposit) < limits.cooldownHours) {
    const remaining = limits.cooldownHours - hoursSince(firstDeposit);
    throw new AppError(`Aguarde ${remaining}h após o primeiro depósito para sacar`);
  }
}
```

---

## 4. Problemas Altos — Fix em 1–2 Semanas

### R04 — Settlement Idempotency Volátil no Redis

**O problema:**

```typescript
// settlement.service.ts
await redis.set(
  `settlement:done:${buyOrderId}:${sellOrderId}`,
  txSig,
  "EX", 7 * 24 * 3600  // TTL de 7 dias — não é permanente
);
```

Se o Redis for reiniciado ou flushed (deploy, falha de memória, migração), todos os
registros de idempotência somem. O settlement vai reprocessar trades já executados
on-chain, gerando transações duplicadas.

**Fix:**

Salvar `settledTx` diretamente no `SettlementQueueItem` (PostgreSQL, durável):

```prisma
model SettlementQueueItem {
  // ... campos existentes ...
  settledTx       String?   // assinatura Solana da tx que liquidou este item
  settledAt       DateTime? // quando foi settled
}
```

```typescript
// Antes de executar: checar no DB
const existing = await prisma.settlementQueueItem.findUnique({
  where: { tradeId: item.tradeId },
  select: { settledTx: true },
});
if (existing?.settledTx) {
  // Já foi executado — apenas marcar como SETTLED sem re-enviar tx
  return existing.settledTx;
}

// Após executar: salvar no DB (além do Redis como cache)
await prisma.settlementQueueItem.update({
  where: { id: item.id },
  data: { settledTx: txSignature, settledAt: new Date() },
});
```

---

### R05 — Race: Cancelamento de Order During Settlement

**O problema:**

```
T=0: Settlement claims order A (status=open) → CLAIMED
T=1: User cancels order A → API checks status=open → marca como cancelled → unlockBalance
T=2: Settlement executa tx on-chain → order A liquidada on-chain
T=3: Settlement tenta atualizar order A → status=cancelled → reconciliação diverge
T=4: Balance do user foi desbloqueado, mas tokens foram entregues on-chain
```

**Fix:**

Ao cancelar, verificar se o item já está na fila de settlement:

```typescript
// cancel-order.use-case.ts
async function cancelOrder(orderId: string, wallet: string): Promise<void> {
  // Usar transação atômica
  await prisma.$transaction(async (tx) => {
    // SELECT FOR UPDATE para garantir exclusividade
    const order = await tx.$queryRaw`
      SELECT * FROM orders WHERE "orderId" = ${orderId} FOR UPDATE
    `;

    if (order.status !== "open" && order.status !== "pending") {
      throw new Error("Order não pode ser cancelada no status atual");
    }

    // Verificar se está sendo processada no settlement
    const inSettlement = await tx.settlementQueueItem.findFirst({
      where: {
        tradeData: { path: ["buyOrderId"], equals: orderId },
        status: { in: ["CLAIMED", "QUEUED"] },
      },
    });

    if (inSettlement?.status === "CLAIMED") {
      throw new Error("Order está sendo liquidada no momento. Aguarde.");
    }

    // Cancelar e desbloquear atomicamente
    await tx.order.update({
      where: { orderId },
      data: { status: "cancelled" },
    });

    const remainingLocked = order.usdcAmount - order.filled * order.price / 100;
    await tx.$executeRaw`
      UPDATE user_accounts
      SET "lockedBalance" = GREATEST(0, "lockedBalance" - ${remainingLocked})
      WHERE "userWallet" = ${wallet}
    `;
  });
}
```

---

### R08 — Exhaustion de Conexões PostgreSQL

**O problema:**

PostgreSQL tem limite de `max_connections` (tipicamente 100–200 em planos básicos).
Cada instância da API abre um pool. Com 10k usuários simultâneos e múltiplas instâncias:

```
3 instâncias × 30 conexões = 90 conexões
→ Perto do limite → queries começam a falhar com "too many clients"
```

**Fix — PgBouncer em transaction pooling mode:**

```yaml
# docker-compose.pgbouncer.yml
pgbouncer:
  image: pgbouncer/pgbouncer:1.21
  environment:
    DATABASES_HOST: postgres
    DATABASES_PORT: 5432
    PGBOUNCER_POOL_MODE: transaction          # ← crítico para APIs stateless
    PGBOUNCER_MAX_CLIENT_CONN: 5000           # conexões da aplicação → PgBouncer
    PGBOUNCER_DEFAULT_POOL_SIZE: 25           # conexões PgBouncer → Postgres
    PGBOUNCER_SERVER_IDLE_TIMEOUT: 600
```

Em `transaction` mode, a conexão só é ocupada durante a transação (não durante toda a
sessão). Permite 5000 clientes com apenas 25 conexões reais ao Postgres.

**Atenção:** `transaction` mode não suporta `SET`, `PREPARE`, e advisory locks por sessão.
Verificar se `withDistributedLock` usa advisory locks (se usar, mudar para `session` mode
ou implementar via `SELECT pg_try_advisory_xact_lock()` que funciona em transaction mode).

---

### R10 — Sem Alerta para Settlement FAILED

Itens que atingem `retryCount >= MAX_RETRIES` e ficam com `status=FAILED` passam
despercebidos. Usuários ficam com ordens abertas e USDC bloqueado indefinidamente.

**Fix:**

```typescript
// settlement.service.ts — após marcar como FAILED
if (item.retryCount >= MAX_RETRIES) {
  await prisma.settlementQueueItem.update({
    where: { id: item.id },
    data: { status: "FAILED" },
  });

  // Alerta imediato
  await alertService.critical("settlement_permanent_failure", {
    tradeId: item.tradeId,
    marketPda: item.tradeData.marketPda,
    buyer: item.tradeData.buyer,
    seller: item.tradeData.seller,
    quantity: item.tradeData.quantity,
    retryCount: item.retryCount,
    lastError: item.lastError,
  });

  // Liberar as orders envolvidas (para que usuários possam re-operar)
  await releaseFailedTrade(item);
}
```

```typescript
// alert.service.ts
export const alertService = {
  async critical(event: string, data: any) {
    // 1. Log estruturado (Datadog / Grafana)
    loggers.app.error({ alert: event, ...data }, `🚨 ALERTA CRÍTICO: ${event}`);

    // 2. Slack webhook
    await fetch(process.env.SLACK_ALERTS_URL!, {
      method: "POST",
      body: JSON.stringify({
        text: `🚨 *${event}*\n\`\`\`${JSON.stringify(data, null, 2)}\`\`\``,
      }),
    });

    // 3. PagerDuty / OpsGenie para incidentes financeiros
    if (FINANCIAL_ALERTS.includes(event)) {
      await pagerDutyService.trigger(event, data);
    }
  },
};
```

---

### R12 — Orders Expiradas com USDC Bloqueado

Orders com `expiresAt < now()` e status `open` mantêm USDC bloqueado para sempre.

**Fix — Cron job de expiração:**

```typescript
// jobs/expire-orders.job.ts
// Executar a cada 5 minutos

export async function expireOrders(): Promise<void> {
  const now = Math.floor(Date.now() / 1000);

  // Buscar orders expiradas em batch (500 por vez)
  const expired = await prisma.order.findMany({
    where: {
      status: { in: ["open", "pending"] },
      expiresAt: { lte: now, not: null },
    },
    take: 500,
    orderBy: { expiresAt: "asc" },
  });

  if (expired.length === 0) return;

  for (const order of expired) {
    await prisma.$transaction(async (tx) => {
      // Marcar como expirada
      await tx.order.update({
        where: { orderId: order.orderId },
        data: { status: "expired" },
      });

      // Desbloquear USDC (somente para BUY orders)
      if (order.type === "buy") {
        const remainingLocked = Number(order.usdcAmount) - Number(order.filled) * Number(order.price) / 100;
        if (remainingLocked > 0) {
          await tx.$executeRaw`
            UPDATE user_accounts
            SET "lockedBalance" = GREATEST(0, "lockedBalance" - ${remainingLocked})
            WHERE "userWallet" = ${order.userWallet}
          `;

          await publishLedgerEvent({
            entryType: "order_unlock_expired",
            userWallet: order.userWallet,
            amount: remainingLocked,
            orderId: order.orderId,
            idempotencyKey: `order_unlock_expired:${order.orderId}`,
          });
        }
      }
    });
  }

  loggers.app.info({ count: expired.length }, "🕐 Orders expiradas liberadas");
}
```

---

### R17 — ReconcilePayouts3xService não está agendado

O serviço existe mas depende de chamada manual. Se o servidor reiniciar, reconciliação
para de rodar.

**Fix — adicionar ao startup da aplicação:**

```typescript
// app.ts ou server.ts — após inicialização
import { CronJob } from "cron";

// Reconciliação 3xChange a cada 5 minutos
new CronJob("*/5 * * * *", async () => {
  try {
    await reconcilePayouts3xService.run();
  } catch (err: any) {
    loggers.app.error({ error: err.message }, "Reconciliação 3xChange falhou");
  }
}, null, true, "America/Sao_Paulo");

// Expiração de orders a cada 5 minutos
new CronJob("*/5 * * * *", async () => {
  try {
    await expireOrders();
  } catch (err: any) {
    loggers.app.error({ error: err.message }, "Expiração de orders falhou");
  }
}, null, true, "America/Sao_Paulo");

// Reconciliação financeira diária às 03:00
new CronJob("0 3 * * *", async () => {
  try {
    await dailyReconciliationService.run();
  } catch (err: any) {
    loggers.app.error({ error: err.message }, "Reconciliação diária falhou");
    await alertService.critical("daily_reconciliation_failed", { error: err.message });
  }
}, null, true, "America/Sao_Paulo");
```

---

### R18 — Sem Bloqueio de Conta

**Fix — campo `status` no User:**

```prisma
model User {
  // ... campos existentes ...
  accountStatus   UserAccountStatus @default(ACTIVE)
  frozenAt        DateTime?
  frozenReason    String?
}

enum UserAccountStatus {
  ACTIVE
  FROZEN        // Bloqueado para depósitos/saques/trades
  RESTRICTED    // Pode ver mas não operar
  SUSPENDED     // Banido permanentemente
}
```

```typescript
// middleware/account-status.middleware.ts
export async function requireActiveAccount(c: Context, next: Next) {
  const user = c.get("user");
  if (user.accountStatus !== "ACTIVE") {
    return c.json({ error: "Conta bloqueada. Entre em contato com o suporte." }, 403);
  }
  return next();
}

// Aplicar em todas as rotas financeiras:
// POST /orders, POST /quote, POST /authorize, POST /execute
```

Endpoint admin para freeze:
```typescript
// POST /api/v1/admin/users/:userId/freeze
adminRoutes.post("/users/:userId/freeze", async (c) => {
  const { reason } = c.req.valid("json");
  await prisma.user.update({
    where: { id: Number(c.req.param("userId")) },
    data: {
      accountStatus: "FROZEN",
      frozenAt: new Date(),
      frozenReason: reason,
    },
  });
  // Cancelar todas as orders abertas do usuário
  await cancelAllOpenOrders(userId);
  return c.json({ ok: true });
});
```

---

## 5. Melhorias de Arquitetura — Médio Prazo

### 5.1 Reconciliação Financeira Diária

Todo sistema financeiro sério tem um relatório de saldo ao final do dia que verifica:

```
Saldo esperado = Σ(depósitos) - Σ(saques) + Σ(ganhos em mercado) - Σ(perdas)
Saldo real (on-chain) = consulta ao Solana
Divergência aceitável = 0 (ou < 0.01 USDC por erros de arredondamento já existentes)
```

```typescript
// services/daily-reconciliation.service.ts

interface ReconciliationReport {
  date: string;
  totalUserBalances: number;       // Σ UserAccount.balance
  totalLockedBalances: number;     // Σ UserAccount.lockedBalance
  totalDepositsToday: number;      // Σ depósitos confirmados no dia
  totalWithdrawalsToday: number;   // Σ saques concluídos no dia
  totalSettlementsToday: number;   // Σ settlements executados no dia
  pendingPayouts: number;          // Payouts em payout_order_placed ou payout_usdc_sent
  pendingSettlements: number;      // SettlementQueue items QUEUED ou CLAIMED
  ledgerBalance: number;           // Saldo calculado via double-entry ledger
  divergence: number;              // |totalUserBalances - ledgerBalance|
  divergenceAcceptable: boolean;   // divergence < 0.01
}

export class DailyReconciliationService {
  async run(): Promise<ReconciliationReport> {
    const [
      { _sum: { balance, lockedBalance } },
      depositTotal,
      withdrawalTotal,
    ] = await Promise.all([
      prisma.userAccount.aggregate({ _sum: { balance: true, lockedBalance: true } }),
      prisma.pixUsdcPayment.aggregate({
        where: { status: "completed", updatedAt: { gte: startOfDay() } },
        _sum: { usdcAmountSent: true },
      }),
      prisma.payout.aggregate({
        where: { status: "payout_completed", updatedAt: { gte: startOfDay() } },
        _sum: { senderAmount: true },
      }),
    ]);

    const ledgerBalance = await calculateLedgerBalance(); // Σ credits - Σ debits
    const divergence = Math.abs(Number(balance) - ledgerBalance);

    const report: ReconciliationReport = { /* ... */ divergence, divergenceAcceptable: divergence < 0.01 };

    if (!report.divergenceAcceptable) {
      await alertService.critical("financial_reconciliation_divergence", {
        divergence,
        totalUserBalances: balance,
        ledgerBalance,
      });
    }

    // Salvar relatório para auditoria
    await prisma.reconciliationReport.create({ data: report as any });

    return report;
  }
}
```

---

### 5.2 Anti-Fraude e Detecção de Lavagem

Em casas de apostas, o padrão mais comum de fraude é:

1. **Depósito → Saque imediato** (teste de cartão clonado)
2. **Wash trading** (comprar e vender contra si mesmo para manipular preços)
3. **Multi-conta** (mesma pessoa com várias contas para burlar limites)
4. **Bonus abuse** (se houver promoções)

```typescript
// services/fraud-detection.service.ts

export class FraudDetectionService {
  // Verifica se o usuário tentou sacar em menos de 24h após o depósito
  async checkDepositWithdrawalPattern(userId: number, withdrawalAmount: number): Promise<FraudCheck> {
    const recentDeposits = await prisma.pixUsdcPayment.findMany({
      where: {
        userId,
        status: "completed",
        updatedAt: { gte: new Date(Date.now() - 24 * 60 * 60 * 1000) },
      },
    });

    const recentDepositTotal = recentDeposits.reduce((sum, d) => sum + Number(d.usdcAmountSent), 0);

    if (recentDepositTotal > 0 && withdrawalAmount >= recentDepositTotal * 0.8) {
      return {
        suspicious: true,
        reason: "Saque >= 80% do depósito nas últimas 24h",
        action: "REQUIRE_MANUAL_REVIEW",  // ou BLOCK
      };
    }

    return { suspicious: false };
  }

  // Detecta wash trading: mesmo usuário nos dois lados de uma trade
  async checkWashTrading(buyerWallet: string, sellerWallet: string): Promise<boolean> {
    // Verificar se buyer e seller têm o mesmo userId (multi-conta detectada via PIX)
    const [buyer, seller] = await Promise.all([
      prisma.user.findFirst({ where: { walletAddress: buyerWallet }, select: { pixKey: true } }),
      prisma.user.findFirst({ where: { walletAddress: sellerWallet }, select: { pixKey: true } }),
    ]);

    if (buyer?.pixKey?.pixKey && buyer.pixKey.pixKey === seller?.pixKey?.pixKey) {
      await alertService.critical("wash_trading_detected", { buyerWallet, sellerWallet });
      return true;
    }

    return false;
  }

  // Verifica se chave PIX já está em uso por outra conta
  async checkPixKeyUniqueness(pixKey: string, userId: number): Promise<boolean> {
    const existing = await prisma.pixKey.findFirst({
      where: { pixKey, userId: { not: userId } },
    });

    if (existing) {
      await alertService.critical("duplicate_pix_key_detected", {
        pixKey: maskPixKey(pixKey),
        userId,
        existingUserId: existing.userId,
      });
      return false;
    }

    return true;
  }
}
```

---

### 5.3 Solana RPC com Fallback Automático

**O problema:** Um único RPC (ex: Helius) pode ter downtime, 429, ou degradação.
Em produção com volume alto, a API fica completamente parada.

```typescript
// config/solana.config.ts — RPC com fallback

const RPC_ENDPOINTS = [
  process.env.SOLANA_RPC_URL!,           // Helius (primário)
  process.env.SOLANA_RPC_FALLBACK_1!,    // QuickNode (fallback)
  process.env.SOLANA_RPC_FALLBACK_2!,    // Alchemy ou Triton (terceiro)
].filter(Boolean);

class ResilientConnection {
  private connections: Connection[];
  private currentIndex = 0;
  private failureCounts = new Map<string, number>();

  constructor(endpoints: string[]) {
    this.connections = endpoints.map((url) => new Connection(url, "confirmed"));
  }

  get current(): Connection {
    return this.connections[this.currentIndex];
  }

  async withFallback<T>(fn: (conn: Connection) => Promise<T>): Promise<T> {
    for (let i = 0; i < this.connections.length; i++) {
      const idx = (this.currentIndex + i) % this.connections.length;
      const conn = this.connections[idx];

      try {
        const result = await fn(conn);
        this.currentIndex = idx; // voltar para este endpoint
        return result;
      } catch (err: any) {
        const is429 = err?.message?.includes("429") || err?.status === 429;
        const isDown = err?.message?.includes("ECONNREFUSED") || err?.message?.includes("timeout");

        if ((is429 || isDown) && i < this.connections.length - 1) {
          loggers.app.warn({ endpoint: conn.rpcEndpoint, error: err.message }, "RPC falhou, tentando fallback");
          continue;
        }
        throw err;
      }
    }
    throw new Error("Todos os RPC endpoints falharam");
  }
}

export const resilientConnection = new ResilientConnection(RPC_ENDPOINTS);
```

---

### 5.4 Queue de Saques com Processamento Assíncrono

Atualmente o fluxo de saque é síncrono: o usuário aguarda o HTTP response enquanto
o servidor assina e envia a transação Solana. Se a transação demorar, o request faz timeout.

**Arquitetura ideal:**

```
POST /execute  →  [Validação + CAS lock]  →  PUT na fila (RabbitMQ/BullMQ)
                                                ↓
                                          Worker assíncrono
                                                ↓
                                     Sign + Send USDC on-chain
                                                ↓
                                       Webhook 3xChange
                                                ↓
                                     WebSocket notification ao usuário
```

```typescript
// use-cases/withdraw/execute-payout-3x.use-case.ts
// Após CAS lock bem-sucedido:

await rabbitmqService.publish("payout.execution.queue", {
  payoutId: payout.id,
  userId: payout.userId,
  walletAddress: payout.userWallet,
  depositAddress: payout.depositAddress,
  senderAmount: payout.senderAmount,
  paymentReference: payout.paymentReference,
  retryCount: 0,
});

// Retornar imediatamente ao usuário
return {
  status: "processing",
  message: "Saque sendo processado. Você será notificado quando concluído.",
  payoutId: payout.id,
};
```

---

### 5.5 Read Replicas para Consultas de Alta Frequência

O endpoint `GET /markets` e `GET /orders/book/:marketPda` são chamados centenas de
vezes por segundo em cada market maker ativo. Isso pressiona a primary DB desnecessariamente.

```typescript
// database/prisma-client.ts
import { PrismaClient } from "@prisma/client";

// Primary — escrita
export const prisma = new PrismaClient({
  datasources: { db: { url: process.env.DATABASE_URL } },
});

// Read replica — leitura
export const prismaRead = new PrismaClient({
  datasources: { db: { url: process.env.DATABASE_REPLICA_URL ?? process.env.DATABASE_URL } },
});
```

Usar `prismaRead` em:
- `GET /markets` (getAllMarkets)
- `GET /markets/:pda` (getMarket)
- `GET /orders/book/:pda` (order book display)
- `GET /users/:wallet/positions`
- Qualquer query de listagem/read-only

Usar `prisma` (primary) em:
- Todas as escritas financeiras
- Order placement
- Payout execution

---

## 6. Observabilidade e Monitoramento

### 6.1 Métricas Mínimas Necessárias

```typescript
// metrics/financial.metrics.ts (Prometheus / Datadog)

// Contadores
counter("settlement_items_processed_total", { status: "settled" | "failed" });
counter("orders_placed_total", { type: "buy" | "sell", side: "yes" | "no" });
counter("payouts_executed_total", { provider: "3xchange" | "blindpay", status });
counter("deposits_processed_total", { provider, status });
counter("webhook_signature_failures_total", { provider });

// Histogramas (latência)
histogram("settlement_batch_duration_ms");
histogram("payout_execute_duration_ms");
histogram("order_placement_duration_ms");
histogram("solana_tx_confirmation_ms");

// Gauges (estado atual)
gauge("settlement_queue_depth", { status: "QUEUED" | "CLAIMED" | "FAILED" });
gauge("payouts_stuck_count");         // status=payout_usdc_sent há > 30 min
gauge("orders_expired_pending_count");
gauge("total_user_balance_usdc");
gauge("total_locked_balance_usdc");
```

### 6.2 Alertas Essenciais

| Alerta | Threshold | Severidade | Ação |
|---|---|---|---|
| Settlement queue FAILED | > 0 items | P1 | PagerDuty imediato |
| Payout stuck | > 5 min em payout_usdc_sent | P1 | PagerDuty imediato |
| Reconciliação divergência | > 0.01 USDC | P1 | PagerDuty imediato |
| Settlement queue depth | > 500 QUEUED | P2 | Slack alerta |
| RPC 429 rate | > 50/min | P2 | Slack alerta |
| Webhook signature failure | > 3/min do mesmo IP | P2 | Slack + bloquear IP |
| Saque > 5x média do usuário | | P2 | Manual review |
| DB connection pool > 80% | | P2 | Slack alerta |
| Erro de arredondamento detectado | qualquer | P1 | PagerDuty |

### 6.3 Health Check Financeiro

```typescript
// routes/health.routes.ts
healthRoutes.get("/financial", async (c) => {
  const [
    settlementFailed,
    payoutsStuck,
    queueDepth,
  ] = await Promise.all([
    prisma.settlementQueueItem.count({ where: { status: "FAILED" } }),
    prisma.payout.count({
      where: {
        status: { in: ["payout_usdc_sent", "payout_executing"] },
        updatedAt: { lte: new Date(Date.now() - 30 * 60 * 1000) },
      },
    }),
    prisma.settlementQueueItem.count({ where: { status: "QUEUED" } }),
  ]);

  const healthy = settlementFailed === 0 && payoutsStuck === 0;

  return c.json({
    healthy,
    settlementFailed,
    payoutsStuck,
    queueDepth,
    timestamp: new Date().toISOString(),
  }, healthy ? 200 : 503);
});
```

---

## 7. Segurança Adicional

### 7.1 Dados Pessoais (LGPD)

Chaves PIX (CPF, CNPJ, telefone) são dados pessoais sensíveis. Atualmente em plaintext.

```typescript
// utils/crypto.ts
import { createCipheriv, createDecipheriv, randomBytes } from "crypto";

const ENCRYPTION_KEY = Buffer.from(process.env.PIX_KEY_ENCRYPTION_KEY!, "hex"); // 32 bytes

export function encryptPixKey(value: string): string {
  const iv = randomBytes(16);
  const cipher = createCipheriv("aes-256-gcm", ENCRYPTION_KEY, iv);
  const encrypted = Buffer.concat([cipher.update(value, "utf8"), cipher.final()]);
  const tag = cipher.getAuthTag();
  return `${iv.toString("hex")}:${tag.toString("hex")}:${encrypted.toString("hex")}`;
}

export function decryptPixKey(encrypted: string): string {
  const [ivHex, tagHex, dataHex] = encrypted.split(":");
  const iv = Buffer.from(ivHex, "hex");
  const tag = Buffer.from(tagHex, "hex");
  const decipher = createDecipheriv("aes-256-gcm", ENCRYPTION_KEY, iv);
  decipher.setAuthTag(tag);
  return decipher.update(Buffer.from(dataHex, "hex")).toString("utf8") + decipher.final("utf8");
}
```

### 7.2 Logs sem Vazamento de Dados

```typescript
// utils/log-sanitizer.ts
const SENSITIVE_FIELDS = ["pixKey", "cpf", "cnpj", "phone", "depositAddress", "walletAddress"];

export function sanitizeForLog(obj: any): any {
  if (!obj || typeof obj !== "object") return obj;

  return Object.fromEntries(
    Object.entries(obj).map(([key, value]) => {
      if (SENSITIVE_FIELDS.some((f) => key.toLowerCase().includes(f.toLowerCase()))) {
        return [key, typeof value === "string" ? `${value.slice(0, 4)}****` : "[REDACTED]"];
      }
      return [key, sanitizeForLog(value)];
    }),
  );
}

// Usar em todos os loggers.app.info/error/warn
loggers.app.info(sanitizeForLog({ pixKey, wallet, amount }), "Payout iniciado");
```

### 7.3 Rate Limiting por Operação Financeira

```typescript
// Limites específicos por tipo de operação
const RATE_LIMITS = {
  "POST /quote":         { window: "1m", max: 5  },   // 5 quotes por minuto
  "POST /authorize":     { window: "1m", max: 3  },   // 3 autorizações por minuto
  "POST /execute":       { window: "5m", max: 3  },   // 3 execuções por 5 minutos
  "POST /orders":        { window: "1m", max: 30 },   // já existe — manter
  "POST /deposits":      { window: "1m", max: 3  },   // 3 depósitos por minuto
  "POST /register-pix":  { window: "1h", max: 3  },   // 3 trocas de chave PIX por hora
};
```

### 7.4 Idempotency Key no Header para Todas as Operações de Escrita

```typescript
// middleware/idempotency.middleware.ts
// O cliente envia "Idempotency-Key: <uuid>" no header
// O servidor processa apenas uma vez e cacheia o response por 24h

export async function idempotencyMiddleware(c: Context, next: Next) {
  const key = c.req.header("Idempotency-Key");
  if (!key) return next(); // opcional para operações não-financeiras

  const cacheKey = `idempotency:${key}`;
  const cached = await redis.get(cacheKey);

  if (cached) {
    return c.json(JSON.parse(cached), 200); // replay da resposta anterior
  }

  await next();

  const status = c.res.status;
  if (status >= 200 && status < 300) {
    const body = await c.res.json();
    await redis.set(cacheKey, JSON.stringify(body), "EX", 86400); // 24h
    return c.json(body, status);
  }
}

// Aplicar em:
// POST /orders
// POST /quote
// POST /authorize
// POST /execute
// POST /deposits/pix
```

---

## 8. Plano de Implementação

### Sprint 1 — Crítico (Esta Semana)

| Tarefa | Arquivo principal | Impacto |
|---|---|---|
| Lock de saldo ao criar payout quote | `create-payout-quote-3x.use-case.ts` + `user-account.repository.ts` | Bloqueia R01, R06 |
| Verificação de assinatura 3xChange webhook | `webhooks.routes.ts` (já parcial) | Bloqueia R02 |
| Adicionar `settledTx` no SettlementQueueItem | `schema.prisma` + `settlement.service.ts` | Bloqueia R04 |
| Limite diário de saque (básico) | novo `withdrawal-limits.service.ts` | Bloqueia R07 |
| Migração Float → Decimal em PixUsdcPayment + Payout | `schema.prisma` + migration SQL | Bloqueia R03 |
| Cron jobs de reconciliação e expiração de orders | `app.ts` ou `server.ts` | Bloqueia R12, R17 |

### Sprint 2 — Alto (Próximas 2 Semanas)

| Tarefa | Impacto |
|---|---|
| PgBouncer em produção | R08 |
| Alertas Slack/PagerDuty para settlement FAILED | R10 |
| Campo `accountStatus` no User + endpoint freeze | R18 |
| Cancelamento atômico de order (SELECT FOR UPDATE) | R05 |
| Log sanitization em todos os loggers financeiros | R15 |
| Idempotency key middleware | Resiliência geral |

### Sprint 3 — Arquitetura (1 Mês)

| Tarefa | Impacto |
|---|---|
| Reconciliação financeira diária automatizada | R13 |
| Detecção de wash trading e lavagem | R14, R20 |
| Solana RPC com fallback automático | R16 |
| Encriptação de chaves PIX (AES-256-GCM) | R19 |
| Read replicas para queries de listagem | Escalabilidade |
| Queue assíncrona para execução de saques | Resiliência |
| Verificação de assinatura BlindPay e Bitso | R02 (completo) |

### Sprint 4 — Escala (2–3 Meses)

| Tarefa | Impacto |
|---|---|
| Load testing com 10k usuários simultâneos | Validação |
| CQRS para order book (read model separado) | Performance |
| Multi-RPC load balancing | Disponibilidade |
| KYC para saques acima de R$5.000 | Compliance |
| Relatórios financeiros para BACEN/CVM | Compliance |
| Runbooks documentados para incidents | Operacional |

---

## 9. Checklist de Revisão por Operação Financeira

Use este checklist ao revisar ou implementar qualquer operação que envolva dinheiro:

```
[ ] A operação é atômica? (única query/transação no DB)
[ ] Existe idempotency key? (previne replay em retry)
[ ] O saldo é verificado e reservado ANTES de qualquer I/O externo?
[ ] Em caso de falha, o saldo é liberado corretamente?
[ ] A operação é registrada no ledger com double-entry?
[ ] Existe mecanismo de reconciliação se o servidor morrer a meio caminho?
[ ] A assinatura do webhook externo é verificada antes de processar?
[ ] Os dados sensíveis estão mascarados nos logs?
[ ] Existe rate limiting específico para esta operação?
[ ] O comportamento em falha do serviço externo está testado?
```

---

## 10. Resumo Executivo

O sistema Oraculo tem uma base sólida para alta concorrência: locks condicionais atômicos,
`FOR UPDATE SKIP LOCKED` no settlement, CAS lock no payout, ledger com idempotencyKey.
A maioria dos mecanismos críticos já existe e funciona.

Os **três riscos que podem causar perda financeira real hoje** são:

1. **R01/R06 — Duplo saque**: um usuário pode abrir duas sessões de saque simultâneas
   e sacar mais do que tem. Fix: reservar saldo ao criar o quote.

2. **R02 — Webhook sem assinatura**: qualquer um pode forjar um depósito PIX.
   Fix: verificar HMAC de cada provider antes de processar.

3. **R03 — Float em campos monetários**: erros de arredondamento acumulam em volume.
   Fix: migrar todos os campos `Float` financeiros para `Decimal`.

Com esses três resolvidos, o sistema pode operar com segurança em escala. Os demais
itens melhoram a resiliência, detecção de fraude e observabilidade ao longo do tempo.
