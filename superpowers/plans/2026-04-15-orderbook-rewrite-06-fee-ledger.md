# Orderbook Rewrite — Plano 6: Fee ledger integration

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Integrar o fee ledger existente (`FeeLedgerService` + `FeeEvent` model) ao fluxo de settlement do trading-v2. Hoje:
1. `IntentClassifier` reserva `notional + fee` em USDC mas o settler consome só `notional`. O "resto" fica travado em `reserved` sem sentido semântico.
2. O usuário na ponta seller paga fee on-chain (via `settle_clob_sell`) mas nosso DB credita o `notional` completo em `free`. Drift DB↔on-chain.
3. Nenhum registro no `FeeEvent` ledger do trading-v2, só do legacy.

Plano 6 fecha essas três lacunas de forma coordenada.

**Architecture:** `feeBps` persistido em `Ob2Order` e copiado pro `Ob2Trade` no match. `StubSettlerService.applyDeltas` passa a computar fee por side, consumir a porção de fee da reservation (quando a reserva é USDC), e ajustar o USDC creditado ao counterparty subtraindo a fee do lado que recebe (seller em TRADE com `buyerRes=USDC`, ambos em MERGE). `SolanaSettlerService` chama `feeLedgerService.record()` NO MESMO tx em que marca SETTLED (junto do ob2_onchain_events_processed insert) — garantindo que fee só é registrada quando on-chain confirmou. `SettlementReverter` não precisa undo de FeeEvent porque revert só ocorre antes de SETTLED (fee ainda não foi registrada).

**Tech Stack:** Bun, TypeScript, Prisma 7, Jest. Reuso do `FeeLedgerService` existente em `api/src/shared/services/fee-ledger.service.ts`.

**Spec:** `docs/superpowers/specs/2026-04-15-orderbook-rewrite-design.md` (§8 fees)
**Planos anteriores:**
- Plano 1 (`c891f0b`) — fundação
- Plano 2 (`c591534`) — orders + matching
- Plano 3 (`fe88950`) — SolanaSettler + reverter + deadline worker
- Plano 4 (`9674e09`) — SolanaOnchainCaller real
- Plano 5 (`0260dd6`) — listener + idempotência via memo

**Próximos planos:**
- Plano 7: WebSocket v2 • Plano 8: MM bot externo • Plano 9: Cutover

**Não-escopo:**
- **Histórico retroativo**: não re-emitimos FeeEvents pra trades já SETTLED antes de Plano 6. Ledger arranca vazio pro trading-v2 daqui pra frente.
- **Fee rate dinâmico por mercado**: hoje todos os trades usam o feeBps do `PlaceOrderInput` (configurável por chamada). Enforcement no nível do market (via `market.fee_bps` central) fica pro futuro.
- **Revert de FeeEvent**: como fee só é registrada no tx de SETTLED (pós-confirmação on-chain), revert (pré-SETTLED) não precisa undo.

---

## File Structure

```
api/
  prisma/schema.prisma                         # MODIFY: +feeBps em Ob2Order, Ob2Trade
  prisma/scripts/trading-v2-fees.sql           # CREATE: CHECK + default pra feeBps
  src/modules/trading-v2/
    types/trade.types.ts                       # MODIFY: TradeRecord.feeBps
    types/order.types.ts                       # MODIFY: OrderView.feeBps; CreateOrderRow já tem
    repositories/
      order.repository.ts                      # MODIFY: persist feeBps on create, map em toView
      trade.repository.ts                      # MODIFY: create accepts feeBps, map em toRecord
    services/
      fee-computation.service.ts               # CREATE: pure — computa fee amounts por side/primitive
      solana-settler.service.ts                # MODIFY: record FeeEvents no tx de SETTLED
      reconcile-settling-trades.service.ts     # MODIFY: settleFromEvent também grava FeeEvents
      stub-settler.service.ts                  # MODIFY: applyDeltas deduz fee do USDC creditado ao counterparty
      matching-engine.service.ts               # MODIFY: copia feeBps da taker order pro trade
    use-cases/
      place-order.use-case.ts                  # MODIFY: passa feeBps pra orders.create
    __tests__/
      fee-computation.unit.test.ts             # CREATE
      stub-settler-with-fees.integration.test.ts # CREATE
      fee-ledger-integration.integration.test.ts # CREATE
    index.ts                                   # MODIFY: export FeeComputation
    README.md                                  # MODIFY: Plano 6 section
```

**Responsabilidades (novos):**

- `FeeComputation` (pure, no IO) — dado `(primitive, priceBps, quantityMicro, feeBps)`, retorna `{ buyerFeeMicro, sellerFeeMicro, buyerUsdcReceivedMicro, sellerUsdcReceivedMicro }`. Única fonte da matemática de fees.
- `FeeEvent` records são despachados via `feeLedgerService.record()` (existente; reuse sem mudar).

---

## Prerequisite check

- [ ] **Step 0: Baseline.**

```bash
cd /Users/rafael/Documents/GitHub/Oraculo/api
bun x jest src/modules/trading-v2 --runInBand
```

Expected: 120 passed + 1 skipped (post-Plano 5).

- [ ] **Step 0.1: Worktree.**

```bash
git worktree add .claude/worktrees/orderbook-rewrite-06-fee-ledger -b worktree-orderbook-rewrite-06-fee-ledger
cp api/.env .claude/worktrees/orderbook-rewrite-06-fee-ledger/api/.env
cd .claude/worktrees/orderbook-rewrite-06-fee-ledger/api
bun install
bun x prisma generate
bun x jest src/modules/trading-v2 --runInBand   # confirma 120 no worktree
```

---

## Task 1: FeeComputation pure service

**Files:**
- Create: `api/src/modules/trading-v2/services/fee-computation.service.ts`
- Create: `api/src/modules/trading-v2/__tests__/fee-computation.unit.test.ts`

Pure function encapsula TODA a matemática de fees. Consumer único: settlers (stub + solana) e reverter (pra validar que não há fee pra reverter — sempre retorna 0 na tabela de revert).

- [ ] **Step 1.1: Failing tests.**

