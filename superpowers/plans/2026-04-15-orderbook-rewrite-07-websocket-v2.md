# Orderbook Rewrite — Plano 7: WebSocket v2

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Entregar WebSocket stateless pro trading-v2 que propaga `order_update`, `trade_settled`, `trade_reverted` e `book_snapshot` (top-N levels) em tempo real, sem nenhum estado próprio — toda a fonte é o Postgres. Multi-instance via Redis Pub/Sub (mesmo padrão do legacy).

**Architecture:** Interface `ITradingV2EventPublisher` abstrai o transport de pub/sub. Implementação real usa Redis (reuso do `redis` já configurado em `shared/database/config/redis-config.ts`). Publishers injetados em `PlaceOrderUseCase`, `CancelOrderUseCase`, `SolanaSettlerService` (emitido dentro do tx de SETTLED) e `SettlementReverter` (emitido ao marcar REVERTED). Subscriber-side: `WsTradingV2Handler` (Bun ServerWebSocket) aceita comandos `subscribe`/`unsubscribe` de markets e users; escuta canais Redis `ob2:market:<pda>` e `ob2:user:<userId>` e repassa eventos pros clients conectados. Book snapshot é computed sob demanda via `BookSnapshotService` a partir de `ob2_orders WHERE status='OPEN'`.

**Tech Stack:** Bun runtime (ServerWebSocket nativo), `ioredis`, Prisma 7, Jest.

**Spec:** `docs/superpowers/specs/2026-04-15-orderbook-rewrite-design.md` (§10 WebSocket)
**Planos anteriores:**
- Plano 1–6 (merged). Último: `5cdf109` — fee ledger integration. 132 testes verdes.

**Próximos planos:**
- Plano 8: MM bot externo • Plano 9: Cutover

**Não-escopo:**
- **Autenticação/autorização no WS**: assumimos mesmo middleware do legacy (extrai Clerk JWT do header). Plan só entrega a shape; wire real de auth fica com a plataforma.
- **Historical events replay**: WS só empurra eventos live. Catch-up após reconexão é responsabilidade do client (fetch via HTTP REST endpoints).
- **Rate limiting por client**: deferido pra infra externa (CloudFlare, nginx).
- **Dashboard rooms** (legacy): fora de escopo; trading-v2 só expõe market + user rooms.

---

## File Structure

```
api/src/modules/trading-v2/
  types/
    events.types.ts                          # CREATE: OrderUpdateEvent, TradeSettledEvent, etc.
  services/
    trading-v2-event-publisher.ts            # CREATE: ITradingV2EventPublisher + RedisPublisher + InMemoryPublisher
    book-snapshot.service.ts                 # CREATE: top-N bids/asks from ob2_orders
    ws-trading-v2-handler.service.ts         # CREATE: WebSocket connection manager
    place-order.use-case.ts                  # MODIFY: emit order_update
    cancel-order.use-case.ts                 # MODIFY: emit order_update
    solana-settler.service.ts                # MODIFY: emit trade_settled (+ order_update if filled)
    settlement-reverter.service.ts           # MODIFY: emit trade_reverted + order_update
    matching-engine.service.ts               # MODIFY: emit order_update on partial fills
  routes/
    ws.routes.ts                             # CREATE: WebSocket endpoint /api/v2/trading/ws
  __tests__/
    trading-v2-event-publisher.unit.test.ts
    book-snapshot.service.integration.test.ts
    ws-trading-v2-handler.unit.test.ts
    event-emission.integration.test.ts       # end-to-end: place-order → event
  index.ts                                   # MODIFY: exports
  README.md                                  # MODIFY: Plano 7 section
```

**Responsabilidades (novos):**

- `events.types.ts` — tipos dos eventos WS com `type` discriminator: `OrderUpdateEvent | TradeSettledEvent | TradeRevertedEvent | BookSnapshotEvent`.
- `trading-v2-event-publisher.ts` — interface `ITradingV2EventPublisher` com `publishMarket(pda, event)` e `publishUser(userId, event)`. Duas impls: `RedisEventPublisher` (prod) e `InMemoryEventPublisher` (tests).
- `book-snapshot.service.ts` — `snapshot(marketPda, depth=20)`: read-only query que agrega `ob2_orders` por preço/side.
- `ws-trading-v2-handler.service.ts` — gerencia conexões Bun WebSocket + subscrições Redis + fan-out.
- `ws.routes.ts` — mount na app entry; parsea upgrade request.

---

## Prerequisite check

- [ ] **Step 0: Baseline.**

```bash
cd /Users/rafael/Documents/GitHub/Oraculo/api
bun x jest src/modules/trading-v2 --runInBand
```

Expected: 132 passed + 1 skipped.

- [ ] **Step 0.1: Worktree.**

```bash
git worktree add .claude/worktrees/orderbook-rewrite-07-websocket-v2 -b worktree-orderbook-rewrite-07-websocket-v2
cp api/.env .claude/worktrees/orderbook-rewrite-07-websocket-v2/api/.env
cd .claude/worktrees/orderbook-rewrite-07-websocket-v2/api
bun install
bun x prisma generate
bun x jest src/modules/trading-v2 --runInBand
```

---

## Task 1: Event types + ITradingV2EventPublisher

**Files:**
- Create: `api/src/modules/trading-v2/types/events.types.ts`
- Create: `api/src/modules/trading-v2/services/trading-v2-event-publisher.ts`
- Create: `api/src/modules/trading-v2/__tests__/trading-v2-event-publisher.unit.test.ts`

### Step 1.1: Event types.

