# Orderbook Rewrite — Plano 2: Orders + Matching Engine

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Entregar orders lifecycle completo (place, cancel, ciclo de estados) + matching engine síncrono (taker path) no módulo `trading-v2`, testável isoladamente com um **stub de settlement** que apenas move estado no DB. Ao final, ordens casam, trades são criados, saldos fluem corretamente entre lados, e as invariantes I1/I2 continuam verdadeiras.

**Architecture:** Schema estende `ob2_*` com `orders`, `trades`, `onchain_events_processed`. `MatchingEngine.tryMatch` usa `SELECT ... FOR UPDATE SKIP LOCKED` pro price-time priority. Decisão de primitiva (TRADE/MINT/MERGE) derivada do `reservations.asset` dos dois lados (spec §6.2 — tabela de decisão). `StubSettlerService` faz o trabalho de settlement localmente no DB (consume reserves, credit assets no lado oposto) — no Plano 3 será substituído pelo caller on-chain real sem mudar a interface.

**Tech Stack:** Bun, TypeScript, Hono + zod-validator, Prisma 7, Jest.

**Spec:** `docs/superpowers/specs/2026-04-15-orderbook-rewrite-design.md`
**Plano anterior:** `docs/superpowers/plans/2026-04-15-orderbook-rewrite-01-foundation.md` (entregue, merge em `c891f0b`)
**Próximos planos:**
- Plano 3: Settlement sync on-chain + async maker + revert + listener — substitui `StubSettler` pelo caller real
- Plano 4: Instrução `settle_fill` no programa Solana
- Plano 5: WebSocket v2 • Plano 6: MM bot externo • Plano 7: Cutover

---

## File Structure

**Novos arquivos (todos em `api/src/modules/trading-v2/`):**

```
api/
  prisma/
    schema.prisma                          # MODIFY: adiciona Ob2Order, Ob2Trade, Ob2OnchainEvent
    scripts/
      trading-v2-orders.sql                # CREATE: CHECKs + índice parcial do book
  src/modules/trading-v2/
    types/
      order.types.ts                       # PlaceOrderInput, OrderView, OrderStatus
      trade.types.ts                       # TradeRecord, MatchResult, primitive types
    repositories/
      order.repository.ts                  # CRUD + findBestMatch
      trade.repository.ts                  # create + query
    services/
      primitive-decider.service.ts         # pure function: (maker.asset, taker.asset) → primitive
      stub-settler.service.ts              # Plan 2 only — replaced in Plan 3
      matching-engine.service.ts           # tryMatch com FOR UPDATE SKIP LOCKED
    use-cases/
      place-order.use-case.ts              # classify → reserve → create order → match
      cancel-order.use-case.ts             # validate → release → mark cancelled
    routes/
      orders.routes.ts                     # POST /v2/orders, DELETE /v2/orders/:id, GET /v2/orders
    schemas/
      place-order.schema.ts                # zod validator
    __tests__/
      order.repository.integration.test.ts
      trade.repository.integration.test.ts
      primitive-decider.unit.test.ts
      matching-engine.integration.test.ts
      place-order.use-case.integration.test.ts
      cancel-order.use-case.integration.test.ts
      matching-scenarios.e2e.test.ts       # cenários multi-user dos 3 primitives
```

**Responsabilidades:**

- `types/`: pure TS (já estabelecido no Plano 1).
- `repositories/`: SQL puro via Prisma. Sem lógica.
- `services/primitive-decider`: função pura — recebe assets das duas reservations, devolve primitiva. Testável em isolamento.
- `services/stub-settler`: Plano 2 only. Faz o que o settlement real fará: consome reservations, credita saldos do lado oposto, marca trade SETTLED. **Não chama Solana.** Plano 3 troca por implementação real mantendo a mesma interface `ISettler`.
- `services/matching-engine`: `tryMatch(takerOrder)` roda loop com FOR UPDATE SKIP LOCKED, cria trades, chama settler, atualiza ordens.
- `use-cases/`: orquestram services. Ponto de entrada das rotas.
- `routes/`: HTTP thin.

---

## Prerequisite check

- [ ] **Step 0: Confirmar estado pós-Plano 1**

```bash
cd /Users/rafael/Documents/GitHub/Oraculo/api
bun x jest src/modules/trading-v2 --runInBand
```

Expected: `Tests: 27 passed, 27 total`. Se não, pare e resolva antes de prosseguir.

- [ ] **Step 0.1: Worktree**

```bash
# Fora do worktree. Usando Claude Code:
# EnterWorktree name="orderbook-rewrite-02-orders-matching"
# Ou manualmente:
git worktree add .claude/worktrees/orderbook-rewrite-02-orders-matching -b worktree-orderbook-rewrite-02-orders-matching
cp api/.env .claude/worktrees/orderbook-rewrite-02-orders-matching/api/.env
```

- [ ] **Step 0.2: Deps no worktree**

```bash
cd .claude/worktrees/orderbook-rewrite-02-orders-matching/api
bun install
bun x prisma generate
bun x jest src/modules/trading-v2 --runInBand  # confirma 27 passed no worktree
```

---

## Task 1: Schema de orders + trades + onchain_events_processed

**Files:**
- Modify: `api/prisma/schema.prisma` (append)
- Create: `api/prisma/scripts/trading-v2-orders.sql`

- [ ] **Step 1.1: Append ao schema.prisma (depois dos models Ob2 existentes):**

```prisma
model Ob2Order {
  id            String          @id @default(uuid())
  userId        String          @map("user_id")
  marketPda     String          @map("market_pda")
  side          Ob2Side
  priceBps      Int             @map("price_bps")          // 1..9999
  quantity      Decimal         @db.Decimal(20, 6)          // YES-normalized micro-units as Decimal
  filled        Decimal         @default(0) @db.Decimal(20, 6)
  status        Ob2OrderStatus
  rejectReason  String?         @map("reject_reason")
  clientOrderId String?         @map("client_order_id")
  reservationId String?         @map("reservation_id")      // FK lógica -> ob2_reservations.id
  createdAt     DateTime        @default(now()) @map("created_at")
  updatedAt     DateTime        @default(now()) @updatedAt @map("updated_at")
  closedAt      DateTime?       @map("closed_at")

  @@unique([userId, clientOrderId], map: "ob2_orders_user_client_idx")
  @@map("ob2_orders")
}

model Ob2Trade {
  id               String           @id @default(uuid())
  marketPda        String           @map("market_pda")
  makerOrderId     String           @map("maker_order_id")
  takerOrderId     String           @map("taker_order_id")
  priceBps         Int              @map("price_bps")
  quantity         Decimal          @db.Decimal(20, 6)
  primitive        Ob2Primitive
  status           Ob2TradeStatus
  sync             Boolean
  settlingDeadline DateTime         @map("settling_deadline")
  txSignature      String?          @unique @map("tx_signature")
  revertReason     String?          @map("revert_reason")
  createdAt        DateTime         @default(now()) @map("created_at")
  settledAt        DateTime?        @map("settled_at")

  @@map("ob2_trades")
}

model Ob2OnchainEvent {
  signature        String
  instructionIndex Int       @map("instruction_index")
  kind             String
  tradeId          String?   @map("trade_id")
  processedAt      DateTime  @default(now()) @map("processed_at")

  @@id([signature, instructionIndex])
  @@map("ob2_onchain_events_processed")
}
```

- [ ] **Step 1.2: `api/prisma/scripts/trading-v2-orders.sql`:**

```sql
-- Idempotente. Rodar APÓS db push.

DO $$ BEGIN
  ALTER TABLE ob2_orders ADD CONSTRAINT ob2_orders_qty_positive CHECK (quantity > 0);
EXCEPTION WHEN duplicate_object THEN NULL; END $$;

DO $$ BEGIN
  ALTER TABLE ob2_orders ADD CONSTRAINT ob2_orders_filled_within_qty CHECK (filled >= 0 AND filled <= quantity);
EXCEPTION WHEN duplicate_object THEN NULL; END $$;

DO $$ BEGIN
  ALTER TABLE ob2_orders ADD CONSTRAINT ob2_orders_price_range CHECK (price_bps > 0 AND price_bps < 10000);
EXCEPTION WHEN duplicate_object THEN NULL; END $$;

DO $$ BEGIN
  ALTER TABLE ob2_trades ADD CONSTRAINT ob2_trades_qty_positive CHECK (quantity > 0);
EXCEPTION WHEN duplicate_object THEN NULL; END $$;

DO $$ BEGIN
  ALTER TABLE ob2_trades ADD CONSTRAINT ob2_trades_price_range CHECK (price_bps > 0 AND price_bps < 10000);
EXCEPTION WHEN duplicate_object THEN NULL; END $$;

-- Book index: busca do melhor preço por lado, só OPEN
CREATE INDEX IF NOT EXISTS ob2_orders_book_idx
  ON ob2_orders (market_pda, side, price_bps, created_at)
  WHERE status = 'OPEN';

-- User orders query
CREATE INDEX IF NOT EXISTS ob2_orders_user_idx
  ON ob2_orders (user_id, market_pda, created_at DESC);

-- Trade lookup por status (para worker de deadline)
CREATE INDEX IF NOT EXISTS ob2_trades_settling_idx
  ON ob2_trades (status, settling_deadline)
  WHERE status = 'SETTLING';

CREATE INDEX IF NOT EXISTS ob2_trades_market_idx
  ON ob2_trades (market_pda, created_at DESC);
```

- [ ] **Step 1.3: Aplicar:**

```bash
cd /Users/rafael/Documents/GitHub/Oraculo/.claude/worktrees/orderbook-rewrite-02-orders-matching/api
bun x prisma db push
bun x prisma generate
DATABASE_URL_PSQL=$(grep "^DATABASE_URL=" .env | sed 's/DATABASE_URL=//; s/"//g')
psql "$DATABASE_URL_PSQL" -f prisma/scripts/trading-v2-orders.sql
```

- [ ] **Step 1.4: Verificar:**

```bash
psql "$DATABASE_URL_PSQL" -c "SELECT tablename FROM pg_tables WHERE tablename LIKE 'ob2_%' ORDER BY tablename;"
psql "$DATABASE_URL_PSQL" -c "SELECT indexname FROM pg_indexes WHERE tablename LIKE 'ob2_%' ORDER BY indexname;"
```

Expected: 5 tables (balances, reservations, orders, trades, onchain_events_processed); índices `ob2_orders_book_idx`, `ob2_orders_user_idx`, `ob2_trades_settling_idx`, `ob2_trades_market_idx` presentes.

- [ ] **Step 1.5: Commit:**

```bash
cd /Users/rafael/Documents/GitHub/Oraculo/.claude/worktrees/orderbook-rewrite-02-orders-matching
git add api/prisma/schema.prisma api/prisma/scripts/trading-v2-orders.sql
git commit -m "feat(trading-v2): schema de orders + trades + onchain events"
```

---

## Task 2: Types — order + trade

**Files:**
- Create: `api/src/modules/trading-v2/types/order.types.ts`
- Create: `api/src/modules/trading-v2/types/trade.types.ts`
- Modify: `api/src/modules/trading-v2/index.ts`

- [ ] **Step 2.1: `types/order.types.ts`:**

```typescript
import type { Ob2OrderStatus, Ob2Side } from "../../../generated/prisma/client";

export { Ob2OrderStatus, Ob2Side };

/**
 * Entrada HTTP de place-order. Side é sempre YES-normalizado pelo caller.
 */
export interface PlaceOrderInput {
  userId: string;
  marketPda: string;
  side: Ob2Side;              // "BUY" | "SELL" (YES-normalized)
  priceBps: number;           // 1..9999
  quantity: bigint;           // micro-units YES
  clientOrderId?: string;     // idempotência
  feeBps: number;             // carregado do market config pelo caller
}

export interface OrderView {
  id: string;
  userId: string;
  marketPda: string;
  side: Ob2Side;
  priceBps: number;
  quantity: bigint;
  filled: bigint;
  status: Ob2OrderStatus;
  rejectReason: string | null;
  clientOrderId: string | null;
  reservationId: string | null;
  createdAt: Date;
  updatedAt: Date;
  closedAt: Date | null;
}

export class OrderRejectedError extends Error {
  constructor(public readonly code: string, message: string) {
    super(message);
    this.name = "OrderRejectedError";
  }
}

export class OrderNotFoundError extends Error {
  constructor(public readonly orderId: string) {
    super(`order ${orderId} not found`);
    this.name = "OrderNotFoundError";
  }
}

export class OrderNotCancellableError extends Error {
  constructor(public readonly status: Ob2OrderStatus) {
    super(`order in status ${status} cannot be cancelled`);
    this.name = "OrderNotCancellableError";
  }
}
```

- [ ] **Step 2.2: `types/trade.types.ts`:**

