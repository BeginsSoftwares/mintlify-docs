# Plano de Implementação — Robustez e Segurança
> Baseado em `concorrencia-e-seguranca.md` | Iniciado em 2026-03-23
> R19 (encriptação PIX AES-256-GCM) excluído do escopo.

**Status de cada task:** `[ ]` pendente · `[~]` em andamento · `[x]` concluído

---

## Sprint 1 — Crítico (Semana 1)
> Risco de perda financeira real se não resolvido. Executar antes de qualquer outra coisa.

---

### TASK-01 · R01 + R06 — Lock de saldo no /quote e debit no /execute
**Risco bloqueado:** Duplo saque simultâneo / saldo não reservado
**Complexidade:** Média

**Arquivos a modificar:**
- `api/src/modules/payments/use-cases/withdraw/create-payout-quote-3x.use-case.ts`
- `api/src/modules/payments/use-cases/withdraw/execute-payout-3x.use-case.ts`
- `api/src/modules/payments/use-cases/withdraw/authorize-payout-3x.use-case.ts`
- `api/src/shared/repositories/user-account.repository.ts`
- `api/prisma/schema.prisma` (index parcial)
- Nova migration SQL: `api/prisma/scripts/feat-payout-balance-lock.sql`

**O que fazer:**
1. Em `user-account.repository.ts`, adicionar:
   - `lockBalanceForPayout(wallet, amount)` — UPDATE condicional (`WHERE balance - lockedBalance >= amount`)
   - `debitAndUnlockForPayout(wallet, amount)` — debita balance + libera locked atomicamente
   - `unlockBalanceForPayout(wallet, amount)` — apenas libera locked (para falha/cancelamento)

2. Em `create-payout-quote-3x.use-case.ts`:
   - Antes de criar o Payout record, chamar `lockBalanceForPayout(wallet, senderAmount)`
   - Se retornar 0 rows → lançar `AppError("Saldo insuficiente para saque")`
   - Salvar `balanceLockedAt` no Payout (para reconciliação de locks órfãos)

3. Em `execute-payout-3x.use-case.ts`:
   - Após envio bem-sucedido da tx Solana, chamar `debitAndUnlockForPayout(wallet, amount)`
   - Em qualquer falha permanente, chamar `unlockBalanceForPayout` e setar status=payout_failed

4. Em `authorize-payout-3x.use-case.ts`:
   - Verificar que o Payout existe e tem status=payout_pending (lock ativo)
   - Se já expirado ou cancelado → liberar lock e rejeitar

5. Migration SQL:
   ```sql
   -- Garantir que só existe 1 payout ativo por usuário
   CREATE UNIQUE INDEX IF NOT EXISTS idx_payouts_one_active_per_user
   ON payouts ("userId")
   WHERE status NOT IN ('payout_completed', 'payout_failed', 'payout_refund_pending');
   ```

**Critério de aceite:**
- [ ] Dois `/quote` simultâneos com saldo insuficiente para os dois → segundo retorna 400
- [ ] Falha no `/execute` → locked liberado, usuário pode iniciar novo saque
- [ ] Index parcial criado e testado com `EXPLAIN`

---

### TASK-02 · R02 — Verificação de assinatura dos webhooks
**Risco bloqueado:** Webhook forjado creditando saldo falso
**Complexidade:** Baixa–Média (depende da documentação de cada provider)

**Arquivos a modificar:**
- `api/src/modules/payments/routes/webhooks.routes.ts`
- `api/src/shared/utils/env.ts` (novas env vars)
- `api/prisma/schema.prisma` (nova tabela WebhookAuditLog)
- Nova migration SQL: `api/prisma/scripts/feat-webhook-audit-log.sql`

**O que fazer:**

1. Criar `api/src/modules/payments/middleware/webhook-signature.middleware.ts`:
   ```
   verifyBlindpaySignature(rawBody, signature, secret) → boolean
   verifyBitsoSignature(rawBody, sourceIp, allowedIps) → boolean
   // 3xChange já tem HMAC — apenas centralizar aqui
   ```
   - Sempre usar `timingSafeEqual` para comparar hashes (previne timing attacks)
   - Ler o body como raw bytes ANTES de qualquer parse de JSON

2. Em `webhooks.routes.ts`:
   - Antes de qualquer handler: verificar assinatura
   - Se falhar: logar em `WebhookAuditLog` com `signatureOk=false` + retornar 200 OK (não expor falha ao atacante; evitar retry storm)
   - Se passar: logar com `signatureOk=true` e processar normalmente

3. Novas env vars:
   ```
   BLINDPAY_WEBHOOK_SECRET=
   BITSO_ALLOWED_IPS=34.x.x.x,52.x.x.x   # obter com Bitso
   ```

4. Schema Prisma — novo model:
   ```prisma
   model WebhookAuditLog {
     id          Int      @id @default(autoincrement())
     provider    String
     eventType   String
     externalId  String?
     rawPayload  String   @db.Text
     sourceIp    String?
     signatureOk Boolean
     reason      String?
     createdAt   DateTime @default(now())
     @@index([provider, signatureOk, createdAt(sort: Desc)])
   }
   ```