```typescript
// events.types.ts
import type { Ob2Side, Ob2OrderStatus } from "../../../generated/prisma/client";
import type { Ob2Primitive } from "../../../generated/prisma/client";

export interface OrderUpdateEvent {
  type: "order_update";
  marketPda: string;
  userId: string;
  orderId: string;
  side: Ob2Side;
  priceBps: number;
  quantity: string;          // bigint serialized
  filled: string;            // bigint serialized
  status: Ob2OrderStatus;
  clientOrderId: string | null;
  timestamp: number;         // unix ms
}

export interface TradeSettledEvent {
  type: "trade_settled";
  marketPda: string;
  tradeId: string;
  priceBps: number;
  quantity: string;
  primitive: Ob2Primitive;
  makerOrderId: string;
  takerOrderId: string;
  txSignature: string | null;
  timestamp: number;
}

export interface TradeRevertedEvent {
  type: "trade_reverted";
  marketPda: string;
  tradeId: string;
  reason: string;
  timestamp: number;
}

export interface BookLevel {
  priceBps: number;
  quantity: string;          // total bigint at this level
}

export interface BookSnapshotEvent {
  type: "book_snapshot";
  marketPda: string;
  bids: BookLevel[];         // highest price first
  asks: BookLevel[];         // lowest price first
  timestamp: number;
}

export type TradingV2Event =
  | OrderUpdateEvent
  | TradeSettledEvent
  | TradeRevertedEvent
  | BookSnapshotEvent;
```

### Step 1.2: Publisher interface + implementations.

```typescript
// trading-v2-event-publisher.ts
import type { Redis } from "ioredis";
import type { TradingV2Event } from "../types/events.types";

export interface ITradingV2EventPublisher {
  publishMarket(marketPda: string, event: TradingV2Event): Promise<void>;
  publishUser(userId: string, event: TradingV2Event): Promise<void>;
}

export class RedisEventPublisher implements ITradingV2EventPublisher {
  constructor(private readonly redis: Redis) {}

  async publishMarket(marketPda: string, event: TradingV2Event): Promise<void> {
    await this.redis.publish(channelMarket(marketPda), JSON.stringify(event));
  }

  async publishUser(userId: string, event: TradingV2Event): Promise<void> {
    await this.redis.publish(channelUser(userId), JSON.stringify(event));
  }
}

/** Test double — records calls in-process, doesn't use Redis. */
export class InMemoryEventPublisher implements ITradingV2EventPublisher {
  public readonly marketCalls: Array<{ pda: string; event: TradingV2Event }> = [];
  public readonly userCalls:   Array<{ userId: string; event: TradingV2Event }> = [];

  async publishMarket(marketPda: string, event: TradingV2Event): Promise<void> {
    this.marketCalls.push({ pda: marketPda, event });
  }
  async publishUser(userId: string, event: TradingV2Event): Promise<void> {
    this.userCalls.push({ userId, event });
  }
}

export function channelMarket(pda: string): string { return `ob2:market:${pda}`; }
export function channelUser(userId: string): string { return `ob2:user:${userId}`; }
```

### Step 1.3: Unit tests.

```typescript
import { InMemoryEventPublisher, channelMarket, channelUser } from "../services/trading-v2-event-publisher";

test("channelMarket formats as ob2:market:<pda>", () => {
  expect(channelMarket("Market1111")).toBe("ob2:market:Market1111");
});

test("channelUser formats as ob2:user:<userId>", () => {
  expect(channelUser("u-123")).toBe("ob2:user:u-123");
});

test("InMemoryEventPublisher records publishMarket and publishUser calls", async () => {
  const pub = new InMemoryEventPublisher();
  const event = {
    type: "order_update" as const,
    marketPda: "Market1", userId: "u1", orderId: "o1",
    side: "BUY" as const, priceBps: 6000, quantity: "100", filled: "0",
    status: "OPEN" as const, clientOrderId: null, timestamp: 1,
  };
  await pub.publishMarket("Market1", event);
  await pub.publishUser("u1", event);
  expect(pub.marketCalls).toHaveLength(1);
  expect(pub.userCalls).toHaveLength(1);
  expect(pub.marketCalls[0].pda).toBe("Market1");
  expect(pub.userCalls[0].userId).toBe("u1");
});
```

### Step 1.4: Run, fail → implement → pass.

```bash
cd /Users/rafael/Documents/GitHub/Oraculo/.claude/worktrees/orderbook-rewrite-07-websocket-v2/api
bun x jest src/modules/trading-v2/__tests__/trading-v2-event-publisher.unit.test.ts --runInBand
```

### Step 1.5: Commit.

```bash
git add api/src/modules/trading-v2/types/events.types.ts \
        api/src/modules/trading-v2/services/trading-v2-event-publisher.ts \
        api/src/modules/trading-v2/__tests__/trading-v2-event-publisher.unit.test.ts
git commit -m "feat(trading-v2): event types + ITradingV2EventPublisher + Redis/InMemory impls"
```

---

## Task 2: BookSnapshotService

**Files:**
- Create: `api/src/modules/trading-v2/services/book-snapshot.service.ts`
- Create: `api/src/modules/trading-v2/__tests__/book-snapshot.service.integration.test.ts`

### Step 2.1: Failing test.

