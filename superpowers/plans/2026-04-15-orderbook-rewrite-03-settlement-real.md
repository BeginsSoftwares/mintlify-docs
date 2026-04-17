# Orderbook Rewrite — Plano 3: Settlement real + revert + deadline worker

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Substituir `StubSettlerService` por um `SolanaSettlerService` real que chama on-chain via abstração `IOnchainCaller` (concrete caller do programa Solana fica pro Plano 4), adicionar lógica de revert e worker de deadline SLA, e corrigir os dois débitos deixados pelo review do Plano 2.

**Architecture:** Nova interface `IOnchainCaller` encapsula a chamada on-chain. Plano 3 entrega um `MockOnchainCaller` pra testes + `SolanaSettlerService` que implementa `ISettler` com retry/timeout. `SettlementReverter` faz o inverso do settler (desfaz consume/credit, reabre ou cancela ordens). `ReconcileSettlingTradesService` é o worker que scan trades em SETTLING além do deadline e dispara revert. Cleanup do P2: fechamos a janela de pending-reservation (reserva primeiro, depois cria a ordem OPEN) e extraímos os conversores Decimal↔bigint para util compartilhado.

**Tech Stack:** Bun, TypeScript, Prisma 7, Jest.

**Spec:** `docs/superpowers/specs/2026-04-15-orderbook-rewrite-design.md` (§7 settlement, §7.3 revert, §7.4 listener — listener fica pro Plano 4 porque depende do programa on-chain)
**Planos anteriores:**
- Plano 1 (merged `c891f0b`) — fundação
- Plano 2 (merged `c591534`) — orders + matching + stub settler

**Próximos planos (após este):**
- Plano 4: instrução `settle_fill` no programa Solana + `SolanaCaller` concreto + listener de eventos on-chain
- Plano 5: WebSocket v2 • Plano 6: MM bot externo • Plano 7: Cutover

**Notas do review do Plano 2 endereçadas aqui:**
1. **Pending-reservation window** em `PlaceOrderUseCase` — Task 1.
2. **DRY dos conversores Decimal↔bigint** — Task 2.
3. **Divergência stub NO(BUY)×USDC(SELL)** — documentada no README do Plano 3 e nos riscos do Plano 7 (cutover) quando o stub cair.

---

## File Structure

**Arquivos afetados:**

```
api/src/modules/trading-v2/
  types/
    balance.types.ts                     # MODIFY: exportar decimal-helpers
    decimal-helpers.ts                   # CREATE: toMicro/fromMicro shared
    onchain-caller.types.ts              # CREATE: IOnchainCaller + SettleFillParams + result shapes
    revert.types.ts                      # CREATE: RevertReason enum, RevertResult
  services/
    solana-settler.service.ts            # CREATE: real ISettler impl (chama IOnchainCaller)
    mock-onchain-caller.service.ts       # CREATE: test double (success/failure/timeout modes)
    settlement-reverter.service.ts       # CREATE: inverse of settler — restore reservations + reopen/cancel
    reconcile-settling-trades.service.ts # CREATE: worker scan trades expired → revert
    stub-settler.service.ts              # MODIFY: mark deprecated, keep for legacy usage if any
    matching-engine.service.ts           # MODIFY: accept any ISettler (already does); no structural change
    balance.repository.ts                # MODIFY: switch to shared decimal helpers
    reservation.service.ts               # MODIFY: switch to shared decimal helpers
  repositories/
    order.repository.ts                  # MODIFY: shared decimals + helper to reset filled on revert
    trade.repository.ts                  # MODIFY: shared decimals
  use-cases/
    place-order.use-case.ts              # MODIFY: reserve BEFORE creating OPEN order (close P2 window)
  routes/
    orders.routes.ts                     # MODIFY: swap StubSettler → SolanaSettler (wired with MockOnchainCaller in test env; real in prod — real caller comes in Plano 4)
  __tests__/
    decimal-helpers.unit.test.ts         # CREATE
    place-order-window.integration.test.ts # CREATE: proves no PENDING/OPEN-with-placeholder window
    solana-settler.integration.test.ts   # CREATE: uses MockOnchainCaller
    settlement-reverter.integration.test.ts # CREATE: revert of each primitive restores state
    reconcile-settling-trades.integration.test.ts # CREATE: deadline scan reverts expired
  index.ts                               # MODIFY: export new services + types
  README.md                              # MODIFY: pos-plano 3 state + documented divergence
```

**Responsabilidades (novos):**

- `types/decimal-helpers.ts` — uma função `toMicro(v: unknown): bigint` e uma `fromMicro(v: bigint): string`. Zero state. Usado por todos os repos e services.
- `types/onchain-caller.types.ts` — interface `IOnchainCaller` com um método `sendSettleFill(params)` que retorna `{ ok: true, signature } | { ok: false, reason }`. Abstrai o transport on-chain; Plano 4 implementa o caller concreto.
- `services/mock-onchain-caller.service.ts` — dublê configurável: modo `success` (retorna sig dummy), `failure` (retorna erro com razão dada), `timeout` (retorna depois de X ms OU nunca, simulando RPC lento). Usado em todos os testes de settler/revert/worker.
- `services/solana-settler.service.ts` — `SolanaSettlerService implements ISettler`. Fluxo: busca trade → chama `IOnchainCaller.sendSettleFill` → se ok, aplica deltas DB idênticos aos do stub + marca SETTLED com signature real; se erro, chama `SettlementReverter`.
- `services/settlement-reverter.service.ts` — inverso do settler. Dado um trade em SETTLING, restaura reservations (volta `reserved`) e ordens (subtrai `filled`, reabre ou marca CANCELLED conforme política), marca trade REVERTED.
- `services/reconcile-settling-trades.service.ts` — `scanAndRevert()`: busca trades em SETTLING com `settlingDeadline < now`, dispara `SettlementReverter.revert(tradeId, "deadline_exceeded")` pra cada. Idempotente.

---

## Prerequisite check

- [ ] **Step 0: Baseline**

```bash
cd /Users/rafael/Documents/GitHub/Oraculo/api
bun x jest src/modules/trading-v2 --runInBand
```
Expected: `Tests: 67 passed`. Se não, resolver antes.

- [ ] **Step 0.1: Worktree**

```bash
git worktree add .claude/worktrees/orderbook-rewrite-03-settlement-real -b worktree-orderbook-rewrite-03-settlement-real
cp api/.env .claude/worktrees/orderbook-rewrite-03-settlement-real/api/.env
```

- [ ] **Step 0.2: Deps**

```bash
cd .claude/worktrees/orderbook-rewrite-03-settlement-real/api
bun install
bun x prisma generate
bun x jest src/modules/trading-v2 --runInBand    # confirma 67 baseline no worktree
```

---

## Task 1: Fechar janela de pending-reservation no PlaceOrderUseCase

**Problema do P2 review:** `PlaceOrderUseCase` cria a ordem com `status="OPEN"` e `reservationId="pending"` ANTES da reserve concluir. Isso expõe a ordem ao book; um taker concorrente pode pegar ela e abortar tentando ler a reserva "pending" (não existe). Sem corrupção mas com failure mode transiente sob carga.

**Solução:** inverter ordem — reservar primeiro, depois criar ordem com `reservationId` real já no `OPEN`. Requer ajuste no `ReservationService.reserve` para aceitar um `orderId` que pode ser gerado antecipadamente pelo caller.

**Files:**
- Modify: `api/src/modules/trading-v2/use-cases/place-order.use-case.ts`
- Create: `api/src/modules/trading-v2/__tests__/place-order-window.integration.test.ts`

- [ ] **Step 1.1: Escrever teste que prova que a janela está fechada:**

```typescript
import "dotenv/config";
import { prisma } from "../../../shared/database/config/prisma-client";
import { BalanceRepository } from "../repositories/balance.repository";
import { ReservationService } from "../services/reservation.service";
import { OrderRepository } from "../repositories/order.repository";
import { TradeRepository } from "../repositories/trade.repository";
import { StubSettlerService } from "../services/stub-settler.service";
import { MatchingEngine } from "../services/matching-engine.service";
import { IntentClassifier } from "../services/intent-classifier.service";
import { PlaceOrderUseCase } from "../use-cases/place-order.use-case";
import { UNIT } from "../types/balance.types";

const balanceRepo = new BalanceRepository(prisma);
const resSvc = new ReservationService(prisma);
const orderRepo = new OrderRepository(prisma);
const tradeRepo = new TradeRepository(prisma);
const settler = new StubSettlerService(prisma);
const engine = new MatchingEngine(prisma, orderRepo, tradeRepo, settler);
const classifier = new IntentClassifier();
const place = new PlaceOrderUseCase(prisma, balanceRepo, resSvc, orderRepo, engine, classifier);

const A = "00000000-0000-0000-0000-000000000001";
const MARKET = "Market1111111111111111111111111111111111111";

beforeEach(async () => {
  await prisma.ob2Trade.deleteMany({});
  await prisma.ob2Order.deleteMany({});
  await prisma.ob2Reservation.deleteMany({});
  await prisma.ob2UserMarketBalance.deleteMany({});
});
afterAll(async () => { await prisma.$disconnect(); });

test("successful place never produces an OPEN order with reservationId='pending'", async () => {
  await balanceRepo.upsertOnchain(A, MARKET, "USDC", 1000n * UNIT, 1n);
  const { order } = await place.execute({
    userId: A, marketPda: MARKET, side: "BUY",
    priceBps: 6000, quantity: 100n * UNIT, feeBps: 50,
  });
  expect(order.status).toBe("OPEN");
  expect(order.reservationId).not.toBe("pending");
  expect(order.reservationId).not.toBeNull();
  // The reservation exists
  const res = await prisma.ob2Reservation.findUnique({ where: { id: order.reservationId! } });
  expect(res).not.toBeNull();
  expect(res!.orderId).toBe(order.id);
});

test("no order row with reservationId='pending' is ever persisted", async () => {
  await balanceRepo.upsertOnchain(A, MARKET, "USDC", 1000n * UNIT, 1n);
  await place.execute({
    userId: A, marketPda: MARKET, side: "BUY",
    priceBps: 6000, quantity: 100n * UNIT, feeBps: 50,
  });
  const pendingRows = await prisma.ob2Order.findMany({ where: { reservationId: "pending" } });
  expect(pendingRows).toHaveLength(0);
});

test("on insufficient balance: no order row is persisted at all (not even REJECTED)", async () => {
  await balanceRepo.upsertOnchain(A, MARKET, "USDC", 5n * UNIT, 1n);
  await expect(place.execute({
    userId: A, marketPda: MARKET, side: "BUY",
    priceBps: 6000, quantity: 100n * UNIT, feeBps: 50,
  })).rejects.toThrow();

  const orders = await prisma.ob2Order.findMany({ where: { userId: A } });
  expect(orders).toHaveLength(0);
});
```