```typescript
import type { Ob2Primitive, Ob2TradeStatus } from "../../../generated/prisma/client";

export { Ob2Primitive, Ob2TradeStatus };

export interface TradeRecord {
  id: string;
  marketPda: string;
  makerOrderId: string;
  takerOrderId: string;
  priceBps: number;
  quantity: bigint;
  primitive: Ob2Primitive;
  status: Ob2TradeStatus;
  sync: boolean;
  settlingDeadline: Date;
  txSignature: string | null;
  revertReason: string | null;
  createdAt: Date;
  settledAt: Date | null;
}

/**
 * Resultado de um match round. Uma taker order pode gerar múltiplos fills
 * contra diferentes makers.
 */
export interface MatchResult {
  trades: TradeRecord[];
  takerFilled: bigint;       // total preenchido do taker nessa chamada
  takerRemaining: bigint;    // quanto sobrou (quantity - filled)
  takerStatus: "FILLED" | "OPEN";
}

/** Contract between MatchingEngine and the settler (stub now, real in Plan 3). */
export interface ISettler {
  /**
   * Executa o settlement de um trade recém-criado. Em plano 2, move saldos no DB.
   * Em plano 3, chama on-chain e aguarda confirmação.
   *
   * Responsável por: consumir reservations (makerRes, takerRes), creditar asset
   * recebido em cada lado, marcar trade SETTLED (ou REVERTED em caso de falha).
   *
   * Deve ser idempotente (receber trade já SETTLED/REVERTED = no-op).
   */
  settle(tradeId: string): Promise<void>;
}
```

- [ ] **Step 2.3: Atualizar barrel `api/src/modules/trading-v2/index.ts`:** append

```typescript
export * from "./types/order.types";
export * from "./types/trade.types";
```

- [ ] **Step 2.4: Type-check:**

```bash
cd /Users/rafael/Documents/GitHub/Oraculo/.claude/worktrees/orderbook-rewrite-02-orders-matching/api
bun x tsc --noEmit 2>&1 | grep "src/modules/trading-v2" || echo "clean"
```
Expected: `clean`.

- [ ] **Step 2.5: Commit:**

```bash
git add api/src/modules/trading-v2/types/ api/src/modules/trading-v2/index.ts
git commit -m "feat(trading-v2): types de order + trade (+ ISettler contract)"
```

---

## Task 3: OrderRepository

**Files:**
- Create: `api/src/modules/trading-v2/repositories/order.repository.ts`
- Create: `api/src/modules/trading-v2/__tests__/order.repository.integration.test.ts`

- [ ] **Step 3.1: Write failing test:**

```typescript
import "dotenv/config";
import { prisma } from "../../../shared/database/config/prisma-client";
import { OrderRepository } from "../repositories/order.repository";
import { UNIT } from "../types/balance.types";

const repo = new OrderRepository(prisma);

const USER = "00000000-0000-0000-0000-000000000001";
const USER2 = "00000000-0000-0000-0000-000000000002";
const MARKET = "Market1111111111111111111111111111111111111";

beforeEach(async () => {
  await prisma.ob2Trade.deleteMany({});
  await prisma.ob2Order.deleteMany({});
});
afterAll(async () => { await prisma.$disconnect(); });

test("create returns OrderView with OPEN status and zero filled", async () => {
  const o = await repo.create({
    userId: USER, marketPda: MARKET, side: "BUY",
    priceBps: 6000, quantity: 100n * UNIT, reservationId: "res-1",
  });
  expect(o.status).toBe("OPEN");
  expect(o.filled).toBe(0n);
  expect(o.quantity).toBe(100n * UNIT);
  expect(o.priceBps).toBe(6000);
});

test("getById returns view or null", async () => {
  const o = await repo.create({
    userId: USER, marketPda: MARKET, side: "SELL",
    priceBps: 4000, quantity: 50n * UNIT, reservationId: "res-2",
  });
  const got = await repo.getById(o.id);
  expect(got!.id).toBe(o.id);

  const none = await repo.getById("00000000-0000-0000-0000-00000000dead");
  expect(none).toBeNull();
});

test("findBestMatch returns null when no opposing orders exist", async () => {
  const match = await repo.findBestMatch(MARKET, "BUY", 6000);
  expect(match).toBeNull();
});

test("findBestMatch for BUY picks lowest SELL price within limit, FIFO tiebreak", async () => {
  await repo.create({ userId: USER,  marketPda: MARKET, side: "SELL", priceBps: 5500, quantity: 10n * UNIT, reservationId: "r1" });
  await new Promise(r => setTimeout(r, 5));
  await repo.create({ userId: USER2, marketPda: MARKET, side: "SELL", priceBps: 5000, quantity: 20n * UNIT, reservationId: "r2" });
  await new Promise(r => setTimeout(r, 5));
  const best = await repo.findBestMatch(MARKET, "BUY", 6000);  // willing to pay ≤ 6000
  expect(best!.priceBps).toBe(5000);
  expect(best!.userId).toBe(USER2);
});

test("findBestMatch for BUY ignores SELLs priced above the BUY limit", async () => {
  await repo.create({ userId: USER, marketPda: MARKET, side: "SELL", priceBps: 7000, quantity: 10n * UNIT, reservationId: "r1" });
  const best = await repo.findBestMatch(MARKET, "BUY", 6000);
  expect(best).toBeNull();
});

test("findBestMatch for SELL picks highest BUY price within limit", async () => {
  await repo.create({ userId: USER,  marketPda: MARKET, side: "BUY", priceBps: 4500, quantity: 10n * UNIT, reservationId: "r1" });
  await repo.create({ userId: USER2, marketPda: MARKET, side: "BUY", priceBps: 5500, quantity: 20n * UNIT, reservationId: "r2" });
  const best = await repo.findBestMatch(MARKET, "SELL", 4000);  // willing to accept ≥ 4000
  expect(best!.priceBps).toBe(5500);
});

test("applyFill increases filled and closes order when filled equals quantity", async () => {
  const o = await repo.create({
    userId: USER, marketPda: MARKET, side: "BUY",
    priceBps: 6000, quantity: 100n * UNIT, reservationId: "r1",
  });
  await repo.applyFill(o.id, 40n * UNIT);
  const mid = await repo.getById(o.id);
  expect(mid!.filled).toBe(40n * UNIT);
  expect(mid!.status).toBe("OPEN");

  await repo.applyFill(o.id, 60n * UNIT);
  const done = await repo.getById(o.id);
  expect(done!.filled).toBe(100n * UNIT);
  expect(done!.status).toBe("FILLED");
  expect(done!.closedAt).not.toBeNull();
});

test("markCancelled moves OPEN order to CANCELLED with closedAt", async () => {
  const o = await repo.create({
    userId: USER, marketPda: MARKET, side: "BUY",
    priceBps: 6000, quantity: 100n * UNIT, reservationId: "r1",
  });
  await repo.markCancelled(o.id);
  const got = await repo.getById(o.id);
  expect(got!.status).toBe("CANCELLED");
  expect(got!.closedAt).not.toBeNull();
});

test("listOpen returns only OPEN orders for a user ordered by createdAt desc", async () => {
  const a = await repo.create({ userId: USER, marketPda: MARKET, side: "BUY", priceBps: 5000, quantity: 10n * UNIT, reservationId: "r1" });
  const b = await repo.create({ userId: USER, marketPda: MARKET, side: "BUY", priceBps: 5500, quantity: 10n * UNIT, reservationId: "r2" });
  await repo.markCancelled(a.id);
  const list = await repo.listByUser(USER, MARKET, { status: "OPEN" });
  expect(list.map(o => o.id)).toEqual([b.id]);
});
```

- [ ] **Step 3.2: Run, must fail.**

```bash
cd /Users/rafael/Documents/GitHub/Oraculo/.claude/worktrees/orderbook-rewrite-02-orders-matching/api
bun x jest src/modules/trading-v2/__tests__/order.repository.integration.test.ts --runInBand
```

- [ ] **Step 3.3: Implement `repositories/order.repository.ts`:**

```typescript
import type { PrismaClient, Ob2Side, Ob2OrderStatus } from "../../../generated/prisma/client";
import type { OrderView } from "../types/order.types";

export interface CreateOrderRow {
  userId: string;
  marketPda: string;
  side: Ob2Side;
  priceBps: number;
  quantity: bigint;
  reservationId: string;
  clientOrderId?: string;
}

export interface ListFilter {
  status?: Ob2OrderStatus;
}

const toBig = (v: unknown): bigint => {
  const s = String(v);
  const [i, f = ""] = s.split(".");
  return BigInt(i + f.padEnd(6, "0").slice(0, 6));
};

const fromBig = (v: bigint): string => {
  const s = v.toString().padStart(7, "0");
  return `${s.slice(0, -6)}.${s.slice(-6)}`;
};

const toView = (row: any): OrderView => ({
  id: row.id,
  userId: row.userId,
  marketPda: row.marketPda,
  side: row.side,
  priceBps: row.priceBps,
  quantity: toBig(row.quantity),
  filled: toBig(row.filled),
  status: row.status,
  rejectReason: row.rejectReason,
  clientOrderId: row.clientOrderId,
  reservationId: row.reservationId,
  createdAt: row.createdAt,
  updatedAt: row.updatedAt,
  closedAt: row.closedAt,
});

export class OrderRepository {
  constructor(private readonly prisma: PrismaClient) {}

  async create(input: CreateOrderRow): Promise<OrderView> {
    const row = await this.prisma.ob2Order.create({
      data: {
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

  async getById(id: string): Promise<OrderView | null> {
    const row = await this.prisma.ob2Order.findUnique({ where: { id } });
    return row ? toView(row) : null;
  }

  /**
   * Pra um taker com `takerSide`, retorna a melhor ordem contra-parte (OPEN)
   * dentro do price limit `takerPriceBps`. Nada de lock aqui — é apenas leitura
   * não-authoritativa (matching engine faz FOR UPDATE SKIP LOCKED).
   */
  async findBestMatch(
    marketPda: string, takerSide: Ob2Side, takerPriceBps: number,
  ): Promise<OrderView | null> {
    const opposing: Ob2Side = takerSide === "BUY" ? "SELL" : "BUY";
    const row = await this.prisma.ob2Order.findFirst({
      where: {
        marketPda, status: "OPEN", side: opposing,
        ...(takerSide === "BUY"
          ? { priceBps: { lte: takerPriceBps } }
          : { priceBps: { gte: takerPriceBps } }),
      },
      orderBy: [
        { priceBps: takerSide === "BUY" ? "asc" : "desc" },
        { createdAt: "asc" },
      ],
    });
    return row ? toView(row) : null;
  }

  /**
   * Adiciona `amount` ao filled. Se atingir quantity, move status pra FILLED.
   * DEVE rodar dentro da mesma tx que o caller (matching engine orquestra).
   */
  async applyFill(orderId: string, amount: bigint): Promise<OrderView> {
    const current = await this.prisma.ob2Order.findUnique({ where: { id: orderId } });
    if (!current) throw new Error(`order ${orderId} not found`);
    const filledBefore = toBig(current.filled);
    const qty = toBig(current.quantity);
    const newFilled = filledBefore + amount;
    if (newFilled > qty) throw new Error(`fill ${amount} exceeds remaining on ${orderId}`);
    const done = newFilled === qty;

    const row = await this.prisma.ob2Order.update({
      where: { id: orderId },
      data: {
        filled: fromBig(newFilled),
        status: done ? "FILLED" : "OPEN",
        closedAt: done ? new Date() : null,
      },
    });
    return toView(row);
  }

  async markCancelled(orderId: string): Promise<OrderView> {
    const row = await this.prisma.ob2Order.update({
      where: { id: orderId },
      data: { status: "CANCELLED", closedAt: new Date() },
    });
    return toView(row);
  }

  async listByUser(userId: string, marketPda: string, filter: ListFilter = {}): Promise<OrderView[]> {
    const rows = await this.prisma.ob2Order.findMany({
      where: { userId, marketPda, ...(filter.status ? { status: filter.status } : {}) },
      orderBy: { createdAt: "desc" },
    });
    return rows.map(toView);
  }
}
```

- [ ] **Step 3.4: Run, all pass.**

Expected: 9 tests passed.

- [ ] **Step 3.5: Commit:**

```bash
git add api/src/modules/trading-v2/repositories/order.repository.ts api/src/modules/trading-v2/__tests__/order.repository.integration.test.ts
git commit -m "feat(trading-v2): OrderRepository + integration tests"
```

---

## Task 4: TradeRepository

**Files:**
- Create: `api/src/modules/trading-v2/repositories/trade.repository.ts`
- Create: `api/src/modules/trading-v2/__tests__/trade.repository.integration.test.ts`

- [ ] **Step 4.1: Failing test:**