```typescript
import "dotenv/config";
import { prisma } from "../../../shared/database/config/prisma-client";
import { OrderRepository } from "../repositories/order.repository";
import { BookSnapshotService } from "../services/book-snapshot.service";
import { UNIT } from "../types/balance.types";

const orderRepo = new OrderRepository(prisma);
const svc = new BookSnapshotService(prisma);
const MARKET = "Market1111111111111111111111111111111111111";

beforeEach(async () => {
  await prisma.ob2Order.deleteMany({});
});
afterAll(async () => { await prisma.$disconnect(); });

test("empty book: both sides empty arrays", async () => {
  const snap = await svc.snapshot(MARKET, 10);
  expect(snap.marketPda).toBe(MARKET);
  expect(snap.bids).toEqual([]);
  expect(snap.asks).toEqual([]);
  expect(snap.timestamp).toBeGreaterThan(0);
});

test("aggregates quantity per price level, OPEN only", async () => {
  await orderRepo.create({ userId: "u1", marketPda: MARKET, side: "BUY", priceBps: 5000, feeBps: 0, quantity: 10n * UNIT, reservationId: "r1" });
  await orderRepo.create({ userId: "u2", marketPda: MARKET, side: "BUY", priceBps: 5000, feeBps: 0, quantity: 20n * UNIT, reservationId: "r2" });
  await orderRepo.create({ userId: "u3", marketPda: MARKET, side: "BUY", priceBps: 4500, feeBps: 0, quantity:  5n * UNIT, reservationId: "r3" });
  await orderRepo.create({ userId: "u4", marketPda: MARKET, side: "SELL", priceBps: 6000, feeBps: 0, quantity: 8n * UNIT, reservationId: "r4" });

  const snap = await svc.snapshot(MARKET, 10);
  // Bids: highest first → 5000 (30), 4500 (5)
  expect(snap.bids).toEqual([
    { priceBps: 5000, quantity: (30n * UNIT).toString() },
    { priceBps: 4500, quantity: ( 5n * UNIT).toString() },
  ]);
  // Asks: lowest first → 6000 (8)
  expect(snap.asks).toEqual([
    { priceBps: 6000, quantity: (8n * UNIT).toString() },
  ]);
});

test("respects depth limit", async () => {
  // Create 5 bid levels
  for (let i = 1; i <= 5; i++) {
    await orderRepo.create({ userId: `u${i}`, marketPda: MARKET, side: "BUY", priceBps: 3000 + i * 100, feeBps: 0, quantity: 1n * UNIT, reservationId: `r${i}` });
  }
  const snap = await svc.snapshot(MARKET, 3);
  expect(snap.bids).toHaveLength(3);
  // Top 3 highest prices: 3500, 3400, 3300
  expect(snap.bids.map(b => b.priceBps)).toEqual([3500, 3400, 3300]);
});

test("excludes FILLED and CANCELLED orders", async () => {
  await orderRepo.create({ userId: "u1", marketPda: MARKET, side: "BUY", priceBps: 5000, feeBps: 0, quantity: 10n * UNIT, reservationId: "r1" });
  await prisma.ob2Order.updateMany({ where: {}, data: { status: "CANCELLED" } });
  const snap = await svc.snapshot(MARKET, 10);
  expect(snap.bids).toEqual([]);
  expect(snap.asks).toEqual([]);
});
```

### Step 2.2: Run, fail.

### Step 2.3: Implement.

```typescript
// book-snapshot.service.ts
import type { PrismaClient } from "../../../generated/prisma/client";
import type { BookSnapshotEvent, BookLevel } from "../types/events.types";
import { toMicro } from "../types/decimal-helpers";

export class BookSnapshotService {
  constructor(private readonly prisma: PrismaClient) {}

  async snapshot(marketPda: string, depth: number): Promise<BookSnapshotEvent> {
    const rows = await this.prisma.ob2Order.findMany({
      where: { marketPda, status: "OPEN" },
      select: { side: true, priceBps: true, quantity: true, filled: true },
    });

    // Group by price/side, summing remaining (quantity - filled)
    const bidMap = new Map<number, bigint>();
    const askMap = new Map<number, bigint>();

    for (const r of rows) {
      const remaining = toMicro(r.quantity) - toMicro(r.filled);
      if (remaining <= 0n) continue;
      const target = r.side === "BUY" ? bidMap : askMap;
      target.set(r.priceBps, (target.get(r.priceBps) ?? 0n) + remaining);
    }

    const bids: BookLevel[] = Array.from(bidMap.entries())
      .sort(([a], [b]) => b - a)
      .slice(0, depth)
      .map(([priceBps, qty]) => ({ priceBps, quantity: qty.toString() }));

    const asks: BookLevel[] = Array.from(askMap.entries())
      .sort(([a], [b]) => a - b)
      .slice(0, depth)
      .map(([priceBps, qty]) => ({ priceBps, quantity: qty.toString() }));

    return {
      type: "book_snapshot",
      marketPda,
      bids,
      asks,
      timestamp: Date.now(),
    };
  }
}
```

### Step 2.4: Run, 4 tests pass.

### Step 2.5: Commit.

```bash
git add api/src/modules/trading-v2/services/book-snapshot.service.ts \
        api/src/modules/trading-v2/__tests__/book-snapshot.service.integration.test.ts
git commit -m "feat(trading-v2): BookSnapshotService (top-N bids/asks)"
```

---

## Task 3: Event emission hooks nos use-cases + services

**Files:**
- Modify: `api/src/modules/trading-v2/use-cases/place-order.use-case.ts`
- Modify: `api/src/modules/trading-v2/use-cases/cancel-order.use-case.ts`
- Modify: `api/src/modules/trading-v2/services/matching-engine.service.ts`
- Modify: `api/src/modules/trading-v2/services/solana-settler.service.ts`
- Modify: `api/src/modules/trading-v2/services/settlement-reverter.service.ts`
- Create: `api/src/modules/trading-v2/__tests__/event-emission.integration.test.ts`

Each of these components gets an **optional** `publisher: ITradingV2EventPublisher` injected via constructor config. When set, emits the appropriate event(s) AFTER the main DB commit — never inside.

### Step 3.1: Failing integration test (end-to-end event emission).