**Critério de aceite:**
- [ ] POST com payload forjado (sem header de assinatura) → 200 OK mas não credita nada
- [ ] WebhookAuditLog registra a tentativa com signatureOk=false
- [ ] Webhook legítimo ainda funciona normalmente

---

### TASK-03 · R03 — Float → Decimal em campos monetários
**Risco bloqueado:** Erros de arredondamento acumulados em volume
**Complexidade:** Média (breaking change — atenção com o código TypeScript existente)

**Arquivos a modificar:**
- `api/prisma/schema.prisma`
- Nova migration SQL: `api/prisma/scripts/feat-decimal-monetary-fields.sql`
- Todos os arquivos que leem/escrevem nesses campos (buscar com grep)

**O que fazer:**

1. No schema Prisma, mudar em `PixUsdcPayment`:
   ```
   amount          Float  →  Decimal @db.Decimal(18, 6)
   usdcAmount      Float  →  Decimal @db.Decimal(18, 6)
   rate            Float  →  Decimal @db.Decimal(18, 8)
   usdcAmountSent  Float  →  Decimal @db.Decimal(18, 6)
   withdrawalFee   Float  →  Decimal @db.Decimal(18, 6)
   ```

2. Em `Payout`:
   ```
   requestAmount   Float  →  Decimal @db.Decimal(18, 6)
   senderAmount    Float  →  Decimal @db.Decimal(18, 6)
   receiverAmount  Float  →  Decimal @db.Decimal(18, 6)
   netAmountBrl    Float  →  Decimal @db.Decimal(18, 6)
   ```

3. Migration SQL:
   ```sql
   ALTER TABLE "pix_usdc_payments"
     ALTER COLUMN "amount" TYPE DECIMAL(18,6) USING "amount"::DECIMAL(18,6),
     ALTER COLUMN "usdcAmount" TYPE DECIMAL(18,6) USING "usdcAmount"::DECIMAL(18,6),
     ALTER COLUMN "rate" TYPE DECIMAL(18,8) USING "rate"::DECIMAL(18,8),
     ALTER COLUMN "usdcAmountSent" TYPE DECIMAL(18,6) USING "usdcAmountSent"::DECIMAL(18,6),
     ALTER COLUMN "withdrawalFee" TYPE DECIMAL(18,6) USING "withdrawalFee"::DECIMAL(18,6);

   ALTER TABLE "payouts"
     ALTER COLUMN "requestAmount" TYPE DECIMAL(18,6) USING "requestAmount"::DECIMAL(18,6),
     ALTER COLUMN "senderAmount" TYPE DECIMAL(18,6) USING "senderAmount"::DECIMAL(18,6),
     ALTER COLUMN "receiverAmount" TYPE DECIMAL(18,6) USING "receiverAmount"::DECIMAL(18,6),
     ALTER COLUMN "netAmountBrl" TYPE DECIMAL(18,6) USING "netAmountBrl"::DECIMAL(18,6);
   ```

4. Atualizar TypeScript:
   - Após `bun prisma generate`, os campos viram `Prisma.Decimal`
   - Grep por `payment.amount`, `payment.usdcAmount`, `payout.senderAmount` etc.
   - Onde for necessário `number`, fazer `Number(field)` ou usar `.toNumber()`
   - Onde comparar, usar `.equals()` ou `Number()` antes de comparar

**Critério de aceite:**
- [ ] Migration roda sem perda de dados em staging
- [ ] `bun prisma generate` sem erros
- [ ] Compilação TypeScript sem erros
- [ ] Valores de 0.1 + 0.2 retornam 0.300000 (não 0.30000000000000004)

---

### TASK-04 · R07 — Limites diários de saque
**Risco bloqueado:** Saque total de conta comprometida sem restrição
**Complexidade:** Baixa–Média

**Arquivos a criar/modificar:**
- `api/prisma/schema.prisma` (novo model `UserWithdrawalLimits`)
- `api/src/modules/payments/services/withdrawal-limits.service.ts` (NOVO)
- `api/src/modules/payments/use-cases/withdraw/create-payout-quote-3x.use-case.ts`
- `api/src/modules/admin/routes/finance.routes.ts` (endpoint para ajustar limites)
- Nova migration SQL: `api/prisma/scripts/feat-withdrawal-limits.sql`

**O que fazer:**

1. Schema:
   ```prisma
   model UserWithdrawalLimits {
     id            Int     @id @default(autoincrement())
     userId        Int     @unique
     perTxLimit    Decimal @db.Decimal(18,6) @default(500)   // por transação
     dailyLimit    Decimal @db.Decimal(18,6) @default(1000)  // por dia (UTC)
     weeklyLimit   Decimal @db.Decimal(18,6) @default(5000)  // por semana
     cooldownHours Int     @default(24)   // carência após 1º depósito
     updatedAt     DateTime @updatedAt
     user          User @relation(fields: [userId], references: [id])
   }
   ```