```typescript
import "dotenv/config";
import { prisma } from "../../../shared/database/config/prisma-client";
import { TradeRepository } from "../repositories/trade.repository";
import { UNIT } from "../types/balance.types";

const repo = new TradeRepository(prisma);

const MARKET = "Market1111111111111111111111111111111111111";
const MAKER_ORD = "00000000-0000-0000-0000-00000000aaaa";
const TAKER_ORD = "00000000-0000-0000-0000-00000000bbbb";

beforeEach(async () => {
  await prisma.ob2Trade.deleteMany({});
});
afterAll(async () => { await prisma.$disconnect(); });

test("create inserts a SETTLING trade with deadline in the future", async () => {
  const t = await repo.create({
    marketPda: MARKET,
    makerOrderId: MAKER_ORD, takerOrderId: TAKER_ORD,
    priceBps: 6000, quantity: 10n * UNIT,
    primitive: "TRADE", sync: true,
  });
  expect(t.status).toBe("SETTLING");
  expect(t.primitive).toBe("TRADE");
  expect(t.sync).toBe(true);
  expect(t.settlingDeadline.getTime()).toBeGreaterThan(Date.now());
});

test("sync=true sets 30s deadline, sync=false sets 120s deadline", async () => {
  const tSync = await repo.create({
    marketPda: MARKET,
    makerOrderId: MAKER_ORD, takerOrderId: TAKER_ORD,
    priceBps: 5000, quantity: 1n * UNIT,
    primitive: "TRADE", sync: true,
  });
  const tAsync = await repo.create({
    marketPda: MARKET,
    makerOrderId: MAKER_ORD, takerOrderId: TAKER_ORD,
    priceBps: 5000, quantity: 1n * UNIT,
    primitive: "TRADE", sync: false,
  });
  const syncMs = tSync.settlingDeadline.getTime() - tSync.createdAt.getTime();
  const asyncMs = tAsync.settlingDeadline.getTime() - tAsync.createdAt.getTime();
  expect(Math.abs(syncMs - 30_000)).toBeLessThan(1000);
  expect(Math.abs(asyncMs - 120_000)).toBeLessThan(1000);
});

test("markSettled sets status and settledAt", async () => {
  const t = await repo.create({
    marketPda: MARKET,
    makerOrderId: MAKER_ORD, takerOrderId: TAKER_ORD,
    priceBps: 6000, quantity: 10n * UNIT,
    primitive: "TRADE", sync: true,
  });
  await repo.markSettled(t.id, "txsig-abc");
  const got = await repo.getById(t.id);
  expect(got!.status).toBe("SETTLED");
  expect(got!.txSignature).toBe("txsig-abc");
  expect(got!.settledAt).not.toBeNull();
});

test("markReverted sets status and reason", async () => {
  const t = await repo.create({
    marketPda: MARKET,
    makerOrderId: MAKER_ORD, takerOrderId: TAKER_ORD,
    priceBps: 6000, quantity: 10n * UNIT,
    primitive: "TRADE", sync: true,
  });
  await repo.markReverted(t.id, "rpc_timeout");
  const got = await repo.getById(t.id);
  expect(got!.status).toBe("REVERTED");
  expect(got!.revertReason).toBe("rpc_timeout");
});

test("listByMarket returns trades ordered by createdAt desc", async () => {
  const a = await repo.create({
    marketPda: MARKET, makerOrderId: MAKER_ORD, takerOrderId: TAKER_ORD,
    priceBps: 6000, quantity: 1n * UNIT, primitive: "TRADE", sync: true,
  });
  await new Promise(r => setTimeout(r, 5));
  const b = await repo.create({
    marketPda: MARKET, makerOrderId: MAKER_ORD, takerOrderId: TAKER_ORD,
    priceBps: 6100, quantity: 1n * UNIT, primitive: "TRADE", sync: true,
  });
  const list = await repo.listByMarket(MARKET, 10);
  expect(list.map(t => t.id)).toEqual([b.id, a.id]);
});
```

- [ ] **Step 4.2: Run, fail.**

- [ ] **Step 4.3: Implement `repositories/trade.repository.ts`:**

```typescript
import type { PrismaClient, Ob2Primitive, Ob2TradeStatus } from "../../../generated/prisma/client";
import type { TradeRecord } from "../types/trade.types";

export interface CreateTradeInput {
  marketPda: string;
  makerOrderId: string;
  takerOrderId: string;
  priceBps: number;
  quantity: bigint;
  primitive: Ob2Primitive;
  sync: boolean;
}

const SLA_MS_SYNC  = 30_000;
const SLA_MS_ASYNC = 120_000;

const toBig = (v: unknown): bigint => {
  const s = String(v);
  const [i, f = ""] = s.split(".");
  return BigInt(i + f.padEnd(6, "0").slice(0, 6));
};

const fromBig = (v: bigint): string => {
  const s = v.toString().padStart(7, "0");
  return `${s.slice(0, -6)}.${s.slice(-6)}`;
};

const toRecord = (row: any): TradeRecord => ({
  id: row.id,
  marketPda: row.marketPda,
  makerOrderId: row.makerOrderId,
  takerOrderId: row.takerOrderId,
  priceBps: row.priceBps,
  quantity: toBig(row.quantity),
  primitive: row.primitive,
  status: row.status,
  sync: row.sync,
  settlingDeadline: row.settlingDeadline,
  txSignature: row.txSignature,
  revertReason: row.revertReason,
  createdAt: row.createdAt,
  settledAt: row.settledAt,
});

export class TradeRepository {
  constructor(private readonly prisma: PrismaClient) {}

  async create(input: CreateTradeInput): Promise<TradeRecord> {
    const slaMs = input.sync ? SLA_MS_SYNC : SLA_MS_ASYNC;
    const deadline = new Date(Date.now() + slaMs);
    const row = await this.prisma.ob2Trade.create({
      data: {
        marketPda: input.marketPda,
        makerOrderId: input.makerOrderId,
        takerOrderId: input.takerOrderId,
        priceBps: input.priceBps,
        quantity: fromBig(input.quantity),
        primitive: input.primitive,
        status: "SETTLING",
        sync: input.sync,
        settlingDeadline: deadline,
      },
    });
    return toRecord(row);
  }

  async getById(id: string): Promise<TradeRecord | null> {
    const row = await this.prisma.ob2Trade.findUnique({ where: { id } });
    return row ? toRecord(row) : null;
  }

  async markSettled(id: string, txSignature: string | null): Promise<void> {
    await this.prisma.ob2Trade.update({
      where: { id },
      data: { status: "SETTLED", txSignature, settledAt: new Date() },
    });
  }

  async markReverted(id: string, reason: string): Promise<void> {
    await this.prisma.ob2Trade.update({
      where: { id },
      data: { status: "REVERTED", revertReason: reason },
    });
  }

  async listByMarket(marketPda: string, limit: number): Promise<TradeRecord[]> {
    const rows = await this.prisma.ob2Trade.findMany({
      where: { marketPda },
      orderBy: { createdAt: "desc" },
      take: limit,
    });
    return rows.map(toRecord);
  }

  async listExpiredSettling(now: Date = new Date()): Promise<TradeRecord[]> {
    const rows = await this.prisma.ob2Trade.findMany({
      where: { status: "SETTLING", settlingDeadline: { lt: now } },
      orderBy: { settlingDeadline: "asc" },
    });
    return rows.map(toRecord);
  }
}
```

- [ ] **Step 4.4: Run, 5 pass.**

- [ ] **Step 4.5: Commit:**

```bash
git add api/src/modules/trading-v2/repositories/trade.repository.ts api/src/modules/trading-v2/__tests__/trade.repository.integration.test.ts
git commit -m "feat(trading-v2): TradeRepository + integration tests"
```

---

## Task 5: PrimitiveDecider (pure)

**Files:**
- Create: `api/src/modules/trading-v2/services/primitive-decider.service.ts`
- Create: `api/src/modules/trading-v2/__tests__/primitive-decider.unit.test.ts`

Spec §6.2: a primitiva vem dos ativos reservados nos dois lados. Função pura.

- [ ] **Step 5.1: Failing test:**

```typescript
import { decidePrimitive, ImpossiblePairError } from "../services/primitive-decider.service";

test("USDC(BUY) × YES(SELL) = TRADE", () => {
  expect(decidePrimitive("BUY", "USDC", "SELL", "YES")).toBe("TRADE");
  expect(decidePrimitive("SELL", "YES", "BUY", "USDC")).toBe("TRADE"); // symmetric
});

test("USDC(BUY) × USDC(SELL) = MINT (both opening)", () => {
  expect(decidePrimitive("BUY", "USDC", "SELL", "USDC")).toBe("MINT");
  expect(decidePrimitive("SELL", "USDC", "BUY", "USDC")).toBe("MINT");
});

test("NO(BUY) × YES(SELL) = MERGE (both closing)", () => {
  expect(decidePrimitive("BUY", "NO", "SELL", "YES")).toBe("MERGE");
  expect(decidePrimitive("SELL", "YES", "BUY", "NO")).toBe("MERGE");
});

test("NO(BUY) × USDC(SELL) = TRADE (taker closes NO short, maker opens short→gets NO)", () => {
  expect(decidePrimitive("BUY", "NO", "SELL", "USDC")).toBe("TRADE");
  expect(decidePrimitive("SELL", "USDC", "BUY", "NO")).toBe("TRADE");
});

test("impossible pairs throw", () => {
  // NO × NO (both closing shorts, but no one holds counter token to destroy)
  expect(() => decidePrimitive("BUY", "NO", "SELL", "NO")).toThrow(ImpossiblePairError);
  // YES(BUY) × YES(SELL) — not possible since BUY with YES reserved is logically
  // "buy YES but I already have YES" which wouldn't happen through classifier
  // (classifier picks NO if freeNo >= qty, else USDC). Guard anyway.
  expect(() => decidePrimitive("BUY", "YES", "SELL", "YES")).toThrow(ImpossiblePairError);
});
```

- [ ] **Step 5.2: Run, fail.**

- [ ] **Step 5.3: Implement:**

```typescript
import type { Ob2Asset, Ob2Primitive, Ob2Side } from "../../../generated/prisma/client";

export class ImpossiblePairError extends Error {
  constructor(takerSide: Ob2Side, takerAsset: Ob2Asset, makerSide: Ob2Side, makerAsset: Ob2Asset) {
    super(`impossible pair: ${takerSide}/${takerAsset} vs ${makerSide}/${makerAsset}`);
    this.name = "ImpossiblePairError";
  }
}

/**
 * Decide a primitiva de settlement a partir dos ativos reservados.
 *
 * Regra derivada do spec §6.2:
 * - USDC × YES   → TRADE (existing YES changes hands)
 * - USDC × USDC  → MINT  (vault mints fresh YES/NO pair)
 * - NO   × YES   → MERGE (vault burns the pair, returns USDC)
 * - NO   × USDC  → TRADE (NO changes hands; one side closes short, other opens)
 *
 * Simetria: a função aceita qualquer ordem (taker, maker) e normaliza.
 */
export function decidePrimitive(
  takerSide: Ob2Side, takerAsset: Ob2Asset,
  makerSide: Ob2Side, makerAsset: Ob2Asset,
): Ob2Primitive {
  const key = normalize(takerSide, takerAsset, makerSide, makerAsset);
  switch (key) {
    case "BUY:USDC|SELL:YES":  return "TRADE";
    case "BUY:USDC|SELL:USDC": return "MINT";
    case "BUY:NO|SELL:YES":    return "MERGE";
    case "BUY:NO|SELL:USDC":   return "TRADE";
    default:
      throw new ImpossiblePairError(takerSide, takerAsset, makerSide, makerAsset);
  }
}

function normalize(ts: Ob2Side, ta: Ob2Asset, ms: Ob2Side, ma: Ob2Asset): string {
  // Sempre retornamos o BUY primeiro, SELL depois — colapsa simetria.
  const buy  = ts === "BUY"  ? `BUY:${ta}`  : `BUY:${ma}`;
  const sell = ts === "SELL" ? `SELL:${ta}` : `SELL:${ma}`;
  return `${buy}|${sell}`;
}
```

- [ ] **Step 5.4: Run, 5 pass.**

- [ ] **Step 5.5: Commit:**

```bash
git add api/src/modules/trading-v2/services/primitive-decider.service.ts api/src/modules/trading-v2/__tests__/primitive-decider.unit.test.ts
git commit -m "feat(trading-v2): PrimitiveDecider (pure, deriva primitiva do ativo reservado)"
```

---

## Task 6: StubSettlerService

**Files:**
- Create: `api/src/modules/trading-v2/services/stub-settler.service.ts`
- Create: `api/src/modules/trading-v2/__tests__/stub-settler.integration.test.ts`

`StubSettler` implementa `ISettler`. Dado um trade SETTLING, faz as mutações de saldo que o settlement real faria, sem chamar on-chain. Substituído no Plano 3.

A lógica por primitiva:

- **TRADE (USDC(BUY) × YES(SELL))**: consome USDC do BUYer (igual a price*qty + fee porção já reservada), consome YES do SELLer. Credita YES pro BUYer (free), credita USDC pro SELLer (free). Em realidade: o vault repassa o USDC pro SELLer e o YES pro BUYer.
- **TRADE (NO(BUY) × USDC(SELL))**: análogo com NO mudando de mão.
- **MINT (USDC(BUY) × USDC(SELL))**: vault consome USDC dos dois, emite YES pro BUYer e NO pro SELLer.
- **MERGE (NO(BUY) × YES(SELL))**: vault consome NO do BUYer e YES do SELLer, credita USDC pros dois proporcionalmente (buyer recebe `qty*(1-price)`, seller recebe `qty*price`).

Para o StubSettler (Plano 2), os fees são **ignorados por enquanto** — tratados no Plano 3 quando integrarmos com o fee ledger existente. Documentar isso explicitamente.

- [ ] **Step 6.1: Failing test:**

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

beforeEach(async () => {
  await prisma.ob2Trade.deleteMany({});
  await prisma.ob2Order.deleteMany({});
  await prisma.ob2Reservation.deleteMany({});
  await prisma.ob2UserMarketBalance.deleteMany({});
});
afterAll(async () => { await prisma.$disconnect(); });

