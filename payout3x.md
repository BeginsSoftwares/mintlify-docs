# Payout via 3xChange — USDC → PIX

Documentação do fluxo de saque do Oraculo utilizando a 3xChange como provider de conversão USDC → BRL via PIX.

---

## Visão Geral

O usuário possui saldo em **USDC na Solana**. Para sacar em reais, o Oraculo:

1. Obtém uma cotação USDC/BRL da 3xChange
2. Cria uma **ordem de payout** na 3xChange, que retorna um endereço de depósito Solana
3. Envia o USDC para esse endereço via transação on-chain assinada pela carteira do usuário
4. A 3xChange detecta o depósito e envia o PIX para a chave cadastrada
5. Um webhook confirma a conclusão (ou falha) da operação

O fluxo é exposto via **3 endpoints públicos** com a mesma superfície de API do provider legado (BlindPay), sem necessidade de mudança no frontend.

---

## Pré-requisitos do Usuário

| Campo | Origem | Obrigatório |
|---|---|---|
| `threeXChangeReceiverId` | Criado no cadastro via `clerk-webhook-processor` | Sim |
| `walletAddress` | Carteira Solana do usuário | Sim |
| `cpf` | Cadastro do usuário | Sim (exigido pela 3xChange para KYC) |
| Chave PIX | Cadastrada via `POST /payments/withdraw/pix-key` | Sim |

> Usuários 3xChange **não criam conta no BlindPay**. O campo `blindpayBankAccountId` é nullable para eles.

---

## Cadastro da Chave PIX

**Endpoint:** `POST /api/v1/payments/withdraw/pix-key`

Para usuários 3xChange, o fluxo de cadastro de chave PIX é simplificado:
- **Não** chama o BlindPay para criar uma conta bancária
- Detecta automaticamente o tipo da chave (`CPF`, `CNPJ`, `EMAIL`, `PHONE`, `RANDOM`)
- Salva localmente no banco com `pixKeyType` preenchido

```
detectPixKeyType("12345678901")   → "CPF"
detectPixKeyType("12345678000199") → "CNPJ"
detectPixKeyType("user@email.com") → "EMAIL"
detectPixKeyType("+5511999999999")  → "PHONE"
(qualquer outro formato)           → "RANDOM"
```

O `pixKeyType` será enviado para a 3xChange na criação da ordem de payout.

---

## Fluxo de Saque — 3 Etapas

```
Frontend                    Oraculo API                    3xChange                 Solana
   │                             │                              │                      │
   │── POST /withdraw/quote ────>│                              │                      │
   │                             │── createPayoutQuote() ──────>│                      │
   │                             │<─ { rate, netAmountBrl } ────│                      │
   │<─ { payoutId, quoteId } ────│                              │                      │
   │                             │                              │                      │
   │── POST /withdraw/authorize ─>│                             │                      │
   │                             │── createPayout() ───────────>│                      │
   │                             │<─ { orderId, depositAddress }│                      │
   │<─ { depositAddress } ───────│                              │                      │
   │                             │                              │                      │
   │── POST /withdraw/execute ──>│                              │                      │
   │                             │── getOrder() [live verify] ─>│                      │
   │                             │<─ { status: AWAITING_DEPOSIT }                      │
   │                             │─── CAS lock (raw SQL) ───────────────────────────── │
   │                             │─── sendUSDC() ──────────────────────────────────────>
   │                             │<── txHash ──────────────────────────────────────────│
   │                             │   [persiste txHash, status: payout_usdc_sent]        │
   │<─ { txHash, status } ───────│                              │                      │
   │                             │                              │                      │
   │                  [async]    │<── Webhook PAYOUT COMPLETED ─│                      │
   │                             │   [status: payout_completed] │                      │
```

---

### Etapa 1 — Cotação (`/quote`)

**Endpoint:** `POST /api/v1/payments/withdraw/quote`

**Use case:** `CreatePayoutQuote3xUseCase`

**Validações:**
- Valor mínimo: **10 USDC**
- Usuário deve ter `threeXChangeReceiverId`
- Chave PIX deve estar cadastrada
- Saldo on-chain verificado diretamente na Solana (não usa cache)

**O que acontece:**
1. Chama `threeXChangeService.createPayoutQuote(receiverId, usdcAmount)` → obtém taxa de câmbio e valor líquido em BRL
2. Gera um `quoteId` (UUID local — não vem da 3xChange)
3. Cria registro `Payout` no banco com `status: payout_pending` e `provider: "3xchange"`

**Retorno:**
```json
{
  "payoutId": "123",
  "quoteId": "550e8400-e29b-41d4-a716-446655440000",
  "amountUsdc": 50.00,
  "netAmountBrl": 285.50,
  "commercialRate": "5.85",
  "rate3x": "5.71"
}
```

> A cotação é **informativa** — a taxa real é travada apenas na etapa `/authorize` quando a ordem é criada na 3xChange.

---

### Etapa 2 — Autorização (`/authorize`)

**Endpoint:** `POST /api/v1/payments/withdraw/authorize`

**Use case:** `AuthorizePayout3xUseCase`