```typescript
import { FeeComputation } from "../services/fee-computation.service";
import { UNIT } from "../types/balance.types";

const fc = new FeeComputation();

// Ajuda: qty=100, price=6000 (0.60), fee=50bps (0.5%)
// buyerUsdcLeg = qty * 6000/10000 = 60 USDC
// sellerUsdcLeg = qty * 4000/10000 = 40 USDC

test("TRADE USDC×YES fees: buyer pays fee on buyerUsdcLeg, seller pays fee on buyerUsdcLeg", () => {
  const r = fc.compute({
    primitive: "TRADE", priceBps: 6000, quantityMicro: 100n * UNIT, feeBps: 50,
    buyerReservationAsset: "USDC", sellerReservationAsset: "YES",
  });
  // Both fees = 60 * 0.005 = 0.30 USDC = 300_000n micro
  expect(r.buyerFeeMicro).toBe(300_000n);
  expect(r.sellerFeeMicro).toBe(300_000n);
  // Buyer consumes: notional + fee = 60.30 from USDC reservation
  expect(r.buyerUsdcConsumedMicro).toBe(60_300_000n);
  // Seller doesn't consume USDC (reservation is YES token, unchanged)
  expect(r.sellerUsdcConsumedMicro).toBe(0n);
  // Seller receives (notional - sellerFee) = 59.70 USDC in free
  expect(r.sellerUsdcReceivedMicro).toBe(59_700_000n);
  expect(r.buyerUsdcReceivedMicro).toBe(0n);
});

test("TRADE NO(BUY)×USDC(SELL) fees: symmetric with sellerUsdcLeg", () => {
  const r = fc.compute({
    primitive: "TRADE", priceBps: 6000, quantityMicro: 100n * UNIT, feeBps: 50,
    buyerReservationAsset: "NO", sellerReservationAsset: "USDC",
  });
  // buyer closes NO short: receives sellerUsdcLeg = 40 USDC, fee = 40 * 0.005 = 0.20
  // seller opens NO short: consumes sellerUsdcLeg + fee = 40.20
  expect(r.buyerFeeMicro).toBe(200_000n);
  expect(r.sellerFeeMicro).toBe(200_000n);
  expect(r.buyerUsdcReceivedMicro).toBe(39_800_000n);   // 40 - 0.20
  expect(r.sellerUsdcConsumedMicro).toBe(40_200_000n);  // 40 + 0.20
  expect(r.buyerUsdcConsumedMicro).toBe(0n);
  expect(r.sellerUsdcReceivedMicro).toBe(0n);
});

test("MINT fees: each side pays fee on its respective USDC leg", () => {
  const r = fc.compute({
    primitive: "MINT", priceBps: 6000, quantityMicro: 100n * UNIT, feeBps: 50,
    buyerReservationAsset: "USDC", sellerReservationAsset: "USDC",
  });
  // buyer: consumes buyerUsdcLeg + fee = 60.30
  // seller: consumes sellerUsdcLeg + fee = 40.20
  expect(r.buyerFeeMicro).toBe(300_000n);
  expect(r.sellerFeeMicro).toBe(200_000n);
  expect(r.buyerUsdcConsumedMicro).toBe(60_300_000n);
  expect(r.sellerUsdcConsumedMicro).toBe(40_200_000n);
  // No USDC received — both get tokens
  expect(r.buyerUsdcReceivedMicro).toBe(0n);
  expect(r.sellerUsdcReceivedMicro).toBe(0n);
});

test("MERGE fees: both sides RECEIVE USDC, pay fee deducted from receipt", () => {
  const r = fc.compute({
    primitive: "MERGE", priceBps: 6000, quantityMicro: 100n * UNIT, feeBps: 50,
    buyerReservationAsset: "NO", sellerReservationAsset: "YES",
  });
  // buyer receives sellerUsdcLeg - fee = 40 - 0.20 = 39.80
  // seller receives buyerUsdcLeg - fee = 60 - 0.30 = 59.70
  expect(r.buyerUsdcReceivedMicro).toBe(39_800_000n);
  expect(r.sellerUsdcReceivedMicro).toBe(59_700_000n);
  expect(r.buyerFeeMicro).toBe(200_000n);
  expect(r.sellerFeeMicro).toBe(300_000n);
  expect(r.buyerUsdcConsumedMicro).toBe(0n);
  expect(r.sellerUsdcConsumedMicro).toBe(0n);
});

test("feeBps=0 means zero fees everywhere", () => {
  const r = fc.compute({
    primitive: "TRADE", priceBps: 6000, quantityMicro: 100n * UNIT, feeBps: 0,
    buyerReservationAsset: "USDC", sellerReservationAsset: "YES",
  });
  expect(r.buyerFeeMicro).toBe(0n);
  expect(r.sellerFeeMicro).toBe(0n);
  expect(r.buyerUsdcConsumedMicro).toBe(60_000_000n);   // exact notional
  expect(r.sellerUsdcReceivedMicro).toBe(60_000_000n);  // exact notional
});

test("rejects negative feeBps and feeBps > 10000", () => {
  expect(() => fc.compute({
    primitive: "TRADE", priceBps: 6000, quantityMicro: 100n * UNIT, feeBps: -1,
    buyerReservationAsset: "USDC", sellerReservationAsset: "YES",
  })).toThrow(/fee/);
  expect(() => fc.compute({
    primitive: "TRADE", priceBps: 6000, quantityMicro: 100n * UNIT, feeBps: 10001,
    buyerReservationAsset: "USDC", sellerReservationAsset: "YES",
  })).toThrow(/fee/);
});
```

- [ ] **Step 1.2: Run, fail.**

```bash
cd /Users/rafael/Documents/GitHub/Oraculo/.claude/worktrees/orderbook-rewrite-06-fee-ledger/api
bun x jest src/modules/trading-v2/__tests__/fee-computation.unit.test.ts --runInBand
```

- [ ] **Step 1.3: Implement.**

```typescript
import type { Ob2Asset, Ob2Primitive } from "../../../generated/prisma/client";

export interface FeeComputationInput {
  primitive: Ob2Primitive;
  priceBps: number;        // 1..9999
  quantityMicro: bigint;
  feeBps: number;          // 0..10000
  buyerReservationAsset: Ob2Asset;
  sellerReservationAsset: Ob2Asset;
}

export interface FeeComputationResult {
  /** Fee em micro-USDC paga pelo BUYER (vai pra fee wallet on-chain). */
  buyerFeeMicro: bigint;
  /** Fee em micro-USDC paga pelo SELLER (vai pra fee wallet on-chain). */
  sellerFeeMicro: bigint;
  /** USDC consumido do buyer: notional + fee (só quando buyer reserva USDC). */
  buyerUsdcConsumedMicro: bigint;
  /** USDC consumido do seller: notional + fee (só quando seller reserva USDC). */
  sellerUsdcConsumedMicro: bigint;
  /** USDC que o buyer RECEBE (credit em free) quando aplicável. */
  buyerUsdcReceivedMicro: bigint;
  /** USDC que o seller RECEBE (credit em free) quando aplicável. */
  sellerUsdcReceivedMicro: bigint;
}

/**
 * Pure: calcula todos os amounts de USDC e fees para um trade.
 * Única fonte da matemática de fees no módulo trading-v2.
 */
export class FeeComputation {
  compute(input: FeeComputationInput): FeeComputationResult {
    if (input.feeBps < 0 || input.feeBps > 10000) {
      throw new Error(`fee out of range: ${input.feeBps}`);
    }

    const qty = input.quantityMicro;
    const buyerUsdcLeg  = (qty * BigInt(input.priceBps)) / 10000n;
    const sellerUsdcLeg = (qty * BigInt(10000 - input.priceBps)) / 10000n;
    const fb = BigInt(input.feeBps);

    const feeOnBuyerLeg  = (buyerUsdcLeg  * fb) / 10000n;
    const feeOnSellerLeg = (sellerUsdcLeg * fb) / 10000n;

    switch (input.primitive) {
      case "TRADE":
        if (input.buyerReservationAsset === "USDC" && input.sellerReservationAsset === "YES") {
          return {
            buyerFeeMicro: feeOnBuyerLeg,
            sellerFeeMicro: feeOnBuyerLeg,
            buyerUsdcConsumedMicro: buyerUsdcLeg + feeOnBuyerLeg,
            sellerUsdcConsumedMicro: 0n,
            buyerUsdcReceivedMicro: 0n,
            sellerUsdcReceivedMicro: buyerUsdcLeg - feeOnBuyerLeg,
          };
        }
        // NO(BUY) × USDC(SELL): buyer receives sellerUsdcLeg minus fee, seller consumes sellerUsdcLeg plus fee
        if (input.buyerReservationAsset === "NO" && input.sellerReservationAsset === "USDC") {
          return {
            buyerFeeMicro: feeOnSellerLeg,
            sellerFeeMicro: feeOnSellerLeg,
            buyerUsdcConsumedMicro: 0n,
            sellerUsdcConsumedMicro: sellerUsdcLeg + feeOnSellerLeg,
            buyerUsdcReceivedMicro: sellerUsdcLeg - feeOnSellerLeg,
            sellerUsdcReceivedMicro: 0n,
          };
        }
        throw new Error(`TRADE with unexpected reservation pair: ${input.buyerReservationAsset}/${input.sellerReservationAsset}`);

      case "MINT":
        // Buyer pays priceYes cents per YES; seller pays priceNo cents per NO. Both pay fee on their leg.
        return {
          buyerFeeMicro: feeOnBuyerLeg,
          sellerFeeMicro: feeOnSellerLeg,
          buyerUsdcConsumedMicro: buyerUsdcLeg + feeOnBuyerLeg,
          sellerUsdcConsumedMicro: sellerUsdcLeg + feeOnSellerLeg,
          buyerUsdcReceivedMicro: 0n,
          sellerUsdcReceivedMicro: 0n,
        };

      case "MERGE":
        // Both sides RECEIVE USDC (vault burns tokens, returns complement).
        // Fee deducted from receipt on each side.
        return {
          buyerFeeMicro: feeOnSellerLeg,       // buyer recebe sellerUsdcLeg
          sellerFeeMicro: feeOnBuyerLeg,       // seller recebe buyerUsdcLeg
          buyerUsdcConsumedMicro: 0n,
          sellerUsdcConsumedMicro: 0n,
          buyerUsdcReceivedMicro: sellerUsdcLeg - feeOnSellerLeg,
          sellerUsdcReceivedMicro: buyerUsdcLeg - feeOnBuyerLeg,
        };
    }
  }
}
```