test("settle of TRADE moves USDC from buyer to seller and YES from seller to buyer", async () => {
  // SEED: BUYER has 1000 USDC free, SELLER has 100 YES free
  await balanceRepo.upsertOnchain(BUYER,  MARKET, "USDC", 1000n * UNIT, 1n);
  await balanceRepo.upsertOnchain(SELLER, MARKET, "YES",  100n  * UNIT, 1n);

  // BUYer reserves 60 USDC (100 qty @ 0.60, ignoring fee)
  const buyerOrder = await orderRepo.create({
    userId: BUYER, marketPda: MARKET, side: "BUY",
    priceBps: 6000, quantity: 100n * UNIT, reservationId: "placeholder",
  });
  const buyerRes = await resSvc.reserve({
    userId: BUYER, marketPda: MARKET, asset: "USDC",
    amount: 60n * UNIT, orderId: buyerOrder.id,
  });
  await prisma.ob2Order.update({ where: { id: buyerOrder.id }, data: { reservationId: buyerRes.id } });

  // SELLer reserves 100 YES
  const sellerOrder = await orderRepo.create({
    userId: SELLER, marketPda: MARKET, side: "SELL",
    priceBps: 6000, quantity: 100n * UNIT, reservationId: "placeholder",
  });
  const sellerRes = await resSvc.reserve({
    userId: SELLER, marketPda: MARKET, asset: "YES",
    amount: 100n * UNIT, orderId: sellerOrder.id,
  });
  await prisma.ob2Order.update({ where: { id: sellerOrder.id }, data: { reservationId: sellerRes.id } });

  // Create trade (taker=buyer)
  const trade = await tradeRepo.create({
    marketPda: MARKET,
    makerOrderId: sellerOrder.id, takerOrderId: buyerOrder.id,
    priceBps: 6000, quantity: 100n * UNIT,
    primitive: "TRADE", sync: true,
  });

  // SETTLE
  await settler.settle(trade.id);

  // Trade is SETTLED
  const t = await tradeRepo.getById(trade.id);
  expect(t!.status).toBe("SETTLED");

  // BUYER: USDC reserved -60 (consumed), USDC free untouched (700->same? no: 1000-60=940 free stays free),
  //        YES free +100
  const buyerUsdc = await balanceRepo.get(BUYER, MARKET, "USDC");
  expect(buyerUsdc!.free).toBe(940n * UNIT);    // 1000 - 60 (reserved originally, now consumed not credited back)
  expect(buyerUsdc!.reserved).toBe(0n);
  const buyerYes = await balanceRepo.get(BUYER, MARKET, "YES");
  expect(buyerYes!.free).toBe(100n * UNIT);

  // SELLER: YES reserved -100 (consumed), USDC free +60 (received from buyer)
  const sellerYes = await balanceRepo.get(SELLER, MARKET, "YES");
  expect(sellerYes!.free).toBe(0n);
  expect(sellerYes!.reserved).toBe(0n);
  const sellerUsdc = await balanceRepo.get(SELLER, MARKET, "USDC");
  expect(sellerUsdc!.free).toBe(60n * UNIT);
});

test("settle is idempotent on second call", async () => {
  await balanceRepo.upsertOnchain(BUYER,  MARKET, "USDC", 1000n * UNIT, 1n);
  await balanceRepo.upsertOnchain(SELLER, MARKET, "YES",  100n  * UNIT, 1n);

  const bo = await orderRepo.create({ userId: BUYER, marketPda: MARKET, side: "BUY", priceBps: 6000, quantity: 100n * UNIT, reservationId: "p" });
  const br = await resSvc.reserve({ userId: BUYER, marketPda: MARKET, asset: "USDC", amount: 60n * UNIT, orderId: bo.id });
  await prisma.ob2Order.update({ where: { id: bo.id }, data: { reservationId: br.id } });
  const so = await orderRepo.create({ userId: SELLER, marketPda: MARKET, side: "SELL", priceBps: 6000, quantity: 100n * UNIT, reservationId: "p" });
  const sr = await resSvc.reserve({ userId: SELLER, marketPda: MARKET, asset: "YES", amount: 100n * UNIT, orderId: so.id });
  await prisma.ob2Order.update({ where: { id: so.id }, data: { reservationId: sr.id } });
  const t = await tradeRepo.create({
    marketPda: MARKET, makerOrderId: so.id, takerOrderId: bo.id,
    priceBps: 6000, quantity: 100n * UNIT, primitive: "TRADE", sync: true,
  });

  await settler.settle(t.id);
  await settler.settle(t.id);  // second call must be no-op

  const buyerUsdc = await balanceRepo.get(BUYER, MARKET, "USDC");
  expect(buyerUsdc!.free).toBe(940n * UNIT);   // not double-credited
  const sellerUsdc = await balanceRepo.get(SELLER, MARKET, "USDC");
  expect(sellerUsdc!.free).toBe(60n * UNIT);
});

test("settle of MINT: both reserved USDC consumed, BUYER gets YES, SELLER gets NO", async () => {
  await balanceRepo.upsertOnchain(BUYER,  MARKET, "USDC", 1000n * UNIT, 1n);
  await balanceRepo.upsertOnchain(SELLER, MARKET, "USDC", 1000n * UNIT, 1n);

  // qty=100, price=0.60 → BUYer reserves 60 USDC; SELLer (opening NO short) reserves 40 USDC.
  const bo = await orderRepo.create({ userId: BUYER,  marketPda: MARKET, side: "BUY",  priceBps: 6000, quantity: 100n * UNIT, reservationId: "p" });
  const br = await resSvc.reserve({ userId: BUYER,  marketPda: MARKET, asset: "USDC", amount: 60n * UNIT, orderId: bo.id });
  await prisma.ob2Order.update({ where: { id: bo.id }, data: { reservationId: br.id } });
  const so = await orderRepo.create({ userId: SELLER, marketPda: MARKET, side: "SELL", priceBps: 6000, quantity: 100n * UNIT, reservationId: "p" });
  const sr = await resSvc.reserve({ userId: SELLER, marketPda: MARKET, asset: "USDC", amount: 40n * UNIT, orderId: so.id });
  await prisma.ob2Order.update({ where: { id: so.id }, data: { reservationId: sr.id } });

  const t = await tradeRepo.create({
    marketPda: MARKET, makerOrderId: so.id, takerOrderId: bo.id,
    priceBps: 6000, quantity: 100n * UNIT, primitive: "MINT", sync: true,
  });
  await settler.settle(t.id);

  expect((await tradeRepo.getById(t.id))!.status).toBe("SETTLED");
  expect((await balanceRepo.get(BUYER,  MARKET, "USDC"))!.free).toBe(940n * UNIT);
  expect((await balanceRepo.get(BUYER,  MARKET, "YES"))!.free).toBe(100n * UNIT);
  expect((await balanceRepo.get(SELLER, MARKET, "USDC"))!.free).toBe(960n * UNIT);
  expect((await balanceRepo.get(SELLER, MARKET, "NO"))!.free).toBe(100n * UNIT);
});

test("settle of MERGE: BUYER's NO + SELLER's YES destroyed, both get USDC proportionally", async () => {
  // BUYer (closing NO short) reserves NO; SELLer (closing YES long) reserves YES.
  await balanceRepo.upsertOnchain(BUYER,  MARKET, "NO",  100n * UNIT, 1n);
  await balanceRepo.upsertOnchain(SELLER, MARKET, "YES", 100n * UNIT, 1n);

  const bo = await orderRepo.create({ userId: BUYER,  marketPda: MARKET, side: "BUY",  priceBps: 6000, quantity: 100n * UNIT, reservationId: "p" });
  const br = await resSvc.reserve({ userId: BUYER,  marketPda: MARKET, asset: "NO",  amount: 100n * UNIT, orderId: bo.id });
  await prisma.ob2Order.update({ where: { id: bo.id }, data: { reservationId: br.id } });
  const so = await orderRepo.create({ userId: SELLER, marketPda: MARKET, side: "SELL", priceBps: 6000, quantity: 100n * UNIT, reservationId: "p" });
  const sr = await resSvc.reserve({ userId: SELLER, marketPda: MARKET, asset: "YES", amount: 100n * UNIT, orderId: so.id });
  await prisma.ob2Order.update({ where: { id: so.id }, data: { reservationId: sr.id } });

  const t = await tradeRepo.create({
    marketPda: MARKET, makerOrderId: so.id, takerOrderId: bo.id,
    priceBps: 6000, quantity: 100n * UNIT, primitive: "MERGE", sync: true,
  });
  await settler.settle(t.id);

  expect((await tradeRepo.getById(t.id))!.status).toBe("SETTLED");
  // BUYer closed NO short @ 0.60: receives qty * (1 - 0.60) = 40 USDC
  expect((await balanceRepo.get(BUYER,  MARKET, "USDC"))!.free).toBe(40n * UNIT);
  expect((await balanceRepo.get(BUYER,  MARKET, "NO"))!.free).toBe(0n);
  // SELLer sold YES @ 0.60: receives qty * 0.60 = 60 USDC
  expect((await balanceRepo.get(SELLER, MARKET, "USDC"))!.free).toBe(60n * UNIT);
  expect((await balanceRepo.get(SELLER, MARKET, "YES"))!.free).toBe(0n);
});
```

- [ ] **Step 6.2: Run, fail.**

- [ ] **Step 6.3: Implement:**

```typescript
import type { PrismaClient, Ob2Asset, Ob2Primitive } from "../../../generated/prisma/client";
import type { ISettler } from "../types/trade.types";

/**
 * Plano 2 stub. Faz o que o settlement real fará, mas sem chamar Solana:
 * - Consome as reservations dos dois lados (debita `reserved`).
 * - Credita o asset "recebido" no `free` do lado oposto.
 * - Marca trade SETTLED.
 *
 * Fees são ignorados neste stub — Plano 3 integra com fee ledger.
 *
 * Interface `ISettler` idêntica ao settler real do Plano 3, que troca este
 * stub sem impacto no MatchingEngine.
 */
export class StubSettlerService implements ISettler {
  constructor(private readonly prisma: PrismaClient) {}

  async settle(tradeId: string): Promise<void> {
    await this.prisma.$transaction(async (tx) => {
      const trade = await tx.ob2Trade.findUnique({ where: { id: tradeId } });
      if (!trade) throw new Error(`trade ${tradeId} not found`);
      if (trade.status !== "SETTLING") return;  // idempotente

      const makerOrder = await tx.ob2Order.findUnique({ where: { id: trade.makerOrderId } });
      const takerOrder = await tx.ob2Order.findUnique({ where: { id: trade.takerOrderId } });
      if (!makerOrder || !takerOrder) throw new Error("orders not found for trade");
      if (!makerOrder.reservationId || !takerOrder.reservationId) {
        throw new Error("orders missing reservationId — programming error");
      }

      const makerRes = await tx.ob2Reservation.findUnique({ where: { id: makerOrder.reservationId } });
      const takerRes = await tx.ob2Reservation.findUnique({ where: { id: takerOrder.reservationId } });
      if (!makerRes || !takerRes) throw new Error("reservations not found");

      const qty = this.fromDecimal(String(trade.quantity));

      // Quanto de cada reservation é consumido neste fill, em função do asset:
      // - Se a reserva é em YES/NO, o valor consumido é exatamente qty.
      // - Se a reserva é em USDC, depende do lado (buyer paga price*qty, seller short paga (1-price)*qty).
      const buyerId  = takerOrder.side === "BUY"  ? takerOrder.userId : makerOrder.userId;
      const sellerId = takerOrder.side === "SELL" ? takerOrder.userId : makerOrder.userId;

      const buyerRes   = takerOrder.side === "BUY"  ? takerRes : makerRes;
      const sellerRes  = takerOrder.side === "SELL" ? takerRes : makerRes;

      const priceBps = trade.priceBps;
      const buyerUsdcLeg  = (qty * BigInt(priceBps)) / 10000n;
      const sellerUsdcLeg = (qty * BigInt(10000 - priceBps)) / 10000n;

      // Consume from buyer reservation and credit the buyer with the asset they are buying
      if (buyerRes.asset === "USDC") {
        await this.debitReserved(tx, buyerId, trade.marketPda, "USDC", buyerUsdcLeg, buyerRes.id);
      } else if (buyerRes.asset === "NO") {
        // buyer is closing NO short — consume NO
        await this.debitReserved(tx, buyerId, trade.marketPda, "NO", qty, buyerRes.id);
      } else {
        throw new Error(`unexpected buyer reservation asset: ${buyerRes.asset}`);
      }

      // Consume from seller reservation
      if (sellerRes.asset === "YES") {
        await this.debitReserved(tx, sellerId, trade.marketPda, "YES", qty, sellerRes.id);
      } else if (sellerRes.asset === "USDC") {
        await this.debitReserved(tx, sellerId, trade.marketPda, "USDC", sellerUsdcLeg, sellerRes.id);
      } else {
        throw new Error(`unexpected seller reservation asset: ${sellerRes.asset}`);
      }

      // Now credit each side with what they receive, based on primitive
      switch (trade.primitive) {
        case "TRADE": {
          // USDC(BUY) × YES(SELL): buyer gets YES, seller gets USDC
          // NO(BUY)   × USDC(SELL): buyer gets NO (closes short via receive),
          //                         seller gets NO (ends up holding it...) NO -
          //                         actually in this symmetric case: buyer already had his
          //                         NO consumed from reserve, seller opens NO short by
          //                         receiving NO credit... that's wrong model.
          //
          // Review: in NO(BUY) × USDC(SELL), the buyer is closing their NO short —
          // their NO is consumed; the seller is opening a NO short — they pay USDC
          // and receive... nothing; their USDC pays the buyer for the USDC leg of
          // closing their NO short. Realistically: buyer receives USDC (refund of
          // short collateral at current price), seller has USDC consumed and
          // "owes" NO to the system (but nobody credits NO to them in a TRADE
          // primitive — this combination is actually a MINT on NO side).
          //
          // Simplification for stub: treat NO(BUY)×USDC(SELL) as TRADE that transfers
          // NO from system to nobody on the SELL side — seller ends up long NO.
          // This is functionally equivalent to the buyer's NO being minted to the seller.
          if (buyerRes.asset === "USDC" && sellerRes.asset === "YES") {
            await this.creditFree(tx, buyerId,  trade.marketPda, "YES",  qty);
            await this.creditFree(tx, sellerId, trade.marketPda, "USDC", buyerUsdcLeg);
          } else if (buyerRes.asset === "NO" && sellerRes.asset === "USDC") {
            await this.creditFree(tx, buyerId,  trade.marketPda, "USDC", sellerUsdcLeg);
            await this.creditFree(tx, sellerId, trade.marketPda, "NO",   qty);
          } else {
            throw new Error(`TRADE primitive with unexpected asset pair`);
          }
          break;
        }
        case "MINT": {
          // USDC(BUY) × USDC(SELL): buyer gets YES, seller gets NO
          await this.creditFree(tx, buyerId,  trade.marketPda, "YES", qty);
          await this.creditFree(tx, sellerId, trade.marketPda, "NO",  qty);
          break;
        }
        case "MERGE": {
          // NO(BUY) × YES(SELL): both get USDC proportionally
          await this.creditFree(tx, buyerId,  trade.marketPda, "USDC", sellerUsdcLeg);
          await this.creditFree(tx, sellerId, trade.marketPda, "USDC", buyerUsdcLeg);
          break;
        }
        default:
          throw new Error(`unknown primitive ${trade.primitive}`);
      }

      await tx.ob2Trade.update({
        where: { id: tradeId },
        data: { status: "SETTLED", settledAt: new Date(), txSignature: `stub:${tradeId.slice(0,8)}` },
      });
    });
  }