2. `withdrawal-limits.service.ts`:
   - `validateWithdrawalLimits(userId, amount)`:
     - Verificar `amount <= perTxLimit`
     - Somar payouts completed nas últimas 24h → `dailyTotal + amount <= dailyLimit`
     - Somar payouts completed nos últimos 7 dias → `weeklyTotal + amount <= weeklyLimit`
     - Verificar carência: primeiro depósito há menos de `cooldownHours` → bloquear
   - `getLimitsForUser(userId)` → retorna limites (criando defaults se não existir)

3. Chamar `validateWithdrawalLimits` no início de `create-payout-quote-3x.use-case.ts`, antes de qualquer operação financeira

4. Endpoint admin para ajustar limites de usuário específico (KYC, usuários verificados):
   - `PATCH /api/v1/admin/users/:userId/withdrawal-limits`

**Critério de aceite:**
- [ ] Saque de 600 USDC com perTxLimit=500 → bloqueado com mensagem clara
- [ ] Dois saques de 600 USDC no mesmo dia com dailyLimit=1000 → segundo bloqueado
- [ ] Admin pode elevar limite de usuário verificado

---

## Sprint 2 — Alto (Semana 2–3)

---

### TASK-05 · R04 — Settlement idempotency em PostgreSQL (não só Redis)
**Risco bloqueado:** Re-processamento de trades após flush do Redis
**Complexidade:** Baixa

**Arquivos a modificar:**
- `api/prisma/schema.prisma` (campos em `SettlementQueueItem`)
- `api/src/modules/prediction-market/trading/services/settlement.service.ts`
- Nova migration SQL: `api/prisma/scripts/feat-settlement-settled-tx.sql`

**O que fazer:**

1. Schema — adicionar campos em `SettlementQueueItem`:
   ```prisma
   model SettlementQueueItem {
     // ... campos existentes ...
     settledTx   String?    // assinatura Solana da tx (persistente)
     settledAt   DateTime?  // quando foi settled
   }
   ```

2. Em `settlement.service.ts`, antes de enviar tx:
   ```typescript
   // Checar no DB antes de checar Redis
   const item = await prisma.settlementQueueItem.findUnique({
     where: { tradeId },
     select: { settledTx: true },
   });
   if (item?.settledTx) return item.settledTx; // já executado, não re-enviar
   ```

3. Após tx confirmada, salvar no DB (além do Redis):
   ```typescript
   await prisma.settlementQueueItem.update({
     where: { id: item.id },
     data: { settledTx: txSignature, settledAt: new Date(), status: "SETTLED" },
   });
   ```

4. Manter Redis como cache de curto prazo (rápido), DB como verdade permanente

**Critério de aceite:**
- [ ] Flush do Redis não causa re-execução de trades settled
- [ ] `settledTx` aparece no DB após settlement bem-sucedido

---

### TASK-06 · R05 — Cancelamento atômico de order (SELECT FOR UPDATE)
**Risco bloqueado:** Race condition entre cancelamento e settlement simultâneos
**Complexidade:** Média

**Arquivos a modificar:**
- `api/src/modules/prediction-market/trading/use-cases/orders/cancel-order.use-case.ts`

**O que fazer:**

1. Envolver toda a lógica de cancelamento em `prisma.$transaction`:
   ```typescript
   await prisma.$transaction(async (tx) => {
     // 1. Lock exclusivo na order
     const order = await tx.$queryRaw`
       SELECT * FROM orders WHERE "orderId" = ${orderId} FOR UPDATE
     `;
     if (!order || !["open", "pending"].includes(order.status)) {
       throw new AppError("Order não pode ser cancelada");
     }

     // 2. Verificar se está em processo de settlement
     const inSettlement = await tx.settlementQueueItem.findFirst({
       where: {
         OR: [
           { tradeData: { path: ["buyOrderId"], equals: orderId } },
           { tradeData: { path: ["sellOrderId"], equals: orderId } },
         ],
         status: "CLAIMED",
       },
     });
     if (inSettlement) throw new AppError("Order está sendo liquidada. Tente em instantes.");

     // 3. Cancelar + desbloquear atomicamente
     await tx.order.update({ where: { orderId }, data: { status: "cancelled" } });

     const remainingLocked = calcRemainingLocked(order);
     if (remainingLocked > 0 && order.type === "buy") {
       await tx.$executeRaw`
         UPDATE user_accounts
         SET "lockedBalance" = GREATEST(0, "lockedBalance" - ${remainingLocked})
         WHERE "userWallet" = ${order.userWallet}
       `;
     }
   }, { isolationLevel: "Serializable" });
   ```

**Critério de aceite:**
- [ ] Cancelamento durante settlement retorna 409 (não corrompe dados)
- [ ] Cancelamento normal libera lockedBalance corretamente

---

### TASK-07 · R08 — PgBouncer em produção
**Risco bloqueado:** Exhaustion de conexões PostgreSQL sob carga
**Complexidade:** Infra (não requer mudança de código, apenas configuração)

**Arquivos a criar/modificar:**
- `docker-compose.yml` ou configuração Railway/Render
- `api/src/shared/database/config/prisma-client.ts`

**O que fazer:**