- [ ] **Step 1.4: Run, 6 tests pass.**

- [ ] **Step 1.5: Commit.**

```bash
git add api/src/modules/trading-v2/services/fee-computation.service.ts \
        api/src/modules/trading-v2/__tests__/fee-computation.unit.test.ts
git commit -m "feat(trading-v2): FeeComputation pure (matemática de fees)"
```

---

## Task 2: Schema — feeBps em Ob2Order e Ob2Trade

**Files:**
- Modify: `api/prisma/schema.prisma`
- Create: `api/prisma/scripts/trading-v2-fees.sql`

- [ ] **Step 2.1: Modify `prisma/schema.prisma`.**

Locate `model Ob2Order` and add after `priceBps`:

```prisma
  feeBps        Int             @default(0) @map("fee_bps")
```

Locate `model Ob2Trade` and add after `priceBps`:

```prisma
  feeBps           Int              @default(0) @map("fee_bps")
```

- [ ] **Step 2.2: Create `prisma/scripts/trading-v2-fees.sql`.**

```sql
-- Idempotent CHECK constraints for fee_bps range (0..10000 inclusive).
-- Prisma default is already 0 so no backfill needed.

DO $$ BEGIN
  ALTER TABLE ob2_orders ADD CONSTRAINT ob2_orders_fee_range CHECK (fee_bps >= 0 AND fee_bps <= 10000);
EXCEPTION WHEN duplicate_object THEN NULL; END $$;

DO $$ BEGIN
  ALTER TABLE ob2_trades ADD CONSTRAINT ob2_trades_fee_range CHECK (fee_bps >= 0 AND fee_bps <= 10000);
EXCEPTION WHEN duplicate_object THEN NULL; END $$;
```

- [ ] **Step 2.3: Apply:**

```bash
cd /Users/rafael/Documents/GitHub/Oraculo/.claude/worktrees/orderbook-rewrite-06-fee-ledger/api
bun x prisma db push
bun x prisma generate
DATABASE_URL_PSQL=$(grep "^DATABASE_URL=" .env | sed 's/DATABASE_URL=//; s/"//g')
psql "$DATABASE_URL_PSQL" -f prisma/scripts/trading-v2-fees.sql
```

- [ ] **Step 2.4: Verify.**

```bash
psql "$DATABASE_URL_PSQL" -c "SELECT column_name, data_type, column_default FROM information_schema.columns WHERE table_name IN ('ob2_orders','ob2_trades') AND column_name='fee_bps';"
```

Expected: 2 rows (one per table), `integer`, default `0`.

- [ ] **Step 2.5: Baseline suite still green.**

```bash
bun x jest src/modules/trading-v2 --runInBand
```

Expected: 120 + 6 (Task 1) = 126 passed + 1 skipped.

- [ ] **Step 2.6: Commit.**

```bash
git add api/prisma/schema.prisma api/prisma/scripts/trading-v2-fees.sql
git commit -m "feat(trading-v2): schema feeBps em Ob2Order e Ob2Trade"
```

---

## Task 3: Propagate feeBps: OrderRepository, MatchingEngine, TradeRepository, types

**Files:**
- Modify: `api/src/modules/trading-v2/types/order.types.ts` (+ feeBps no OrderView)
- Modify: `api/src/modules/trading-v2/types/trade.types.ts` (+ feeBps no TradeRecord)
- Modify: `api/src/modules/trading-v2/repositories/order.repository.ts` (persist + view)
- Modify: `api/src/modules/trading-v2/repositories/trade.repository.ts` (persist + view)
- Modify: `api/src/modules/trading-v2/use-cases/place-order.use-case.ts` (passa feeBps)
- Modify: `api/src/modules/trading-v2/services/matching-engine.service.ts` (copia pro trade)

- [ ] **Step 3.1: `types/order.types.ts`** — append `feeBps: number;` em `OrderView`.

Locate `OrderView` interface and add:

```typescript
  feeBps: number;
```

- [ ] **Step 3.2: `types/trade.types.ts`** — append `feeBps: number;` em `TradeRecord`.

- [ ] **Step 3.3: `order.repository.ts`:**

Modify `CreateOrderRow`:

```typescript
export interface CreateOrderRow {
  id?: string;
  userId: string;
  marketPda: string;
  side: Ob2Side;
  priceBps: number;
  feeBps: number;                      // NEW (no longer optional — caller must provide)
  quantity: bigint;
  reservationId: string;
  clientOrderId?: string;
}
```

Update `create` body — add to `data: {...}`:

```typescript
        feeBps: input.feeBps,
```

Update `toView`:

```typescript
  priceBps: row.priceBps,
  feeBps: row.feeBps,                // NEW
  quantity: toBig(row.quantity),
```

- [ ] **Step 3.4: `trade.repository.ts`:**

Modify `CreateTradeInput`:

```typescript
export interface CreateTradeInput {
  marketPda: string;
  makerOrderId: string;
  takerOrderId: string;
  priceBps: number;
  feeBps: number;                      // NEW
  quantity: bigint;
  primitive: Ob2Primitive;
  sync: boolean;
}
```

Update `create` body — add to `data`:

```typescript
        feeBps: input.feeBps,
```

Update `toRecord` — add `feeBps: row.feeBps` after `priceBps`.

- [ ] **Step 3.5: `use-cases/place-order.use-case.ts`:**

Find the `orders.create` call. Update to pass feeBps:

```typescript
    const order = await this.orders.create({
      id: orderId,
      userId: input.userId, marketPda: input.marketPda, side: input.side,
      priceBps: input.priceBps, feeBps: input.feeBps,
      quantity: input.quantity,
      reservationId: reservation.id, clientOrderId: input.clientOrderId,
    });
```

- [ ] **Step 3.6: `matching-engine.service.ts`:**

Find the block that creates the trade inside `tryMatch`'s `$transaction`. The trade is created via `tx.ob2Trade.create({ data: { ... } })`. The taker's feeBps (available from `taker.feeBps` since we now have it on OrderView) must be copied:

```typescript
        const tradeRow = await tx.ob2Trade.create({
          data: {
            marketPda: taker!.marketPda,
            makerOrderId: makerRow.id,
            takerOrderId: taker!.id,
            priceBps: makerRow.price_bps,
            feeBps: taker!.feeBps,          // NEW — taker's fee rate applies
            quantity: this.toDecimal(fillQty),
            primitive,
            status: "SETTLING",
            sync: true,
            settlingDeadline: new Date(Date.now() + 30_000),
          },
        });
```

- [ ] **Step 3.7: Fix test setups that create orders directly.**

Run:

```bash
bun x jest src/modules/trading-v2 --runInBand 2>&1 | tail -20
```

Errors like "Property 'feeBps' is missing in type 'CreateOrderRow'" will appear. For EACH test file that calls `orderRepo.create({ ... })` directly (e.g., `order.repository.integration.test.ts`, various integration tests), add `feeBps: 0` to the object.

Similarly for `tradeRepo.create({ ... })` calls — add `feeBps: 0`.

Expected fixes needed in: `order.repository.integration.test.ts`, `trade.repository.integration.test.ts`, `stub-settler.integration.test.ts`, `matching-engine.integration.test.ts`, `settlement-reverter.integration.test.ts`, `solana-settler-idempotency.integration.test.ts`, `solana-settler.integration.test.ts`, `reconcile-settling-trades.integration.test.ts`, `reconcile-settling-trades-with-event.integration.test.ts`, `onchain-event-listener.unit.test.ts`.

This is mechanical: find every `{ userId: ..., marketPda: ..., side: ..., priceBps: ..., quantity: ... }` for order creation and add `feeBps: 0` right after `priceBps`. Same for trade creation.

- [ ] **Step 3.8: Verify all tests still green.**

```bash
bun x jest src/modules/trading-v2 --runInBand
```

Expected: 126 passed + 1 skipped. If any fail due to new feeBps requirement, fix the test setup.

- [ ] **Step 3.9: tsc clean.**

```bash
bun x tsc --noEmit 2>&1 | grep trading-v2 || echo "clean"
```

- [ ] **Step 3.10: Commit.**

```bash
git add api/src/modules/trading-v2/types/ \
        api/src/modules/trading-v2/repositories/ \
        api/src/modules/trading-v2/use-cases/place-order.use-case.ts \
        api/src/modules/trading-v2/services/matching-engine.service.ts \
        api/src/modules/trading-v2/__tests__/
git commit -m "feat(trading-v2): propaga feeBps por OrderView/TradeRecord"
```

---

## Task 4: StubSettler usa FeeComputation; ajusta credits

**Files:**
- Modify: `api/src/modules/trading-v2/services/stub-settler.service.ts`
- Create: `api/src/modules/trading-v2/__tests__/stub-settler-with-fees.integration.test.ts`

Hoje `applyDeltas` hard-codes `buyerUsdcLeg = qty * priceBps / 10000` e credita isso full ao seller em TRADE. Plano 6: delega pra `FeeComputation.compute(...)` e usa o result pra consume + credit com fees aplicados.

- [ ] **Step 4.1: Failing test (fee integration with feeBps=50 on trade).**