  private async debitReserved(
    tx: any, userId: string, marketPda: string, asset: Ob2Asset, amount: bigint, reservationId: string,
  ): Promise<void> {
    const amountStr = this.toDecimal(amount);
    await tx.$executeRawUnsafe(
      `UPDATE ob2_user_market_balances
          SET reserved = reserved - $4::numeric,
              updated_at = now()
        WHERE user_id = $1 AND market_pda = $2 AND asset = $3::"Ob2Asset"`,
      userId, marketPda, asset, amountStr,
    );
    // Shrink or release the reservation row
    const res = await tx.ob2Reservation.findUnique({ where: { id: reservationId } });
    if (!res) return;
    const current = this.fromDecimal(String(res.amount));
    const remaining = current - amount;
    if (remaining <= 0n) {
      await tx.ob2Reservation.update({ where: { id: reservationId }, data: { releasedAt: new Date() } });
    } else {
      await tx.ob2Reservation.update({ where: { id: reservationId }, data: { amount: this.toDecimal(remaining) } });
    }
  }

  private async creditFree(
    tx: any, userId: string, marketPda: string, asset: Ob2Asset, amount: bigint,
  ): Promise<void> {
    const amountStr = this.toDecimal(amount);
    await tx.ob2UserMarketBalance.upsert({
      where: { userId_marketPda_asset: { userId, marketPda, asset } },
      create: { userId, marketPda, asset, free: amountStr, reserved: "0", onchainTotal: "0" },
      update: {}, // separate raw update below
    });
    await tx.$executeRawUnsafe(
      `UPDATE ob2_user_market_balances
          SET free = free + $4::numeric,
              updated_at = now()
        WHERE user_id = $1 AND market_pda = $2 AND asset = $3::"Ob2Asset"`,
      userId, marketPda, asset, amountStr,
    );
  }

  private toDecimal(v: bigint): string {
    const s = v.toString().padStart(7, "0");
    return `${s.slice(0, -6)}.${s.slice(-6)}`;
  }

  private fromDecimal(s: string): bigint {
    const [i, f = ""] = s.split(".");
    return BigInt(i + f.padEnd(6, "0").slice(0, 6));
  }
}
```

**Important design note to the implementer:** the upsert-then-update for `creditFree` is two roundtrips to guarantee the row exists before the raw increment. A cleaner single-statement variant is welcome if you can verify it with the tests. The embedded comment block in the TRADE primitive explains the NO(BUY)×USDC(SELL) semantic choice — if you find that interpretation wrong, STOP and escalate before implementing; don't silently change it.

- [ ] **Step 6.4: Run, 4 tests pass.**

- [ ] **Step 6.5: Commit:**

```bash
git add api/src/modules/trading-v2/services/stub-settler.service.ts api/src/modules/trading-v2/__tests__/stub-settler.integration.test.ts
git commit -m "feat(trading-v2): StubSettlerService (plano 2, sem on-chain)"
```

---

## Task 7: MatchingEngine.tryMatch

**Files:**
- Create: `api/src/modules/trading-v2/services/matching-engine.service.ts`
- Create: `api/src/modules/trading-v2/__tests__/matching-engine.integration.test.ts`

Engine usa `FOR UPDATE SKIP LOCKED` via raw SQL para localizar e travar a melhor contra-parte atomicamente. Cria o trade, aplica fill nas duas ordens, delega pro `ISettler`.

- [ ] **Step 7.1: Failing test:**

```typescript
import "dotenv/config";
import { prisma } from "../../../shared/database/config/prisma-client";
import { BalanceRepository } from "../repositories/balance.repository";
import { ReservationService } from "../services/reservation.service";
import { OrderRepository } from "../repositories/order.repository";
import { TradeRepository } from "../repositories/trade.repository";
import { StubSettlerService } from "../services/stub-settler.service";
import { MatchingEngine } from "../services/matching-engine.service";
import { UNIT } from "../types/balance.types";

const balanceRepo = new BalanceRepository(prisma);
const resSvc = new ReservationService(prisma);
const orderRepo = new OrderRepository(prisma);
const tradeRepo = new TradeRepository(prisma);
const settler = new StubSettlerService(prisma);
const engine = new MatchingEngine(prisma, orderRepo, tradeRepo, settler);

const A = "00000000-0000-0000-0000-00000000000a";
const B = "00000000-0000-0000-0000-00000000000b";
const MARKET = "Market1111111111111111111111111111111111111";

async function seedAndPlace(
  userId: string, asset: "USDC" | "YES" | "NO", free: bigint,
  side: "BUY" | "SELL", priceBps: number, qty: bigint, reserveAmt: bigint,
) {
  await balanceRepo.upsertOnchain(userId, MARKET, asset, free, 1n);
  const ord = await orderRepo.create({ userId, marketPda: MARKET, side, priceBps, quantity: qty, reservationId: "p" });
  const res = await resSvc.reserve({ userId, marketPda: MARKET, asset, amount: reserveAmt, orderId: ord.id });
  await prisma.ob2Order.update({ where: { id: ord.id }, data: { reservationId: res.id } });
  return ord.id;
}

beforeEach(async () => {
  await prisma.ob2Trade.deleteMany({});
  await prisma.ob2Order.deleteMany({});
  await prisma.ob2Reservation.deleteMany({});
  await prisma.ob2UserMarketBalance.deleteMany({});
});
afterAll(async () => { await prisma.$disconnect(); });

test("tryMatch on empty book leaves taker OPEN", async () => {
  const takerId = await seedAndPlace(A, "USDC", 1000n * UNIT, "BUY", 6000, 100n * UNIT, 60n * UNIT);
  const result = await engine.tryMatch(takerId);
  expect(result.trades).toHaveLength(0);
  expect(result.takerStatus).toBe("OPEN");
  expect(result.takerFilled).toBe(0n);
});

test("tryMatch fills exactly when resting SELL qty equals taker qty", async () => {
  // Maker (SELL 100 @ 0.50) resting first
  await seedAndPlace(B, "YES", 100n * UNIT, "SELL", 5000, 100n * UNIT, 100n * UNIT);
  // Taker (BUY 100 @ 0.60) arrives
  const takerId = await seedAndPlace(A, "USDC", 1000n * UNIT, "BUY", 6000, 100n * UNIT, 60n * UNIT);

  const result = await engine.tryMatch(takerId);
  expect(result.trades).toHaveLength(1);
  expect(result.trades[0].priceBps).toBe(5000);  // maker's price wins
  expect(result.trades[0].primitive).toBe("TRADE");
  expect(result.takerStatus).toBe("FILLED");
  expect(result.takerFilled).toBe(100n * UNIT);

  // Taker (buyer) got YES, maker (seller) got USDC at 0.50 (maker's price)
  expect((await balanceRepo.get(A, MARKET, "YES"))!.free).toBe(100n * UNIT);
  // Taker reserved 60 USDC but only 50 was spent at maker's price — refund 10 to free
  expect((await balanceRepo.get(A, MARKET, "USDC"))!.free).toBe(950n * UNIT);  // 1000 - 50 spent
  expect((await balanceRepo.get(B, MARKET, "USDC"))!.free).toBe(50n * UNIT);
});

test("tryMatch consumes multiple resting orders up to taker qty", async () => {
  await seedAndPlace(B, "YES", 200n * UNIT, "SELL", 5000, 30n * UNIT, 30n * UNIT);
  await seedAndPlace(B, "YES", 200n * UNIT, "SELL", 5200, 40n * UNIT, 40n * UNIT);
  const takerId = await seedAndPlace(A, "USDC", 1000n * UNIT, "BUY", 6000, 100n * UNIT, 60n * UNIT);
  const result = await engine.tryMatch(takerId);

  expect(result.trades.length).toBeGreaterThanOrEqual(2);
  expect(result.takerFilled).toBe(70n * UNIT);
  expect(result.takerStatus).toBe("OPEN");      // 30 remaining
  const taker = await orderRepo.getById(takerId);
  expect(taker!.filled).toBe(70n * UNIT);
  expect(taker!.status).toBe("OPEN");
});

test("tryMatch does not cross limit price", async () => {
  await seedAndPlace(B, "YES", 100n * UNIT, "SELL", 7000, 100n * UNIT, 100n * UNIT); // too expensive
  const takerId = await seedAndPlace(A, "USDC", 1000n * UNIT, "BUY", 6000, 100n * UNIT, 60n * UNIT);
  const result = await engine.tryMatch(takerId);
  expect(result.trades).toHaveLength(0);
  expect(result.takerStatus).toBe("OPEN");
});

test("tryMatch refunds price-improvement to taker's free when maker price is better than taker limit", async () => {
  await seedAndPlace(B, "YES", 100n * UNIT, "SELL", 4000, 100n * UNIT, 100n * UNIT);
  const takerId = await seedAndPlace(A, "USDC", 1000n * UNIT, "BUY", 6000, 100n * UNIT, 60n * UNIT);
  await engine.tryMatch(takerId);
  // Taker paid 40 USDC at 0.40 (better than 0.60 limit). Started with 1000 free, reserved 60, executed at 40, refund 20 to free.
  // Invariant: balance free+reserved == (initial 1000 - 40 spent) = 960
  const usdc = await balanceRepo.get(A, MARKET, "USDC");
  expect(usdc!.free + usdc!.reserved).toBe(960n * UNIT);
  expect(usdc!.reserved).toBe(0n);
  expect(usdc!.free).toBe(960n * UNIT);
});
```

- [ ] **Step 7.2: Run, fail.**

- [ ] **Step 7.3: Implement:**

```typescript
import type { PrismaClient } from "../../../generated/prisma/client";
import type { OrderRepository } from "../repositories/order.repository";
import type { TradeRepository } from "../repositories/trade.repository";
import type { ISettler, MatchResult, TradeRecord } from "../types/trade.types";
import type { OrderView } from "../types/order.types";
import { decidePrimitive } from "./primitive-decider.service";

/**
 * Matching engine — price-time priority com FOR UPDATE SKIP LOCKED.
 *
 * Para cada round:
 *   1. Em uma tx curta, SELECT ... FOR UPDATE SKIP LOCKED da melhor contra-parte.
 *   2. Calcula fill = min(makerRemaining, takerRemaining).
 *   3. Decide primitiva via reservations.asset dos dois lados.
 *   4. Aplica fill nas duas ordens.
 *   5. Cria trade SETTLING.
 *   6. COMMIT.
 *   7. Chama settler.settle(tradeId) (fora da tx — settler abre a própria).
 *   8. Se taker ainda tem remaining e ainda há contra-parte viável, repete.
 *
 * Split settle em tx própria mantém a lock do book curta — settler roda em paralelo
 * a outros matches se forem em ordens diferentes.
 */
export class MatchingEngine {
  constructor(
    private readonly prisma: PrismaClient,
    private readonly orderRepo: OrderRepository,
    private readonly tradeRepo: TradeRepository,
    private readonly settler: ISettler,
  ) {}