1. Deploy do PgBouncer em frente ao PostgreSQL:
   ```yaml
   # docker-compose.pgbouncer.yml
   pgbouncer:
     image: pgbouncer/pgbouncer:1.21
     environment:
       DATABASES_HOST: ${DB_HOST}
       DATABASES_PORT: 5432
       DATABASES_DBNAME: ${DB_NAME}
       DATABASES_USER: ${DB_USER}
       DATABASES_PASSWORD: ${DB_PASSWORD}
       PGBOUNCER_POOL_MODE: transaction
       PGBOUNCER_MAX_CLIENT_CONN: 5000
       PGBOUNCER_DEFAULT_POOL_SIZE: 25
       PGBOUNCER_SERVER_IDLE_TIMEOUT: 600
     ports:
       - "6432:6432"
   ```

2. Atualizar `DATABASE_URL` da API para apontar para PgBouncer (`host:6432`) ao invés de Postgres diretamente

3. **Atenção:** `transaction` pool mode não suporta `LISTEN/NOTIFY` nem advisory locks de sessão. Verificar se `withDistributedLock` usa `pg_advisory_lock` (session-level) — se sim, mudar para `pg_try_advisory_xact_lock` (transaction-level, compatível com pgbouncer)

4. Reduzir pool size do Prisma Client para evitar sobrecarregar PgBouncer:
   ```typescript
   // prisma-client.ts
   datasourceUrl: `${DATABASE_URL}?connection_limit=5`
   ```

**Critério de aceite:**
- [ ] PgBouncer rodando em staging
- [ ] 500 conexões simultâneas simuladas sem "too many clients"
- [ ] Todos os testes passando com PgBouncer no caminho

---

### TASK-08 · R10 — Sistema de alertas (Slack + critical events)
**Risco bloqueado:** Settlement FAILED passando despercebido
**Complexidade:** Baixa

**Arquivos a criar/modificar:**
- `api/src/shared/services/alert.service.ts` (NOVO)
- `api/src/shared/utils/env.ts` (SLACK_ALERTS_WEBHOOK_URL)
- `api/src/modules/prediction-market/trading/services/settlement.service.ts`
- `api/src/modules/payments/use-cases/withdraw/execute-payout-3x.use-case.ts`

**O que fazer:**

1. `alert.service.ts`:
   ```typescript
   type AlertLevel = "info" | "warning" | "critical";

   export const alertService = {
     async send(level: AlertLevel, event: string, data: Record<string, any>) {
       // 1. Log estruturado sempre
       loggers.app[level === "critical" ? "error" : level]({ alert: event, ...data }, `ALERT: ${event}`);

       // 2. Slack para warning e critical
       if (level !== "info" && process.env.SLACK_ALERTS_WEBHOOK_URL) {
         const emoji = level === "critical" ? "🚨" : "⚠️";
         await fetch(process.env.SLACK_ALERTS_WEBHOOK_URL, {
           method: "POST",
           headers: { "Content-Type": "application/json" },
           body: JSON.stringify({
             text: `${emoji} *[${level.toUpperCase()}] ${event}*`,
             attachments: [{ text: JSON.stringify(data, null, 2), color: level === "critical" ? "danger" : "warning" }],
           }),
         }).catch((err) => loggers.app.warn({ err }, "Falha ao enviar alerta Slack"));
       }
     },
   };
   ```

2. Eventos críticos para alertar:
   - `settlement_permanent_failure` — item FAILED após MAX_RETRIES (settlement.service.ts)
   - `payout_stuck` — payout em `payout_usdc_sent` há > 30 min (reconciliação)
   - `financial_reconciliation_divergence` — divergência na reconciliação diária
   - `webhook_signature_failure_spike` — > 3 falhas/min do mesmo IP
   - `wash_trading_detected` — mesmo usuário nos dois lados (fraud detection)
   - `duplicate_pix_key` — mesma chave PIX em contas diferentes
   - `daily_reconciliation_failed` — erro no job de reconciliação

3. Adicionar chamadas `alertService.send("critical", ...)` em cada ponto acima

**Critério de aceite:**
- [ ] Settlement FAILED → mensagem no Slack em < 30s
- [ ] Alert service não bloqueia o fluxo principal (falhas de Slack são capturadas)

---

### TASK-09 · R11 — PIX key vinculada ao usuário (unicidade)
**Risco bloqueado:** Dois usuários com a mesma chave PIX (fraude de saída)
**Complexidade:** Baixa

**Arquivos a modificar:**
- `api/src/modules/payments/use-cases/register-pix-key.use-case.ts`
- `api/prisma/schema.prisma` (index único em pixKey)

**O que fazer:**

1. Antes de salvar a chave PIX, verificar unicidade global:
   ```typescript
   const existing = await prisma.pixKey.findFirst({
     where: { pixKey: normalizedKey, userId: { not: userId } },
   });
   if (existing) {
     await alertService.send("warning", "duplicate_pix_key_attempt", {
       pixKey: maskPixKey(normalizedKey),
       newUserId: userId,
       existingUserId: existing.userId,
     });
     throw new AppError("Essa chave PIX já está cadastrada em outra conta.");
   }
   ```

2. Adicionar constraint de unicidade no schema:
   ```prisma
   model PixKey {
     // ...
     @@unique([pixKey]) // uma chave PIX = um usuário no sistema
   }
   ```