```typescript
import "dotenv/config";
import { prisma } from "../../../shared/database/config/prisma-client";
import { BalanceRepository } from "../repositories/balance.repository";
import { ReservationService } from "../services/reservation.service";
import { OrderRepository } from "../repositories/order.repository";
import { TradeRepository } from "../repositories/trade.repository";
import { StubSettlerService } from "../services/stub-settler.service";
import { UNIT } from "../types/balance.types";

const balanceRepo = new BalanceRepository(prisma);
const resSvc = new ReservationService(prisma);
const orderRepo = new OrderRepository(prisma);
const tradeRepo = new TradeRepository(prisma);
const settler = new StubSettlerService(prisma);

const BUYER  = "00000000-0000-0000-0000-000000000001";
const SELLER = "00000000-0000-0000-0000-000000000002";
const MARKET = "Market1111111111111111111111111111111111111";
const FEE_BPS = 50;   // 0.5%

async function setupWithFees(
  primitive: "TRADE" | "MINT" | "MERGE",
): Promise<{ tradeId: string; buyerUsdcReserved: bigint; sellerUsdcReserved: bigint }> {
  await prisma.ob2Trade.deleteMany({});
  await prisma.ob2Order.deleteMany({});
  await prisma.ob2Reservation.deleteMany({});
  await prisma.ob2UserMarketBalance.deleteMany({});

  // Seed according to primitive
  if (primitive === "TRADE") {
    await balanceRepo.upsertOnchain(BUYER,  MARKET, "USDC", 1000n * UNIT, 1n);
    await balanceRepo.upsertOnchain(SELLER, MARKET, "YES",  100n  * UNIT, 1n);
  } else if (primitive === "MINT") {
    await balanceRepo.upsertOnchain(BUYER,  MARKET, "USDC", 1000n * UNIT, 1n);
    await balanceRepo.upsertOnchain(SELLER, MARKET, "USDC", 1000n * UNIT, 1n);
  } else {
    await balanceRepo.upsertOnchain(BUYER,  MARKET, "NO",  100n * UNIT, 1n);
    await balanceRepo.upsertOnchain(SELLER, MARKET, "YES", 100n * UNIT, 1n);
  }

  const buyerResAsset  = primitive === "MERGE" ? "NO"   : "USDC";
  const sellerResAsset = primitive === "MERGE" ? "YES"  : (primitive === "MINT" ? "USDC" : "YES");

  // Reservation amounts (matching what the IntentClassifier would produce for feeBps=50)
  // qty=100, price=0.60 → buyerUsdcLeg=60, sellerUsdcLeg=40
  // TRADE: buyer USDC = 60 + 0.30 = 60.30; seller YES = 100
  // MINT:  buyer USDC = 60 + 0.30 = 60.30; seller USDC = 40 + 0.20 = 40.20
  // MERGE: buyer NO = 100; seller YES = 100
  const buyerResAmt = primitive === "MERGE" ? 100n * UNIT
                    : primitive === "MINT"  ? 60_300_000n
                    : 60_300_000n;
  const sellerResAmt = primitive === "MERGE" ? 100n * UNIT
                    : primitive === "MINT"   ? 40_200_000n
                    : 100n * UNIT;

  const bo = await orderRepo.create({
    userId: BUYER, marketPda: MARKET, side: "BUY",
    priceBps: 6000, feeBps: FEE_BPS, quantity: 100n * UNIT, reservationId: "p",
  });
  const br = await resSvc.reserve({ userId: BUYER, marketPda: MARKET, asset: buyerResAsset, amount: buyerResAmt, orderId: bo.id });
  await prisma.ob2Order.update({ where: { id: bo.id }, data: { reservationId: br.id } });

  const so = await orderRepo.create({
    userId: SELLER, marketPda: MARKET, side: "SELL",
    priceBps: 6000, feeBps: FEE_BPS, quantity: 100n * UNIT, reservationId: "p",
  });
  const sr = await resSvc.reserve({ userId: SELLER, marketPda: MARKET, asset: sellerResAsset, amount: sellerResAmt, orderId: so.id });
  await prisma.ob2Order.update({ where: { id: so.id }, data: { reservationId: sr.id } });

  const t = await tradeRepo.create({
    marketPda: MARKET, makerOrderId: so.id, takerOrderId: bo.id,
    priceBps: 6000, feeBps: FEE_BPS, quantity: 100n * UNIT,
    primitive, sync: true,
  });

  return {
    tradeId: t.id,
    buyerUsdcReserved: buyerResAsset === "USDC" ? buyerResAmt : 0n,
    sellerUsdcReserved: sellerResAsset === "USDC" ? sellerResAmt : 0n,
  };
}

afterAll(async () => { await prisma.$disconnect(); });

test("TRADE with 50bps fee: buyer consumes notional+fee, seller receives notional-fee", async () => {
  const { tradeId } = await setupWithFees("TRADE");
  await settler.settle(tradeId);

  // Buyer: reserved 60.30 USDC, consumed 60.30 (all of it). Free untouched except initial.
  const buyerUsdc = await balanceRepo.get(BUYER, MARKET, "USDC");
  expect(buyerUsdc!.reserved).toBe(0n);
  expect(buyerUsdc!.free).toBe(1000n * UNIT - 60_300_000n);   // 939.70
  // Buyer receives YES
  expect((await balanceRepo.get(BUYER, MARKET, "YES"))!.free).toBe(100n * UNIT);

  // Seller: consumed YES qty=100, received USDC = 60 - 0.30 = 59.70
  expect((await balanceRepo.get(SELLER, MARKET, "YES"))!.free).toBe(0n);
  expect((await balanceRepo.get(SELLER, MARKET, "USDC"))!.free).toBe(59_700_000n);
});

test("MERGE with 50bps fee: both sides get USDC minus their respective fee", async () => {
  const { tradeId } = await setupWithFees("MERGE");
  await settler.settle(tradeId);

  // Buyer receives sellerUsdcLeg - feeOnSeller = 40 - 0.20 = 39.80
  expect((await balanceRepo.get(BUYER, MARKET, "USDC"))!.free).toBe(39_800_000n);
  // Seller receives buyerUsdcLeg - feeOnBuyer = 60 - 0.30 = 59.70
  expect((await balanceRepo.get(SELLER, MARKET, "USDC"))!.free).toBe(59_700_000n);
});

test("feeBps=0 preserves old behavior", async () => {
  await prisma.ob2Trade.deleteMany({});
  await prisma.ob2Order.deleteMany({});
  await prisma.ob2Reservation.deleteMany({});
  await prisma.ob2UserMarketBalance.deleteMany({});
  await balanceRepo.upsertOnchain(BUYER,  MARKET, "USDC", 1000n * UNIT, 1n);
  await balanceRepo.upsertOnchain(SELLER, MARKET, "YES",  100n  * UNIT, 1n);

  const bo = await orderRepo.create({ userId: BUYER, marketPda: MARKET, side: "BUY", priceBps: 6000, feeBps: 0, quantity: 100n * UNIT, reservationId: "p" });
  const br = await resSvc.reserve({ userId: BUYER, marketPda: MARKET, asset: "USDC", amount: 60n * UNIT, orderId: bo.id });
  await prisma.ob2Order.update({ where: { id: bo.id }, data: { reservationId: br.id } });
  const so = await orderRepo.create({ userId: SELLER, marketPda: MARKET, side: "SELL", priceBps: 6000, feeBps: 0, quantity: 100n * UNIT, reservationId: "p" });
  const sr = await resSvc.reserve({ userId: SELLER, marketPda: MARKET, asset: "YES", amount: 100n * UNIT, orderId: so.id });
  await prisma.ob2Order.update({ where: { id: so.id }, data: { reservationId: sr.id } });
  const t = await tradeRepo.create({
    marketPda: MARKET, makerOrderId: so.id, takerOrderId: bo.id,
    priceBps: 6000, feeBps: 0, quantity: 100n * UNIT, primitive: "TRADE", sync: true,
  });

  await settler.settle(t.id);

  expect((await balanceRepo.get(BUYER, MARKET, "USDC"))!.free).toBe(1000n * UNIT - 60n * UNIT);   // 940
  expect((await balanceRepo.get(SELLER, MARKET, "USDC"))!.free).toBe(60n * UNIT);                  // exact notional
});
```

- [ ] **Step 4.2: Run, fail.**

- [ ] **Step 4.3: Modify `stub-settler.service.ts`.**

Add at top:

```typescript
import { FeeComputation } from "./fee-computation.service";
```

Add a private instance in the class:

```typescript
  private readonly fees = new FeeComputation();
```

In `applyDeltas`, replace the per-primitive block with FeeComputation-driven logic.

The method currently computes `buyerUsdcLeg`, `sellerUsdcLeg` inline and dispatches per primitive. Replace the entire switch with:

```typescript
    const fees = this.fees.compute({
      primitive: trade.primitive,
      priceBps: trade.priceBps,
      quantityMicro: qty,
      feeBps: trade.feeBps,
      buyerReservationAsset: buyerRes.asset,
      sellerReservationAsset: sellerRes.asset,
    });

    // Debit from reservations (per side, only USDC legs have consume)
    if (buyerRes.asset === "USDC") {
      await this.debitReservedAndShrink(tx, buyerOrder.userId, trade.marketPda, "USDC", fees.buyerUsdcConsumedMicro, buyerRes.id);
    }
    if (sellerRes.asset === "USDC") {
      await this.debitReservedAndShrink(tx, sellerOrder.userId, trade.marketPda, "USDC", fees.sellerUsdcConsumedMicro, sellerRes.id);
    }
    // Token reservations are always consumed at full qty
    if (buyerRes.asset === "NO") {
      await this.debitReservedAndShrink(tx, buyerOrder.userId, trade.marketPda, "NO", qty, buyerRes.id);
    }
    if (sellerRes.asset === "YES") {
      await this.debitReservedAndShrink(tx, sellerOrder.userId, trade.marketPda, "YES", qty, sellerRes.id);
    }

    // Credit free on receiving side (per primitive)
    switch (trade.primitive) {
      case "TRADE":
        if (buyerRes.asset === "USDC" && sellerRes.asset === "YES") {
          await this.creditFree(tx, buyerOrder.userId,  trade.marketPda, "YES",  qty);
          await this.creditFree(tx, sellerOrder.userId, trade.marketPda, "USDC", fees.sellerUsdcReceivedMicro);
        } else if (buyerRes.asset === "NO" && sellerRes.asset === "USDC") {
          await this.creditFree(tx, buyerOrder.userId,  trade.marketPda, "USDC", fees.buyerUsdcReceivedMicro);
          await this.creditFree(tx, sellerOrder.userId, trade.marketPda, "NO",   qty);
        } else {
          throw new Error(`TRADE primitive with unexpected asset pair ${buyerRes.asset}/${sellerRes.asset}`);
        }
        break;
      case "MINT":
        await this.creditFree(tx, buyerOrder.userId,  trade.marketPda, "YES", qty);
        await this.creditFree(tx, sellerOrder.userId, trade.marketPda, "NO",  qty);
        break;
      case "MERGE":
        await this.creditFree(tx, buyerOrder.userId,  trade.marketPda, "USDC", fees.buyerUsdcReceivedMicro);
        await this.creditFree(tx, sellerOrder.userId, trade.marketPda, "USDC", fees.sellerUsdcReceivedMicro);
        break;
      default:
        throw new Error(`unknown primitive ${trade.primitive}`);
    }
```

**Important:** remove the previous in-method computation of `buyerUsdcLeg` / `sellerUsdcLeg` (if any) — FeeComputation is now the source. Keep `qty = toMicro(trade.quantity)` just before calling `this.fees.compute(...)`.

- [ ] **Step 4.4: Run the new fee tests, 3 pass.**

- [ ] **Step 4.5: Run full suite — earlier tests must still pass (they used feeBps=0).**

```bash
bun x jest src/modules/trading-v2 --runInBand
```

Expected: 126 + 3 = 129 passed + 1 skipped.

- [ ] **Step 4.6: Commit.**

```bash
git add api/src/modules/trading-v2/services/stub-settler.service.ts \
        api/src/modules/trading-v2/__tests__/stub-settler-with-fees.integration.test.ts
git commit -m "feat(trading-v2): StubSettler usa FeeComputation + ajusta credits com fee"
```

---

## Task 5: SolanaSettler grava FeeEvents no tx de SETTLED

**Files:**
- Modify: `api/src/modules/trading-v2/services/solana-settler.service.ts`
- Modify: `api/src/modules/trading-v2/services/reconcile-settling-trades.service.ts`
- Create: `api/src/modules/trading-v2/__tests__/fee-ledger-integration.integration.test.ts`

FeeEvent SÓ é gravado quando o trade transiciona pra SETTLED. Portanto: 2 pontos de emissão:
1. `SolanaSettler` na branch `if (res.ok)` (caminho feliz).
2. `ReconcileSettlingTradesService.settleFromEvent` (recovery via listener).

Ambos após on-chain success confirmado. Nenhum revert precisa undo.

- [ ] **Step 5.1: Failing test.**

```typescript
import "dotenv/config";
import { prisma } from "../../../shared/database/config/prisma-client";
import { BalanceRepository } from "../repositories/balance.repository";
import { ReservationService } from "../services/reservation.service";
import { OrderRepository } from "../repositories/order.repository";
import { TradeRepository } from "../repositories/trade.repository";
import { StubSettlerService } from "../services/stub-settler.service";
import { SettlementReverter } from "../services/settlement-reverter.service";
import { SolanaSettlerService } from "../services/solana-settler.service";
import { MockOnchainCaller } from "../services/mock-onchain-caller.service";
import { OnchainEventRepository } from "../repositories/onchain-event.repository";
import { UNIT } from "../types/balance.types";

const balanceRepo = new BalanceRepository(prisma);
const resSvc = new ReservationService(prisma);
const orderRepo = new OrderRepository(prisma);
const tradeRepo = new TradeRepository(prisma);
const stub = new StubSettlerService(prisma);
const reverter = new SettlementReverter(prisma, tradeRepo, orderRepo);
const events = new OnchainEventRepository(prisma);

const BUYER  = "00000000-0000-0000-0000-000000000001";
const SELLER = "00000000-0000-0000-0000-000000000002";
const BUYER_WALLET  = "BuyerWalletxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx";
const SELLER_WALLET = "SellerWalletxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx";
const MARKET = "Market1111111111111111111111111111111111111";

async function setupWithUsers(): Promise<string> {
  await prisma.feeEvent.deleteMany({ where: { marketPda: MARKET } });
  await prisma.ob2OnchainEvent.deleteMany({});
  await prisma.ob2Trade.deleteMany({});
  await prisma.ob2Order.deleteMany({});
  await prisma.ob2Reservation.deleteMany({});
  await prisma.ob2UserMarketBalance.deleteMany({});
  // Insert users with wallet addresses so FeeLedger can resolve them.
  // NOTE: project-specific — if User model requires more fields, adjust.
  await prisma.user.upsert({
    where: { id: 1 },
    create: { id: 1, walletAddress: BUYER_WALLET } as any,
    update: { walletAddress: BUYER_WALLET } as any,
  }).catch(() => {});
  await prisma.user.upsert({
    where: { id: 2 },
    create: { id: 2, walletAddress: SELLER_WALLET } as any,
    update: { walletAddress: SELLER_WALLET } as any,
  }).catch(() => {});

  await balanceRepo.upsertOnchain(BUYER,  MARKET, "USDC", 1000n * UNIT, 1n);
  await balanceRepo.upsertOnchain(SELLER, MARKET, "YES",  100n  * UNIT, 1n);

  const bo = await orderRepo.create({ userId: BUYER, marketPda: MARKET, side: "BUY", priceBps: 6000, feeBps: 50, quantity: 100n * UNIT, reservationId: "p" });
  const br = await resSvc.reserve({ userId: BUYER, marketPda: MARKET, asset: "USDC", amount: 60_300_000n, orderId: bo.id });
  await prisma.ob2Order.update({ where: { id: bo.id }, data: { reservationId: br.id } });

  const so = await orderRepo.create({ userId: SELLER, marketPda: MARKET, side: "SELL", priceBps: 6000, feeBps: 50, quantity: 100n * UNIT, reservationId: "p" });
  const sr = await resSvc.reserve({ userId: SELLER, marketPda: MARKET, asset: "YES", amount: 100n * UNIT, orderId: so.id });
  await prisma.ob2Order.update({ where: { id: so.id }, data: { reservationId: sr.id } });

  const t = await tradeRepo.create({
    marketPda: MARKET, makerOrderId: so.id, takerOrderId: bo.id,
    priceBps: 6000, feeBps: 50, quantity: 100n * UNIT, primitive: "TRADE", sync: true,
  });
  return t.id;
}

afterAll(async () => { await prisma.$disconnect(); });

test("SolanaSettler success path: 2 FeeEvent rows created (buyer + seller)", async () => {
  const tradeId = await setupWithUsers();
  const caller = new MockOnchainCaller({ mode: "success" });
  const settler = new SolanaSettlerService(prisma, tradeRepo, stub, reverter, caller, { eventRepo: events });

  await settler.settle(tradeId);

  const feeEvents = await prisma.feeEvent.findMany({ where: { tradeId } });
  expect(feeEvents).toHaveLength(2);
  // Amounts: buyerFee = 0.30; sellerFee = 0.30
  const amounts = feeEvents.map(e => String(e.amountUsdc)).sort();
  expect(amounts).toEqual(["0.3", "0.3"]);
  const wallets = feeEvents.map(e => e.userWallet).sort();
  expect(wallets).toEqual([BUYER_WALLET, SELLER_WALLET].sort());
  feeEvents.forEach(e => {
    expect(e.eventType).toBe("trading_taker");
    expect(e.destination).toBe("platform_fee_wallet");
  });
});

test("non-retryable failure: no FeeEvent recorded", async () => {
  const tradeId = await setupWithUsers();
  const caller = new MockOnchainCaller({ mode: "failure", reason: "program_error", retryable: false });
  const settler = new SolanaSettlerService(prisma, tradeRepo, stub, reverter, caller, { eventRepo: events });

  await settler.settle(tradeId);

  const feeEvents = await prisma.feeEvent.findMany({ where: { tradeId } });
  expect(feeEvents).toHaveLength(0);
});
```