- [ ] **Step 1.2: Run, must fail on test 1 (current code sets reservationId="pending") and test 3 (current code creates REJECTED row).**

```bash
cd /Users/rafael/Documents/GitHub/Oraculo/.claude/worktrees/orderbook-rewrite-03-settlement-real/api
bun x jest src/modules/trading-v2/__tests__/place-order-window.integration.test.ts --runInBand
```

- [ ] **Step 1.3: Refactor `place-order.use-case.ts` — reserve-first using a pre-generated orderId.**

Dependencies: `ReservationService.reserve` already takes `orderId` as input. We'll generate the UUID here, reserve with it, then `orders.create` with both `id` (new param in `CreateOrderRow`) and `reservationId`.

First, update `OrderRepository.create` to accept an optional pre-generated id. Change `api/src/modules/trading-v2/repositories/order.repository.ts`:

```typescript
export interface CreateOrderRow {
  id?: string;                 // NEW: optional — caller may pre-generate to tie to reservation
  userId: string;
  marketPda: string;
  side: Ob2Side;
  priceBps: number;
  quantity: bigint;
  reservationId: string;
  clientOrderId?: string;
}
```

And inside `create`:

```typescript
  async create(input: CreateOrderRow): Promise<OrderView> {
    const row = await this.prisma.ob2Order.create({
      data: {
        ...(input.id ? { id: input.id } : {}),
        userId: input.userId,
        marketPda: input.marketPda,
        side: input.side,
        priceBps: input.priceBps,
        quantity: fromBig(input.quantity),
        filled: "0",
        status: "OPEN",
        reservationId: input.reservationId,
        clientOrderId: input.clientOrderId ?? null,
      },
    });
    return toView(row);
  }
```

Then rewrite `place-order.use-case.ts`'s `execute`:

```typescript
  async execute(input: PlaceOrderInput): Promise<PlaceOrderResult> {
    if (input.priceBps <= 0 || input.priceBps >= 10000) {
      throw new OrderRejectedError("invalid_price", `priceBps must be in (0, 10000), got ${input.priceBps}`);
    }
    if (input.quantity <= 0n) {
      throw new OrderRejectedError("invalid_quantity", `quantity must be positive`);
    }

    if (input.clientOrderId) {
      const existing = await this.prisma.ob2Order.findUnique({
        where: { userId_clientOrderId: { userId: input.userId, clientOrderId: input.clientOrderId } },
      });
      if (existing) {
        throw new OrderRejectedError("duplicate_client_order_id", `clientOrderId ${input.clientOrderId} already used`);
      }
    }

    const [usdc, yes, no] = await Promise.all([
      this.balanceRepo.get(input.userId, input.marketPda, "USDC"),
      this.balanceRepo.get(input.userId, input.marketPda, "YES"),
      this.balanceRepo.get(input.userId, input.marketPda, "NO"),
    ]);
    const { asset, amount } = this.classifier.classify({
      side: input.side,
      quantity: input.quantity,
      priceBps: input.priceBps,
      feeBps: input.feeBps,
      freeYes: yes?.free ?? 0n,
      freeNo:  no?.free  ?? 0n,
      freeUsdc: usdc?.free ?? 0n,
    });

    // Pre-generate the orderId so reservation.orderId can point at it
    // BEFORE the order row exists. Reserve first — if it fails, no order is created.
    const orderId = (await import("crypto")).randomUUID();

    const reservation = await this.reservations.reserve({
      userId: input.userId, marketPda: input.marketPda, asset, amount, orderId,
    });

    // Reservation succeeded; create the OPEN order with real reservationId.
    const order = await this.orders.create({
      id: orderId,
      userId: input.userId, marketPda: input.marketPda, side: input.side,
      priceBps: input.priceBps, quantity: input.quantity,
      reservationId: reservation.id, clientOrderId: input.clientOrderId,
    });

    const matchResult = await this.engine.tryMatch(order.id);
    const finalOrder = await this.orders.getById(order.id);
    return { order: finalOrder!, trades: matchResult.trades };
  }
```

Remove the `try/catch` that marked the order REJECTED — no order to reject anymore.

Note on the dynamic import: top-level `import { randomUUID } from "crypto"` works too; either is fine. Static import is cleaner — add to the top of the file and remove the dynamic import inside `execute`:

```typescript
import { randomUUID } from "crypto";
// ... and in execute: const orderId = randomUUID();
```

- [ ] **Step 1.4: Run — 3 new tests pass + existing place-order tests still pass.**

```bash
bun x jest src/modules/trading-v2/__tests__/place-order-window.integration.test.ts src/modules/trading-v2/__tests__/place-order.use-case.integration.test.ts --runInBand
```

Expected: 3 new + 4 existing = 7 passed. **If `place-order.use-case.integration.test.ts` has a test expecting REJECTED row on insufficient balance, update the expectation to "no order row created".** Adjust only the assertions needed, not unrelated tests.

- [ ] **Step 1.5: Run full trading-v2 suite to confirm nothing else broke.**

```bash
bun x jest src/modules/trading-v2 --runInBand
```

Expected: `Tests: 69 passed` (67 baseline − 1 dropped assertion + 3 new). If a different number, investigate.

- [ ] **Step 1.6: Commit.**

```bash
cd /Users/rafael/Documents/GitHub/Oraculo/.claude/worktrees/orderbook-rewrite-03-settlement-real
git add api/src/modules/trading-v2/use-cases/place-order.use-case.ts \
        api/src/modules/trading-v2/repositories/order.repository.ts \
        api/src/modules/trading-v2/__tests__/place-order-window.integration.test.ts \
        api/src/modules/trading-v2/__tests__/place-order.use-case.integration.test.ts
git commit -m "fix(trading-v2): fecha janela de pending-reservation (reserve antes de criar ordem)"
```

---

## Task 2: DRY dos conversores Decimal↔bigint

**Files:**
- Create: `api/src/modules/trading-v2/types/decimal-helpers.ts`
- Create: `api/src/modules/trading-v2/__tests__/decimal-helpers.unit.test.ts`
- Modify: `api/src/modules/trading-v2/repositories/balance.repository.ts`
- Modify: `api/src/modules/trading-v2/repositories/order.repository.ts`
- Modify: `api/src/modules/trading-v2/repositories/trade.repository.ts`
- Modify: `api/src/modules/trading-v2/services/reservation.service.ts`
- Modify: `api/src/modules/trading-v2/services/balance.service.ts`
- Modify: `api/src/modules/trading-v2/services/stub-settler.service.ts`
- Modify: `api/src/modules/trading-v2/services/matching-engine.service.ts`

- [ ] **Step 2.1: Criar `types/decimal-helpers.ts`:**

```typescript
/**
 * Conversores únicos Decimal(20,6) ↔ bigint (micro-units, 6 decimais fixos).
 *
 * Todo o módulo trading-v2 usa essas duas funções. Não reescreva inline — se
 * achar um novo uso, import daqui. O teste em __tests__/decimal-helpers.unit.test.ts
 * trava o comportamento.
 */

/** Prisma Decimal (string ou Decimal object) → bigint micro-units. */
export function toMicro(v: unknown): bigint {
  const s = String(v);
  const [intPart, fracPart = ""] = s.split(".");
  const frac = fracPart.padEnd(6, "0").slice(0, 6);
  return BigInt(intPart + frac);
}

/** bigint micro-units → string "XXXX.XXXXXX" para enviar ao Decimal do Prisma. */
export function fromMicro(v: bigint): string {
  const neg = v < 0n;
  const abs = neg ? -v : v;
  const s = abs.toString().padStart(7, "0");
  const out = `${s.slice(0, -6)}.${s.slice(-6)}`;
  return neg ? `-${out}` : out;
}
```

- [ ] **Step 2.2: Testes unitários:**

```typescript
import { toMicro, fromMicro } from "../types/decimal-helpers";

test("toMicro accepts integer string", () => {
  expect(toMicro("100")).toBe(100_000_000n);
});

test("toMicro accepts decimal string with < 6 digits (right-pads)", () => {
  expect(toMicro("100.5")).toBe(100_500_000n);
  expect(toMicro("100.50")).toBe(100_500_000n);
});

test("toMicro truncates more than 6 fractional digits", () => {
  expect(toMicro("100.1234567")).toBe(100_123_456n);  // last 7 dropped
});

test("toMicro handles zero and very small numbers", () => {
  expect(toMicro("0")).toBe(0n);
  expect(toMicro("0.000001")).toBe(1n);
  expect(toMicro("0.0000001")).toBe(0n);      // rounds down
});

test("fromMicro produces 6-fractional-digit string", () => {
  expect(fromMicro(100_000_000n)).toBe("100.000000");
  expect(fromMicro(1n)).toBe("0.000001");
  expect(fromMicro(0n)).toBe("0.000000");
});

test("fromMicro and toMicro roundtrip", () => {
  for (const v of [0n, 1n, 999_999n, 1_000_000n, 123_456_789_012n]) {
    expect(toMicro(fromMicro(v))).toBe(v);
  }
});

test("fromMicro handles negative bigint", () => {
  expect(fromMicro(-100_000_000n)).toBe("-100.000000");
  expect(toMicro(fromMicro(-1n))).toBe(-1n);
});
```

- [ ] **Step 2.3: Run, fail.**

```bash
bun x jest src/modules/trading-v2/__tests__/decimal-helpers.unit.test.ts --runInBand
```

- [ ] **Step 2.4: Run, passes** (after Step 2.1).

- [ ] **Step 2.5: Refactor each file to use the shared helpers.**

For each of the 7 modified files, replace the inline `toBig`/`fromBig` / `toDecimal`/`fromDecimal` / `fromBig`/`toBig` local helpers with an import from `../types/decimal-helpers`.