```typescript
import "dotenv/config";
import { prisma } from "../../../shared/database/config/prisma-client";
import { BalanceRepository } from "../repositories/balance.repository";
import { ReservationService } from "../services/reservation.service";
import { OrderRepository } from "../repositories/order.repository";
import { TradeRepository } from "../repositories/trade.repository";
import { StubSettlerService } from "../services/stub-settler.service";
import { SettlementReverter } from "../services/settlement-reverter.service";
import { MatchingEngine } from "../services/matching-engine.service";
import { IntentClassifier } from "../services/intent-classifier.service";
import { PlaceOrderUseCase } from "../use-cases/place-order.use-case";
import { CancelOrderUseCase } from "../use-cases/cancel-order.use-case";
import { InMemoryEventPublisher } from "../services/trading-v2-event-publisher";
import { UNIT } from "../types/balance.types";

const publisher = new InMemoryEventPublisher();
const balanceRepo = new BalanceRepository(prisma);
const resSvc = new ReservationService(prisma);
const orderRepo = new OrderRepository(prisma);
const tradeRepo = new TradeRepository(prisma);
const stub = new StubSettlerService(prisma, { publisher });
const reverter = new SettlementReverter(prisma, tradeRepo, orderRepo, { publisher });
const engine = new MatchingEngine(prisma, orderRepo, tradeRepo, stub, { publisher });
const classifier = new IntentClassifier();
const place = new PlaceOrderUseCase(prisma, balanceRepo, resSvc, orderRepo, engine, classifier, { publisher });
const cancel = new CancelOrderUseCase(prisma, orderRepo, resSvc, { publisher });

const A = "00000000-0000-0000-0000-000000000001";
const B = "00000000-0000-0000-0000-000000000002";
const MARKET = "Market1111111111111111111111111111111111111";

beforeEach(async () => {
  publisher.marketCalls.length = 0;
  publisher.userCalls.length = 0;
  await prisma.ob2Trade.deleteMany({});
  await prisma.ob2Order.deleteMany({});
  await prisma.ob2Reservation.deleteMany({});
  await prisma.ob2UserMarketBalance.deleteMany({});
});
afterAll(async () => { await prisma.$disconnect(); });

test("place-order OPEN: emits order_update to market + user", async () => {
  await balanceRepo.upsertOnchain(A, MARKET, "USDC", 1000n * UNIT, 1n);
  const { order } = await place.execute({
    userId: A, marketPda: MARKET, side: "BUY",
    priceBps: 6000, quantity: 100n * UNIT, feeBps: 0,
  });

  // At least one market emit + one user emit
  const orderEvents = publisher.marketCalls.filter(c => c.event.type === "order_update");
  expect(orderEvents.length).toBeGreaterThanOrEqual(1);
  expect(orderEvents[0].pda).toBe(MARKET);
  expect(orderEvents[0].event.type).toBe("order_update");
  const userOrderEvents = publisher.userCalls.filter(c => c.event.type === "order_update");
  expect(userOrderEvents.length).toBeGreaterThanOrEqual(1);
  expect(userOrderEvents[0].userId).toBe(A);
});

test("cancel-order: emits CANCELLED order_update", async () => {
  await balanceRepo.upsertOnchain(A, MARKET, "USDC", 1000n * UNIT, 1n);
  const { order } = await place.execute({
    userId: A, marketPda: MARKET, side: "BUY",
    priceBps: 6000, quantity: 100n * UNIT, feeBps: 0,
  });
  publisher.marketCalls.length = 0;
  publisher.userCalls.length = 0;

  await cancel.execute({ userId: A, orderId: order.id });

  const marketCancel = publisher.marketCalls.find(c => c.event.type === "order_update" && (c.event as any).status === "CANCELLED");
  expect(marketCancel).toBeDefined();
  const userCancel = publisher.userCalls.find(c => c.event.type === "order_update" && (c.event as any).status === "CANCELLED");
  expect(userCancel).toBeDefined();
});

test("match → settlement: emits trade_settled for both users + market", async () => {
  await balanceRepo.upsertOnchain(B, MARKET, "YES", 100n * UNIT, 1n);
  await place.execute({ userId: B, marketPda: MARKET, side: "SELL", priceBps: 5000, quantity: 100n * UNIT, feeBps: 0 });
  await balanceRepo.upsertOnchain(A, MARKET, "USDC", 1000n * UNIT, 1n);
  publisher.marketCalls.length = 0;
  publisher.userCalls.length = 0;

  await place.execute({ userId: A, marketPda: MARKET, side: "BUY", priceBps: 6000, quantity: 100n * UNIT, feeBps: 0 });

  const marketSettled = publisher.marketCalls.find(c => c.event.type === "trade_settled");
  expect(marketSettled).toBeDefined();
  const usersNotified = new Set(publisher.userCalls.filter(c => c.event.type === "trade_settled").map(c => c.userId));
  expect(usersNotified.has(A)).toBe(true);
  expect(usersNotified.has(B)).toBe(true);
});
```

### Step 3.2: Run, fail.

### Step 3.3: Implement emission hooks.

**For each of the 5 components**, add an optional publisher via config:

```typescript
// generic pattern
export interface XxxConfig {
  publisher?: ITradingV2EventPublisher;
  // ... other existing config fields
}

class XxxService {
  private readonly publisher: ITradingV2EventPublisher | null;

  constructor(/* existing */, config: XxxConfig = {}) {
    this.publisher = config.publisher ?? null;
  }
}
```

And emit events after DB commits.

**place-order.use-case.ts:** After `this.orders.getById(order.id)` and building result, BEFORE `return`:

```typescript
    if (this.publisher && finalOrder) {
      const event = orderToEvent(finalOrder);
      await Promise.all([
        this.publisher.publishMarket(finalOrder.marketPda, event),
        this.publisher.publishUser(finalOrder.userId, event),
      ]);
    }
```

Define a shared helper `orderToEvent(order: OrderView): OrderUpdateEvent` in a new file `api/src/modules/trading-v2/services/event-builders.ts`:

```typescript
import type { OrderView } from "../types/order.types";
import type { OrderUpdateEvent, TradeSettledEvent, TradeRevertedEvent } from "../types/events.types";
import type { TradeRecord } from "../types/trade.types";

export function orderToEvent(o: OrderView): OrderUpdateEvent {
  return {
    type: "order_update",
    marketPda: o.marketPda,
    userId: o.userId,
    orderId: o.id,
    side: o.side,
    priceBps: o.priceBps,
    quantity: o.quantity.toString(),
    filled: o.filled.toString(),
    status: o.status,
    clientOrderId: o.clientOrderId,
    timestamp: Date.now(),
  };
}

export function tradeToSettledEvent(t: TradeRecord): TradeSettledEvent {
  return {
    type: "trade_settled",
    marketPda: t.marketPda,
    tradeId: t.id,
    priceBps: t.priceBps,
    quantity: t.quantity.toString(),
    primitive: t.primitive,
    makerOrderId: t.makerOrderId,
    takerOrderId: t.takerOrderId,
    txSignature: t.txSignature,
    timestamp: Date.now(),
  };
}

export function tradeToRevertedEvent(t: TradeRecord, reason: string): TradeRevertedEvent {
  return {
    type: "trade_reverted",
    marketPda: t.marketPda,
    tradeId: t.id,
    reason,
    timestamp: Date.now(),
  };
}
```

**cancel-order.use-case.ts:** After `this.orders.markCancelled(...)` returns `result`, emit `orderToEvent(result)` to market + user.

**matching-engine.service.ts:** After each successful fill round (when `trade` is created and settler called), emit:
- `orderToEvent(updatedMakerOrder)` to market + maker's user
- `orderToEvent(updatedTakerOrder)` to market + taker's user

Load the updated orders fresh via `orderRepo.getById` since they may have changed status to FILLED.

**solana-settler.service.ts:** Inside `if (res.ok)` block, AFTER the SETTLED commit (and AFTER `emitFeeEvents`), emit:
- `tradeToSettledEvent(trade)` to market
- `tradeToSettledEvent(trade)` to BOTH users (buyer + seller — look up via orders)

Fetch the fresh trade via `this.trades.getById(tradeId)` so `txSignature` is populated.

**settlement-reverter.service.ts:** After `tx.ob2Trade.update({ ... status: REVERTED })`, emit:
- `tradeToRevertedEvent(trade, reason)` to market
- Same to both users (fetch user IDs from the orders).

### Step 3.4: Run all tests (new + existing).

Expected: 132 baseline + new events test count. Existing tests don't pass `publisher` config → the null guard skips emission → no regression. New test passes.

### Step 3.5: Commit.

```bash
cd /Users/rafael/Documents/GitHub/Oraculo/.claude/worktrees/orderbook-rewrite-07-websocket-v2
git add api/src/modules/trading-v2/services/event-builders.ts \
        api/src/modules/trading-v2/use-cases/ \
        api/src/modules/trading-v2/services/matching-engine.service.ts \
        api/src/modules/trading-v2/services/solana-settler.service.ts \
        api/src/modules/trading-v2/services/settlement-reverter.service.ts \
        api/src/modules/trading-v2/__tests__/event-emission.integration.test.ts
git commit -m "feat(trading-v2): event emission em place-order/cancel/match/settle/revert"
```

---

## Task 4: WsTradingV2Handler (WebSocket connection manager)

**Files:**
- Create: `api/src/modules/trading-v2/services/ws-trading-v2-handler.service.ts`
- Create: `api/src/modules/trading-v2/__tests__/ws-trading-v2-handler.unit.test.ts`

Handler para o `Bun.serve({ websocket: ... })`. Mantém Map de `ws → Set<subscription>`, gerencia subscribe/unsubscribe, repassa eventos Redis pros clients. **Stateless**: se o processo cai, reconexão recria todo estado.

### Step 4.1: Failing unit test (com fakes em memória, sem Redis).

```typescript
import { WsTradingV2Handler } from "../services/ws-trading-v2-handler.service";
import type { OrderUpdateEvent } from "../types/events.types";

// Fake ServerWebSocket — captura sends, simula close.
type FakeWs = {
  sent: string[];
  closed: boolean;
  send(data: string): void;
  close(): void;
};
const mkWs = (): FakeWs => ({
  sent: [], closed: false,
  send(data) { this.sent.push(data); },
  close() { this.closed = true; },
});

// Fake Redis subscriber — manual trigger via publishTo.
class FakeRedisSub {
  listeners = new Map<string, Array<(ch: string, msg: string) => void>>();
  subscribed = new Set<string>();
  async subscribe(channel: string) { this.subscribed.add(channel); }
  async unsubscribe(channel: string) { this.subscribed.delete(channel); }
  on(event: string, cb: (ch: string, msg: string) => void) {
    if (event === "message") {
      const list = this.listeners.get("message") ?? [];
      list.push(cb);
      this.listeners.set("message", list);
    }
  }
  /** Test helper: simulates an incoming Redis message. */
  async __publishTo(channel: string, msg: string) {
    const cbs = this.listeners.get("message") ?? [];
    for (const cb of cbs) await cb(channel, msg);
  }
}

const fakeSub = () => new FakeRedisSub() as any;

test("subscribe command: subscribes client to market channel + calls redis.subscribe", async () => {
  const sub = new FakeRedisSub();
  const handler = new WsTradingV2Handler({ redisSubscriber: sub as any });
  const ws = mkWs();
  await handler.onOpen(ws as any);
  await handler.onMessage(ws as any, JSON.stringify({ op: "subscribe", kind: "market", target: "Market1" }));

  expect(sub.subscribed.has("ob2:market:Market1")).toBe(true);
  // Ack sent
  const ack = JSON.parse(ws.sent[0]);
  expect(ack.op).toBe("subscribed");
  expect(ack.kind).toBe("market");
  expect(ack.target).toBe("Market1");
});

test("Redis message on subscribed channel is fan-out to client", async () => {
  const sub = new FakeRedisSub();
  const handler = new WsTradingV2Handler({ redisSubscriber: sub as any });
  const ws = mkWs();
  await handler.onOpen(ws as any);
  await handler.onMessage(ws as any, JSON.stringify({ op: "subscribe", kind: "market", target: "Market1" }));
  ws.sent.length = 0;   // clear ack

  const event: OrderUpdateEvent = {
    type: "order_update", marketPda: "Market1", userId: "u1", orderId: "o1",
    side: "BUY", priceBps: 6000, quantity: "100", filled: "0",
    status: "OPEN", clientOrderId: null, timestamp: 1,
  };
  await sub.__publishTo("ob2:market:Market1", JSON.stringify(event));

  expect(ws.sent).toHaveLength(1);
  expect(JSON.parse(ws.sent[0])).toEqual(event);
});

test("unsubscribe removes client from channel; no more fan-out", async () => {
  const sub = new FakeRedisSub();
  const handler = new WsTradingV2Handler({ redisSubscriber: sub as any });
  const ws = mkWs();
  await handler.onOpen(ws as any);
  await handler.onMessage(ws as any, JSON.stringify({ op: "subscribe", kind: "market", target: "Market1" }));
  ws.sent.length = 0;

  await handler.onMessage(ws as any, JSON.stringify({ op: "unsubscribe", kind: "market", target: "Market1" }));
  await sub.__publishTo("ob2:market:Market1", JSON.stringify({ type: "order_update" }));

  // No message after unsubscribe (except the unsub ack)
  const events = ws.sent.filter(s => {
    const p = JSON.parse(s);
    return p.type === "order_update";
  });
  expect(events).toHaveLength(0);
});

test("onClose removes client from all subscriptions (no redis.unsubscribe if other clients still want it)", async () => {
  const sub = new FakeRedisSub();
  const handler = new WsTradingV2Handler({ redisSubscriber: sub as any });
  const ws1 = mkWs();
  const ws2 = mkWs();
  await handler.onOpen(ws1 as any);
  await handler.onOpen(ws2 as any);
  await handler.onMessage(ws1 as any, JSON.stringify({ op: "subscribe", kind: "market", target: "Market1" }));
  await handler.onMessage(ws2 as any, JSON.stringify({ op: "subscribe", kind: "market", target: "Market1" }));

  await handler.onClose(ws1 as any);
  // ws2 still connected — don't unsubscribe from Redis
  expect(sub.subscribed.has("ob2:market:Market1")).toBe(true);

  await handler.onClose(ws2 as any);
  // Last client gone — unsubscribe
  expect(sub.subscribed.has("ob2:market:Market1")).toBe(false);
});

test("malformed command JSON: sends error ack, doesn't crash", async () => {
  const sub = new FakeRedisSub();
  const handler = new WsTradingV2Handler({ redisSubscriber: sub as any });
  const ws = mkWs();
  await handler.onOpen(ws as any);
  await handler.onMessage(ws as any, "not json");

  const ack = JSON.parse(ws.sent[0]);
  expect(ack.op).toBe("error");
});
```