**Note on `prisma.user.upsert`:** the `User` model in this project may have additional required fields. If the upsert fails due to missing columns, catch it silently — we just need SOME record that maps userId → wallet for the FeeEvent. If the test fails because lookup returns null, inline the wallet addresses by stubbing `walletLookup`. See the implementation step.

- [ ] **Step 5.2: Run, fail.**

- [ ] **Step 5.3: Modify `solana-settler.service.ts`.**

Add imports:

```typescript
import { feeLedgerService } from "@/shared/services/fee-ledger.service";
import { FeeComputation } from "./fee-computation.service";
```

Add private `fees = new FeeComputation()` in the class.

Add a `UserWalletLookup`-like lookup for the settler — or inject it. Simplest: add optional `userWalletLookup?: (userId: string) => Promise<string | null>` in the config; if missing, use a default inline Prisma lookup. The lookup returns the wallet address (string) or null; null wallets cause FeeEvent to be skipped (fee-ledger is non-blocking per its own implementation).

Extend `SolanaSettlerConfig`:

```typescript
export interface SolanaSettlerConfig {
  maxRetries?: number;
  retryDelayMs?: number;
  eventRepo?: OnchainEventRepository;
  userWalletLookup?: (userId: string) => Promise<string | null>;
}
```

Add field + constructor assignment (with a default that does `prisma.user.findFirst`):

```typescript
  private readonly userWalletLookup: (userId: string) => Promise<string | null>;

  constructor(
    // ... existing params ...
    config: SolanaSettlerConfig = {},
  ) {
    this.maxRetries = config.maxRetries ?? 3;
    this.retryDelayMs = config.retryDelayMs ?? 200;
    this.eventRepo = config.eventRepo ?? null;
    this.userWalletLookup = config.userWalletLookup ?? (async (userId: string) => {
      const idInt = Number.parseInt(userId, 10);
      const where = Number.isFinite(idInt) ? { id: idInt } : { sessionId: userId };
      const u = await this.prisma.user.findFirst({ where, select: { walletAddress: true } });
      return u?.walletAddress ?? null;
    });
  }
```

Inside the `if (res.ok) { ... }` branch, AFTER the SETTLED mark, emit the fee events (outside the `$transaction` since feeLedger is non-blocking and swallows errors):

```typescript
      if (res.ok) {
        await this.prisma.$transaction(async (tx) => {
          // ... existing: check status, applyDeltas skipped (already done pre-loop),
          // insert onchain event, mark SETTLED ...
        });
        // Fee events — emitted outside the transaction since fee-ledger is
        // non-blocking audit. Must be AFTER SETTLED commit (we don't want to
        // record fees if mark-SETTLED rolled back).
        await this.emitFeeEvents(tradeId, res.signature);
        return;
      }
```

Add `emitFeeEvents` method:

```typescript
  private async emitFeeEvents(tradeId: string, signature: string): Promise<void> {
    const trade = await this.trades.getById(tradeId);
    if (!trade) return;
    const makerOrder = await this.prisma.ob2Order.findUnique({ where: { id: trade.makerOrderId } });
    const takerOrder = await this.prisma.ob2Order.findUnique({ where: { id: trade.takerOrderId } });
    if (!makerOrder || !takerOrder) return;
    const buyerOrder  = takerOrder.side === "BUY"  ? takerOrder : makerOrder;
    const sellerOrder = takerOrder.side === "SELL" ? takerOrder : makerOrder;
    const makerRes = await this.prisma.ob2Reservation.findFirst({ where: { orderId: makerOrder.id } });
    const takerRes = await this.prisma.ob2Reservation.findFirst({ where: { orderId: takerOrder.id } });
    if (!makerRes || !takerRes) return;
    const buyerRes  = buyerOrder.id  === takerOrder.id ? takerRes : makerRes;
    const sellerRes = sellerOrder.id === takerOrder.id ? takerRes : makerRes;

    const fees = this.fees.compute({
      primitive: trade.primitive,
      priceBps: trade.priceBps,
      quantityMicro: toMicro(trade.quantity),
      feeBps: trade.feeBps,
      buyerReservationAsset: buyerRes.asset,
      sellerReservationAsset: sellerRes.asset,
    });

    const [buyerWallet, sellerWallet] = await Promise.all([
      this.userWalletLookup(buyerOrder.userId),
      this.userWalletLookup(sellerOrder.userId),
    ]);

    if (buyerWallet && fees.buyerFeeMicro > 0n) {
      await feeLedgerService.record({
        eventType: "trading_taker",
        tradeId,
        userWallet: buyerWallet,
        marketPda: trade.marketPda,
        amountUsdc: Number(fees.buyerFeeMicro) / 1_000_000,
        destination: "platform_fee_wallet",
        txSignature: signature,
        metadata: { side: "buyer", primitive: trade.primitive },
      });
    }
    if (sellerWallet && fees.sellerFeeMicro > 0n) {
      await feeLedgerService.record({
        eventType: "trading_taker",
        tradeId,
        userWallet: sellerWallet,
        marketPda: trade.marketPda,
        amountUsdc: Number(fees.sellerFeeMicro) / 1_000_000,
        destination: "platform_fee_wallet",
        txSignature: signature,
        metadata: { side: "seller", primitive: trade.primitive },
      });
    }
  }
```

Import `toMicro` from `../types/decimal-helpers` if not already.

- [ ] **Step 5.4: Modify `reconcile-settling-trades.service.ts`.**

Similar logic — when `settleFromEvent` marks a trade SETTLED via the recovery path, emit fee events using the same pattern. Easiest: extract the `emitFeeEvents` method into a new file `fee-emitter.service.ts` and call from both services. For Plano 6, duplicate inline if extraction feels heavy.

In `ReconcileConfig` add `userWalletLookup` + `feeComputation` fields. In `settleFromEvent`, after the `ob2Trade.update` that marks SETTLED, copy the emit logic from above (reuse a shared private method if it was extracted).