  async tryMatch(takerOrderId: string): Promise<MatchResult> {
    const trades: TradeRecord[] = [];

    // Re-read taker outside loop to inspect final state
    let taker = await this.orderRepo.getById(takerOrderId);
    if (!taker) throw new Error(`taker order ${takerOrderId} not found`);
    if (taker.status !== "OPEN") {
      return { trades, takerFilled: taker.filled, takerRemaining: taker.quantity - taker.filled,
               takerStatus: taker.status === "FILLED" ? "FILLED" : "OPEN" };
    }

    while (true) {
      taker = await this.orderRepo.getById(takerOrderId);
      if (!taker || taker.status !== "OPEN") break;
      const takerRemaining = taker.quantity - taker.filled;
      if (takerRemaining <= 0n) break;

      const fillResult = await this.prisma.$transaction(async (tx) => {
        // Seleciona e trava a melhor contra-parte disponível.
        // SELECT ... FOR UPDATE SKIP LOCKED em raw SQL porque o Prisma client não
        // expõe SKIP LOCKED. A query retorna uma única linha; se nenhuma disponível,
        // volta vazio e saímos do loop.
        const opposing = taker!.side === "BUY" ? "SELL" : "BUY";
        const priceFilter = taker!.side === "BUY" ? "price_bps <= $3" : "price_bps >= $3";
        const orderBy = taker!.side === "BUY"
          ? "price_bps ASC, created_at ASC"
          : "price_bps DESC, created_at ASC";

        const rows: any[] = await tx.$queryRawUnsafe(
          `SELECT id, user_id, market_pda, side, price_bps, quantity, filled, reservation_id, client_order_id
             FROM ob2_orders
            WHERE market_pda = $1
              AND status = 'OPEN'
              AND side = $2::"Ob2Side"
              AND ${priceFilter}
            ORDER BY ${orderBy}
            LIMIT 1
            FOR UPDATE SKIP LOCKED`,
          taker!.marketPda, opposing, taker!.priceBps,
        );
        if (rows.length === 0) return null;

        const makerRow = rows[0];
        const makerQty = this.fromDecimal(String(makerRow.quantity));
        const makerFilled = this.fromDecimal(String(makerRow.filled));
        const makerRemaining = makerQty - makerFilled;
        const fillQty = makerRemaining < takerRemaining ? makerRemaining : takerRemaining;
        if (fillQty <= 0n) return null;

        // Decide primitive via reservations
        const makerRes = await tx.ob2Reservation.findUnique({ where: { id: makerRow.reservation_id } });
        const takerRes = await tx.ob2Reservation.findUnique({ where: { id: taker!.reservationId! } });
        if (!makerRes || !takerRes) throw new Error("reservation missing — cannot match");

        const primitive = decidePrimitive(taker!.side, takerRes.asset, makerRow.side, makerRes.asset);

        // Apply fills
        await this.applyFillRaw(tx, makerRow.id, fillQty, makerQty);
        await this.applyFillRaw(tx, taker!.id,    fillQty, taker!.quantity);

        // Create trade (status SETTLING, sync = true porque chegou via taker path)
        const tradeRow = await tx.ob2Trade.create({
          data: {
            marketPda: taker!.marketPda,
            makerOrderId: makerRow.id,
            takerOrderId: taker!.id,
            priceBps: makerRow.price_bps,           // maker price wins
            quantity: this.toDecimal(fillQty),
            primitive,
            status: "SETTLING",
            sync: true,
            settlingDeadline: new Date(Date.now() + 30_000),
          },
        });
        return {
          tradeId: tradeRow.id,
          fillQty,
          makerPriceBps: makerRow.price_bps,
          takerReservationId: takerRes.id,
          takerReservationAsset: takerRes.asset,
        };
      });

      if (!fillResult) break;

      // Settle outside the book-lock tx. StubSettler is idempotent.
      await this.settler.settle(fillResult.tradeId);

      // Price-improvement refund for the taker (if USDC-reserved and match was below limit):
      // We reserved at taker's priceBps; if maker price was better, refund the difference
      // to taker's free balance. Same logic for NO(BUY)×USDC(SELL) is handled inside StubSettler via
      // the usdcLeg calculation; this refund step applies when the TAKER reserved USDC
      // at a higher price than the actual fill price.
      if (fillResult.takerReservationAsset === "USDC") {
        await this.refundPriceImprovement(
          taker!.id, fillResult.takerReservationId,
          taker!.priceBps, fillResult.makerPriceBps, fillResult.fillQty, taker!.side,
        );
      }

      const trade = await this.tradeRepo.getById(fillResult.tradeId);
      if (trade) trades.push(trade);
    }

    const final = await this.orderRepo.getById(takerOrderId);
    return {
      trades,
      takerFilled: final?.filled ?? 0n,
      takerRemaining: (final?.quantity ?? 0n) - (final?.filled ?? 0n),
      takerStatus: final?.status === "FILLED" ? "FILLED" : "OPEN",
    };
  }

  /**
   * Quando o taker reservou USDC baseado no SEU limit price mas o fill ocorreu a
   * preço melhor (maker price vence), devolvemos a diferença pro free.
   */
  private async refundPriceImprovement(
    orderId: string, reservationId: string,
    takerLimitBps: number, makerPriceBps: number, fillQty: bigint, takerSide: "BUY" | "SELL",
  ): Promise<void> {
    let reservedAtLimit: bigint;
    let spentAtMakerPrice: bigint;
    if (takerSide === "BUY") {
      reservedAtLimit    = (fillQty * BigInt(takerLimitBps)) / 10000n;
      spentAtMakerPrice  = (fillQty * BigInt(makerPriceBps)) / 10000n;
    } else {
      // SELL (short) reserved at (1 - limit), fill at (1 - maker)
      reservedAtLimit    = (fillQty * BigInt(10000 - takerLimitBps)) / 10000n;
      spentAtMakerPrice  = (fillQty * BigInt(10000 - makerPriceBps)) / 10000n;
    }
    const refund = reservedAtLimit - spentAtMakerPrice;
    if (refund <= 0n) return;

    await this.prisma.$transaction(async (tx) => {
      // Move `refund` de reserved -> free, e diminui o amount da reservation
      const res = await tx.ob2Reservation.findUnique({ where: { id: reservationId } });
      if (!res || res.releasedAt) return;
      const current = this.fromDecimal(String(res.amount));
      if (refund > current) return;  // defensivo
      const amountStr = this.toDecimal(refund);
      await tx.$executeRawUnsafe(
        `UPDATE ob2_user_market_balances
            SET reserved = reserved - $4::numeric,
                free     = free     + $4::numeric,
                updated_at = now()
          WHERE user_id = (SELECT user_id FROM ob2_orders WHERE id = $1)
            AND market_pda = $2 AND asset = $3::"Ob2Asset"`,
        orderId, res.marketPda, res.asset, amountStr,
      );
      const remaining = current - refund;
      if (remaining === 0n) {
        await tx.ob2Reservation.update({ where: { id: reservationId }, data: { releasedAt: new Date() } });
      } else {
        await tx.ob2Reservation.update({ where: { id: reservationId }, data: { amount: this.toDecimal(remaining) } });
      }
    });
  }

  private async applyFillRaw(tx: any, orderId: string, amount: bigint, quantity: bigint): Promise<void> {
    const amountStr = this.toDecimal(amount);
    const qtyStr = this.toDecimal(quantity);
    await tx.$executeRawUnsafe(
      `UPDATE ob2_orders
          SET filled = filled + $2::numeric,
              status = CASE WHEN (filled + $2::numeric) >= $3::numeric THEN 'FILLED'::"Ob2OrderStatus"
                            ELSE status END,
              closed_at = CASE WHEN (filled + $2::numeric) >= $3::numeric THEN now() ELSE closed_at END,
              updated_at = now()
        WHERE id = $1`,
      orderId, amountStr, qtyStr,
    );
  }

  private toDecimal(v: bigint): string {
    const s = v.toString().padStart(7, "0");
    return `${s.slice(0, -6)}.${s.slice(-6)}`;
  }

  private fromDecimal(s: string): bigint {
    const [i, f = ""] = s.split(".");
    return BigInt(i + f.padEnd(6, "0").slice(0, 6));
  }
}
```

- [ ] **Step 7.4: Run, all pass.** Expected: 5 tests.

- [ ] **Step 7.5: Commit:**

```bash
git add api/src/modules/trading-v2/services/matching-engine.service.ts api/src/modules/trading-v2/__tests__/matching-engine.integration.test.ts
git commit -m "feat(trading-v2): MatchingEngine (FOR UPDATE SKIP LOCKED + price improvement)"
```

---

## Task 8: PlaceOrder use case

**Files:**
- Create: `api/src/modules/trading-v2/use-cases/place-order.use-case.ts`
- Create: `api/src/modules/trading-v2/__tests__/place-order.use-case.integration.test.ts`

Orquestra: classify intent → reserve → create order → tryMatch.

- [ ] **Step 8.1: Failing test:**

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
import { OrderRejectedError } from "../types/order.types";
import { InsufficientBalanceError } from "../types/reservation.types";
import { UNIT } from "../types/balance.types";

const balanceRepo = new BalanceRepository(prisma);
const resSvc = new ReservationService(prisma);
const orderRepo = new OrderRepository(prisma);
const tradeRepo = new TradeRepository(prisma);
const settler = new StubSettlerService(prisma);
const engine = new MatchingEngine(prisma, orderRepo, tradeRepo, settler);
const classifier = new IntentClassifier();

const useCase = new PlaceOrderUseCase(prisma, balanceRepo, resSvc, orderRepo, engine, classifier);

const A = "00000000-0000-0000-0000-000000000001";
const MARKET = "Market1111111111111111111111111111111111111";

beforeEach(async () => {
  await prisma.ob2Trade.deleteMany({});
  await prisma.ob2Order.deleteMany({});
  await prisma.ob2Reservation.deleteMany({});
  await prisma.ob2UserMarketBalance.deleteMany({});
});
afterAll(async () => { await prisma.$disconnect(); });

test("happy path: BUY with USDC available → OPEN order with reservation", async () => {
  await balanceRepo.upsertOnchain(A, MARKET, "USDC", 1000n * UNIT, 1n);
  const out = await useCase.execute({
    userId: A, marketPda: MARKET, side: "BUY",
    priceBps: 6000, quantity: 100n * UNIT, feeBps: 50,
  });
  expect(out.order.status).toBe("OPEN");
  expect(out.order.reservationId).not.toBeNull();
  expect(out.trades).toHaveLength(0);

  const usdc = await balanceRepo.get(A, MARKET, "USDC");
  // Reserve = 100 * 0.60 + fee(0.5%) = 60.30
  expect(usdc!.reserved).toBe(60_300_000n);
  expect(usdc!.free).toBe(1000n * UNIT - 60_300_000n);
});

test("rejects with InsufficientBalanceError when free < required", async () => {
  await balanceRepo.upsertOnchain(A, MARKET, "USDC", 10n * UNIT, 1n);
  await expect(useCase.execute({
    userId: A, marketPda: MARKET, side: "BUY",
    priceBps: 6000, quantity: 100n * UNIT, feeBps: 50,
  })).rejects.toBeInstanceOf(InsufficientBalanceError);
});

test("rejects clientOrderId duplicate", async () => {
  await balanceRepo.upsertOnchain(A, MARKET, "USDC", 1000n * UNIT, 1n);
  await useCase.execute({
    userId: A, marketPda: MARKET, side: "BUY",
    priceBps: 6000, quantity: 10n * UNIT, feeBps: 50, clientOrderId: "abc",
  });
  await expect(useCase.execute({
    userId: A, marketPda: MARKET, side: "BUY",
    priceBps: 6000, quantity: 10n * UNIT, feeBps: 50, clientOrderId: "abc",
  })).rejects.toBeInstanceOf(OrderRejectedError);
});

test("rejects invalid priceBps", async () => {
  await balanceRepo.upsertOnchain(A, MARKET, "USDC", 1000n * UNIT, 1n);
  await expect(useCase.execute({
    userId: A, marketPda: MARKET, side: "BUY",
    priceBps: 0, quantity: 100n * UNIT, feeBps: 50,
  })).rejects.toBeInstanceOf(OrderRejectedError);
  await expect(useCase.execute({
    userId: A, marketPda: MARKET, side: "BUY",
    priceBps: 10000, quantity: 100n * UNIT, feeBps: 50,
  })).rejects.toBeInstanceOf(OrderRejectedError);
});
```

- [ ] **Step 8.2: Run, fail.**

- [ ] **Step 8.3: Implement:**

```typescript
import type { PrismaClient } from "../../../generated/prisma/client";
import type { BalanceRepository } from "../repositories/balance.repository";
import type { ReservationService } from "../services/reservation.service";
import type { OrderRepository } from "../repositories/order.repository";
import type { MatchingEngine } from "../services/matching-engine.service";
import type { IntentClassifier } from "../services/intent-classifier.service";
import { OrderRejectedError, type PlaceOrderInput, type OrderView } from "../types/order.types";
import type { TradeRecord } from "../types/trade.types";

export interface PlaceOrderResult {
  order: OrderView;
  trades: TradeRecord[];
}

export class PlaceOrderUseCase {
  constructor(
    private readonly prisma: PrismaClient,
    private readonly balanceRepo: BalanceRepository,
    private readonly reservations: ReservationService,
    private readonly orders: OrderRepository,
    private readonly engine: MatchingEngine,
    private readonly classifier: IntentClassifier,
  ) {}

  async execute(input: PlaceOrderInput): Promise<PlaceOrderResult> {
    // Validate input
    if (input.priceBps <= 0 || input.priceBps >= 10000) {
      throw new OrderRejectedError("invalid_price", `priceBps must be in (0, 10000), got ${input.priceBps}`);
    }
    if (input.quantity <= 0n) {
      throw new OrderRejectedError("invalid_quantity", `quantity must be positive`);
    }

    // Idempotência por clientOrderId
    if (input.clientOrderId) {
      const existing = await this.prisma.ob2Order.findUnique({
        where: { userId_clientOrderId: { userId: input.userId, clientOrderId: input.clientOrderId } },
      });
      if (existing) {
        throw new OrderRejectedError("duplicate_client_order_id", `clientOrderId ${input.clientOrderId} already used`);
      }
    }

    // Classify intent — reads free balances
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

    // Create the order shell + reservation. Two-step because order.id is the FK.
    const order = await this.orders.create({
      userId: input.userId, marketPda: input.marketPda, side: input.side,
      priceBps: input.priceBps, quantity: input.quantity,
      reservationId: "pending", clientOrderId: input.clientOrderId,
    });

    let reservation;
    try {
      reservation = await this.reservations.reserve({
        userId: input.userId, marketPda: input.marketPda, asset, amount, orderId: order.id,
      });
    } catch (e) {
      // Rollback the order shell
      await this.prisma.ob2Order.update({
        where: { id: order.id },
        data: { status: "REJECTED", rejectReason: "insufficient_balance", closedAt: new Date() },
      });
      throw e;
    }

    // Link the reservation to the order
    await this.prisma.ob2Order.update({
      where: { id: order.id }, data: { reservationId: reservation.id },
    });

    // Now attempt matching
    const matchResult = await this.engine.tryMatch(order.id);

    const finalOrder = await this.orders.getById(order.id);
    return { order: finalOrder!, trades: matchResult.trades };
  }
}
```