3. `maskPixKey(key)` — função que masca os dados: `"123.456.789-00"` → `"123.***-00"`

**Critério de aceite:**
- [ ] Tentar cadastrar CPF já usado por outro usuário → erro claro
- [ ] Alerta Slack emitido tentativa duplicada

---

### TASK-10 · R12 + R17 — Cron jobs: expiração de orders + reconciliação
**Risco bloqueado:** USDC locked para sempre (R12) + payouts órfãos sem reconciliação (R17)
**Complexidade:** Baixa

**Arquivos a criar/modificar:**
- `api/src/jobs/expire-orders.job.ts` (NOVO)
- `api/src/jobs/index.ts` (NOVO — registra todos os crons)
- `api/src/app.ts` ou `api/src/server.ts` (inicialização dos jobs)

**O que fazer:**

1. `expire-orders.job.ts`:
   - A cada 5 min: buscar orders com `expiresAt < now()` e `status IN (open, pending)`
   - Para cada order: `$transaction` → status=expired + desbloquear USDC + publicar evento ledger
   - Processar em batches de 500 para não travar o DB

2. `index.ts` — registrar todos os crons:
   ```typescript
   import { CronJob } from "cron";

   export function startBackgroundJobs() {
     // Reconciliação 3xChange
     new CronJob("*/5 * * * *", () => reconcilePayouts3xService.run().catch(logErr), null, true);

     // Expiração de orders
     new CronJob("*/5 * * * *", () => expireOrders().catch(logErr), null, true);

     // Reconciliação financeira diária (03:00 BRT)
     new CronJob("0 3 * * *", () => dailyReconciliationService.run().catch(alertAndLog), null, true);

     // Health check financeiro (a cada 10 min — alerta se fila FAILED > 0)
     new CronJob("*/10 * * * *", () => financialHealthCheck().catch(logErr), null, true);
   }
   ```

3. Chamar `startBackgroundJobs()` no `server.ts` após DB conectar

**Critério de aceite:**
- [ ] Order com `expiresAt` no passado → expirada automaticamente em < 10 min
- [ ] `ReconcilePayouts3xService` roda a cada 5 min sem intervenção manual
- [ ] Jobs logam início e fim de cada execução

---

### TASK-11 · R18 — Freeze/bloqueio de conta de usuário
**Risco bloqueado:** Conta suspeita operar livremente durante investigação
**Complexidade:** Baixa

**Arquivos a modificar:**
- `api/prisma/schema.prisma` (campos em `User`)
- `api/src/modules/identity/repositories/user.repository.ts`
- `api/src/shared/middleware/account-status.middleware.ts` (NOVO)
- `api/src/modules/admin/routes/users.routes.ts` (endpoint de freeze)
- Rotas financeiras que precisam do middleware
- Nova migration SQL: `api/prisma/scripts/feat-account-status.sql`

**O que fazer:**

1. Schema:
   ```prisma
   model User {
     // ... campos existentes ...
     accountStatus  UserAccountStatus @default(ACTIVE)
     frozenAt       DateTime?
     frozenReason   String?
   }

   enum UserAccountStatus {
     ACTIVE
     FROZEN       // bloqueado para todas as operações financeiras
     RESTRICTED   // pode visualizar, não pode operar
   }
   ```

2. `account-status.middleware.ts`:
   ```typescript
   export async function requireActiveAccount(c: Context, next: Next) {
     const userId = c.get("userId");
     const user = await userRepository.findById(userId, { select: { accountStatus: true } });
     if (user?.accountStatus !== "ACTIVE") {
       return c.json({ error: "Conta suspensa. Entre em contato com o suporte." }, 403);
     }
     return next();
   }
   ```

3. Aplicar `requireActiveAccount` em:
   - `POST /orders` (criar order)
   - `POST /withdraw/quote`, `/authorize`, `/execute`
   - `POST /deposits/pix` (iniciar depósito)

4. Endpoint admin:
   ```
   POST /api/v1/admin/users/:userId/freeze   → { reason: string }
   POST /api/v1/admin/users/:userId/unfreeze → {}
   ```
   - freeze: setar accountStatus=FROZEN, frozenAt, frozenReason + cancelar orders abertas

**Critério de aceite:**
- [ ] Conta FROZEN não consegue criar orders nem iniciar saque (retorna 403)
- [ ] Admin pode freeze e unfreeze via API
- [ ] Freeze cancela orders abertas do usuário e libera USDC

---

### TASK-12 · R13 — Reconciliação financeira diária automatizada
**Risco bloqueado:** Divergência financeira não detectada
**Complexidade:** Média

**Arquivos a criar/modificar:**
- `api/src/modules/finance/services/daily-reconciliation.service.ts` (NOVO)
- `api/prisma/schema.prisma` (novo model `ReconciliationReport`)
- `api/src/modules/admin/routes/finance.routes.ts` (endpoint para rodar manualmente)
- Nova migration SQL

**O que fazer:**