**Input:** `{ quoteId }`

**Validações:**
- Payout deve existir e pertencer ao usuário
- `provider === "3xchange"`
- `status === "payout_pending"`
- Usuário deve ter `cpf` preenchido

**O que acontece:**
1. Chama `threeXChangeService.createPayout()` com:
   - `receiverId` do usuário
   - `amountCrypto` (USDC)
   - `pixKey` + `pixKeyType` + `recipientName` (da chave PIX cadastrada)
   - `recipientDocument` (CPF do usuário)
   - `externalId = payoutId` (garante idempotência: se chamado duas vezes, a 3xChange retorna a mesma ordem)
2. Recebe `{ orderId, depositAddress }` da 3xChange
3. Persiste no banco: `threexchangeOrderId`, `depositAddress`, `netAmountBrl`, `status: payout_order_placed`

**Retorno:**
```json
{
  "payoutId": "123",
  "depositAddress": "GfAa...xK9",
  "amountUsdc": 50.00,
  "netAmountBrl": 285.50,
  "status": "payout_order_placed"
}
```

**Idempotência:** Se `/authorize` for chamado novamente com o mesmo `quoteId` e a ordem já existir (`payout_order_placed`), retorna os dados da ordem existente sem recriar na 3xChange.

---

### Etapa 3 — Execução (`/execute`)

**Endpoint:** `POST /api/v1/payments/withdraw/execute`

**Use case:** `ExecutePayout3xUseCase`

**Input:** `{ payoutId }`

Esta é a etapa crítica — envolve uma transação Solana irreversível. Implementa 4 checkpoints de segurança:

#### Checkpoint 1 — Ownership & Status

- Verifica que o payout existe, pertence ao usuário e tem `provider === "3xchange"`
- Idempotente: se status já for `payout_usdc_sent`, `payout_pix_processing` ou `payout_completed`, retorna o resultado existente sem re-executar

#### Checkpoint 2 — CAS Lock (Compare-And-Swap)

Prevenção de double-spend via SQL atômico:

```sql
UPDATE payouts
SET status = 'payout_executing',
    locked_at = NOW(),
    updated_at = NOW()
WHERE id = $payoutId
  AND status = 'payout_order_placed'
  AND (locked_at IS NULL OR locked_at < NOW() - INTERVAL '5 minutes')
```

- Se `affected_rows = 0`: outro processo já está executando → retorna erro `409 PAYOUT_LOCKED`
- O lock expira em **5 minutos** — protege contra crash mid-execution
- Todos os paths de falha pós-lock chamam `releaseLock()` para reverter o status

#### Checkpoint 3 — Live Verification na 3xChange

Antes de mover qualquer USDC, consulta a ordem diretamente na 3xChange:

- `status` deve ser `AWAITING_DEPOSIT` — qualquer outro status (EXPIRED, FAILED, etc.) aborta
- `depositAddress` da resposta deve ser **idêntico** ao endereço salvo no banco
- Se houver divergência de endereço: **security alert** em log + erro `DEPOSIT_ADDRESS_MISMATCH`

#### Checkpoint 4 — Envio USDC na Solana

Monta e envia a transação SPL Token:

1. Calcula o ATA (Associated Token Account) de origem e destino para o mint do USDC (`EPjFWdd5AufqSSqeM2qN1xzybapC8G4wEGGkZwyTDt1v`)
2. Se o ATA de destino não existir, adiciona instrução `createAssociatedTokenAccountInstruction`
3. Adiciona instrução `createTransferInstruction` com `amount = usdcAmount * 10^6` (6 decimais)
4. Assina via `getWalletSigner()` (prioridade: 3xChange → Privy → AWS legacy)
5. **Persiste `transactionHash` e `status: payout_usdc_sent` ANTES de retornar** — se o processo morrer após o envio, o hash está salvo

**Retorno:**
```json
{
  "payoutId": "123",
  "transactionHash": "3xK...abc",
  "status": "payout_usdc_sent"
}
```

---

## Estados do Payout (State Machine)

```
payout_pending
    │
    │  /authorize → cria ordem na 3xChange
    ▼
payout_order_placed
    │
    │  /execute → CAS lock
    ▼
payout_executing  ──── (lock expira em 5 min → volta para payout_order_placed via Reconcile)
    │
    │  USDC enviado on-chain
    ▼
payout_usdc_sent
    │
    │  3xChange detecta depósito, inicia PIX
    ▼
payout_pix_processing
    │
    ├─── PIX entregue ──────────────────────────> payout_completed ✅
    │
    └─── PIX falhou (USDC já enviado) ──────────> payout_refund_pending ⚠️
         (requer intervenção manual ou refund automático)
```

**Outros estados:**
- `payout_failed` — ordem expirou antes do envio USDC (sem prejuízo financeiro)

---

## Webhook de Conclusão

**Endpoint:** `POST /api/v1/payments/webhooks/3xchange`

A 3xChange envia eventos com `type: "PAYOUT"` para notificar a conclusão ou falha:

### Evento: COMPLETED