- [ ] **Step 8.4: Run, 4 pass.**

- [ ] **Step 8.5: Commit:**

```bash
git add api/src/modules/trading-v2/use-cases/place-order.use-case.ts api/src/modules/trading-v2/__tests__/place-order.use-case.integration.test.ts
git commit -m "feat(trading-v2): PlaceOrderUseCase (classify → reserve → match)"
```

---

## Task 9: CancelOrder use case

**Files:**
- Create: `api/src/modules/trading-v2/use-cases/cancel-order.use-case.ts`
- Create: `api/src/modules/trading-v2/__tests__/cancel-order.use-case.integration.test.ts`

- [ ] **Step 9.1: Failing test:**

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
import { CancelOrderUseCase } from "../use-cases/cancel-order.use-case";
import { OrderNotFoundError, OrderNotCancellableError } from "../types/order.types";
import { UNIT } from "../types/balance.types";

const balanceRepo = new BalanceRepository(prisma);
const resSvc = new ReservationService(prisma);
const orderRepo = new OrderRepository(prisma);
const tradeRepo = new TradeRepository(prisma);
const settler = new StubSettlerService(prisma);
const engine = new MatchingEngine(prisma, orderRepo, tradeRepo, settler);
const classifier = new IntentClassifier();
const place = new PlaceOrderUseCase(prisma, balanceRepo, resSvc, orderRepo, engine, classifier);
const cancel = new CancelOrderUseCase(prisma, orderRepo, resSvc);

const A = "00000000-0000-0000-0000-000000000001";
const MARKET = "Market1111111111111111111111111111111111111";

beforeEach(async () => {
  await prisma.ob2Trade.deleteMany({});
  await prisma.ob2Order.deleteMany({});
  await prisma.ob2Reservation.deleteMany({});
  await prisma.ob2UserMarketBalance.deleteMany({});
});
afterAll(async () => { await prisma.$disconnect(); });

test("cancel OPEN order: releases reservation, marks CANCELLED", async () => {
  await balanceRepo.upsertOnchain(A, MARKET, "USDC", 1000n * UNIT, 1n);
  const { order } = await place.execute({
    userId: A, marketPda: MARKET, side: "BUY",
    priceBps: 6000, quantity: 100n * UNIT, feeBps: 50,
  });
  expect(order.status).toBe("OPEN");

  const result = await cancel.execute({ userId: A, orderId: order.id });
  expect(result.status).toBe("CANCELLED");

  const usdc = await balanceRepo.get(A, MARKET, "USDC");
  expect(usdc!.free).toBe(1000n * UNIT);
  expect(usdc!.reserved).toBe(0n);
});

test("cancel of not-owned order throws OrderNotFoundError", async () => {
  await balanceRepo.upsertOnchain(A, MARKET, "USDC", 1000n * UNIT, 1n);
  const { order } = await place.execute({
    userId: A, marketPda: MARKET, side: "BUY",
    priceBps: 6000, quantity: 100n * UNIT, feeBps: 50,
  });
  await expect(cancel.execute({
    userId: "00000000-0000-0000-0000-000000000009",
    orderId: order.id,
  })).rejects.toBeInstanceOf(OrderNotFoundError);
});

test("cancel of missing orderId throws OrderNotFoundError", async () => {
  await expect(cancel.execute({
    userId: A, orderId: "00000000-0000-0000-0000-000000000dead",
  })).rejects.toBeInstanceOf(OrderNotFoundError);
});

test("cancel of already-CANCELLED order throws OrderNotCancellableError", async () => {
  await balanceRepo.upsertOnchain(A, MARKET, "USDC", 1000n * UNIT, 1n);
  const { order } = await place.execute({
    userId: A, marketPda: MARKET, side: "BUY",
    priceBps: 6000, quantity: 100n * UNIT, feeBps: 50,
  });
  await cancel.execute({ userId: A, orderId: order.id });
  await expect(cancel.execute({ userId: A, orderId: order.id }))
    .rejects.toBeInstanceOf(OrderNotCancellableError);
});
```

- [ ] **Step 9.2: Run, fail.**

- [ ] **Step 9.3: Implement:**

```typescript
import type { PrismaClient } from "../../../generated/prisma/client";
import type { OrderRepository } from "../repositories/order.repository";
import type { ReservationService } from "../services/reservation.service";
import { OrderNotFoundError, OrderNotCancellableError, type OrderView } from "../types/order.types";

export interface CancelOrderInput {
  userId: string;
  orderId: string;
}

export class CancelOrderUseCase {
  constructor(
    private readonly prisma: PrismaClient,
    private readonly orders: OrderRepository,
    private readonly reservations: ReservationService,
  ) {}

  async execute(input: CancelOrderInput): Promise<OrderView> {
    const order = await this.orders.getById(input.orderId);
    if (!order) throw new OrderNotFoundError(input.orderId);
    if (order.userId !== input.userId) throw new OrderNotFoundError(input.orderId); // no disclosure
    if (order.status !== "OPEN") throw new OrderNotCancellableError(order.status);

    // Release the remaining reservation and mark cancelled.
    // Both actions must happen together — if release fails, don't cancel.
    if (order.reservationId) {
      await this.reservations.release(order.reservationId);
    }
    return this.orders.markCancelled(order.id);
  }
}
```

- [ ] **Step 9.4: Run, 4 pass.**

- [ ] **Step 9.5: Commit:**

```bash
git add api/src/modules/trading-v2/use-cases/cancel-order.use-case.ts api/src/modules/trading-v2/__tests__/cancel-order.use-case.integration.test.ts
git commit -m "feat(trading-v2): CancelOrderUseCase"
```

---

## Task 10: HTTP Routes (Hono)

**Files:**
- Create: `api/src/modules/trading-v2/schemas/place-order.schema.ts`
- Create: `api/src/modules/trading-v2/routes/orders.routes.ts`
- Modify: `api/src/app.ts` (mount /v2 routes)

- [ ] **Step 10.1: Zod schema:**

```typescript
import { z } from "zod";

export const placeOrderSchema = z.object({
  marketPda: z.string().min(32).max(44),
  side: z.enum(["BUY", "SELL"]),
  priceBps: z.number().int().min(1).max(9999),
  quantity: z.string().regex(/^\d+$/),    // bigint-as-string in micro-units
  feeBps: z.number().int().min(0).max(10000).optional().default(50),
  clientOrderId: z.string().max(64).optional(),
});

export type PlaceOrderRequestBody = z.infer<typeof placeOrderSchema>;
```

- [ ] **Step 10.2: Routes:**

```typescript
/**
 * Rotas v2 — operam sobre o novo orderbook (trading-v2).
 * NÃO integradas com prediction-market/trading/ antigo. Em paralelo.
 */
import { Hono } from "hono";
import { zValidator } from "@hono/zod-validator";
import { prisma } from "@/shared/database/config/prisma-client";
import { getAuthenticatedUser } from "@/shared/middleware/clerk-auth.middleware";
import { errorResponse, successResponse } from "@/shared/utils/response";
import { BalanceRepository } from "../repositories/balance.repository";
import { ReservationService } from "../services/reservation.service";
import { OrderRepository } from "../repositories/order.repository";
import { TradeRepository } from "../repositories/trade.repository";
import { StubSettlerService } from "../services/stub-settler.service";
import { MatchingEngine } from "../services/matching-engine.service";
import { IntentClassifier } from "../services/intent-classifier.service";
import { PlaceOrderUseCase } from "../use-cases/place-order.use-case";
import { CancelOrderUseCase } from "../use-cases/cancel-order.use-case";
import {
  OrderRejectedError, OrderNotFoundError, OrderNotCancellableError,
} from "../types/order.types";
import { InsufficientBalanceError } from "../types/reservation.types";
import { placeOrderSchema } from "../schemas/place-order.schema";

const balanceRepo = new BalanceRepository(prisma);
const resSvc = new ReservationService(prisma);
const orderRepo = new OrderRepository(prisma);
const tradeRepo = new TradeRepository(prisma);
const settler = new StubSettlerService(prisma);
const engine = new MatchingEngine(prisma, orderRepo, tradeRepo, settler);
const classifier = new IntentClassifier();
const placeOrder = new PlaceOrderUseCase(prisma, balanceRepo, resSvc, orderRepo, engine, classifier);
const cancelOrder = new CancelOrderUseCase(prisma, orderRepo, resSvc);

const orders = new Hono();

orders.post("/orders", zValidator("json", placeOrderSchema), async (c) => {
  const user = await getAuthenticatedUser(c);
  if (!user) return errorResponse(c, "não autenticado", null, 401);

  const body = c.req.valid("json");
  try {
    const result = await placeOrder.execute({
      userId: user.id,
      marketPda: body.marketPda,
      side: body.side,
      priceBps: body.priceBps,
      quantity: BigInt(body.quantity),
      feeBps: body.feeBps,
      clientOrderId: body.clientOrderId,
    });
    return successResponse(c, {
      order: serializeOrder(result.order),
      trades: result.trades.map(serializeTrade),
    });
  } catch (e) {
    if (e instanceof OrderRejectedError)         return errorResponse(c, e.message, { code: e.code }, 400);
    if (e instanceof InsufficientBalanceError)   return errorResponse(c, e.message, { code: "insufficient_balance" }, 400);
    throw e;
  }
});

orders.delete("/orders/:id", async (c) => {
  const user = await getAuthenticatedUser(c);
  if (!user) return errorResponse(c, "não autenticado", null, 401);

  const orderId = c.req.param("id");
  try {
    const cancelled = await cancelOrder.execute({ userId: user.id, orderId });
    return successResponse(c, serializeOrder(cancelled));
  } catch (e) {
    if (e instanceof OrderNotFoundError)          return errorResponse(c, "not found", null, 404);
    if (e instanceof OrderNotCancellableError)    return errorResponse(c, e.message, { status: e.status }, 409);
    throw e;
  }
});

orders.get("/orders", async (c) => {
  const user = await getAuthenticatedUser(c);
  if (!user) return errorResponse(c, "não autenticado", null, 401);
  const marketPda = c.req.query("marketPda");
  const status = c.req.query("status");
  if (!marketPda) return errorResponse(c, "marketPda required", null, 400);
  const list = await orderRepo.listByUser(user.id, marketPda,
    status === "OPEN" ? { status: "OPEN" } : {},
  );
  return successResponse(c, list.map(serializeOrder));
});

function serializeOrder(o: any) {
  return {
    id: o.id, userId: o.userId, marketPda: o.marketPda, side: o.side,
    priceBps: o.priceBps, quantity: o.quantity.toString(), filled: o.filled.toString(),
    status: o.status, clientOrderId: o.clientOrderId,
    createdAt: o.createdAt, closedAt: o.closedAt,
  };
}

function serializeTrade(t: any) {
  return {
    id: t.id, marketPda: t.marketPda, priceBps: t.priceBps,
    quantity: t.quantity.toString(), primitive: t.primitive,
    status: t.status, createdAt: t.createdAt,
  };
}