1. Schema:
   ```prisma
   model ReconciliationReport {
     id                    Int      @id @default(autoincrement())
     date                  String   @unique // "2026-03-23"
     totalUserBalance      Decimal  @db.Decimal(18,6)
     totalLockedBalance    Decimal  @db.Decimal(18,6)
     totalDepositsDay      Decimal  @db.Decimal(18,6)
     totalWithdrawalsDay   Decimal  @db.Decimal(18,6)
     pendingPayouts        Int
     pendingSettlements    Int
     settlementFailed      Int
     ledgerBalance         Decimal  @db.Decimal(18,6)
     divergence            Decimal  @db.Decimal(18,6)
     divergenceAcceptable  Boolean
     runAt                 DateTime @default(now())
   }
   ```

2. `daily-reconciliation.service.ts`:
   - Agregar: `Σ UserAccount.balance`, `Σ deposits hoje`, `Σ saques hoje`
   - Calcular `ledgerBalance` = Σ (credit - debit) de toda a tabela LedgerEntry
   - Calcular `divergence = |totalUserBalance - ledgerBalance|`
   - Se `divergence > 0.01` → `alertService.send("critical", "financial_divergence", ...)`
   - Salvar `ReconciliationReport`

3. Endpoint admin: `POST /api/v1/admin/finance/reconciliation/run` (triggera manualmente)

4. Registrar no cron (já coberto na TASK-10)

**Critério de aceite:**
- [ ] Relatório gerado todo dia às 03:00
- [ ] Divergência > 0.01 USDC dispara alerta Slack crítico
- [ ] Relatórios visíveis via `GET /api/v1/admin/finance/reconciliation`

---

## Sprint 3 — Médio / Arquitetura (Semana 3–4)

---

### TASK-13 · R14 — Detecção de lavagem e fraude
**Risco bloqueado:** Depósito → saque imediato, multi-conta, wash trading
**Complexidade:** Média

**Arquivos a criar/modificar:**
- `api/src/modules/fraud/services/fraud-detection.service.ts` (NOVO)
- `api/src/modules/payments/use-cases/withdraw/create-payout-quote-3x.use-case.ts`
- `api/src/modules/prediction-market/trading/use-cases/orders/create-order.use-case.ts`

**O que fazer:**

1. `fraud-detection.service.ts`:

   **a) Depósito → saque imediato:**
   ```typescript
   async checkDepositWithdrawalPattern(userId, amount): Promise<FraudCheck> {
     const deposits24h = await getDepositsLast24h(userId);
     const total24h = deposits24h.reduce((s, d) => s + d, 0);
     if (total24h > 0 && amount >= total24h * 0.8) {
       return { flag: true, reason: "Saque ≥ 80% do depósito nas últimas 24h", action: "MANUAL_REVIEW" };
     }
     return { flag: false };
   }
   ```

   **b) Wash trading (mesmo usuário nos dois lados):**
   ```typescript
   async checkWashTrading(buyerWallet, sellerWallet): Promise<boolean> {
     // Verificar se mesma chave PIX (CPF) vinculada aos dois wallets
     const [buyerPixKey, sellerPixKey] = await Promise.all([
       getPixKeyByWallet(buyerWallet),
       getPixKeyByWallet(sellerWallet),
     ]);
     if (buyerPixKey && sellerPixKey && buyerPixKey === sellerPixKey) {
       await alertService.send("critical", "wash_trading_detected", { buyerWallet, sellerWallet });
       return true;
     }
     return false;
   }
   ```

   **c) Velocidade de saque:**
   - Mais de 3 saques em 1h → `MANUAL_REVIEW`
   - Saque logo após o primeiro depósito (< cooldownHours) → já coberto em TASK-04

2. Chamar `checkDepositWithdrawalPattern` no início do `/quote`

3. Chamar `checkWashTrading` no settlement antes de processar o match

4. Ação para `MANUAL_REVIEW`:
   - Criar registro em tabela `FraudReview`
   - Setar `accountStatus=RESTRICTED` temporariamente
   - Alertar Slack + time de compliance

**Critério de aceite:**
- [ ] Depósito de 100 USDC + saque de 85 USDC na sequência → flaggeado
- [ ] Wash trading detectado → alerta crítico + operação bloqueada

---

### TASK-14 · R15 — Log sanitization
**Risco bloqueado:** Dados sensíveis vazando em logs (PIX keys, wallets, endereços)
**Complexidade:** Baixa

**Arquivos a criar/modificar:**
- `api/src/shared/utils/log-sanitizer.ts` (NOVO)
- Todos os arquivos que chamam `loggers.app.info/error/warn` com dados financeiros

**O que fazer:**

1. `log-sanitizer.ts`:
   ```typescript
   const SENSITIVE_KEYS = ["pixKey", "cpf", "cnpj", "phone", "depositAddress",
                           "walletAddress", "pixEndToEndId", "accountNumber"];

   export function sanitize(obj: unknown): unknown {
     if (!obj || typeof obj !== "object") return obj;
     if (Array.isArray(obj)) return obj.map(sanitize);
     return Object.fromEntries(
       Object.entries(obj as Record<string, unknown>).map(([k, v]) => {
         if (SENSITIVE_KEYS.some((s) => k.toLowerCase().includes(s.toLowerCase()))) {
           return [k, typeof v === "string" ? `${String(v).slice(0, 4)}****` : "[REDACTED]"];
         }
         return [k, sanitize(v)];
       }),
     );
   }
   ```