Per file:

**`balance.repository.ts`**: delete local `toBig` and `fromBig`, import at top:
```typescript
import { toMicro as toBig, fromMicro as fromBig } from "../types/decimal-helpers";
```
(Aliasing keeps the rest of the file unchanged — simpler diff.)

**`order.repository.ts`**: same pattern — delete local helpers, import with alias.

**`trade.repository.ts`**: same.

**`reservation.service.ts`**: the file has `toDecimal`/`fromDecimal` as private methods. Delete them and use:
```typescript
import { toMicro, fromMicro } from "../types/decimal-helpers";
```
Then replace `this.toDecimal(x)` → `fromMicro(x)` and `this.fromDecimal(x)` → `toMicro(x)` in the class body.

**`balance.service.ts`**: same pattern — has private `fromDecimal`, replace with `toMicro` import.

**`stub-settler.service.ts`**: has private `toDecimal`/`fromDecimal`, replace with `fromMicro`/`toMicro`.

**`matching-engine.service.ts`**: has private `toDecimal`/`fromDecimal`, replace with `fromMicro`/`toMicro`.

- [ ] **Step 2.6: Full suite must stay green.**

```bash
bun x jest src/modules/trading-v2 --runInBand
```

Expected: `Tests: 76 passed` (69 from Task 1 + 7 new decimal tests).

- [ ] **Step 2.7: tsc clean.**

```bash
bun x tsc --noEmit 2>&1 | grep "src/modules/trading-v2" || echo "clean"
```

- [ ] **Step 2.8: Commit.**

```bash
cd /Users/rafael/Documents/GitHub/Oraculo/.claude/worktrees/orderbook-rewrite-03-settlement-real
git add api/src/modules/trading-v2/types/decimal-helpers.ts \
        api/src/modules/trading-v2/__tests__/decimal-helpers.unit.test.ts \
        api/src/modules/trading-v2/repositories/ \
        api/src/modules/trading-v2/services/balance.service.ts \
        api/src/modules/trading-v2/services/reservation.service.ts \
        api/src/modules/trading-v2/services/stub-settler.service.ts \
        api/src/modules/trading-v2/services/matching-engine.service.ts
git commit -m "refactor(trading-v2): helper único de conversão Decimal↔bigint"
```

---

## Task 3: Types — IOnchainCaller + RevertReason

**Files:**
- Create: `api/src/modules/trading-v2/types/onchain-caller.types.ts`
- Create: `api/src/modules/trading-v2/types/revert.types.ts`
- Modify: `api/src/modules/trading-v2/index.ts` (append exports)

- [ ] **Step 3.1: `types/onchain-caller.types.ts`:**

```typescript
import type { Ob2Primitive, Ob2Asset } from "../../../generated/prisma/client";

/**
 * Parâmetros pra chamada on-chain de settle_fill.
 * A maior parte vem do trade. `buyerReservationAsset`/`sellerReservationAsset`
 * são derivados das reservations (determinam a primitiva do ponto de vista
 * do programa — o caller pode passar como hint).
 */
export interface SettleFillParams {
  tradeId: string;
  marketPda: string;
  buyerUserId: string;
  sellerUserId: string;
  buyerReservationAsset: Ob2Asset;
  sellerReservationAsset: Ob2Asset;
  priceBps: number;
  quantityMicro: bigint;
  primitive: Ob2Primitive;
}

export type SettleFillResult =
  | { ok: true; signature: string }
  | { ok: false; reason: string; retryable: boolean };

/**
 * Interface que abstrai o transport on-chain.
 *
 * Plano 3 entrega só MockOnchainCaller. Plano 4 entrega SolanaOnchainCaller
 * real (RPC, Anchor IDL, etc.).
 *
 * Contrato:
 * - `sendSettleFill` retorna um result discriminado; NUNCA throws exceto em
 *   erro de programação.
 * - `retryable=true` indica falha transiente (timeout, RPC down) — settler
 *   pode repetir com backoff.
 * - `retryable=false` indica falha definitiva (program error, insufficient
 *   vault balance on-chain) — settler deve revert.
 */
export interface IOnchainCaller {
  sendSettleFill(params: SettleFillParams): Promise<SettleFillResult>;
}
```

- [ ] **Step 3.2: `types/revert.types.ts`:**

```typescript
/** Motivos padronizados — usado no campo trade.revertReason e no log. */
export const RevertReason = {
  DeadlineExceeded:       "deadline_exceeded",
  OnchainFailed:          "onchain_failed",
  OnchainTimeout:         "onchain_timeout",
  OnchainProgramError:    "onchain_program_error",
  Manual:                 "manual",
} as const;

export type RevertReason = typeof RevertReason[keyof typeof RevertReason];

export interface RevertResult {
  tradeId: string;
  reason: RevertReason;
  reopened: { makerOrderId: boolean; takerOrderId: boolean };
}
```

- [ ] **Step 3.3: Barrel `index.ts` — append:**

```typescript
export * from "./types/onchain-caller.types";
export * from "./types/revert.types";
export * from "./types/decimal-helpers";
```

- [ ] **Step 3.4: tsc clean.**

```bash
bun x tsc --noEmit 2>&1 | grep "src/modules/trading-v2" || echo "clean"
```

- [ ] **Step 3.5: Commit.**

```bash
git add api/src/modules/trading-v2/types/ api/src/modules/trading-v2/index.ts
git commit -m "feat(trading-v2): types IOnchainCaller + RevertReason"
```

---

## Task 4: MockOnchainCaller

Test double configurável. Três modos: success, failure (transient/permanent), timeout.

**Files:**
- Create: `api/src/modules/trading-v2/services/mock-onchain-caller.service.ts`
- Create: `api/src/modules/trading-v2/__tests__/mock-onchain-caller.unit.test.ts`

- [ ] **Step 4.1: Testes:**

```typescript
import { MockOnchainCaller } from "../services/mock-onchain-caller.service";
import { UNIT } from "../types/balance.types";

const baseParams = {
  tradeId: "trade-1",
  marketPda: "Market1111111111111111111111111111111111111",
  buyerUserId: "u1", sellerUserId: "u2",
  buyerReservationAsset: "USDC" as const,
  sellerReservationAsset: "YES" as const,
  priceBps: 6000, quantityMicro: 100n * UNIT,
  primitive: "TRADE" as const,
};

test("success mode returns ok with generated signature", async () => {
  const c = new MockOnchainCaller({ mode: "success" });
  const r = await c.sendSettleFill(baseParams);
  expect(r.ok).toBe(true);
  if (r.ok) expect(r.signature).toMatch(/^mock:/);
});

test("failure mode returns ok=false with given reason and retryable flag", async () => {
  const c = new MockOnchainCaller({ mode: "failure", reason: "rpc_down", retryable: true });
  const r = await c.sendSettleFill(baseParams);
  expect(r.ok).toBe(false);
  if (!r.ok) {
    expect(r.reason).toBe("rpc_down");
    expect(r.retryable).toBe(true);
  }
});

test("timeout mode rejects after configured ms via promise race", async () => {
  const c = new MockOnchainCaller({ mode: "timeout", timeoutMs: 20 });
  const r = await c.sendSettleFill(baseParams);
  expect(r.ok).toBe(false);
  if (!r.ok) {
    expect(r.reason).toMatch(/timeout/);
    expect(r.retryable).toBe(true);
  }
});

test("sequential success-then-failure via script sequence", async () => {
  const c = new MockOnchainCaller({
    mode: "sequence",
    sequence: [
      { ok: true, signature: "mock:1" },
      { ok: false, reason: "program_error", retryable: false },
    ],
  });
  const r1 = await c.sendSettleFill(baseParams);
  const r2 = await c.sendSettleFill(baseParams);
  expect(r1.ok).toBe(true);
  expect(r2.ok).toBe(false);
});

test("history captures all calls", async () => {
  const c = new MockOnchainCaller({ mode: "success" });
  await c.sendSettleFill(baseParams);
  await c.sendSettleFill({ ...baseParams, tradeId: "trade-2" });
  expect(c.history).toHaveLength(2);
  expect(c.history[0].tradeId).toBe("trade-1");
  expect(c.history[1].tradeId).toBe("trade-2");
});
```

- [ ] **Step 4.2: Run, fail.**

- [ ] **Step 4.3: Implement `services/mock-onchain-caller.service.ts`:**

```typescript
import type { IOnchainCaller, SettleFillParams, SettleFillResult } from "../types/onchain-caller.types";

export type MockConfig =
  | { mode: "success" }
  | { mode: "failure"; reason: string; retryable: boolean }
  | { mode: "timeout"; timeoutMs: number }
  | { mode: "sequence"; sequence: SettleFillResult[] };

/**
 * Test double for IOnchainCaller. See MockConfig for modes.
 *
 * - `success`: returns ok=true with a synthetic signature.
 * - `failure`: returns ok=false with configured reason and retryable flag.
 * - `timeout`: waits `timeoutMs`, returns a retryable timeout failure.
 * - `sequence`: pops results from the array in order; throws if exhausted.
 *
 * Records every call into `history` for inspection.
 */
export class MockOnchainCaller implements IOnchainCaller {
  public readonly history: SettleFillParams[] = [];
  private seqIdx = 0;

  constructor(private readonly config: MockConfig) {}

  async sendSettleFill(params: SettleFillParams): Promise<SettleFillResult> {
    this.history.push(params);
    switch (this.config.mode) {
      case "success":
        return { ok: true, signature: `mock:${params.tradeId.slice(0, 8)}` };
      case "failure":
        return { ok: false, reason: this.config.reason, retryable: this.config.retryable };
      case "timeout":
        await new Promise(r => setTimeout(r, this.config.timeoutMs));
        return { ok: false, reason: "timeout", retryable: true };
      case "sequence": {
        if (this.seqIdx >= this.config.sequence.length) {
          throw new Error("MockOnchainCaller sequence exhausted");
        }
        return this.config.sequence[this.seqIdx++];
      }
    }
  }
}
```

- [ ] **Step 4.4: Run, 5 pass.**

- [ ] **Step 4.5: Commit.**

```bash
git add api/src/modules/trading-v2/services/mock-onchain-caller.service.ts \
        api/src/modules/trading-v2/__tests__/mock-onchain-caller.unit.test.ts
git commit -m "feat(trading-v2): MockOnchainCaller (test double configurável)"
```