**Implementation flexibility:** if duplicating, add a TODO comment pointing to the consolidated emitter future cleanup. If extracting, create `fee-emitter.service.ts` with a single `emitForSettledTrade(tradeId, signature)` method that both services call.

- [ ] **Step 5.5: Run tests, 2 pass.**

```bash
bun x jest src/modules/trading-v2/__tests__/fee-ledger-integration.integration.test.ts --runInBand
```

Note: if User insertion fails due to missing required columns, the test can inline `config.userWalletLookup` to return hardcoded BUYER_WALLET/SELLER_WALLET based on userId. Adjust the test — it's about verifying the FeeLedger call pattern, not the user lookup.

- [ ] **Step 5.6: Full suite still green.**

```bash
bun x jest src/modules/trading-v2 --runInBand
```

Expected: 129 + 2 = 131 passed + 1 skipped.

- [ ] **Step 5.7: Commit.**

```bash
git add api/src/modules/trading-v2/services/solana-settler.service.ts \
        api/src/modules/trading-v2/services/reconcile-settling-trades.service.ts \
        api/src/modules/trading-v2/__tests__/fee-ledger-integration.integration.test.ts
git commit -m "feat(trading-v2): SolanaSettler + recovery gravam FeeEvents"
```

---

## Task 6: Routes wiring + barrel + README

**Files:**
- Modify: `api/src/modules/trading-v2/routes/orders.routes.ts`
- Modify: `api/src/modules/trading-v2/index.ts`
- Modify: `api/src/modules/trading-v2/README.md`

- [ ] **Step 6.1: Confirm `orders.routes.ts` still works.**

No structural change needed — the settler auto-creates a default `userWalletLookup` via Prisma. If the route's own `userLookup` has better logic (int-parse + sessionId fallback), inject that into the settler config for consistency:

```typescript
const settler = new SolanaSettlerService(prisma, tradeRepo, stub, reverter, caller, {
  eventRepo,
  userWalletLookup: userLookup.getWalletAddress.bind(userLookup),
});
```

Actually `userLookup` returns `Promise<string>` (throws on missing), not `Promise<string | null>`. Wrap:

```typescript
  userWalletLookup: async (userId: string) => {
    try { return await userLookup.getWalletAddress(userId); }
    catch { return null; }
  },
```

- [ ] **Step 6.2: Update barrel `index.ts`:**

```typescript
export { FeeComputation } from "./services/fee-computation.service";
export type { FeeComputationInput, FeeComputationResult } from "./services/fee-computation.service";
```

- [ ] **Step 6.3: Append to `README.md` (depois de "Settlement"):**

```markdown
## Fee ledger (Plano 6)

Taxas são:
- **Reservadas** pelo `IntentClassifier` quando usuário posta ordem USDC-reservante (notional + fee).
- **Consumidas** pelo `StubSettlerService.applyDeltas` atomic com o settlement; para o counterparty, o USDC recebido é `notional - feeContraparte`.
- **Registradas** no `FeeEvent` model (via `feeLedgerService.record`) apenas quando o trade transiciona para SETTLED — garantindo fee ledger reflete on-chain real.

Math centralizada em `FeeComputation` (pure, sem IO). Dado `{primitive, priceBps, qty, feeBps, buyer/sellerAsset}`, retorna `{buyerFee, sellerFee, buyerUsdcConsumed, sellerUsdcConsumed, buyerUsdcReceived, sellerUsdcReceived}`. Consumer único: settler e recovery path.

Emissão de `FeeEvent`:
- `SolanaSettlerService` no caminho feliz após SETTLED commit.
- `ReconcileSettlingTradesService.settleFromEvent` no caminho de recuperação via listener.

Revert não emite fee events inversos — como fee só registra pós-SETTLED e revert só ocorre pré-SETTLED, não há desbalanço.

### Fee destinations

- `trading_taker` → `platform_fee_wallet` — fees das duas pontas do trade.
```

- [ ] **Step 6.4: Rodar suíte inteira.**

```bash
cd /Users/rafael/Documents/GitHub/Oraculo/.claude/worktrees/orderbook-rewrite-06-fee-ledger/api
bun x jest src/modules/trading-v2 --runInBand
```

Expected: 131 passed + 1 skipped.

- [ ] **Step 6.5: tsc clean.**

```bash
bun x tsc --noEmit 2>&1 | grep trading-v2 || echo "clean"
```

- [ ] **Step 6.6: Commit.**

```bash
git add api/src/modules/trading-v2/routes/orders.routes.ts \
        api/src/modules/trading-v2/index.ts \
        api/src/modules/trading-v2/README.md
git commit -m "docs(trading-v2): README fee ledger + barrel + routes wire"
```

---

## Critérios de aceitação

1. ✅ `feeBps` persistido em `Ob2Order` e `Ob2Trade` com CHECK 0..10000.
2. ✅ `FeeComputation` pure cobre 4 casos (TRADE×2, MINT, MERGE) + edge cases (feeBps=0, out-of-range).
3. ✅ `StubSettler` consume fee portion da reservation USDC + reduz USDC creditado ao counterparty quando aplicável.
4. ✅ `SolanaSettler` emite `FeeEvent` no tx pós-SETTLED (caminho feliz).
5. ✅ `ReconcileSettlingTradesService.settleFromEvent` emite `FeeEvent` (recovery path).
6. ✅ Revert NÃO emite fee events — documentado.
7. ✅ Suíte ≥ 131 passed + 1 skipped; tsc clean.

---

## Riscos e mitigações

| Risco | Mitigação |
|---|---|
| Conversão `Number(bigint) / 1_000_000` perde precisão pra valores grandes | Fee amounts realistas são sub-cent (micros). Se preocupação, trocar por string conversion via `fromMicro`. |
| User lookup falha em ambiente de teste por User model ter campos required | `userWalletLookup` retorna null silenciosamente; FeeEvent é skipado. Teste usa config override com lookup hardcoded. |
| IntentClassifier calcula fee com `feeBps` mas o trade persistido pode ter `feeBps` diferente (se taker.feeBps != classifier.feeBps) | Aderência ao `PlaceOrderInput.feeBps` em ambos os pontos (Plano 1 classifier + Plano 6 persist). São a mesma variável por request. |
| FeeEvent duplicado se settler + listener ambos emitirem | Fee ledger é apenas audit (não tem unique constraint em tradeId). Adicionar `@@unique([tradeId, userWallet, eventType])` migration futura se duplicação virar problema. Por enquanto: rare race, aceitável. |
| Tests existentes que não passam `feeBps` quebram | Task 3.7 varre e adiciona `feeBps: 0` em todos os pontos de criação. |

---

## O que NÃO está neste plano

- **Fee cap / min**: Plan N — hoje feeBps é percentual puro.
- **Fee por market (via market.fee_bps do schema)**: hoje pass-through via `PlaceOrderInput.feeBps`. Centralizar por market é refactor futuro.
- **WebSocket emit do FeeEvent**: Plano 7 WS.
- **Dashboard de fee revenue**: consumer do FeeEvent ledger, fora de scope do rewrite.