### Step 4.2: Run, fail.

### Step 4.3: Implement.

```typescript
// ws-trading-v2-handler.service.ts
import type { Redis } from "ioredis";
import { channelMarket, channelUser } from "./trading-v2-event-publisher";

export interface WsClient {
  send(data: string): void;
  close(): void;
}

export interface WsHandlerConfig {
  redisSubscriber: Redis;
}

interface Command {
  op: "subscribe" | "unsubscribe";
  kind: "market" | "user";
  target: string;
}

export class WsTradingV2Handler {
  private readonly redis: Redis;
  // client → set of channels it's subscribed to
  private clientSubs = new Map<WsClient, Set<string>>();
  // channel → set of clients subscribed
  private channelClients = new Map<string, Set<WsClient>>();

  constructor(config: WsHandlerConfig) {
    this.redis = config.redisSubscriber;
    this.redis.on("message", (channel: string, msg: string) => {
      this.fanOut(channel, msg);
    });
  }

  async onOpen(ws: WsClient): Promise<void> {
    this.clientSubs.set(ws, new Set());
  }

  async onMessage(ws: WsClient, raw: string): Promise<void> {
    let cmd: Command;
    try {
      cmd = JSON.parse(raw);
    } catch {
      ws.send(JSON.stringify({ op: "error", reason: "invalid_json" }));
      return;
    }
    if (cmd.op === "subscribe") {
      await this.subscribe(ws, cmd.kind, cmd.target);
      ws.send(JSON.stringify({ op: "subscribed", kind: cmd.kind, target: cmd.target }));
    } else if (cmd.op === "unsubscribe") {
      await this.unsubscribe(ws, cmd.kind, cmd.target);
      ws.send(JSON.stringify({ op: "unsubscribed", kind: cmd.kind, target: cmd.target }));
    } else {
      ws.send(JSON.stringify({ op: "error", reason: "unknown_op" }));
    }
  }

  async onClose(ws: WsClient): Promise<void> {
    const channels = this.clientSubs.get(ws);
    if (!channels) return;
    for (const ch of channels) {
      const clients = this.channelClients.get(ch);
      if (!clients) continue;
      clients.delete(ws);
      if (clients.size === 0) {
        this.channelClients.delete(ch);
        await this.redis.unsubscribe(ch);
      }
    }
    this.clientSubs.delete(ws);
  }

  private async subscribe(ws: WsClient, kind: "market" | "user", target: string): Promise<void> {
    const channel = kind === "market" ? channelMarket(target) : channelUser(target);
    const subs = this.clientSubs.get(ws) ?? new Set();
    if (subs.has(channel)) return;
    subs.add(channel);
    this.clientSubs.set(ws, subs);

    const clients = this.channelClients.get(channel) ?? new Set();
    const firstClient = clients.size === 0;
    clients.add(ws);
    this.channelClients.set(channel, clients);

    if (firstClient) await this.redis.subscribe(channel);
  }

  private async unsubscribe(ws: WsClient, kind: "market" | "user", target: string): Promise<void> {
    const channel = kind === "market" ? channelMarket(target) : channelUser(target);
    const subs = this.clientSubs.get(ws);
    if (!subs || !subs.has(channel)) return;
    subs.delete(channel);

    const clients = this.channelClients.get(channel);
    if (!clients) return;
    clients.delete(ws);
    if (clients.size === 0) {
      this.channelClients.delete(channel);
      await this.redis.unsubscribe(channel);
    }
  }

  private fanOut(channel: string, msg: string): void {
    const clients = this.channelClients.get(channel);
    if (!clients) return;
    for (const ws of clients) {
      try { ws.send(msg); }
      catch { /* client disconnected mid-send; cleanup happens on onClose */ }
    }
  }
}
```