---

## Task 5: SettlementReverter

Inverso do settler. Dado um trade em SETTLING: restaura saldos (devolve o que foi consumido, cobra o que foi creditado), restaura a reservation (recria ou extende), subtrai o `filled` das ordens, e:
- Se a ordem maker estava OPEN → continua OPEN.
- Se havia sido marcada FILLED por este trade → reabre como OPEN (filled decrementa) OU cancela, por política default = **cancel** (spec §7.3).
- Marca trade REVERTED com razão.

**Files:**
- Create: `api/src/modules/trading-v2/services/settlement-reverter.service.ts`
- Create: `api/src/modules/trading-v2/__tests__/settlement-reverter.integration.test.ts`

- [ ] **Step 5.1: Testes:**

```typescript
import "dotenv/config";
import { prisma } from "../../../shared/database/config/prisma-client";
import { BalanceRepository } from "../repositories/balance.repository";
import { ReservationService } from "../services/reservation.service";
import { OrderRepository } from "../repositories/order.repository";
import { TradeRepository } from "../repositories/trade.repository";
import { StubSettlerService } from "../services/stub-settler.service";
import { SettlementReverter } from "../services/settlement-reverter.service";
import { RevertReason } from "../types/revert.types";
import { UNIT } from "../types/balance.types";

const balanceRepo = new BalanceRepository(prisma);
const resSvc = new ReservationService(prisma);
const orderRepo = new OrderRepository(prisma);
const tradeRepo = new TradeRepository(prisma);
const settler = new StubSettlerService(prisma);
const reverter = new SettlementReverter(prisma, tradeRepo, orderRepo);

const BUYER  = "00000000-0000-0000-0000-000000000001";
const SELLER = "00000000-0000-0000-0000-000000000002";
const MARKET = "Market1111111111111111111111111111111111111";

async function setupTradeReadyToRevert(primitive: "TRADE" | "MINT" | "MERGE") {
  await prisma.ob2Trade.deleteMany({});
  await prisma.ob2Order.deleteMany({});
  await prisma.ob2Reservation.deleteMany({});
  await prisma.ob2UserMarketBalance.deleteMany({});

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

  const buyerResAsset  = primitive === "MERGE" ? "NO"   : (primitive === "MINT" ? "USDC" : "USDC");
  const buyerResAmt    = primitive === "MERGE" ? 100n * UNIT : (primitive === "MINT" ? 60n * UNIT : 60n * UNIT);
  const sellerResAsset = primitive === "MERGE" ? "YES"  : (primitive === "MINT" ? "USDC" : "YES");
  const sellerResAmt   = primitive === "MERGE" ? 100n * UNIT : (primitive === "MINT" ? 40n * UNIT : 100n * UNIT);

  const buyerOrder = await orderRepo.create({
    userId: BUYER, marketPda: MARKET, side: "BUY", priceBps: 6000, quantity: 100n * UNIT, reservationId: "p",
  });
  const buyerRes = await resSvc.reserve({
    userId: BUYER, marketPda: MARKET, asset: buyerResAsset, amount: buyerResAmt, orderId: buyerOrder.id,
  });
  await prisma.ob2Order.update({ where: { id: buyerOrder.id }, data: { reservationId: buyerRes.id } });

  const sellerOrder = await orderRepo.create({
    userId: SELLER, marketPda: MARKET, side: "SELL", priceBps: 6000, quantity: 100n * UNIT, reservationId: "p",
  });
  const sellerRes = await resSvc.reserve({
    userId: SELLER, marketPda: MARKET, asset: sellerResAsset, amount: sellerResAmt, orderId: sellerOrder.id,
  });
  await prisma.ob2Order.update({ where: { id: sellerOrder.id }, data: { reservationId: sellerRes.id } });

  // Create trade and settle via stub — this establishes "settled" state that we'll then revert
  const trade = await tradeRepo.create({
    marketPda: MARKET,
    makerOrderId: sellerOrder.id, takerOrderId: buyerOrder.id,
    priceBps: 6000, quantity: 100n * UNIT,
    primitive, sync: true,
  });
  // Apply fills to both orders (simulate what matching engine did)
  await prisma.ob2Order.update({ where: { id: buyerOrder.id  }, data: { filled: "100", status: "FILLED", closedAt: new Date() } });
  await prisma.ob2Order.update({ where: { id: sellerOrder.id }, data: { filled: "100", status: "FILLED", closedAt: new Date() } });
  await settler.settle(trade.id);
  // Put trade back into SETTLING to simulate "about to be reverted"
  await prisma.ob2Trade.update({ where: { id: trade.id }, data: { status: "SETTLING", settledAt: null, txSignature: null } });
  return { tradeId: trade.id, buyerOrderId: buyerOrder.id, sellerOrderId: sellerOrder.id };
}

afterAll(async () => { await prisma.$disconnect(); });

test("revert TRADE restores balances and cancels FILLED orders by default", async () => {
  const { tradeId, buyerOrderId, sellerOrderId } = await setupTradeReadyToRevert("TRADE");

  await reverter.revert(tradeId, RevertReason.OnchainFailed);

  const t = await tradeRepo.getById(tradeId);
  expect(t!.status).toBe("REVERTED");
  expect(t!.revertReason).toBe(RevertReason.OnchainFailed);

  // Balances restored: buyer's USDC back to 1000 (minus any still-reserved amount, but there is none here after revert), seller's YES back
  const buyerUsdc = await balanceRepo.get(BUYER, MARKET, "USDC");
  const sellerYes = await balanceRepo.get(SELLER, MARKET, "YES");
  expect(buyerUsdc!.free + buyerUsdc!.reserved).toBe(1000n * UNIT);
  expect(sellerYes!.free + sellerYes!.reserved).toBe(100n * UNIT);
  expect((await balanceRepo.get(BUYER, MARKET, "YES"))?.free ?? 0n).toBe(0n);
  expect((await balanceRepo.get(SELLER, MARKET, "USDC"))?.free ?? 0n).toBe(0n);

  // Orders moved to CANCELLED (default policy)
  const b = await orderRepo.getById(buyerOrderId);
  const s = await orderRepo.getById(sellerOrderId);
  expect(b!.status).toBe("CANCELLED");
  expect(s!.status).toBe("CANCELLED");
});

test("revert MINT reverses both USDC consumption and both token mints", async () => {
  const { tradeId } = await setupTradeReadyToRevert("MINT");
  await reverter.revert(tradeId, RevertReason.OnchainTimeout);

  expect((await balanceRepo.get(BUYER,  MARKET, "USDC"))!.free + (await balanceRepo.get(BUYER,  MARKET, "USDC"))!.reserved).toBe(1000n * UNIT);
  expect((await balanceRepo.get(SELLER, MARKET, "USDC"))!.free + (await balanceRepo.get(SELLER, MARKET, "USDC"))!.reserved).toBe(1000n * UNIT);
  expect((await balanceRepo.get(BUYER,  MARKET, "YES"))?.free ?? 0n).toBe(0n);
  expect((await balanceRepo.get(SELLER, MARKET, "NO"))?.free ?? 0n).toBe(0n);
});

test("revert MERGE restores NO to buyer and YES to seller, removes USDC credit", async () => {
  const { tradeId } = await setupTradeReadyToRevert("MERGE");
  await reverter.revert(tradeId, RevertReason.DeadlineExceeded);

  expect((await balanceRepo.get(BUYER,  MARKET, "NO"))!.free  + (await balanceRepo.get(BUYER,  MARKET, "NO"))!.reserved).toBe(100n * UNIT);
  expect((await balanceRepo.get(SELLER, MARKET, "YES"))!.free + (await balanceRepo.get(SELLER, MARKET, "YES"))!.reserved).toBe(100n * UNIT);
  expect((await balanceRepo.get(BUYER,  MARKET, "USDC"))?.free ?? 0n).toBe(0n);
  expect((await balanceRepo.get(SELLER, MARKET, "USDC"))?.free ?? 0n).toBe(0n);
});

test("revert is idempotent: second call is no-op", async () => {
  const { tradeId } = await setupTradeReadyToRevert("TRADE");
  await reverter.revert(tradeId, RevertReason.OnchainFailed);
  await reverter.revert(tradeId, RevertReason.OnchainFailed);   // must not double-restore
  const buyerUsdc = await balanceRepo.get(BUYER, MARKET, "USDC");
  expect(buyerUsdc!.free + buyerUsdc!.reserved).toBe(1000n * UNIT);
});

test("revert of SETTLED trade refuses (not a SETTLING trade)", async () => {
  const { tradeId } = await setupTradeReadyToRevert("TRADE");
  await prisma.ob2Trade.update({ where: { id: tradeId }, data: { status: "SETTLED" } });
  // No-op — idempotente (mesmo comportamento de revert idempotente).
  await reverter.revert(tradeId, RevertReason.Manual);
  const t = await tradeRepo.getById(tradeId);
  expect(t!.status).toBe("SETTLED");  // unchanged
});
```

- [ ] **Step 5.2: Run, fail.**

- [ ] **Step 5.3: Implement `services/settlement-reverter.service.ts`:**