2. Criar wrapper de logger que aplica sanitize automaticamente:
   ```typescript
   // Ao invés de loggers.app.info diretamente, usar:
   export const safeLog = {
     info:  (data: object, msg: string) => loggers.app.info(sanitize(data), msg),
     warn:  (data: object, msg: string) => loggers.app.warn(sanitize(data), msg),
     error: (data: object, msg: string) => loggers.app.error(sanitize(data), msg),
   };
   ```

3. Substituir nos arquivos de pagamento/saque (payout use-cases, webhook handlers, deposit service)

**Critério de aceite:**
- [ ] Log de payout não mostra depositAddress completo
- [ ] Log de webhook não mostra pixKey em plaintext

---

### TASK-15 · R16 — Solana RPC com fallback automático
**Risco bloqueado:** RPC único como single-point-of-failure
**Complexidade:** Média

**Arquivos a modificar:**
- `api/src/shared/config/solana.config.ts`
- `api/src/shared/utils/env.ts`

**O que fazer:**

1. Novas env vars:
   ```
   SOLANA_RPC_URL=https://mainnet.helius-rpc.com/...      # primário
   SOLANA_RPC_FALLBACK_1=https://...quicknode.com/...     # fallback 1
   SOLANA_RPC_FALLBACK_2=https://...alchemy.com/...       # fallback 2
   ```

2. Classe `ResilientConnection` em `solana.config.ts`:
   - Lista de endpoints ordenada por prioridade
   - `withFallback(fn)`: tenta cada endpoint em ordem
   - Em 429 ou timeout: tenta próximo sem lançar erro
   - Salva qual endpoint está funcionando (volta para o primário quando disponível)
   - Circuit breaker por endpoint: se falhar 3x seguidas, remove temporariamente (5 min)

3. Trocar todas as chamadas `getConnection()` por `resilientConnection.current` ou `resilientConnection.withFallback(...)`

**Critério de aceite:**
- [ ] Simular downtime do RPC primário → API continua funcionando com fallback
- [ ] Log indica qual RPC está sendo usado
- [ ] RPC primário volta a ser usado quando disponível

---

### TASK-16 · R20 — Proteção contra wash trading no order book
**Risco bloqueado:** Manipulação de preço via ordens contra si mesmo
**Complexidade:** Baixa (aproveitar TASK-13)

**Arquivos a modificar:**
- `api/src/modules/prediction-market/trading/services/orderbook-engine.service.ts` (ou equivalente)
- `api/src/modules/prediction-market/trading/services/settlement.service.ts`

**O que fazer:**

1. No matching engine, antes de confirmar um match entre buy e sell:
   ```typescript
   if (buyOrder.userWallet === sellOrder.userWallet) {
     // Auto-match do mesmo usuário → não executar
     loggers.app.warn({ orderId: buyOrder.orderId }, "Self-match bloqueado");
     return null; // skip match
   }
   ```

2. No settlement, chamar `fraudDetectionService.checkWashTrading(buyer, seller)`

3. Considerar também: se dois userIds diferentes mas mesma chave PIX → wash trading cross-conta

**Critério de aceite:**
- [ ] Order de compra e venda do mesmo wallet no mesmo mercado → não matchea
- [ ] Tentativa de wash trading cross-conta (mesma PIX, wallets diferentes) → bloqueada

---

### TASK-17 · R09 — Settlement idempotency key mais robusto
**Risco bloqueado:** Colisão de chave em edge cases
**Complexidade:** Baixa

**Arquivos a modificar:**
- `api/src/modules/prediction-market/trading/services/settlement.service.ts`

**O que fazer:**

1. Chave atual: `settlement:done:${buyOrderId}:${sellOrderId}` — pode colidir se o mesmo par for re-queueado
2. Chave nova: `settlement:done:${tradeId}:${settlementQueueItemId}` — tradeId é UUID único por definição
3. Ou melhor: usar o `settledTx` do DB (TASK-05), que é definitivo por natureza

**Critério de aceite:**
- [ ] Nenhuma colisão possível mesmo com milhares de settlements simultâneos

---

### TASK-18 · Idempotency key middleware para operações de escrita
**Risco bloqueado:** Re-envio de formulários / retries do cliente gerando duplicatas
**Complexidade:** Baixa–Média

**Arquivos a criar/modificar:**
- `api/src/shared/middleware/idempotency.middleware.ts` (NOVO)
- Rotas: `/orders`, `/withdraw/quote`, `/withdraw/authorize`, `/withdraw/execute`

**O que fazer:**

1. Middleware:
   - Header: `Idempotency-Key: <uuid-v4>` (gerado pelo cliente)
   - Se presente: checar Redis por `idempotency:{key}:{userId}`
   - Se cache hit: retornar resposta armazenada (sem re-executar)
   - Se miss: executar, armazenar response (status 200–299) por 24h
   - TTL 24h: após isso, mesma key pode ser re-usada (janela de deduplicação razoável)

2. O cliente (frontend/mobile) deve:
   - Gerar UUID antes de enviar o request
   - Re-usar o mesmo UUID em retries do mesmo request
   - Gerar UUID novo para cada nova operação