### Step 4.4: Run, 5 tests pass.

### Step 4.5: Commit.

```bash
git add api/src/modules/trading-v2/services/ws-trading-v2-handler.service.ts \
        api/src/modules/trading-v2/__tests__/ws-trading-v2-handler.unit.test.ts
git commit -m "feat(trading-v2): WsTradingV2Handler (connection manager + Redis fan-out)"
```

---

## Task 5: WS route mount + routes wiring

**Files:**
- Create: `api/src/modules/trading-v2/routes/ws.routes.ts`
- Modify: `api/src/modules/trading-v2/routes/orders.routes.ts` (injetar publisher nos services)
- Modify: `api/index.ts` (mount WS endpoint)

### Step 5.1: Create `ws.routes.ts`.

```typescript
/**
 * WebSocket endpoint setup pro trading-v2.
 *
 * Bun ServerWebSocket handler factory. Mountado em `api/index.ts` via
 * `Bun.serve({ websocket: { ... } })`.
 *
 * Protocolo client:
 *   → { op: "subscribe",   kind: "market"|"user", target: string }
 *   → { op: "unsubscribe", kind: "market"|"user", target: string }
 *
 * Server ack:
 *   ← { op: "subscribed"|"unsubscribed", kind, target }
 *   ← { op: "error", reason: string }
 *
 * Event push (qualquer momento após subscribe):
 *   ← OrderUpdateEvent | TradeSettledEvent | TradeRevertedEvent | BookSnapshotEvent
 */
import type { ServerWebSocket } from "bun";
import { Redis } from "ioredis";
import { WsTradingV2Handler } from "../services/ws-trading-v2-handler.service";

let handler: WsTradingV2Handler | null = null;

export function getOrCreateHandler(redisUrl: string): WsTradingV2Handler {
  if (handler) return handler;
  const sub = new Redis(redisUrl);
  handler = new WsTradingV2Handler({ redisSubscriber: sub });
  return handler;
}

/** Bun ServerWebSocket handler config. Mount em Bun.serve({ websocket }). */
export function buildWebSocketHandlers(redisUrl: string) {
  return {
    open(ws: ServerWebSocket) {
      void getOrCreateHandler(redisUrl).onOpen(ws as any);
    },
    message(ws: ServerWebSocket, raw: string | Buffer) {
      const text = typeof raw === "string" ? raw : raw.toString("utf8");
      void getOrCreateHandler(redisUrl).onMessage(ws as any, text);
    },
    close(ws: ServerWebSocket) {
      void getOrCreateHandler(redisUrl).onClose(ws as any);
    },
  };
}
```

### Step 5.2: Modify `orders.routes.ts` to inject RedisEventPublisher.

Find the wiring block. Add:

```typescript
import { redis } from "@/shared/database/config/redis-config";
import { RedisEventPublisher } from "../services/trading-v2-event-publisher";

const publisher = new RedisEventPublisher(redis);
```

Then thread `{ publisher }` into every service that accepts it:
- `new StubSettlerService(prisma, { publisher })`
- `new SettlementReverter(prisma, tradeRepo, orderRepo, { publisher })`
- `new MatchingEngine(prisma, orderRepo, tradeRepo, settler, { publisher })`
- `new SolanaSettlerService(prisma, tradeRepo, stub, reverter, caller, { eventRepo, userWalletLookup: ..., publisher })`
- `new PlaceOrderUseCase(prisma, balanceRepo, resSvc, orderRepo, engine, classifier, { publisher })`
- `new CancelOrderUseCase(prisma, orderRepo, resSvc, { publisher })`

**IMPORTANT:** the first arg and positional args of existing constructors must match what's already there. Look at each class's constructor to place `{ publisher }` in the right config slot. Several services already take a trailing config object (Plano 3-6); extend it with publisher.

### Step 5.3: Mount WS in `api/index.ts`.

Grep for the existing `Bun.serve({ ... })` or equivalent. If the project uses Hono with `upgradeWebSocket`, use that pattern. If raw Bun.serve, do:

```typescript
import { buildWebSocketHandlers } from "@/modules/trading-v2/routes/ws.routes";

// Near existing Bun.serve:
const wsHandlers = buildWebSocketHandlers(process.env.REDIS_URL ?? "redis://localhost:6379");

Bun.serve({
  // ... existing config ...
  fetch(req, server) {
    if (new URL(req.url).pathname === "/api/v2/trading/ws") {
      if (server.upgrade(req)) return;
      return new Response("Upgrade failed", { status: 500 });
    }
    // ... existing fetch logic ...
  },
  websocket: wsHandlers,
});
```

**If the project already has a different WS mount pattern** (legacy socket.io, etc.), adapt to match — don't override existing WS. Report the actual mount pattern used.

### Step 5.4: Smoke test.

```bash
cd /Users/rafael/Documents/GitHub/Oraculo/.claude/worktrees/orderbook-rewrite-07-websocket-v2/api
PORT=3999 bun run index.ts > /tmp/smoke.log 2>&1 &
SERVER_PID=$!
sleep 3
# Try WebSocket upgrade — expect 101 Switching Protocols if wired, else 404/500
curl -s -o /dev/null -w "%{http_code}" -i -N -H "Connection: Upgrade" -H "Upgrade: websocket" -H "Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==" -H "Sec-WebSocket-Version: 13" http://localhost:3999/api/v2/trading/ws
kill $SERVER_PID 2>/dev/null
```

Expected: `101` (upgrade success) or `400` (validation error). NOT `404`.

### Step 5.5: Full trading-v2 suite still green.

```bash
bun x jest src/modules/trading-v2 --runInBand
```

Expected: baseline + new tests. Should be 132 baseline + ~13 new = ~145 passed.