```typescript
import type { PrismaClient, Ob2Asset } from "../../../generated/prisma/client";
import type { OrderRepository } from "../repositories/order.repository";
import type { TradeRepository } from "../repositories/trade.repository";
import { toMicro, fromMicro } from "../types/decimal-helpers";
import { RevertReason, type RevertResult } from "../types/revert.types";

/**
 * Desfaz as mutações aplicadas por um settler (stub OU real) pra um trade em SETTLING.
 *
 * Política: ordens que haviam sido marcadas FILLED por este trade são CANCELLED
 * por default (não reabertas como OPEN) — conforme spec §7.3.
 *
 * Idempotente: chamadas após a primeira (status != SETTLING) são no-op.
 */
export class SettlementReverter {
  constructor(
    private readonly prisma: PrismaClient,
    private readonly trades: TradeRepository,
    private readonly orders: OrderRepository,
  ) {}

  async revert(tradeId: string, reason: RevertReason): Promise<RevertResult | null> {
    return this.prisma.$transaction(async (tx) => {
      const trade = await tx.ob2Trade.findUnique({ where: { id: tradeId } });
      if (!trade) throw new Error(`trade ${tradeId} not found`);
      if (trade.status !== "SETTLING") return null;  // idempotente

      const makerOrder = await tx.ob2Order.findUnique({ where: { id: trade.makerOrderId } });
      const takerOrder = await tx.ob2Order.findUnique({ where: { id: trade.takerOrderId } });
      if (!makerOrder || !takerOrder) throw new Error("orders not found");

      const buyerOrder  = takerOrder.side === "BUY"  ? takerOrder : makerOrder;
      const sellerOrder = takerOrder.side === "SELL" ? takerOrder : makerOrder;

      // Read reservations to know which assets were consumed per side
      const makerRes = await tx.ob2Reservation.findFirst({ where: { orderId: makerOrder.id } });
      const takerRes = await tx.ob2Reservation.findFirst({ where: { orderId: takerOrder.id } });
      if (!makerRes || !takerRes) throw new Error("reservations not found");
      const buyerRes  = buyerOrder.id  === takerOrder.id ? takerRes : makerRes;
      const sellerRes = sellerOrder.id === takerOrder.id ? takerRes : makerRes;

      const qty = toMicro(trade.quantity);
      const buyerUsdcLeg  = (qty * BigInt(trade.priceBps)) / 10000n;
      const sellerUsdcLeg = (qty * BigInt(10000 - trade.priceBps)) / 10000n;

      // Consumed amount per side (same values the settler subtracted from `reserved`)
      const buyerConsumed  = buyerRes.asset  === "USDC" ? buyerUsdcLeg  : qty;
      const sellerConsumed = sellerRes.asset === "USDC" ? sellerUsdcLeg : qty;

      // Undo the settler's work: for each primitive, credit back what was consumed
      // from `reserved` AND debit back what was added to `free` on the receiving side.
      switch (trade.primitive) {
        case "TRADE":
          if (buyerRes.asset === "USDC" && sellerRes.asset === "YES") {
            await this.creditReserved(tx, buyerOrder.userId,  trade.marketPda, "USDC", buyerUsdcLeg);
            await this.creditReserved(tx, sellerOrder.userId, trade.marketPda, "YES",  qty);
            await this.debitFree     (tx, buyerOrder.userId,  trade.marketPda, "YES",  qty);
            await this.debitFree     (tx, sellerOrder.userId, trade.marketPda, "USDC", buyerUsdcLeg);
          } else if (buyerRes.asset === "NO" && sellerRes.asset === "USDC") {
            await this.creditReserved(tx, buyerOrder.userId,  trade.marketPda, "NO",   qty);
            await this.creditReserved(tx, sellerOrder.userId, trade.marketPda, "USDC", sellerUsdcLeg);
            await this.debitFree     (tx, buyerOrder.userId,  trade.marketPda, "USDC", sellerUsdcLeg);
            await this.debitFree     (tx, sellerOrder.userId, trade.marketPda, "NO",   qty);
          } else {
            throw new Error(`revert TRADE: unexpected reservation pair ${buyerRes.asset}/${sellerRes.asset}`);
          }
          break;
        case "MINT":
          await this.creditReserved(tx, buyerOrder.userId,  trade.marketPda, "USDC", buyerUsdcLeg);
          await this.creditReserved(tx, sellerOrder.userId, trade.marketPda, "USDC", sellerUsdcLeg);
          await this.debitFree     (tx, buyerOrder.userId,  trade.marketPda, "YES",  qty);
          await this.debitFree     (tx, sellerOrder.userId, trade.marketPda, "NO",   qty);
          break;
        case "MERGE":
          await this.creditReserved(tx, buyerOrder.userId,  trade.marketPda, "NO",   qty);
          await this.creditReserved(tx, sellerOrder.userId, trade.marketPda, "YES",  qty);
          await this.debitFree     (tx, buyerOrder.userId,  trade.marketPda, "USDC", sellerUsdcLeg);
          await this.debitFree     (tx, sellerOrder.userId, trade.marketPda, "USDC", buyerUsdcLeg);
          break;
      }

      // Re-inflate the reservation rows so I2 (reserved == sum(active reservations)) holds.
      // Map each side back to maker/taker orders.
      const makerReinflate = makerOrder.id === buyerOrder.id  ? buyerConsumed  : sellerConsumed;
      const takerReinflate = takerOrder.id === buyerOrder.id  ? buyerConsumed  : sellerConsumed;
      await this.reinflateReservation(tx, makerRes.id, makerReinflate);
      await this.reinflateReservation(tx, takerRes.id, takerReinflate);

      // Cancel both orders (default policy — spec §7.3).
      // Also roll back `filled` by the trade quantity so accounting is clean.
      await this.decrementFilledAndCancel(tx, makerOrder.id, qty);
      await this.decrementFilledAndCancel(tx, takerOrder.id, qty);

      await tx.ob2Trade.update({
        where: { id: tradeId },
        data: { status: "REVERTED", revertReason: reason },
      });

      return {
        tradeId,
        reason,
        reopened: { makerOrderId: false, takerOrderId: false },
      };
    });
  }

  private async creditReserved(
    tx: any, userId: string, marketPda: string, asset: Ob2Asset, amount: bigint,
  ): Promise<void> {
    if (amount === 0n) return;
    await tx.ob2UserMarketBalance.upsert({
      where: { userId_marketPda_asset: { userId, marketPda, asset } },
      create: { userId, marketPda, asset, free: "0", reserved: "0", onchainTotal: "0" },
      update: {},
    });
    await tx.$executeRawUnsafe(
      `UPDATE ob2_user_market_balances
          SET reserved = reserved + $4::numeric, updated_at = now()
        WHERE user_id = $1 AND market_pda = $2 AND asset = $3::"Ob2Asset"`,
      userId, marketPda, asset, fromMicro(amount),
    );
  }

  private async debitFree(
    tx: any, userId: string, marketPda: string, asset: Ob2Asset, amount: bigint,
  ): Promise<void> {
    if (amount === 0n) return;
    await tx.$executeRawUnsafe(
      `UPDATE ob2_user_market_balances
          SET free = free - $4::numeric, updated_at = now()
        WHERE user_id = $1 AND market_pda = $2 AND asset = $3::"Ob2Asset"
          AND free >= $4::numeric`,
      userId, marketPda, asset, fromMicro(amount),
    );
  }

  private async reinflateReservation(tx: any, reservationId: string, amount: bigint): Promise<void> {
    const res = await tx.ob2Reservation.findUnique({ where: { id: reservationId } });
    if (!res) return;
    const current = toMicro(res.amount);
    const newAmount = current + amount;
    await tx.ob2Reservation.update({
      where: { id: reservationId },
      data: {
        amount: fromMicro(newAmount),
        releasedAt: null,
      },
    });
  }

  private async decrementFilledAndCancel(tx: any, orderId: string, by: bigint): Promise<void> {
    const o = await tx.ob2Order.findUnique({ where: { id: orderId } });
    if (!o) return;
    const filled = toMicro(o.filled);
    const newFilled = filled - by;
    await tx.ob2Order.update({
      where: { id: orderId },
      data: {
        filled: fromMicro(newFilled < 0n ? 0n : newFilled),
        status: "CANCELLED",
        closedAt: new Date(),
      },
    });
  }
}
```

- [ ] **Step 5.4: Run, 5 pass.**

```bash
bun x jest src/modules/trading-v2/__tests__/settlement-reverter.integration.test.ts --runInBand
```

- [ ] **Step 5.5: Confirm full suite ainda passa.**

```bash
bun x jest src/modules/trading-v2 --runInBand
```

Expected: 76 + 5 = 81 tests.

- [ ] **Step 5.6: Commit.**

```bash
git add api/src/modules/trading-v2/services/settlement-reverter.service.ts \
        api/src/modules/trading-v2/__tests__/settlement-reverter.integration.test.ts
git commit -m "feat(trading-v2): SettlementReverter (inverso do settler, cancela ordens)"
```

---

## Task 6: SolanaSettlerService (real ISettler via IOnchainCaller)

**Files:**
- Create: `api/src/modules/trading-v2/services/solana-settler.service.ts`
- Create: `api/src/modules/trading-v2/__tests__/solana-settler.integration.test.ts`

Fluxo:
1. Busca trade (SETTLING).
2. Deriva SettleFillParams (userIds, reservation assets, primitive, qty, price).
3. Chama `IOnchainCaller.sendSettleFill`.
4. Se ok: aplica deltas DB (reutiliza a lógica do StubSettler — pode até chamar StubSettler internamente OU extrair o delta-apply) + marca SETTLED com signature.
5. Se !ok && retryable: retry com backoff (max 3 tentativas em 30s total por SLA).
6. Se !ok && !retryable (ou retries esgotados): chama `SettlementReverter.revert(tradeId, reason)`.

Implementação pragmática: delegamos a aplicação de deltas pro `StubSettlerService` (renomeamos o método público `applyDeltas` pra ser reutilizável). Assim mantemos a lógica de per-primitive delta em UM lugar.

- [ ] **Step 6.1: Refactor `stub-settler.service.ts` — extrair a aplicação de deltas pra método público.**

Transforme o corpo de `settle()` num método `applyDeltas(tx, tradeId): Promise<void>` que NÃO muda o status do trade. O `settle()` fica:

```typescript
  async settle(tradeId: string): Promise<void> {
    await this.prisma.$transaction(async (tx) => {
      const trade = await tx.ob2Trade.findUnique({ where: { id: tradeId } });
      if (!trade || trade.status !== "SETTLING") return;
      await this.applyDeltas(tx, tradeId);
      await tx.ob2Trade.update({
        where: { id: tradeId },
        data: { status: "SETTLED", settledAt: new Date(), txSignature: `stub:${tradeId.slice(0,8)}` },
      });
    });
  }

  /** Exported for reuse by SolanaSettlerService. Expects trade to be in SETTLING. */
  async applyDeltas(tx: any, tradeId: string): Promise<void> {
    // move the original body here (everything that was inside the transaction but
    // ends BEFORE the final tx.ob2Trade.update that marks SETTLED)
    ...
  }
```

Confirm `stub-settler.integration.test.ts` ainda passa (comportamento inalterado).

- [ ] **Step 6.2: Testes SolanaSettler:**

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
import { UNIT } from "../types/balance.types";
import { RevertReason } from "../types/revert.types";