export default orders;
```

- [ ] **Step 10.3: Mount routes in `api/src/app.ts`.**

Locate the section where other route modules are mounted (grep for `app.route(` or `app.mount(`). Add:

```typescript
import v2Orders from "./modules/trading-v2/routes/orders.routes";
// ...
app.route("/api/v2/trading", v2Orders);
```

Place it near other module mounts, not interleaved with middleware.

- [ ] **Step 10.4: Smoke test via `bun run`:**

```bash
cd /Users/rafael/Documents/GitHub/Oraculo/.claude/worktrees/orderbook-rewrite-02-orders-matching/api
# Start server on a temp port and check the routes exist
PORT=3999 bun run index.ts &
SERVER_PID=$!
sleep 2
curl -s -X POST http://localhost:3999/api/v2/trading/orders -H "Content-Type: application/json" -d '{}' | head -c 500
kill $SERVER_PID 2>/dev/null
```

Expected: 401 response (not authenticated) — proves the route is mounted and responds. If you get 404, the mount is wrong.

- [ ] **Step 10.5: Commit:**

```bash
git add api/src/modules/trading-v2/schemas/ api/src/modules/trading-v2/routes/ api/src/app.ts
git commit -m "feat(trading-v2): HTTP routes /api/v2/trading/orders"
```

---

## Task 11: E2E scenarios (3 primitives, multi-user)

**Files:**
- Create: `api/src/modules/trading-v2/__tests__/matching-scenarios.e2e.test.ts`

Teste end-to-end que exercita cada primitiva no use-case level (não HTTP), com dois usuários reais.

- [ ] **Step 11.1: Test file:**

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

const A = "00000000-0000-0000-0000-00000000000a";
const B = "00000000-0000-0000-0000-00000000000b";
const MARKET = "Market1111111111111111111111111111111111111";

beforeEach(async () => {
  await prisma.ob2Trade.deleteMany({});
  await prisma.ob2Order.deleteMany({});
  await prisma.ob2Reservation.deleteMany({});
  await prisma.ob2UserMarketBalance.deleteMany({});
});
afterAll(async () => { await prisma.$disconnect(); });

test("TRADE: A buys YES from B, B had YES", async () => {
  await balanceRepo.upsertOnchain(A, MARKET, "USDC", 1000n * UNIT, 1n);
  await balanceRepo.upsertOnchain(B, MARKET, "YES",  100n  * UNIT, 1n);

  // B posts SELL @ 0.50
  await place.execute({ userId: B, marketPda: MARKET, side: "SELL", priceBps: 5000, quantity: 100n * UNIT, feeBps: 0 });
  // A buys @ 0.60 (willing to pay up to)
  const r = await place.execute({ userId: A, marketPda: MARKET, side: "BUY",  priceBps: 6000, quantity: 100n * UNIT, feeBps: 0 });
  expect(r.trades).toHaveLength(1);
  expect(r.trades[0].primitive).toBe("TRADE");
  expect((await balanceRepo.get(A, MARKET, "YES"))!.free).toBe(100n * UNIT);
  expect((await balanceRepo.get(B, MARKET, "USDC"))!.free).toBe(50n * UNIT);
});

test("MINT: A buys YES, B sells without having YES → both pay USDC, vault mints pair", async () => {
  await balanceRepo.upsertOnchain(A, MARKET, "USDC", 1000n * UNIT, 1n);
  await balanceRepo.upsertOnchain(B, MARKET, "USDC", 1000n * UNIT, 1n);

  // B posts SELL @ 0.60 with no YES — classifier will reserve USDC (short open)
  await place.execute({ userId: B, marketPda: MARKET, side: "SELL", priceBps: 6000, quantity: 100n * UNIT, feeBps: 0 });
  const r = await place.execute({ userId: A, marketPda: MARKET, side: "BUY",  priceBps: 6000, quantity: 100n * UNIT, feeBps: 0 });
  expect(r.trades).toHaveLength(1);
  expect(r.trades[0].primitive).toBe("MINT");
  // A gets YES, B gets NO
  expect((await balanceRepo.get(A, MARKET, "YES"))!.free).toBe(100n * UNIT);
  expect((await balanceRepo.get(B, MARKET, "NO"))!.free).toBe(100n * UNIT);
});

test("MERGE: A holds NO, B holds YES, both close → both get USDC", async () => {
  // Seed: A has NO, B has YES, both want to close (A sends BUY side → will fill with NO reserve per classifier; B sends SELL with YES)
  await balanceRepo.upsertOnchain(A, MARKET, "NO",  100n * UNIT, 1n);
  await balanceRepo.upsertOnchain(B, MARKET, "YES", 100n * UNIT, 1n);

  await place.execute({ userId: B, marketPda: MARKET, side: "SELL", priceBps: 6000, quantity: 100n * UNIT, feeBps: 0 });
  const r = await place.execute({ userId: A, marketPda: MARKET, side: "BUY",  priceBps: 6000, quantity: 100n * UNIT, feeBps: 0 });
  expect(r.trades).toHaveLength(1);
  expect(r.trades[0].primitive).toBe("MERGE");
  // A (was short YES = long NO) at 0.60 receives (1-0.60)*100 = 40 USDC
  expect((await balanceRepo.get(A, MARKET, "USDC"))!.free).toBe(40n * UNIT);
  // B (was long YES) receives 0.60*100 = 60 USDC
  expect((await balanceRepo.get(B, MARKET, "USDC"))!.free).toBe(60n * UNIT);
});

test("Invariant I2 holds across random multi-user sequence of 30 orders", async () => {
  await balanceRepo.upsertOnchain(A, MARKET, "USDC", 10_000n * UNIT, 1n);
  await balanceRepo.upsertOnchain(B, MARKET, "USDC", 10_000n * UNIT, 1n);

  let rng = 123;
  const next = () => { rng = (rng * 1103515245 + 12345) & 0x7fffffff; return rng; };

  for (let i = 0; i < 30; i++) {
    const u = (next() % 2) === 0 ? A : B;
    const side = (next() % 2) === 0 ? "BUY" as const : "SELL" as const;
    const priceBps = 3000 + (next() % 4000);
    const qty = BigInt((next() % 20) + 1) * UNIT;
    try {
      await place.execute({ userId: u, marketPda: MARKET, side, priceBps, quantity: qty, feeBps: 0 });
    } catch { /* insufficient balance acceptable as random inputs exceed budget */ }
  }

  // Invariant I2 per user per asset: reserved == sum of active reservations
  for (const u of [A, B]) {
    for (const asset of ["USDC", "YES", "NO"] as const) {
      const bal = await balanceRepo.get(u, MARKET, asset);
      if (!bal) continue;
      const activeSum = await prisma.ob2Reservation.aggregate({
        _sum: { amount: true },
        where: { userId: u, marketPda: MARKET, asset, releasedAt: null },
      });
      const sumMicro = activeSum._sum.amount
        ? BigInt(String(activeSum._sum.amount).replace(".", "").padEnd(7, "0").slice(0, -6) + "000000")
        : 0n;
      // Simpler: just use the comparable decimals
      const reservedStr = bal.reserved.toString();
      const sumStr = String(activeSum._sum.amount ?? "0");
      // Normalize both to bigint micro
      const reservedMicro = bal.reserved;
      const expectedMicro = sumStr === "0" ? 0n : (() => {
        const [ii, ff = ""] = sumStr.split(".");
        return BigInt(ii + ff.padEnd(6, "0").slice(0, 6));
      })();
      expect(reservedMicro).toBe(expectedMicro);
    }
  }
});
```

- [ ] **Step 11.2: Run:**

```bash
cd /Users/rafael/Documents/GitHub/Oraculo/.claude/worktrees/orderbook-rewrite-02-orders-matching/api
bun x jest src/modules/trading-v2/__tests__/matching-scenarios.e2e.test.ts --runInBand
```

Expected: 4 tests passed.

- [ ] **Step 11.3: Commit:**

```bash
git add api/src/modules/trading-v2/__tests__/matching-scenarios.e2e.test.ts
git commit -m "test(trading-v2): e2e dos 3 primitives + invariante I2 multi-user"
```

---

## Task 12: Suíte final + README

**Files:**
- Modify: `api/src/modules/trading-v2/index.ts` (export novos services)
- Modify: `api/src/modules/trading-v2/README.md`

- [ ] **Step 12.1: Update `index.ts`:**

```typescript
// Types
export * from "./types/balance.types";
export * from "./types/reservation.types";
export * from "./types/intent.types";
export * from "./types/order.types";
export * from "./types/trade.types";

// Repositories
export { BalanceRepository } from "./repositories/balance.repository";
export { OrderRepository } from "./repositories/order.repository";
export { TradeRepository } from "./repositories/trade.repository";

// Services
export { BalanceService } from "./services/balance.service";
export { ReservationService } from "./services/reservation.service";
export { IntentClassifier } from "./services/intent-classifier.service";
export { ReconciliationService, DUST_MICRO } from "./services/reconciliation.service";
export { decidePrimitive, ImpossiblePairError } from "./services/primitive-decider.service";
export { StubSettlerService } from "./services/stub-settler.service";
export { MatchingEngine } from "./services/matching-engine.service";

// Use cases
export { PlaceOrderUseCase } from "./use-cases/place-order.use-case";
export { CancelOrderUseCase } from "./use-cases/cancel-order.use-case";
```

- [ ] **Step 12.2: Update `README.md`:** substitua por

```markdown
# trading-v2

Novo orderbook. Planos 1 e 2 entregues: fundação (balances/reservations/intent)
+ orders lifecycle + matching engine síncrono com stub de settlement.

Spec: `docs/superpowers/specs/2026-04-15-orderbook-rewrite-design.md`
Planos:
- 1: `docs/superpowers/plans/2026-04-15-orderbook-rewrite-01-foundation.md` (merged)
- 2: `docs/superpowers/plans/2026-04-15-orderbook-rewrite-02-orders-matching.md` (este)

## Endpoints HTTP

- `POST /api/v2/trading/orders`   — place order
- `DELETE /api/v2/trading/orders/:id` — cancel
- `GET /api/v2/trading/orders?marketPda=X[&status=OPEN]` — list

## Invariantes verificadas

- **I1** (reserva obrigatória atômica): `__tests__/reservation.service.invariant-i1.test.ts`
- **I2** (conservação): `__tests__/balance.service.invariant-i2.test.ts` + cenário multi-user em `matching-scenarios.e2e.test.ts`

## Settlement

`StubSettlerService` faz as mutações locais no DB que o settlement real fará.
Plano 3 substitui por caller on-chain mantendo a interface `ISettler`.

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

- 3: Settlement sync/async real on-chain + listener + revert
- 4: `settle_fill` no programa Solana
- 5: WebSocket v2
- 6: MM bot externo
- 7: Cutover
```

- [ ] **Step 12.3: Rodar suíte inteira:**

```bash
cd /Users/rafael/Documents/GitHub/Oraculo/.claude/worktrees/orderbook-rewrite-02-orders-matching/api
bun x jest src/modules/trading-v2 --runInBand
```

Expected (27 do Plano 1 + novos):
- `balance.repository.integration.test.ts` — 3
- `reservation.service.integration.test.ts` — 10
- `reservation.service.invariant-i1.test.ts` — 2
- `balance.service.invariant-i2.test.ts` — 2
- `intent-classifier.unit.test.ts` — 5
- `reconciliation.service.unit.test.ts` — 3
- `snapshot-onchain-balances.integration.test.ts` — 2
- `order.repository.integration.test.ts` — 9
- `trade.repository.integration.test.ts` — 5
- `primitive-decider.unit.test.ts` — 5
- `stub-settler.integration.test.ts` — 4
- `matching-engine.integration.test.ts` — 5
- `place-order.use-case.integration.test.ts` — 4
- `cancel-order.use-case.integration.test.ts` — 4
- `matching-scenarios.e2e.test.ts` — 4

**Total esperado: 67 tests.**

- [ ] **Step 12.4: Type-check:**

```bash
bun x tsc --noEmit 2>&1 | grep "src/modules/trading-v2" || echo "clean"
```

Expected: `clean`.

- [ ] **Step 12.5: Commit:**

```bash
git add api/src/modules/trading-v2/index.ts api/src/modules/trading-v2/README.md
git commit -m "docs(trading-v2): barrel export + README pós-plano 2"
```

---

## Critérios de aceitação do plano

1. ✅ 8 novos arquivos de teste (+ 2 ampliados do plano 1 se necessário), total 40 testes novos (67 no módulo inteiro).
2. ✅ Schema adicional aplicado (`orders`, `trades`, `onchain_events_processed`) sem tocar schema antigo.
3. ✅ `OrderRepository`, `TradeRepository`, `PrimitiveDecider`, `StubSettler`, `MatchingEngine`, `PlaceOrderUseCase`, `CancelOrderUseCase` implementados.
4. ✅ Rotas HTTP `/api/v2/trading/*` expostas e testadas via smoke test (401 comprova mount).
5. ✅ Matching exercitado nos 3 primitives (TRADE, MINT, MERGE) em cenário e2e multi-user.
6. ✅ Invariante I2 mantida ao longo de sequência aleatória multi-user de 30 orders.
7. ✅ `ISettler` é interface estável — Plano 3 apenas troca implementação.
8. ✅ Zero modificação em `prediction-market/trading/` antigo.

---

## Riscos e mitigações

| Risco | Mitigação |
|---|---|
| `FOR UPDATE SKIP LOCKED` não disponível ou se comporta diferente no Prisma adapter | Confirmar via teste dedicado: dois takers concorrentes contra 1 maker devem resultar em 1 trade, não 2 duplicados (adicionar se necessário). Raw SQL via `$queryRawUnsafe` garante o SKIP LOCKED literal. |
| StubSettler diverge da semântica real que o Plano 3 implementará | O spec §6.2 é a fonte; StubSettler replica a mesma tabela de decisão. Quando Plano 3 escrever o settler real, os mesmos cenários e2e devem passar. Se divergirem, é bug no real, não no stub. |
| Price-improvement refund pode introduzir drift I2 | Refund é feito atomicamente (tx) dentro do engine, com consumePartial equivalente da reserva. Property test de I2 multi-user cobre isso. |
| Order shell criado antes da reservation pode deixar "orphan REJECTED" na tabela | Aceitável: serve de trilha de auditoria. Query padrão do usuário filtra `status IN (OPEN, FILLED, CANCELLED)`. |
| Rotas HTTP ainda não cobertas por teste automatizado | Smoke test (Step 10.4) só verifica mount. E2E HTTP full-flow fica para um follow-up; o use-case está testado, que é onde mora a lógica. |
| `refundPriceImprovement` calcula refund ignorando a porção de fee dentro da reserva | Plan 2 e2e usam `feeBps=0` e o cálculo bate. Plano 3 integra com fee ledger — momento de ajustar a conta do refund pra descontar a parcela de fee proporcional. Documentar claramente quando fee for introduzido. |

---

## O que NÃO está neste plano

- **Settlement on-chain real**: Plano 3.
- **Async maker path / worker de deadline**: Plano 3.
- **Listener on-chain**: Plano 3.
- **Fee ledger integration**: Plano 3 (o `feeBps` já flui pela API mas não é debitado pra fee wallet neste plano).
- **WebSocket**: Plano 5.
- **MM bot**: Plano 6.
- **Cutover e migração dos mercados atuais**: Plano 7.