```json
{
  "type": "PAYOUT",
  "event": "order.completed",
  "status": "COMPLETED",
  "orderId": "uuid-da-ordem",
  "pixEndToEndId": "E00400000...",
  "timestamp": "2026-03-21T14:00:00Z"
}
```

→ Atualiza payout para `payout_completed`, persiste `pixEndToEndId`

### Evento: FAILED

```json
{
  "type": "PAYOUT",
  "event": "order.failed",
  "status": "FAILED",
  "orderId": "uuid-da-ordem"
}
```

→ Atualiza payout para `payout_refund_pending`
→ O USDC **já foi enviado** — necessário reembolso ao usuário

**Deduplicação:** Webhooks duplicados são ignorados se o payout já estiver em `payout_completed` ou `payout_failed`.

**Busca do payout:** por `threexchangeOrderId` primário, com fallback por `externalId` (= nosso `payoutId`).

---

## Reconciliação Periódica

**Service:** `ReconcilePayouts3xService`

Safety net para saques presos em estados intermediários. Deve ser executado a cada **5 minutos** via cron.

### O que faz:

**1. Libera locks órfãos**

```sql
UPDATE payouts
SET status = 'payout_order_placed', locked_at = NULL
WHERE provider = '3xchange'
  AND status = 'payout_executing'
  AND locked_at < NOW() - INTERVAL '5 minutes'
```

Detecta processos que morreram entre o lock e o envio USDC.

**2. Reconcilia saques presos**

Busca payouts com status `payout_usdc_sent` ou `payout_pix_processing` com mais de 15 minutos sem atualização. Para cada um:
- Consulta o status real da ordem na 3xChange via `getOrder(orderId)`
- Aplica a transição de estado correspondente:

| Status 3xChange | Novo status Oraculo |
|---|---|
| `PIX_SENDING` | `payout_pix_processing` |
| `COMPLETED` | `payout_completed` |
| `FAILED` | `payout_refund_pending` |
| `EXPIRED` | `payout_failed` |
| `AWAITING_DEPOSIT` com payout_usdc_sent > 15min | **alerta** — verificar on-chain |

---

## Segurança — Resumo

| Ameaça | Mitigação |
|---|---|
| Double-spend (dois processos executando simultaneamente) | CAS lock via raw SQL atômico |
| Endereço de depósito adulterado no banco | Live verification antes do envio + security alert se divergir |
| Processo morre após enviar USDC mas antes de persistir | txHash salvo antes do retorno; reconciliação detecta via `getOrder()` |
| Webhook duplicado processado duas vezes | Deduplicação por status (`payout_completed` / `payout_failed`) |
| Lock nunca liberado (crash mid-flight) | Lock expira em 5 min; reconciliação libera locks órfãos |
| Ordem expirada sendo executada | Checkpoint 3 verifica `status === AWAITING_DEPOSIT` ao vivo |
| Replay de `/execute` após USDC enviado | Idempotência: retorna resultado existente para status terminais |

---

## Roteamento de Provider

O backend detecta automaticamente qual provider usar — **sem mudança no frontend**:

- **`/quote` e `/authorize`:** verifica `user.threeXChangeReceiverId`
  - Se preenchido → 3xChange use-cases
  - Se ausente → BlindPay use-cases (legado)

- **`/execute`:** verifica `payout.provider` diretamente no banco
  - `"3xchange"` → `ExecutePayout3xUseCase`
  - ausente/outro → `ExecutePayoutUseCase` (BlindPay)

---

## Tabelas Afetadas

### `payouts`

| Campo | Tipo | Descrição |
|---|---|---|
| `provider` | `VARCHAR` | `"3xchange"` ou null (BlindPay) |
| `threexchange_order_id` | `VARCHAR UNIQUE` | UUID da ordem na 3xChange |
| `deposit_address` | `VARCHAR` | Endereço Solana para envio USDC |
| `net_amount_brl` | `DOUBLE PRECISION` | Valor líquido em BRL após taxas |
| `locked_at` | `TIMESTAMP` | Timestamp do CAS lock (null = sem lock) |
| `pix_end_to_end_id` | `VARCHAR` | ID E2E do PIX (após conclusão) |

### `pix_keys`

| Campo | Tipo | Descrição |
|---|---|---|
| `blindpay_bank_account_id` | `VARCHAR NULL` | Nullable — usuários 3xChange não têm conta BlindPay |
| `pix_key_type` | `VARCHAR` | CPF / CNPJ / EMAIL / PHONE / RANDOM |

### Migration SQL

```bash
psql $DATABASE_URL -f prisma/scripts/feat-3xchange-payout.sql
```

---

## Pendências

- [ ] **Migration SQL não aplicada em prod/staging** — executar `feat-3xchange-payout.sql`
- [ ] **Cron do Reconcile** — `reconcilePayouts3xService.run()` precisa ser chamado a cada 5 minutos
- [ ] **Notificação WebSocket** — notificar usuário via socket quando webhook `payout_completed` chegar
- [ ] **Fase 6 (Cleanup)** — remover Privy do Oraculo após migração completa (`bun remove @privy-io/node`, deletar `privy.service.ts`, remover env vars `PRIVY_*`)