const balanceRepo = new BalanceRepository(prisma);
const resSvc = new ReservationService(prisma);
const orderRepo = new OrderRepository(prisma);
const tradeRepo = new TradeRepository(prisma);
const stub = new StubSettlerService(prisma);
const reverter = new SettlementReverter(prisma, tradeRepo, orderRepo);

const BUYER = "00000000-0000-0000-0000-000000000001";
const SELLER = "00000000-0000-0000-0000-000000000002";
const MARKET = "Market1111111111111111111111111111111111111";

async function setupTRADE() {
  await prisma.ob2Trade.deleteMany({});
  await prisma.ob2Order.deleteMany({});
  await prisma.ob2Reservation.deleteMany({});
  await prisma.ob2UserMarketBalance.deleteMany({});
  await balanceRepo.upsertOnchain(BUYER,  MARKET, "USDC", 1000n * UNIT, 1n);
  await balanceRepo.upsertOnchain(SELLER, MARKET, "YES",  100n  * UNIT, 1n);
  const bo = await orderRepo.create({ userId: BUYER,  marketPda: MARKET, side: "BUY",  priceBps: 6000, quantity: 100n * UNIT, reservationId: "p" });
  const br = await resSvc.reserve({ userId: BUYER,  marketPda: MARKET, asset: "USDC", amount: 60n * UNIT, orderId: bo.id });
  await prisma.ob2Order.update({ where: { id: bo.id }, data: { reservationId: br.id } });
  const so = await orderRepo.create({ userId: SELLER, marketPda: MARKET, side: "SELL", priceBps: 6000, quantity: 100n * UNIT, reservationId: "p" });
  const sr = await resSvc.reserve({ userId: SELLER, marketPda: MARKET, asset: "YES", amount: 100n * UNIT, orderId: so.id });
  await prisma.ob2Order.update({ where: { id: so.id }, data: { reservationId: sr.id } });
  await prisma.ob2Order.update({ where: { id: bo.id }, data: { filled: "100", status: "FILLED", closedAt: new Date() } });
  await prisma.ob2Order.update({ where: { id: so.id }, data: { filled: "100", status: "FILLED", closedAt: new Date() } });
  const t = await tradeRepo.create({
    marketPda: MARKET, makerOrderId: so.id, takerOrderId: bo.id,
    priceBps: 6000, quantity: 100n * UNIT, primitive: "TRADE", sync: true,
  });
  return { tradeId: t.id };
}

afterAll(async () => { await prisma.$disconnect(); });

test("on-chain success: trade SETTLED with real signature, balances moved", async () => {
  const { tradeId } = await setupTRADE();
  const caller = new MockOnchainCaller({ mode: "success" });
  const settler = new SolanaSettlerService(prisma, tradeRepo, stub, reverter, caller);

  await settler.settle(tradeId);

  const t = await tradeRepo.getById(tradeId);
  expect(t!.status).toBe("SETTLED");
  expect(t!.txSignature).toMatch(/^mock:/);
  expect((await balanceRepo.get(BUYER, MARKET, "YES"))!.free).toBe(100n * UNIT);
  expect((await balanceRepo.get(SELLER, MARKET, "USDC"))!.free).toBe(60n * UNIT);
  expect(caller.history).toHaveLength(1);
});

test("non-retryable failure: trade REVERTED with reason from caller, balances restored", async () => {
  const { tradeId } = await setupTRADE();
  const caller = new MockOnchainCaller({ mode: "failure", reason: "program_error", retryable: false });
  const settler = new SolanaSettlerService(prisma, tradeRepo, stub, reverter, caller);

  await settler.settle(tradeId);

  const t = await tradeRepo.getById(tradeId);
  expect(t!.status).toBe("REVERTED");
  expect(t!.revertReason).toBe(RevertReason.OnchainProgramError);
  // Buyer balance back to 1000 USDC
  const buyerUsdc = await balanceRepo.get(BUYER, MARKET, "USDC");
  expect(buyerUsdc!.free + buyerUsdc!.reserved).toBe(1000n * UNIT);
});

test("retryable failure then success within retry budget", async () => {
  const { tradeId } = await setupTRADE();
  const caller = new MockOnchainCaller({
    mode: "sequence",
    sequence: [
      { ok: false, reason: "rpc_down", retryable: true },
      { ok: false, reason: "rpc_down", retryable: true },
      { ok: true, signature: "mock:retry-ok" },
    ],
  });
  const settler = new SolanaSettlerService(prisma, tradeRepo, stub, reverter, caller);

  await settler.settle(tradeId);

  const t = await tradeRepo.getById(tradeId);
  expect(t!.status).toBe("SETTLED");
  expect(t!.txSignature).toBe("mock:retry-ok");
  expect(caller.history).toHaveLength(3);
});

test("retryable failure exhausting retries → REVERTED with OnchainFailed", async () => {
  const { tradeId } = await setupTRADE();
  const caller = new MockOnchainCaller({
    mode: "sequence",
    sequence: [
      { ok: false, reason: "rpc_down", retryable: true },
      { ok: false, reason: "rpc_down", retryable: true },
      { ok: false, reason: "rpc_down", retryable: true },
    ],
  });
  const settler = new SolanaSettlerService(prisma, tradeRepo, stub, reverter, caller, { maxRetries: 3, retryDelayMs: 1 });

  await settler.settle(tradeId);

  const t = await tradeRepo.getById(tradeId);
  expect(t!.status).toBe("REVERTED");
  expect(t!.revertReason).toBe(RevertReason.OnchainFailed);
});

test("settle on already SETTLED trade: no-op", async () => {
  const { tradeId } = await setupTRADE();
  await prisma.ob2Trade.update({ where: { id: tradeId }, data: { status: "SETTLED", settledAt: new Date() } });
  const caller = new MockOnchainCaller({ mode: "success" });
  const settler = new SolanaSettlerService(prisma, tradeRepo, stub, reverter, caller);

  await settler.settle(tradeId);

  expect(caller.history).toHaveLength(0);  // no on-chain call
});
```

- [ ] **Step 6.3: Run, fail.**

- [ ] **Step 6.4: Implement `services/solana-settler.service.ts`:**

```typescript
import type { PrismaClient } from "../../../generated/prisma/client";
import type { TradeRepository } from "../repositories/trade.repository";
import type { StubSettlerService } from "./stub-settler.service";
import type { SettlementReverter } from "./settlement-reverter.service";
import type { IOnchainCaller, SettleFillParams } from "../types/onchain-caller.types";
import type { ISettler } from "../types/trade.types";
import { RevertReason } from "../types/revert.types";
import { toMicro } from "../types/decimal-helpers";

export interface SolanaSettlerConfig {
  maxRetries?: number;       // default 3
  retryDelayMs?: number;     // default 200ms, exponential backoff (delay * 2^attempt)
}

/**
 * Real ISettler. Fluxo:
 *   1. Trade em SETTLING? Monta SettleFillParams.
 *   2. Call IOnchainCaller.sendSettleFill com retry pra retryable failures (até maxRetries).
 *   3. Sucesso → aplica deltas DB (via stub.applyDeltas) + marca SETTLED com real signature.
 *   4. Falha definitiva ou retries exaustos → SettlementReverter.revert.
 *
 * Trade já SETTLED/REVERTED: no-op.
 */
export class SolanaSettlerService implements ISettler {
  private readonly maxRetries: number;
  private readonly retryDelayMs: number;

  constructor(
    private readonly prisma: PrismaClient,
    private readonly trades: TradeRepository,
    private readonly stub: StubSettlerService,   // reuse applyDeltas for per-primitive math
    private readonly reverter: SettlementReverter,
    private readonly caller: IOnchainCaller,
    config: SolanaSettlerConfig = {},
  ) {
    this.maxRetries = config.maxRetries ?? 3;
    this.retryDelayMs = config.retryDelayMs ?? 200;
  }

  async settle(tradeId: string): Promise<void> {
    const trade = await this.trades.getById(tradeId);
    if (!trade) throw new Error(`trade ${tradeId} not found`);
    if (trade.status !== "SETTLING") return;

    // Build params
    const makerOrder = await this.prisma.ob2Order.findUnique({ where: { id: trade.makerOrderId } });
    const takerOrder = await this.prisma.ob2Order.findUnique({ where: { id: trade.takerOrderId } });
    if (!makerOrder || !takerOrder) throw new Error("orders not found");
    const makerRes = await this.prisma.ob2Reservation.findFirst({ where: { orderId: makerOrder.id } });
    const takerRes = await this.prisma.ob2Reservation.findFirst({ where: { orderId: takerOrder.id } });
    if (!makerRes || !takerRes) throw new Error("reservations not found");

    const buyerIsTaker = takerOrder.side === "BUY";
    const params: SettleFillParams = {
      tradeId,
      marketPda: trade.marketPda,
      buyerUserId:  buyerIsTaker ? takerOrder.userId : makerOrder.userId,
      sellerUserId: buyerIsTaker ? makerOrder.userId : takerOrder.userId,
      buyerReservationAsset:  buyerIsTaker ? takerRes.asset : makerRes.asset,
      sellerReservationAsset: buyerIsTaker ? makerRes.asset : takerRes.asset,
      priceBps: trade.priceBps,
      quantityMicro: toMicro(trade.quantity),
      primitive: trade.primitive,
    };

    // Call with retry
    let lastReason = "unknown";
    for (let attempt = 0; attempt < this.maxRetries; attempt++) {
      const res = await this.caller.sendSettleFill(params);
      if (res.ok) {
        await this.prisma.$transaction(async (tx) => {
          const current = await tx.ob2Trade.findUnique({ where: { id: tradeId } });
          if (!current || current.status !== "SETTLING") return;  // idempotente
          await this.stub.applyDeltas(tx, tradeId);
          await tx.ob2Trade.update({
            where: { id: tradeId },
            data: { status: "SETTLED", settledAt: new Date(), txSignature: res.signature },
          });
        });
        return;
      }
      lastReason = res.reason;
      if (!res.retryable) {
        await this.reverter.revert(tradeId, this.mapReason(res.reason));
        return;
      }
      if (attempt < this.maxRetries - 1) {
        await new Promise(r => setTimeout(r, this.retryDelayMs * Math.pow(2, attempt)));
      }
    }
    // Exhausted retries
    await this.reverter.revert(tradeId, RevertReason.OnchainFailed);
  }