### Step 5.6: Commit.

```bash
cd /Users/rafael/Documents/GitHub/Oraculo/.claude/worktrees/orderbook-rewrite-07-websocket-v2
git add api/src/modules/trading-v2/routes/ws.routes.ts \
        api/src/modules/trading-v2/routes/orders.routes.ts \
        api/index.ts
git commit -m "feat(trading-v2): WS route /api/v2/trading/ws + wire RedisEventPublisher"
```

---

## Task 6: Barrel + README + final suite

**Files:**
- Modify: `api/src/modules/trading-v2/index.ts`
- Modify: `api/src/modules/trading-v2/README.md`

### Step 6.1: Update barrel.

```typescript
// Append:
export * from "./types/events.types";
export {
  RedisEventPublisher, InMemoryEventPublisher,
  channelMarket, channelUser,
} from "./services/trading-v2-event-publisher";
export type { ITradingV2EventPublisher } from "./services/trading-v2-event-publisher";
export { BookSnapshotService } from "./services/book-snapshot.service";
export { WsTradingV2Handler } from "./services/ws-trading-v2-handler.service";
export { orderToEvent, tradeToSettledEvent, tradeToRevertedEvent } from "./services/event-builders";
```

### Step 6.2: Append `README.md` section after "Fee ledger":

```markdown
## WebSocket v2 (Plano 7)

Stateless WS em `/api/v2/trading/ws`. Multi-instance via Redis Pub/Sub.

### Protocolo client

```json
// Subscribe
{ "op": "subscribe",   "kind": "market" | "user", "target": "<pda or userId>" }
{ "op": "unsubscribe", "kind": "market" | "user", "target": "<pda or userId>" }
```

### Eventos recebidos

- `order_update` — criação, fill parcial, FILLED, CANCELLED, REJECTED.
- `trade_settled` — emitido post-SETTLED (buyer + seller + market).
- `trade_reverted` — emitido post-REVERTED.
- `book_snapshot` — sob demanda via HTTP `/api/v2/trading/book?marketPda=...` (follow-up).

### Canais Redis

- `ob2:market:<pda>` — todos os eventos desse mercado.
- `ob2:user:<userId>` — eventos específicos desse user.

### Guarantees

- **Live only**: sem replay. Client faz catch-up via HTTP REST antes/depois.
- **At-most-once delivery**: Redis Pub/Sub. Se client reconectar após downtime, perde eventos do intervalo.
- **No auth enforcement no Plano 7**: client pode sub a qualquer channel. Autenticação/autorização via middleware Clerk fica pro follow-up.
```

### Step 6.3: Full suite.

```bash
cd /Users/rafael/Documents/GitHub/Oraculo/.claude/worktrees/orderbook-rewrite-07-websocket-v2/api
bun x jest src/modules/trading-v2 --runInBand
```

Expected: ~145 passed + 1 skipped. Report actual breakdown.

### Step 6.4: tsc clean.

```bash
bun x tsc --noEmit 2>&1 | grep trading-v2 || echo "clean"
```

### Step 6.5: Commit.

```bash
git add api/src/modules/trading-v2/index.ts api/src/modules/trading-v2/README.md
git commit -m "docs(trading-v2): README WebSocket v2 + barrel"
```

---

## Critérios de aceitação

1. ✅ `ITradingV2EventPublisher` + Redis/InMemory impls + canais `ob2:market:<pda>` / `ob2:user:<userId>`.
2. ✅ `BookSnapshotService` agrega por price/side, respecting depth limit, excludes FILLED/CANCELLED.
3. ✅ Eventos emitidos em: place-order OPEN, cancel-order CANCELLED, match (partial/full fill), settler SETTLED, reverter REVERTED.
4. ✅ `WsTradingV2Handler` gerencia subscribe/unsubscribe + fan-out + dedup de Redis subscriptions (multi-client pra mesmo channel → single Redis sub).
5. ✅ Rota `/api/v2/trading/ws` montada via Bun.serve; smoke test retorna 101/400 (não 404).
6. ✅ Eventos emitidos AFTER DB commit — se publisher falha, DB state NÃO é afetado (publisher não é blocking).
7. ✅ Suite ≥ 145 passed + 1 skipped; tsc clean.

---

## Riscos e mitigações

| Risco | Mitigação |
|---|---|
| Publisher joga exception e afeta tx do settler/place-order | Wrapper `try/catch` silent em cada emit — publisher é audit/UX, não crítico. Alternativa: chamar emits DEPOIS do return (já fora da tx). |
| Redis cai — publish falha silenciosamente | Clients perdem eventos até Redis voltar. Aceitável pra audit/UX. Para reconciliação, client deve catch-up via HTTP REST. |
| Event emission adiciona latência mensurável ao place-order | Emits são fire-and-forget Redis PUBLISH (~sub-ms). Se notável em prod, mover pro próximo tick via `setImmediate`. |
| Multiple client instances do mesmo user geram eventos duplicados | Não — emit é feito pelo server-side do place-order/settler, não pelo WS. WS só ouve e repassa. Cada user event fica uma única vez no canal. |
| Infinite loop se event handling acionar novo event | Garantido que handlers não mutam DB — são só fan-out pra sockets. Sem loop possível. |
| WS upgrade handler não disponível em todos os HTTP frameworks | Task 5.3 define pattern pra Bun.serve nativo. Se o projeto usa Hono upgrade-websocket pattern em vez disso, adaptar. |

---

## O que NÃO está neste plano

- **Auth/authz na WS**: fica pro follow-up (Clerk middleware no upgrade).
- **Historical replay / catch-up**: clients responsáveis por fetch HTTP antes de conectar.
- **Rate limiting por client**: infra externa.
- **Per-connection backpressure**: se client não consome, sends silentemente dropados. Buffering pode vir depois.
- **Dashboard-wide broadcast (legacy)**: fora do trading-v2. Legacy continua servindo isso.