**Critério de aceite:**
- [ ] Dois POSTs com o mesmo `Idempotency-Key` do mesmo usuário → segundo retorna resposta cacheada sem criar nova order
- [ ] POST sem header → funciona normalmente (backward compatible)

---

### TASK-19 · Rate limiting específico por operação financeira
**Risco bloqueado:** Abuso nos endpoints de saque/depósito
**Complexidade:** Baixa

**Arquivos a modificar:**
- `api/src/shared/middleware/rate-limit.middleware.ts` (ou equivalente)
- Rotas financeiras

**O que fazer:**

Aplicar limites específicos por rota (mais restritivos que o global):

| Rota | Janela | Máximo |
|---|---|---|
| `POST /withdraw/quote` | 1 minuto | 5 |
| `POST /withdraw/authorize` | 1 minuto | 3 |
| `POST /withdraw/execute` | 5 minutos | 3 |
| `POST /deposits/pix` | 1 minuto | 3 |
| `POST /pix-key/register` | 1 hora | 3 |
| `POST /orders` | 1 minuto | 30 (já existe) |

Usar o middleware existente de rate limiting com `walletAddress` como identificador (não IP, para evitar problemas com proxies/CDN).

**Critério de aceite:**
- [ ] Quarto `/quote` no mesmo minuto → 429 Too Many Requests
- [ ] Rate limit reseta corretamente após a janela

---

## Sprint 4 — Escalabilidade (Mês 2)

---

### TASK-20 · Read replicas para queries de listagem
**Risco bloqueado:** Primary DB sobrecarregado com queries de leitura
**Complexidade:** Infra + código médio

**O que fazer:**
1. Configurar read replica no provider de DB (Railway, Supabase, etc.)
2. Criar `prismaRead` apontando para replica URL
3. Usar `prismaRead` em: `getAllMarkets`, `getMarket`, `GET /orders/book`, `GET /positions`
4. Manter `prisma` (primary) para todas as escritas

---

### TASK-21 · Queue assíncrona para execução de saques
**Risco bloqueado:** Timeout de request durante envio de tx Solana
**Complexidade:** Alta

**O que fazer:**
1. Ao chamar `/execute`: CAS lock → publicar em fila RabbitMQ → retornar `{ status: "processing" }`
2. Worker assíncrono: consome fila → sign + send tx → update status
3. WebSocket notifica usuário quando payout_usdc_sent

---

### TASK-22 · Health check financeiro endpoint
**Risco bloqueado:** Deploys com problemas financeiros sem detecção imediata
**Complexidade:** Baixa

**O que fazer:**
1. `GET /health/financial` → retorna status de: fila settlement, payouts stuck, divergência de reconciliação
2. Integrar com monitoramento de uptime (Uptime Robot, Grafana, etc.)
3. Retorna 503 se qualquer indicador crítico estiver vermelho

---

## Dependências entre Tasks

```
TASK-01 (lock payout)
  └─→ TASK-12 (reconciliação detecta locks não liberados)

TASK-02 (webhook signature)
  └─→ TASK-08 (alertas — webhooks rejeitados geram alerta)

TASK-03 (Decimal)
  └─→ TASK-12 (reconciliação precisa de math preciso)

TASK-04 (limites de saque)
  └─→ TASK-13 (fraude — limites são a primeira linha de defesa)

TASK-05 (settlement no DB)
  └─→ TASK-17 (idempotency key mais robusto)

TASK-08 (alertas)
  └─→ TASK-10 (crons), TASK-12 (reconciliação), TASK-13 (fraude)
  └─→ Todas as demais tasks usam alertService

TASK-10 (crons)
  └─→ TASK-12 (reconciliação roda via cron)

TASK-11 (freeze conta)
  └─→ TASK-13 (fraude pode triggerar freeze automático)
```

---

## Checklist de Validação por Task

Antes de marcar qualquer task como `[x]`:

```
[ ] Migration SQL criada e testada em staging
[ ] bun prisma generate sem erros
[ ] TypeScript compila sem erros
[ ] Testes unitários dos casos críticos (happy path + failure path)
[ ] Testado manualmente em staging com dados reais
[ ] Não quebrou nenhuma funcionalidade existente
[ ] Documentação (JSDoc ou comentário) nos pontos não óbvios
[ ] Alerta de monitoramento configurado (se aplicável)
```

---

## Resumo por Sprint

| Sprint | Tasks | Riscos bloqueados |
|---|---|---|
| **Sprint 1** (semana 1) | TASK-01, 02, 03, 04 | R01, R02, R03, R06, R07 — os mais críticos |
| **Sprint 2** (semana 2–3) | TASK-05 a 12 | R04, R05, R08, R10, R11, R12, R13, R17, R18 |
| **Sprint 3** (semana 3–4) | TASK-13 a 19 | R09, R14, R15, R16, R20 + idempotency + rate limits |
| **Sprint 4** (mês 2) | TASK-20, 21, 22 | Escalabilidade e arquitetura |

**Total: 22 tasks** cobrindo todos os 20 riscos identificados (exceto R19 — encriptação PIX).