  private mapReason(raw: string): RevertReason {
    if (raw.includes("timeout")) return RevertReason.OnchainTimeout;
    if (raw.includes("program")) return RevertReason.OnchainProgramError;
    return RevertReason.OnchainFailed;
  }
}
```

- [ ] **Step 6.5: Run, 5 pass.**

```bash
bun x jest src/modules/trading-v2/__tests__/solana-settler.integration.test.ts --runInBand
```

- [ ] **Step 6.6: Confirm full suite passes (stub refactor didn't break anything).**

```bash
bun x jest src/modules/trading-v2 --runInBand
```

Expected: 81 + 5 = 86 tests.

- [ ] **Step 6.7: Commit.**

```bash
git add api/src/modules/trading-v2/services/solana-settler.service.ts \
        api/src/modules/trading-v2/services/stub-settler.service.ts \
        api/src/modules/trading-v2/__tests__/solana-settler.integration.test.ts
git commit -m "feat(trading-v2): SolanaSettlerService (real ISettler via IOnchainCaller)"
```

---

## Task 7: ReconcileSettlingTradesService (deadline worker)

**Files:**
- Create: `api/src/modules/trading-v2/services/reconcile-settling-trades.service.ts`
- Create: `api/src/modules/trading-v2/__tests__/reconcile-settling-trades.integration.test.ts`

- [ ] **Step 7.1: Testes:**

```typescript
import "dotenv/config";
import { prisma } from "../../../shared/database/config/prisma-client";
import { BalanceRepository } from "../repositories/balance.repository";
import { ReservationService } from "../services/reservation.service";
import { OrderRepository } from "../repositories/order.repository";
import { TradeRepository } from "../repositories/trade.repository";
import { StubSettlerService } from "../services/stub-settler.service";
import { SettlementReverter } from "../services/settlement-reverter.service";
import { ReconcileSettlingTradesService } from "../services/reconcile-settling-trades.service";
import { RevertReason } from "../types/revert.types";
import { UNIT } from "../types/balance.types";

const balanceRepo = new BalanceRepository(prisma);
const resSvc = new ReservationService(prisma);
const orderRepo = new OrderRepository(prisma);
const tradeRepo = new TradeRepository(prisma);
const stub = new StubSettlerService(prisma);
const reverter = new SettlementReverter(prisma, tradeRepo, orderRepo);
const worker = new ReconcileSettlingTradesService(tradeRepo, reverter);

const BUYER = "00000000-0000-0000-0000-000000000001";
const SELLER = "00000000-0000-0000-0000-000000000002";
const MARKET = "Market1111111111111111111111111111111111111";

async function makeSettlingTrade(deadlineOffsetMs: number): Promise<string> {
  await balanceRepo.upsertOnchain(BUYER,  MARKET, "USDC", 1000n * UNIT, 1n);
  await balanceRepo.upsertOnchain(SELLER, MARKET, "YES",  100n  * UNIT, 1n);
  const bo = await orderRepo.create({ userId: BUYER,  marketPda: MARKET, side: "BUY",  priceBps: 6000, quantity: 100n * UNIT, reservationId: "p" });
  const br = await resSvc.reserve({ userId: BUYER, marketPda: MARKET, asset: "USDC", amount: 60n * UNIT, orderId: bo.id });
  await prisma.ob2Order.update({ where: { id: bo.id }, data: { reservationId: br.id } });
  const so = await orderRepo.create({ userId: SELLER, marketPda: MARKET, side: "SELL", priceBps: 6000, quantity: 100n * UNIT, reservationId: "p" });
  const sr = await resSvc.reserve({ userId: SELLER, marketPda: MARKET, asset: "YES", amount: 100n * UNIT, orderId: so.id });
  await prisma.ob2Order.update({ where: { id: so.id }, data: { reservationId: sr.id } });
  await prisma.ob2Order.update({ where: { id: bo.id }, data: { filled: "100", status: "FILLED", closedAt: new Date() } });
  await prisma.ob2Order.update({ where: { id: so.id }, data: { filled: "100", status: "FILLED", closedAt: new Date() } });
  const t = await tradeRepo.create({
    marketPda: MARKET, makerOrderId: so.id, takerOrderId: bo.id,
    priceBps: 6000, quantity: 100n * UNIT, primitive: "TRADE", sync: true,
  });
  // Manually set the deadline
  await prisma.ob2Trade.update({
    where: { id: t.id },
    data: { settlingDeadline: new Date(Date.now() + deadlineOffsetMs) },
  });
  return t.id;
}

beforeEach(async () => {
  await prisma.ob2Trade.deleteMany({});
  await prisma.ob2Order.deleteMany({});
  await prisma.ob2Reservation.deleteMany({});
  await prisma.ob2UserMarketBalance.deleteMany({});
});
afterAll(async () => { await prisma.$disconnect(); });

test("scan finds no trades when none expired", async () => {
  await makeSettlingTrade(60_000);    // 1 min in the future
  const result = await worker.scanAndRevert();
  expect(result.revertedCount).toBe(0);
});

test("scan reverts trade past deadline with reason DeadlineExceeded", async () => {
  const tradeId = await makeSettlingTrade(-10_000);    // 10s in the past
  const result = await worker.scanAndRevert();
  expect(result.revertedCount).toBe(1);

  const t = await tradeRepo.getById(tradeId);
  expect(t!.status).toBe("REVERTED");
  expect(t!.revertReason).toBe(RevertReason.DeadlineExceeded);
});

test("scan is idempotent on already-REVERTED trades", async () => {
  await makeSettlingTrade(-10_000);
  await worker.scanAndRevert();
  const result2 = await worker.scanAndRevert();
  expect(result2.revertedCount).toBe(0);
});

test("scan processes multiple expired trades in one pass", async () => {
  await makeSettlingTrade(-10_000);
  await new Promise(r => setTimeout(r, 10));
  await makeSettlingTrade(-20_000);
  const result = await worker.scanAndRevert();
  expect(result.revertedCount).toBe(2);
});
```

- [ ] **Step 7.2: Run, fail.**

- [ ] **Step 7.3: Implement `services/reconcile-settling-trades.service.ts`:**

```typescript
import type { TradeRepository } from "../repositories/trade.repository";
import type { SettlementReverter } from "./settlement-reverter.service";
import { RevertReason } from "../types/revert.types";

export interface ScanResult {
  scannedCount: number;
  revertedCount: number;
  errors: Array<{ tradeId: string; err: string }>;
}

/**
 * Worker de SLA: scan trades em SETTLING com deadline no passado e dispara revert.
 *
 * Rodado por um cron/timer externo (ex.: a cada 30s). Idempotente por construção
 * (SettlementReverter.revert é idempotente).
 *
 * NÃO chama o on-chain caller — assume que se um trade está em SETTLING além do
 * deadline, o settlement falhou ou o evento se perdeu, e o usuário deve ser
 * devolvido ao estado pré-match.
 */
export class ReconcileSettlingTradesService {
  constructor(
    private readonly trades: TradeRepository,
    private readonly reverter: SettlementReverter,
  ) {}

  async scanAndRevert(): Promise<ScanResult> {
    const expired = await this.trades.listExpiredSettling();
    const errors: ScanResult["errors"] = [];
    let revertedCount = 0;

    for (const trade of expired) {
      try {
        const r = await this.reverter.revert(trade.id, RevertReason.DeadlineExceeded);
        if (r) revertedCount++;
      } catch (e) {
        errors.push({ tradeId: trade.id, err: e instanceof Error ? e.message : String(e) });
      }
    }

    return { scannedCount: expired.length, revertedCount, errors };
  }
}
```

- [ ] **Step 7.4: Run, 4 pass.**

- [ ] **Step 7.5: Commit.**

```bash
git add api/src/modules/trading-v2/services/reconcile-settling-trades.service.ts \
        api/src/modules/trading-v2/__tests__/reconcile-settling-trades.integration.test.ts
git commit -m "feat(trading-v2): ReconcileSettlingTradesService (deadline worker)"
```

---

## Task 8: Swap StubSettler → SolanaSettler nas rotas + barrel

**Files:**
- Modify: `api/src/modules/trading-v2/routes/orders.routes.ts`
- Modify: `api/src/modules/trading-v2/index.ts`

- [ ] **Step 8.1: Atualizar `orders.routes.ts`.** Substituir o wiring atual:

```typescript
import { MockOnchainCaller } from "../services/mock-onchain-caller.service";
import { SolanaSettlerService } from "../services/solana-settler.service";
import { SettlementReverter } from "../services/settlement-reverter.service";

const stub = new StubSettlerService(prisma);
const reverter = new SettlementReverter(prisma, tradeRepo, orderRepo);

// Plano 4 troca MockOnchainCaller por SolanaOnchainCaller.
// Até lá, usamos o Mock com modo "success" como stand-in em prod — mesmo comportamento
// do StubSettler que já estava em uso.
const caller = new MockOnchainCaller({ mode: "success" });
const settler = new SolanaSettlerService(prisma, tradeRepo, stub, reverter, caller);
const engine = new MatchingEngine(prisma, orderRepo, tradeRepo, settler);
```

Remover o import antigo isolado de `StubSettlerService` pro engine (mantemos o `stub` instance pra reuso do `applyDeltas`). O `MatchingEngine` agora recebe `SolanaSettlerService` diretamente — a interface `ISettler` é a mesma.

**Atenção:** isso muda o comportamento runtime. Antes o engine chamava `StubSettler` que aplicava deltas e marcava SETTLED inline. Agora chama `SolanaSettler` que chama `MockOnchainCaller(success)` e, no ok, delega a `stub.applyDeltas`. Comportamento observacional **idêntico** porque o mock retorna ok imediatamente. Os testes e2e existentes devem passar sem mudança.

- [ ] **Step 8.2: Rodar full suite.**

```bash
cd /Users/rafael/Documents/GitHub/Oraculo/.claude/worktrees/orderbook-rewrite-03-settlement-real/api
bun x jest src/modules/trading-v2 --runInBand
```

Expected: 86 + 0 (no novos testes nesta task) = 86 passed. Se algum cenário e2e falhar, investigar a diferença de timing (SolanaSettler faz uma camada extra de await — se algum teste depender de timing exato, revisar).

- [ ] **Step 8.3: Atualizar barrel `index.ts` — append:**

```typescript
export { MockOnchainCaller } from "./services/mock-onchain-caller.service";
export { SolanaSettlerService } from "./services/solana-settler.service";
export { SettlementReverter } from "./services/settlement-reverter.service";
export { ReconcileSettlingTradesService } from "./services/reconcile-settling-trades.service";
```

- [ ] **Step 8.4: Commit.**

```bash
git add api/src/modules/trading-v2/routes/orders.routes.ts api/src/modules/trading-v2/index.ts
git commit -m "feat(trading-v2): wiring HTTP usa SolanaSettler+MockCaller (Plano 4 troca por real)"
```

---

## Task 9: Smoke test HTTP + README + full suite + deprecation note

**Files:**
- Modify: `api/src/modules/trading-v2/README.md`
- Modify: `api/src/modules/trading-v2/services/stub-settler.service.ts` (@deprecated JSDoc)

- [ ] **Step 9.1: Marcar StubSettlerService como deprecated (não remover — testes ainda o usam diretamente):**

No topo da classe, acima do `export class StubSettlerService`:

```typescript
/**
 * @deprecated (desde Plano 3) — use `SolanaSettlerService` + `MockOnchainCaller`
 * em testes, ou (Plano 4) `SolanaSettlerService` + `SolanaOnchainCaller` em prod.
 *
 * Ainda mantido porque: (a) `SolanaSettlerService` reutiliza `StubSettlerService.applyDeltas`
 * pra não duplicar a lógica de per-primitive balance math, e (b) alguns testes de
 * integração (stub-settler.integration.test.ts) testam o stub diretamente.
 */
export class StubSettlerService implements ISettler {
```

- [ ] **Step 9.2: Sobrescrever `README.md`:**

```markdown
# trading-v2

Novo orderbook. Planos 1–3 entregues.

- **Plano 1:** fundação (balances/reservations/intent classifier/reconciliation)
- **Plano 2:** orders lifecycle + matching engine síncrono + stub de settlement
- **Plano 3:** SolanaSettler (via IOnchainCaller abstrato), SettlementReverter,
  worker de deadline SLA + fechamento de janelas residuais do Plano 2

Spec: `docs/superpowers/specs/2026-04-15-orderbook-rewrite-design.md`
Planos:
- 1: `docs/superpowers/plans/2026-04-15-orderbook-rewrite-01-foundation.md` (merged)
- 2: `docs/superpowers/plans/2026-04-15-orderbook-rewrite-02-orders-matching.md` (merged)
- 3: `docs/superpowers/plans/2026-04-15-orderbook-rewrite-03-settlement-real.md` (este)

## Endpoints HTTP

- `POST /api/v2/trading/orders` — place order
- `DELETE /api/v2/trading/orders/:id` — cancel
- `GET /api/v2/trading/orders?marketPda=X[&status=OPEN]` — list

## Settlement

- `ISettler` — contrato com um método `settle(tradeId)`.
- `SolanaSettlerService` — implementação real. Flui via `IOnchainCaller`
  (abstract); chama on-chain com retry (max 3), backoff exponencial; aplica
  deltas DB no ok; em falha definitiva ou exaustão de retries, chama
  `SettlementReverter.revert(tradeId, reason)`.
- `IOnchainCaller` — abstração do transport on-chain. Plano 4 entrega caller
  concreto `SolanaOnchainCaller`. Plano 3 usa `MockOnchainCaller` configurável.
- `SettlementReverter` — inverso do settler. Política default: ordens afetadas
  pelo trade são CANCELLED (não reabertas), per spec §7.3. Idempotente.
- `ReconcileSettlingTradesService` — worker que scan trades SETTLING expirados
  e dispara revert. Deve ser invocado por um cron/timer (não incluído ainda —
  integração no app entry vem no Plano 7 junto do reconcile de saldos diário).

## Divergência stub vs real (pra atenção no cutover)

O `StubSettlerService` (marcado `@deprecated`) implementa o caso `TRADE NO(BUY)×USDC(SELL)`
creditando NO ao seller sem fonte — efetivamente minting. O `SolanaOnchainCaller`
real (Plano 4) modelará esse caso como short-open, onde o seller não recebe NO
no `free` balance mas um position state. **Reconciliação de cutover (Plano 7)
deve compensar essa diferença** para usuários que abriram shorts via o stub.

## Invariantes verificadas

- **I1** (reserva obrigatória atômica): `__tests__/reservation.service.invariant-i1.test.ts`
- **I2** (conservação): `__tests__/balance.service.invariant-i2.test.ts`, multi-user em `matching-scenarios.e2e.test.ts`

## Rodar testes

```bash
cd api
bun x jest src/modules/trading-v2 --runInBand
```

Precisa de Postgres local (docker compose up postgres) com schema aplicado:
```bash
bun x prisma db push
psql "$DATABASE_URL" -f prisma/scripts/trading-v2-foundation.sql
psql "$DATABASE_URL" -f prisma/scripts/trading-v2-orders.sql
```

## Próximos planos

- 4: `settle_fill` no programa Solana + `SolanaOnchainCaller` concreto + listener de eventos
- 5: WebSocket v2 • 6: MM bot externo • 7: Cutover + reconciliação diária
```

- [ ] **Step 9.3: Rodar suíte inteira:**

```bash
cd /Users/rafael/Documents/GitHub/Oraculo/.claude/worktrees/orderbook-rewrite-03-settlement-real/api
bun x jest src/modules/trading-v2 --runInBand
```

Expected final total: **90 tests passed** (Plano 2: 67, Plano 3 deltas: +3 place-order window, −1 dropped REJECTED assertion, +7 decimal helpers, +5 solana-settler, +5 revert, +4 reconcile-deadline, +5 mock-onchain-caller = +28 net from Plano 2 baseline).

Actual breakdown por arquivo (você deve confirmar):
- balance.repository.integration.test.ts — 3
- reservation.service.integration.test.ts — 10
- reservation.service.invariant-i1.test.ts — 2
- balance.service.invariant-i2.test.ts — 2
- intent-classifier.unit.test.ts — 5
- reconciliation.service.unit.test.ts — 3
- snapshot-onchain-balances.integration.test.ts — 2
- order.repository.integration.test.ts — 9
- trade.repository.integration.test.ts — 5
- primitive-decider.unit.test.ts — 5
- stub-settler.integration.test.ts — 4
- matching-engine.integration.test.ts — 5
- place-order.use-case.integration.test.ts — 3 (após adjust do Task 1)
- cancel-order.use-case.integration.test.ts — 4
- matching-scenarios.e2e.test.ts — 4
- place-order-window.integration.test.ts — 3
- decimal-helpers.unit.test.ts — 7
- mock-onchain-caller.unit.test.ts — 5
- solana-settler.integration.test.ts — 5
- settlement-reverter.integration.test.ts — 5
- reconcile-settling-trades.integration.test.ts — 4
- **Total: 95**

Se 95 bater, perfeito. Se o placeholder de `place-order.use-case` ficou com 4 testes (não adaptou o REJECTED), total = 96.

- [ ] **Step 9.4: tsc clean.**

```bash
bun x tsc --noEmit 2>&1 | grep "src/modules/trading-v2" || echo "clean"
```

- [ ] **Step 9.5: Commit.**

```bash
git add api/src/modules/trading-v2/services/stub-settler.service.ts \
        api/src/modules/trading-v2/README.md
git commit -m "docs(trading-v2): README pós-plano 3 + deprecate StubSettlerService"
```

---

## Critérios de aceitação do plano

1. ✅ Janela de pending-reservation fechada; zero orders com `reservationId="pending"`.
2. ✅ Helper único de conversão Decimal↔bigint, aplicado a todos os repos/services.
3. ✅ `IOnchainCaller` abstração + `MockOnchainCaller` configurável.
4. ✅ `SolanaSettlerService` com retry/backoff, sucesso → aplica deltas + SETTLED; falha → revert.
5. ✅ `SettlementReverter` desfaz deltas de cada primitive, cancela ordens afetadas.
6. ✅ `ReconcileSettlingTradesService` reverte trades SETTLING além do deadline (SLA I4).
7. ✅ HTTP routes usam o novo wiring (SolanaSettler + MockCaller); troca pra caller real em Plano 4.
8. ✅ Suíte trading-v2 ≥ 95 verde; tsc clean.
9. ✅ README atualizado com divergência stub vs real.

---

## Riscos e mitigações

| Risco | Mitigação |
|---|---|
| `SolanaSettler.settle` chamando `stub.applyDeltas` inside sua própria tx — o `stub.applyDeltas` agora é método exportado que recebe `tx` externo | Refactor do Task 6.1 garante que `applyDeltas(tx, tradeId)` não abre sua própria transação; caller orquestra. Testes do stub (inalterados) validam. |
| SettlementReverter pode introduzir drift se chamado em paralelo ao settler | Revert usa $transaction e checa `status === "SETTLING"` na entrada; dupla execução cai no idempotent return. Settler idem. As duas operações são mutuamente exclusivas por status. |
| Deadline worker acidentalmente reverte trade que o settler ainda está processando | O caller síncrono (HTTP request) completa em < 30s (sync SLA). Worker default roda a cada 30s+ e checa deadline = 30s pro sync; janela mínima de sobreposição. Se settler demora mais que SLA, worker toma o trade — esse é o comportamento correto (falha de SLA). |
| Política de cancel em vez de reabrir na revert pode frustrar usuários com maker partial fills | Default = cancel é spec §7.3. Flag pra reabrir ficará em Plano 5 (WS) quando tivermos UI pra explicar. |

---

## O que NÃO está neste plano

- **Caller on-chain concreto (`SolanaOnchainCaller`)**: Plano 4, junto da instrução `settle_fill` no programa.
- **Listener on-chain** (observar confirmações e atualizar ob2_onchain_events_processed): Plano 4.
- **Cron/timer pra rodar o reconcile worker**: integração com scheduler do app fica no Plano 7.
- **Fee ledger real**: ainda ignorado pelo stub. Plano 3.5 opcional ou integrado no Plano 4.
- **Refund de price-improvement quando fee > 0**: P2 risco conhecido; endereçado junto com fee ledger.
