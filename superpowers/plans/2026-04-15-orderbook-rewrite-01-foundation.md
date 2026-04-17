# Orderbook Rewrite — Plano 1: Fundação (Schema + Balance + Reservation)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Entregar a camada de fundação do novo orderbook: schema Postgres novo (isolado do atual), `BalanceService` e `ReservationService` com atomicidade garantida, e suíte de testes das invariantes I1 (reserva obrigatória) e I2 (conservação de saldo). Ao final, essas duas invariantes são verdadeiras por construção e testadas automaticamente.

**Architecture:** Novas tabelas convivem com as antigas num namespace `ob2_*` (orderbook v2) durante toda a construção — o sistema atual continua rodando intocado. Todo código novo mora em `api/src/modules/trading-v2/` sem importar do `prediction-market/trading/` antigo. Cutover acontece em um plano posterior.

**Tech Stack:** Bun, TypeScript, Hono, Prisma 7 (postgres), Jest, `@prisma/adapter-pg`.

**Spec:** `docs/superpowers/specs/2026-04-15-orderbook-rewrite-design.md`

**Planos subsequentes (fora deste):**
- Plano 2: Order lifecycle (place, cancel, estados) + matching engine sync
- Plano 3: Settlement sync/async + listener on-chain + revert
- Plano 4: Instrução `settle_fill` no programa Solana
- Plano 5: WebSocket sobre novo modelo
- Plano 6: MM bot externo
- Plano 7: Cutover + reconciliação diária

---

## File Structure

**Novos arquivos (todos em `api/src/modules/trading-v2/`):**

```
api/
  prisma/
    schema.prisma                          # MODIFY: adiciona enums ob2_* + 2 tabelas
    migrations/
      <timestamp>_trading_v2_foundation/   # CREATE: migration SQL
        migration.sql
  src/modules/trading-v2/
    index.ts                               # CREATE: barrel export
    types/
      balance.types.ts                     # CREATE: Asset, Ob2MarketBalance, etc.
      reservation.types.ts                 # CREATE: ReservationInput/Result
      intent.types.ts                      # CREATE: Intent classification types
    repositories/
      balance.repository.ts                # CREATE: reads/writes user_market_balances
      reservation.repository.ts            # CREATE: reads/writes reservations
    services/
      balance.service.ts                   # CREATE: free/reserved operations
      reservation.service.ts               # CREATE: reserve/release com atomicidade
      intent-classifier.service.ts         # CREATE: §5.1.1 classify()
      reconciliation.service.ts            # CREATE: detecta drift (no auto-fix)
    scripts/
      snapshot-onchain-balances.ts         # CREATE: importa saldos on-chain p/ DB
    __tests__/
      balance.repository.integration.test.ts
      reservation.service.integration.test.ts
      reservation.service.invariant-i1.test.ts
      balance.service.invariant-i2.test.ts
      intent-classifier.unit.test.ts
      reconciliation.service.unit.test.ts
```

**Responsabilidades:**

- `types/` — só types/interfaces, sem runtime.
- `repositories/` — SQL puro via Prisma. Sem regra de negócio. Cada método = 1 query ou transação pequena.
- `services/` — regras de negócio. Não chamam Prisma direto; usam repos.
- `scripts/` — CLI one-off executável via `bun run`.
- `__tests__/` — um arquivo por componente testado + arquivos dedicados pras invariantes.

---

## Prerequisite check

- [ ] **Step 0: Confirmar setup**

Run:
```bash
cd /Users/rafael/Documents/GitHub/Oraculo/api
bun --version     # Expected: 1.x
bun x prisma --version  # Expected: 7.x
ls src/generated/prisma  # Expected: directory exists
```

Se qualquer comando falhar, pare e resolva antes de prosseguir.

---

## Task 1: Adicionar enums e tabelas ob2_* ao schema Prisma

**Files:**
- Modify: `api/prisma/schema.prisma` (append ao final)
- Create: `api/prisma/migrations/<auto>_trading_v2_foundation/migration.sql` (gerada pelo Prisma)

- [ ] **Step 1.1: Acrescentar ao final de `api/prisma/schema.prisma`:**

```prisma
// ─────────────────────────────────────────────
// ORDERBOOK V2 (trading-v2) — novas tabelas
// Ver docs/superpowers/specs/2026-04-15-orderbook-rewrite-design.md §4
// ─────────────────────────────────────────────

enum Ob2Asset {
  USDC
  YES
  NO
}

enum Ob2Side {
  BUY
  SELL
}

enum Ob2OrderStatus {
  OPEN
  FILLED
  CANCELLED
  REJECTED
}

enum Ob2TradeStatus {
  SETTLING
  SETTLED
  REVERTED
}

enum Ob2Primitive {
  TRADE
  MINT
  MERGE
}

model Ob2UserMarketBalance {
  userId        String    @map("user_id")
  marketPda     String    @map("market_pda")
  asset         Ob2Asset
  free          Decimal   @default(0) @db.Decimal(20, 6)
  reserved      Decimal   @default(0) @db.Decimal(20, 6)
  onchainTotal  Decimal   @default(0) @map("onchain_total") @db.Decimal(20, 6)
  onchainSlot   BigInt?   @map("onchain_slot")
  updatedAt     DateTime  @default(now()) @map("updated_at")

  @@id([userId, marketPda, asset])
  @@map("ob2_user_market_balances")
}

model Ob2Reservation {
  id         String    @id @default(uuid())
  userId     String    @map("user_id")
  marketPda  String    @map("market_pda")
  asset      Ob2Asset
  amount     Decimal   @db.Decimal(20, 6)
  orderId    String    @map("order_id")
  releasedAt DateTime? @map("released_at")
  createdAt  DateTime  @default(now()) @map("created_at")

  @@index([orderId], map: "ob2_reservations_order_idx")
  @@map("ob2_reservations")
}
```

Nota: `orders` e `trades` não entram neste plano — são do Plano 2. Mantemos o schema enxuto pra desacoplar entregas.

- [ ] **Step 1.2: Criar script SQL de constraints/índices extra**

O projeto usa `prisma db push` + scripts SQL em `prisma/scripts/` (padrão visto em `feat-3xchange-payout.sql`). Prisma schema não expressa CHECK constraints nem índices parciais.

Create `api/prisma/scripts/trading-v2-foundation.sql`:

```sql
-- Idempotente: pode ser rodado quantas vezes. Cada bloco testa existência antes.

DO $$ BEGIN
  ALTER TABLE ob2_user_market_balances
    ADD CONSTRAINT ob2_balance_free_nonneg CHECK (free >= 0);
EXCEPTION WHEN duplicate_object THEN NULL; END $$;

DO $$ BEGIN
  ALTER TABLE ob2_user_market_balances
    ADD CONSTRAINT ob2_balance_reserved_nonneg CHECK (reserved >= 0);
EXCEPTION WHEN duplicate_object THEN NULL; END $$;

DO $$ BEGIN
  ALTER TABLE ob2_reservations
    ADD CONSTRAINT ob2_reservation_amount_positive CHECK (amount > 0);
EXCEPTION WHEN duplicate_object THEN NULL; END $$;

CREATE INDEX IF NOT EXISTS ob2_reservations_active_idx
  ON ob2_reservations (user_id, market_pda, asset)
  WHERE released_at IS NULL;
```

- [ ] **Step 1.3: Aplicar schema + script**

Run:
```bash
cd /Users/rafael/Documents/GitHub/Oraculo/.claude/worktrees/orderbook-rewrite-01-foundation/api
bun x prisma db push --skip-generate
bun x prisma generate
# Aplicar CHECKs e índices
DATABASE_URL_PSQL=$(grep "^DATABASE_URL=" .env | sed 's/DATABASE_URL=//; s/"//g')
psql "$DATABASE_URL_PSQL" -f prisma/scripts/trading-v2-foundation.sql
```

Expected: tabelas `ob2_user_market_balances` e `ob2_reservations` criadas; enums `Ob2Asset`, `Ob2Side`, `Ob2OrderStatus`, `Ob2TradeStatus`, `Ob2Primitive` criados; client regenerado; script SQL aplicado sem erro.

- [ ] **Step 1.5: Verificar schema no DB**

Run:
```bash
cd /Users/rafael/Documents/GitHub/Oraculo/api
bun x prisma db execute --stdin <<'SQL'
SELECT column_name, data_type FROM information_schema.columns
  WHERE table_name = 'ob2_user_market_balances' ORDER BY ordinal_position;
SQL
```

Expected: 7 colunas: user_id, market_pda, asset, free, reserved, onchain_total, onchain_slot, updated_at.

- [ ] **Step 1.6: Commit**

```bash
git add prisma/schema.prisma prisma/scripts/trading-v2-foundation.sql
git commit -m "feat(trading-v2): schema fundação (balances + reservations)"
```

---

## Task 2: Types de domínio

**Files:**
- Create: `api/src/modules/trading-v2/types/balance.types.ts`
- Create: `api/src/modules/trading-v2/types/reservation.types.ts`
- Create: `api/src/modules/trading-v2/types/intent.types.ts`
- Create: `api/src/modules/trading-v2/index.ts`

- [ ] **Step 2.1: `types/balance.types.ts`**

```typescript
import type { Ob2Asset } from "../../../generated/prisma/client";

export { Ob2Asset };

export interface MarketBalance {
  userId: string;
  marketPda: string;
  asset: Ob2Asset;
  free: bigint;      // unidades mínimas (6 decimais * 10^6)
  reserved: bigint;
  onchainTotal: bigint;
  onchainSlot: bigint | null;
}

/**
 * Conversão Decimal (Prisma) ↔ bigint micro-units.
 * Por que bigint: Decimal arithmetic é propensa a erro de cast; bigint
 * com unidades mínimas (6 decimais fixos) é unambiguous.
 */
export const DECIMALS = 6;
export const UNIT = 1_000_000n;
```

- [ ] **Step 2.2: `types/reservation.types.ts`**

```typescript
import type { Ob2Asset } from "../../../generated/prisma/client";

export interface ReservationInput {
  userId: string;
  marketPda: string;
  asset: Ob2Asset;
  amount: bigint;         // micro-units
  orderId: string;
}

export interface Reservation {
  id: string;
  userId: string;
  marketPda: string;
  asset: Ob2Asset;
  amount: bigint;
  orderId: string;
  releasedAt: Date | null;
  createdAt: Date;
}

export class InsufficientBalanceError extends Error {
  constructor(public readonly asset: Ob2Asset, public readonly need: bigint, public readonly have: bigint) {
    super(`insufficient_${asset.toLowerCase()}: need ${need}, have ${have}`);
    this.name = "InsufficientBalanceError";
  }
}
```

- [ ] **Step 2.3: `types/intent.types.ts`**

```typescript
import type { Ob2Asset, Ob2Side } from "../../../generated/prisma/client";

export { Ob2Side };

/**
 * Contexto puro para classificação. Saldos já foram lidos pelo caller;
 * o classifier é função pura (sem IO).
 */
export interface ClassifyCtx {
  side: Ob2Side;
  quantity: bigint;        // micro-units of YES (book é YES-normalizado)
  priceBps: number;        // 1..9999 (price of YES em basis points de USDC)
  feeBps: number;          // 0..10000
  freeYes: bigint;
  freeNo: bigint;
  freeUsdc: bigint;        // mantido na interface por simetria, pode ser ignorado
}

export interface ClassifyResult {
  asset: Ob2Asset;
  amount: bigint;
}
```

- [ ] **Step 2.4: `index.ts`**

```typescript
export * from "./types/balance.types";
export * from "./types/reservation.types";
export * from "./types/intent.types";
```

- [ ] **Step 2.5: Type-check**

Run:
```bash
cd /Users/rafael/Documents/GitHub/Oraculo/api
bun x tsc --noEmit
```

Expected: zero erros referentes a `src/modules/trading-v2/`. Erros pré-existentes em outros módulos são OK (não introduzimos).

- [ ] **Step 2.6: Commit**

```bash
git add src/modules/trading-v2/
git commit -m "feat(trading-v2): types de domínio (balance, reservation, intent)"
```

---

## Task 3: BalanceRepository

**Files:**
- Create: `api/src/modules/trading-v2/repositories/balance.repository.ts`
- Create: `api/src/modules/trading-v2/__tests__/balance.repository.integration.test.ts`

Repositório é SQL puro. Zero regra. Uma query por método.

- [ ] **Step 3.1: Escrever o teste primeiro**

`api/src/modules/trading-v2/__tests__/balance.repository.integration.test.ts`:

```typescript
import { PrismaClient } from "../../../generated/prisma/client";
import { BalanceRepository } from "../repositories/balance.repository";
import { UNIT } from "../types/balance.types";

const prisma = new PrismaClient();
const repo = new BalanceRepository(prisma);

const USER = "00000000-0000-0000-0000-000000000001";
const MARKET = "Market1111111111111111111111111111111111111";

beforeEach(async () => {
  await prisma.ob2Reservation.deleteMany({});
  await prisma.ob2UserMarketBalance.deleteMany({});
});
afterAll(async () => { await prisma.$disconnect(); });

test("upsertOnchain sets free=total and reserved=0 for new row", async () => {
  await repo.upsertOnchain(USER, MARKET, "USDC", 1000n * UNIT, 12345n);
  const b = await repo.get(USER, MARKET, "USDC");
  expect(b).not.toBeNull();
  expect(b!.free).toBe(1000n * UNIT);
  expect(b!.reserved).toBe(0n);
  expect(b!.onchainTotal).toBe(1000n * UNIT);
});

test("upsertOnchain on existing row only updates onchain snapshot (not free/reserved)", async () => {
  await repo.upsertOnchain(USER, MARKET, "USDC", 1000n * UNIT, 12345n);
  // simulate a reserve between snapshots
  await prisma.ob2UserMarketBalance.update({
    where: { userId_marketPda_asset: { userId: USER, marketPda: MARKET, asset: "USDC" } },
    data: { free: 600n * UNIT, reserved: 400n * UNIT },
  });
  await repo.upsertOnchain(USER, MARKET, "USDC", 1000n * UNIT, 99999n);
  const b = await repo.get(USER, MARKET, "USDC");
  expect(b!.free).toBe(600n * UNIT);
  expect(b!.reserved).toBe(400n * UNIT);
  expect(b!.onchainSlot).toBe(99999n);
});

test("get returns null when row missing", async () => {
  const b = await repo.get(USER, MARKET, "NO");
  expect(b).toBeNull();
});
```

- [ ] **Step 3.2: Rodar teste (deve falhar)**

```bash
cd /Users/rafael/Documents/GitHub/Oraculo/api
bun x jest src/modules/trading-v2/__tests__/balance.repository.integration.test.ts
```
Expected: FAIL — módulo `../repositories/balance.repository` não existe.

- [ ] **Step 3.3: Implementar repo**

`api/src/modules/trading-v2/repositories/balance.repository.ts`:

```typescript
import type { PrismaClient, Ob2Asset } from "../../../generated/prisma/client";
import type { MarketBalance } from "../types/balance.types";

const toBig = (v: unknown): bigint => {
  // Prisma Decimal -> string -> bigint micro-units
  // Decimal(20,6) já vem como string da lib; multiplicamos por 10^6
  const s = String(v);
  const [intPart, fracPart = ""] = s.split(".");
  const frac = fracPart.padEnd(6, "0").slice(0, 6);
  return BigInt(intPart + frac);
};

const fromBig = (v: bigint): string => {
  const s = v.toString().padStart(7, "0");
  const intPart = s.slice(0, -6);
  const fracPart = s.slice(-6);
  return `${intPart}.${fracPart}`;
};

export class BalanceRepository {
  constructor(private readonly prisma: PrismaClient) {}

  async get(userId: string, marketPda: string, asset: Ob2Asset): Promise<MarketBalance | null> {
    const row = await this.prisma.ob2UserMarketBalance.findUnique({
      where: { userId_marketPda_asset: { userId, marketPda, asset } },
    });
    if (!row) return null;
    return {
      userId: row.userId,
      marketPda: row.marketPda,
      asset: row.asset,
      free: toBig(row.free),
      reserved: toBig(row.reserved),
      onchainTotal: toBig(row.onchainTotal),
      onchainSlot: row.onchainSlot,
    };
  }

  /**
   * Cria a linha se não existe (free=total, reserved=0).
   * Se existe, atualiza apenas onchain_total e onchain_slot.
   */
  async upsertOnchain(
    userId: string, marketPda: string, asset: Ob2Asset,
    onchainTotal: bigint, onchainSlot: bigint,
  ): Promise<void> {
    await this.prisma.ob2UserMarketBalance.upsert({
      where: { userId_marketPda_asset: { userId, marketPda, asset } },
      create: {
        userId, marketPda, asset,
        free: fromBig(onchainTotal),
        reserved: "0",
        onchainTotal: fromBig(onchainTotal),
        onchainSlot,
      },
      update: {
        onchainTotal: fromBig(onchainTotal),
        onchainSlot,
      },
    });
  }
}
```

- [ ] **Step 3.4: Rodar teste (deve passar)**

```bash
bun x jest src/modules/trading-v2/__tests__/balance.repository.integration.test.ts
```
Expected: 3 tests passed.

- [ ] **Step 3.5: Commit**

```bash
git add src/modules/trading-v2/repositories/ src/modules/trading-v2/__tests__/balance.repository.integration.test.ts
git commit -m "feat(trading-v2): BalanceRepository + integration tests"
```

---

## Task 4: ReservationService.reserve() atômico

**Files:**
- Create: `api/src/modules/trading-v2/services/reservation.service.ts`
- Create: `api/src/modules/trading-v2/__tests__/reservation.service.integration.test.ts`

Este é o coração da invariante I1. A reserva **deve** ser um `UPDATE ... WHERE free >= amount` que aborta a transação se afeta 0 linhas. Nada menos serve.

- [ ] **Step 4.1: Escrever teste de reserve básico**

`api/src/modules/trading-v2/__tests__/reservation.service.integration.test.ts`:

```typescript
import { PrismaClient } from "../../../generated/prisma/client";
import { BalanceRepository } from "../repositories/balance.repository";
import { ReservationService } from "../services/reservation.service";
import { InsufficientBalanceError, UNIT } from "../types";

const prisma = new PrismaClient();
const balanceRepo = new BalanceRepository(prisma);
const svc = new ReservationService(prisma);

const USER = "00000000-0000-0000-0000-000000000001";
const MARKET = "Market1111111111111111111111111111111111111";
const ORDER = "00000000-0000-0000-0000-0000000000aa";

beforeEach(async () => {
  await prisma.ob2Reservation.deleteMany({});
  await prisma.ob2UserMarketBalance.deleteMany({});
  await balanceRepo.upsertOnchain(USER, MARKET, "USDC", 1000n * UNIT, 1n);
});
afterAll(async () => { await prisma.$disconnect(); });

test("reserve sufficient amount: debits free, credits reserved, creates reservation row", async () => {
  const r = await svc.reserve({
    userId: USER, marketPda: MARKET, asset: "USDC", amount: 300n * UNIT, orderId: ORDER,
  });
  expect(r.id).toBeDefined();

  const b = await balanceRepo.get(USER, MARKET, "USDC");
  expect(b!.free).toBe(700n * UNIT);
  expect(b!.reserved).toBe(300n * UNIT);

  const rows = await prisma.ob2Reservation.findMany({ where: { orderId: ORDER } });
  expect(rows).toHaveLength(1);
  expect(rows[0].releasedAt).toBeNull();
});

test("reserve insufficient: throws InsufficientBalanceError, no state change", async () => {
  await expect(svc.reserve({
    userId: USER, marketPda: MARKET, asset: "USDC", amount: 2000n * UNIT, orderId: ORDER,
  })).rejects.toBeInstanceOf(InsufficientBalanceError);

  const b = await balanceRepo.get(USER, MARKET, "USDC");
  expect(b!.free).toBe(1000n * UNIT);
  expect(b!.reserved).toBe(0n);
  const rows = await prisma.ob2Reservation.findMany({});
  expect(rows).toHaveLength(0);
});

test("reserve rejects zero or negative amount", async () => {
  await expect(svc.reserve({
    userId: USER, marketPda: MARKET, asset: "USDC", amount: 0n, orderId: ORDER,
  })).rejects.toThrow(/amount must be positive/);
});

test("reserve auto-creates balance row with zeros if missing", async () => {
  await expect(svc.reserve({
    userId: USER, marketPda: MARKET, asset: "YES", amount: 1n, orderId: ORDER,
  })).rejects.toBeInstanceOf(InsufficientBalanceError);
  const b = await balanceRepo.get(USER, MARKET, "YES");
  // É OK se o row não existir; é OK se existir com zeros. Testamos: reserva não "cria do nada".
  if (b) {
    expect(b.free).toBe(0n);
    expect(b.reserved).toBe(0n);
  }
});
```

- [ ] **Step 4.2: Rodar teste — deve falhar**

```bash
bun x jest src/modules/trading-v2/__tests__/reservation.service.integration.test.ts
```
Expected: FAIL — módulo não existe.

- [ ] **Step 4.3: Implementar ReservationService.reserve()**

`api/src/modules/trading-v2/services/reservation.service.ts`:

```typescript
import type { PrismaClient } from "../../../generated/prisma/client";
import { InsufficientBalanceError, type ReservationInput, type Reservation } from "../types/reservation.types";
import { randomUUID } from "crypto";

export class ReservationService {
  constructor(private readonly prisma: PrismaClient) {}

  /**
   * Atômico: debita free, credita reserved, cria linha em ob2_reservations.
   * Se free < amount, aborta com InsufficientBalanceError e zero mutações.
   * Implementado via UPDATE condicional + RETURNING — uma ida ao DB.
   */
  async reserve(input: ReservationInput): Promise<Reservation> {
    if (input.amount <= 0n) throw new Error("amount must be positive");

    const amountStr = this.toDecimal(input.amount);
    const id = randomUUID();

    return this.prisma.$transaction(async (tx) => {
      // Conditional UPDATE. affected = 0 se free < amount OU linha não existe.
      const result = await tx.$executeRawUnsafe(
        `UPDATE ob2_user_market_balances
            SET free = free - $4::numeric,
                reserved = reserved + $4::numeric,
                updated_at = now()
          WHERE user_id = $1 AND market_pda = $2 AND asset = $3::"Ob2Asset"
            AND free >= $4::numeric`,
        input.userId, input.marketPda, input.asset, amountStr,
      );

      if (result === 0) {
        // Descobrir o saldo real pra erro informativo
        const row = await tx.ob2UserMarketBalance.findUnique({
          where: { userId_marketPda_asset: {
            userId: input.userId, marketPda: input.marketPda, asset: input.asset,
          } },
        });
        const have = row ? this.fromDecimal(String(row.free)) : 0n;
        throw new InsufficientBalanceError(input.asset, input.amount, have);
      }

      const created = await tx.ob2Reservation.create({
        data: {
          id,
          userId: input.userId,
          marketPda: input.marketPda,
          asset: input.asset,
          amount: amountStr,
          orderId: input.orderId,
        },
      });

      return {
        id: created.id,
        userId: created.userId,
        marketPda: created.marketPda,
        asset: created.asset,
        amount: this.fromDecimal(String(created.amount)),
        orderId: created.orderId,
        releasedAt: created.releasedAt,
        createdAt: created.createdAt,
      };
    });
  }

  private toDecimal(micro: bigint): string {
    const s = micro.toString().padStart(7, "0");
    return `${s.slice(0, -6)}.${s.slice(-6)}`;
  }

  private fromDecimal(s: string): bigint {
    const [i, f = ""] = s.split(".");
    return BigInt(i + f.padEnd(6, "0").slice(0, 6));
  }
}
```

- [ ] **Step 4.4: Rodar teste — deve passar**

```bash
bun x jest src/modules/trading-v2/__tests__/reservation.service.integration.test.ts
```
Expected: 4 tests passed.

- [ ] **Step 4.5: Commit**

```bash
git add src/modules/trading-v2/services/reservation.service.ts src/modules/trading-v2/__tests__/reservation.service.integration.test.ts
git commit -m "feat(trading-v2): ReservationService.reserve() atômico"
```

---

## Task 5: ReservationService.release() + releasePartial()

**Files:**
- Modify: `api/src/modules/trading-v2/services/reservation.service.ts`
- Modify: `api/src/modules/trading-v2/__tests__/reservation.service.integration.test.ts`

- [ ] **Step 5.1: Acrescentar testes ao arquivo existente**

Append no `reservation.service.integration.test.ts`:

```typescript
test("release returns full amount to free and marks reservation released", async () => {
  const r = await svc.reserve({
    userId: USER, marketPda: MARKET, asset: "USDC", amount: 300n * UNIT, orderId: ORDER,
  });
  await svc.release(r.id);

  const b = await balanceRepo.get(USER, MARKET, "USDC");
  expect(b!.free).toBe(1000n * UNIT);
  expect(b!.reserved).toBe(0n);

  const row = await prisma.ob2Reservation.findUnique({ where: { id: r.id } });
  expect(row!.releasedAt).not.toBeNull();
});

test("release is idempotent: second call is no-op", async () => {
  const r = await svc.reserve({
    userId: USER, marketPda: MARKET, asset: "USDC", amount: 300n * UNIT, orderId: ORDER,
  });
  await svc.release(r.id);
  await svc.release(r.id);  // must not throw, must not double-credit free

  const b = await balanceRepo.get(USER, MARKET, "USDC");
  expect(b!.free).toBe(1000n * UNIT);
  expect(b!.reserved).toBe(0n);
});

test("releasePartial returns amount to free and shrinks reservation", async () => {
  const r = await svc.reserve({
    userId: USER, marketPda: MARKET, asset: "USDC", amount: 300n * UNIT, orderId: ORDER,
  });
  await svc.releasePartial(r.id, 100n * UNIT);

  const b = await balanceRepo.get(USER, MARKET, "USDC");
  expect(b!.free).toBe(800n * UNIT);
  expect(b!.reserved).toBe(200n * UNIT);

  const row = await prisma.ob2Reservation.findUnique({ where: { id: r.id } });
  expect(row!.releasedAt).toBeNull();
  expect(String(row!.amount)).toBe("200.000000");
});

test("releasePartial more than reserved throws", async () => {
  const r = await svc.reserve({
    userId: USER, marketPda: MARKET, asset: "USDC", amount: 300n * UNIT, orderId: ORDER,
  });
  await expect(svc.releasePartial(r.id, 500n * UNIT)).rejects.toThrow(/exceeds/);
});
```

- [ ] **Step 5.2: Rodar — deve falhar nos 4 novos**

```bash
bun x jest src/modules/trading-v2/__tests__/reservation.service.integration.test.ts
```
Expected: 4 novos falham, os 4 anteriores continuam passando.

- [ ] **Step 5.3: Implementar release() e releasePartial()**

Acrescentar dentro da classe `ReservationService`:

```typescript
  /**
   * Idempotente: se já released, retorna sem efeito.
   */
  async release(reservationId: string): Promise<void> {
    await this.prisma.$transaction(async (tx) => {
      const res = await tx.ob2Reservation.findUnique({ where: { id: reservationId } });
      if (!res) throw new Error(`reservation ${reservationId} not found`);
      if (res.releasedAt) return;  // idempotente

      const amountStr = String(res.amount);

      await tx.$executeRawUnsafe(
        `UPDATE ob2_user_market_balances
            SET reserved = reserved - $4::numeric,
                free = free + $4::numeric,
                updated_at = now()
          WHERE user_id = $1 AND market_pda = $2 AND asset = $3::"Ob2Asset"`,
        res.userId, res.marketPda, res.asset, amountStr,
      );

      await tx.ob2Reservation.update({
        where: { id: reservationId },
        data: { releasedAt: new Date() },
      });
    });
  }

  /**
   * Libera uma fração da reserva (usado em fills parciais).
   * Amount ≤ reservation.amount atual. Amount == full equivale a release().
   */
  async releasePartial(reservationId: string, amount: bigint): Promise<void> {
    if (amount <= 0n) throw new Error("amount must be positive");

    await this.prisma.$transaction(async (tx) => {
      const res = await tx.ob2Reservation.findUnique({ where: { id: reservationId } });
      if (!res) throw new Error(`reservation ${reservationId} not found`);
      if (res.releasedAt) throw new Error("reservation already released");

      const current = this.fromDecimal(String(res.amount));
      if (amount > current) {
        throw new Error(`amount ${amount} exceeds reservation ${current}`);
      }

      const amountStr = this.toDecimal(amount);
      const remaining = current - amount;

      await tx.$executeRawUnsafe(
        `UPDATE ob2_user_market_balances
            SET reserved = reserved - $4::numeric,
                free = free + $4::numeric,
                updated_at = now()
          WHERE user_id = $1 AND market_pda = $2 AND asset = $3::"Ob2Asset"`,
        res.userId, res.marketPda, res.asset, amountStr,
      );

      if (remaining === 0n) {
        await tx.ob2Reservation.update({
          where: { id: reservationId },
          data: { releasedAt: new Date() },
        });
      } else {
        await tx.ob2Reservation.update({
          where: { id: reservationId },
          data: { amount: this.toDecimal(remaining) },
        });
      }
    });
  }

  /**
   * "Consome" parte da reserva: debita do reserved SEM devolver pro free.
   * Usado pelo settlement pra marcar que o token/USDC saiu pro outro lado.
   */
  async consumePartial(reservationId: string, amount: bigint): Promise<void> {
    if (amount <= 0n) throw new Error("amount must be positive");

    await this.prisma.$transaction(async (tx) => {
      const res = await tx.ob2Reservation.findUnique({ where: { id: reservationId } });
      if (!res) throw new Error(`reservation ${reservationId} not found`);
      if (res.releasedAt) throw new Error("reservation already released");

      const current = this.fromDecimal(String(res.amount));
      if (amount > current) {
        throw new Error(`amount ${amount} exceeds reservation ${current}`);
      }

      const amountStr = this.toDecimal(amount);
      const remaining = current - amount;

      await tx.$executeRawUnsafe(
        `UPDATE ob2_user_market_balances
            SET reserved = reserved - $4::numeric,
                updated_at = now()
          WHERE user_id = $1 AND market_pda = $2 AND asset = $3::"Ob2Asset"`,
        res.userId, res.marketPda, res.asset, amountStr,
      );

      if (remaining === 0n) {
        await tx.ob2Reservation.update({
          where: { id: reservationId },
          data: { releasedAt: new Date() },
        });
      } else {
        await tx.ob2Reservation.update({
          where: { id: reservationId },
          data: { amount: this.toDecimal(remaining) },
        });
      }
    });
  }
```

- [ ] **Step 5.4: Rodar — tudo deve passar**

```bash
bun x jest src/modules/trading-v2/__tests__/reservation.service.integration.test.ts
```
Expected: 8 tests passed.

- [ ] **Step 5.5: Commit**

```bash
git add src/modules/trading-v2/services/reservation.service.ts src/modules/trading-v2/__tests__/reservation.service.integration.test.ts
git commit -m "feat(trading-v2): reservation release + releasePartial + consumePartial"
```

---

## Task 6: Teste de Invariante I1 (concorrência)

**Files:**
- Create: `api/src/modules/trading-v2/__tests__/reservation.service.invariant-i1.test.ts`

Invariante I1 só vale se o UPDATE condicional for atômico sob concorrência. Teste dispara N reservas concorrentes cujo total excede o saldo e verifica que **só as suficientes passam**.

- [ ] **Step 6.1: Escrever o teste**

```typescript
import { PrismaClient } from "../../../generated/prisma/client";
import { BalanceRepository } from "../repositories/balance.repository";
import { ReservationService } from "../services/reservation.service";
import { InsufficientBalanceError, UNIT } from "../types";

const prisma = new PrismaClient();
const balanceRepo = new BalanceRepository(prisma);
const svc = new ReservationService(prisma);

const USER = "00000000-0000-0000-0000-000000000001";
const MARKET = "Market1111111111111111111111111111111111111";

beforeEach(async () => {
  await prisma.ob2Reservation.deleteMany({});
  await prisma.ob2UserMarketBalance.deleteMany({});
});
afterAll(async () => { await prisma.$disconnect(); });

test("I1: 50 concurrent reserves of 100 USDC against 1000 USDC balance -> exactly 10 succeed", async () => {
  await balanceRepo.upsertOnchain(USER, MARKET, "USDC", 1000n * UNIT, 1n);

  const attempts = Array.from({ length: 50 }, (_, i) =>
    svc.reserve({
      userId: USER, marketPda: MARKET, asset: "USDC",
      amount: 100n * UNIT,
      orderId: `00000000-0000-0000-0000-00000000${i.toString().padStart(4, "0")}`,
    }).then(() => "ok").catch(e => e instanceof InsufficientBalanceError ? "insufficient" : "error")
  );

  const results = await Promise.all(attempts);
  const ok = results.filter(r => r === "ok").length;
  const ins = results.filter(r => r === "insufficient").length;
  const err = results.filter(r => r === "error").length;

  expect(err).toBe(0);
  expect(ok).toBe(10);
  expect(ins).toBe(40);

  const b = await balanceRepo.get(USER, MARKET, "USDC");
  expect(b!.free).toBe(0n);
  expect(b!.reserved).toBe(1000n * UNIT);
});

test("I1: concurrent mixed reserve+release keeps invariant free+reserved==onchainTotal", async () => {
  await balanceRepo.upsertOnchain(USER, MARKET, "USDC", 1000n * UNIT, 1n);

  const reserveIds: string[] = [];
  const reserveOps: Promise<unknown>[] = [];
  for (let i = 0; i < 20; i++) {
    reserveOps.push(svc.reserve({
      userId: USER, marketPda: MARKET, asset: "USDC", amount: 10n * UNIT,
      orderId: `00000000-0000-0000-0000-a000000${i.toString().padStart(5, "0")}`,
    }).then(r => reserveIds.push(r.id)).catch(() => {}));
  }
  await Promise.all(reserveOps);

  // Release metade concorrentemente
  const releaseOps = reserveIds.slice(0, reserveIds.length / 2).map(id => svc.release(id));
  await Promise.all(releaseOps);

  const b = await balanceRepo.get(USER, MARKET, "USDC");
  expect(b!.free + b!.reserved).toBe(1000n * UNIT);
});
```

- [ ] **Step 6.2: Rodar — deve passar na primeira**

```bash
bun x jest src/modules/trading-v2/__tests__/reservation.service.invariant-i1.test.ts --runInBand
```
Expected: 2 tests passed. Se algum falhar → a atomicidade do UPDATE não está garantida; investigar antes de seguir.

- [ ] **Step 6.3: Commit**

```bash
git add src/modules/trading-v2/__tests__/reservation.service.invariant-i1.test.ts
git commit -m "test(trading-v2): invariante I1 sob concorrência"
```

---

## Task 7: BalanceService (fachada de leitura)

**Files:**
- Create: `api/src/modules/trading-v2/services/balance.service.ts`
- Create: `api/src/modules/trading-v2/__tests__/balance.service.invariant-i2.test.ts`

Serviço simples por cima do repo: leitura por `(user, market)` agregando os 3 assets + exposição da invariante I2 como função.

- [ ] **Step 7.1: Escrever o teste I2**

`api/src/modules/trading-v2/__tests__/balance.service.invariant-i2.test.ts`:

```typescript
import { PrismaClient } from "../../../generated/prisma/client";
import { BalanceRepository } from "../repositories/balance.repository";
import { BalanceService } from "../services/balance.service";
import { ReservationService } from "../services/reservation.service";
import { UNIT } from "../types";

const prisma = new PrismaClient();
const balanceRepo = new BalanceRepository(prisma);
const resSvc = new ReservationService(prisma);
const balSvc = new BalanceService(prisma);

const USER = "00000000-0000-0000-0000-000000000001";
const MARKET = "Market1111111111111111111111111111111111111";

beforeEach(async () => {
  await prisma.ob2Reservation.deleteMany({});
  await prisma.ob2UserMarketBalance.deleteMany({});
  await balanceRepo.upsertOnchain(USER, MARKET, "USDC", 1000n * UNIT, 1n);
});
afterAll(async () => { await prisma.$disconnect(); });

test("I2: invariant holds through random sequence of reserves and releases", async () => {
  const seed = 42;
  let rng = seed;
  const next = () => { rng = (rng * 1103515245 + 12345) & 0x7fffffff; return rng; };

  const liveRes: string[] = [];
  for (let i = 0; i < 100; i++) {
    const action = next() % 3;
    try {
      if (action === 0 || liveRes.length === 0) {
        const amt = BigInt((next() % 50) + 1) * UNIT;
        const r = await resSvc.reserve({
          userId: USER, marketPda: MARKET, asset: "USDC", amount: amt,
          orderId: `00000000-0000-0000-0000-${i.toString(16).padStart(12, "0")}`,
        });
        liveRes.push(r.id);
      } else if (action === 1) {
        const idx = next() % liveRes.length;
        await resSvc.release(liveRes[idx]);
        liveRes.splice(idx, 1);
      } else {
        const idx = next() % liveRes.length;
        const amt = BigInt((next() % 10) + 1) * UNIT;
        await resSvc.releasePartial(liveRes[idx], amt).catch(() => {}); // pode exceder, ok
      }
    } catch { /* insufficient, esperado */ }

    // Invariante após cada operação
    const inv = await balSvc.checkInvariantI2(USER, MARKET, "USDC");
    expect(inv.holds).toBe(true);
    if (!inv.holds) {
      throw new Error(`I2 broke at step ${i}: ${JSON.stringify(inv)}`);
    }
  }
});

test("I2: detects broken invariant when manipulated directly", async () => {
  // Simula corrupção: mexe no DB sem passar pelos services
  await prisma.ob2UserMarketBalance.update({
    where: { userId_marketPda_asset: { userId: USER, marketPda: MARKET, asset: "USDC" } },
    data: { reserved: "500" },  // reserved sem reservation row correspondente
  });
  const inv = await balSvc.checkInvariantI2(USER, MARKET, "USDC");
  expect(inv.holds).toBe(false);
});
```

- [ ] **Step 7.2: Rodar — deve falhar (service ainda não existe)**

```bash
bun x jest src/modules/trading-v2/__tests__/balance.service.invariant-i2.test.ts
```
Expected: FAIL.

- [ ] **Step 7.3: Implementar BalanceService**

`api/src/modules/trading-v2/services/balance.service.ts`:

```typescript
import type { PrismaClient, Ob2Asset } from "../../../generated/prisma/client";
import { BalanceRepository } from "../repositories/balance.repository";
import type { MarketBalance } from "../types/balance.types";

export interface InvariantI2Result {
  holds: boolean;
  free: bigint;
  reserved: bigint;
  sumOfActiveReservations: bigint;
  onchainTotal: bigint;
}

export class BalanceService {
  private readonly repo: BalanceRepository;
  constructor(private readonly prisma: PrismaClient) {
    this.repo = new BalanceRepository(prisma);
  }

  async get(userId: string, marketPda: string, asset: Ob2Asset): Promise<MarketBalance | null> {
    return this.repo.get(userId, marketPda, asset);
  }

  async getAll(userId: string, marketPda: string): Promise<MarketBalance[]> {
    const rows = await this.prisma.ob2UserMarketBalance.findMany({
      where: { userId, marketPda },
    });
    return rows.map(r => ({
      userId: r.userId,
      marketPda: r.marketPda,
      asset: r.asset,
      free: this.fromDecimal(String(r.free)),
      reserved: this.fromDecimal(String(r.reserved)),
      onchainTotal: this.fromDecimal(String(r.onchainTotal)),
      onchainSlot: r.onchainSlot,
    }));
  }

  /**
   * Checa I2 localmente: soma de reservations ativas == reserved.
   * (Checagem contra on-chain é separada, vive em ReconciliationService.)
   */
  async checkInvariantI2(
    userId: string, marketPda: string, asset: Ob2Asset,
  ): Promise<InvariantI2Result> {
    const bal = await this.repo.get(userId, marketPda, asset);
    const resSum = await this.prisma.ob2Reservation.aggregate({
      _sum: { amount: true },
      where: { userId, marketPda, asset, releasedAt: null },
    });
    const sumOfActive = resSum._sum.amount
      ? this.fromDecimal(String(resSum._sum.amount))
      : 0n;

    const free = bal?.free ?? 0n;
    const reserved = bal?.reserved ?? 0n;
    const onchainTotal = bal?.onchainTotal ?? 0n;

    return {
      holds: reserved === sumOfActive,
      free, reserved, sumOfActiveReservations: sumOfActive, onchainTotal,
    };
  }

  private fromDecimal(s: string): bigint {
    const [i, f = ""] = s.split(".");
    return BigInt(i + f.padEnd(6, "0").slice(0, 6));
  }
}
```

- [ ] **Step 7.4: Rodar — deve passar**

```bash
bun x jest src/modules/trading-v2/__tests__/balance.service.invariant-i2.test.ts --runInBand
```
Expected: 2 tests passed.

- [ ] **Step 7.5: Commit**

```bash
git add src/modules/trading-v2/services/balance.service.ts src/modules/trading-v2/__tests__/balance.service.invariant-i2.test.ts
git commit -m "feat(trading-v2): BalanceService + invariante I2 (property test)"
```

---

## Task 8: IntentClassifierService (§5.1.1)

**Files:**
- Create: `api/src/modules/trading-v2/services/intent-classifier.service.ts`
- Create: `api/src/modules/trading-v2/__tests__/intent-classifier.unit.test.ts`

Pura (sem IO). Dado estado de saldo + input de ordem, devolve `(asset, amount)` a reservar.

- [ ] **Step 8.1: Escrever testes**

```typescript
import { IntentClassifier } from "../services/intent-classifier.service";
import { UNIT } from "../types";

const classifier = new IntentClassifier();

// price=6000bps=0.60, fee=50bps=0.5%
const base = { priceBps: 6000, feeBps: 50 };

test("SELL with YES >= qty → reserve YES", () => {
  const r = classifier.classify({
    side: "SELL", quantity: 100n * UNIT,
    freeYes: 500n * UNIT, freeNo: 0n, freeUsdc: 0n,
    ...base,
  });
  expect(r.asset).toBe("YES");
  expect(r.amount).toBe(100n * UNIT);
});

test("SELL with YES < qty → reserve USDC at (1-price)*qty + fee", () => {
  const r = classifier.classify({
    side: "SELL", quantity: 100n * UNIT,
    freeYes: 0n, freeNo: 0n, freeUsdc: 1000n * UNIT,
    ...base,
  });
  // notional = 100 * (1 - 0.60) = 40; fee = 40 * 0.005 = 0.20; total = 40.20
  expect(r.asset).toBe("USDC");
  expect(r.amount).toBe(40_200_000n);
});

test("BUY with NO >= qty → reserve NO", () => {
  const r = classifier.classify({
    side: "BUY", quantity: 100n * UNIT,
    freeYes: 0n, freeNo: 500n * UNIT, freeUsdc: 0n,
    ...base,
  });
  expect(r.asset).toBe("NO");
  expect(r.amount).toBe(100n * UNIT);
});

test("BUY with NO < qty → reserve USDC at price*qty + fee", () => {
  const r = classifier.classify({
    side: "BUY", quantity: 100n * UNIT,
    freeYes: 0n, freeNo: 0n, freeUsdc: 1000n * UNIT,
    ...base,
  });
  // notional = 100 * 0.60 = 60; fee = 60 * 0.005 = 0.30; total = 60.30
  expect(r.asset).toBe("USDC");
  expect(r.amount).toBe(60_300_000n);
});

test("rejects priceBps out of (0, 10000)", () => {
  expect(() => classifier.classify({
    side: "BUY", quantity: 1n,
    freeYes: 0n, freeNo: 0n, freeUsdc: 0n,
    priceBps: 0, feeBps: 0,
  })).toThrow(/price/);
  expect(() => classifier.classify({
    side: "BUY", quantity: 1n,
    freeYes: 0n, freeNo: 0n, freeUsdc: 0n,
    priceBps: 10000, feeBps: 0,
  })).toThrow(/price/);
});
```

- [ ] **Step 8.2: Rodar — FAIL**

```bash
bun x jest src/modules/trading-v2/__tests__/intent-classifier.unit.test.ts
```

- [ ] **Step 8.3: Implementar**

`api/src/modules/trading-v2/services/intent-classifier.service.ts`:

```typescript
import type { ClassifyCtx, ClassifyResult } from "../types/intent.types";

export class IntentClassifier {
  classify(ctx: ClassifyCtx): ClassifyResult {
    if (ctx.priceBps <= 0 || ctx.priceBps >= 10000) {
      throw new Error(`price out of range: ${ctx.priceBps}`);
    }
    if (ctx.feeBps < 0 || ctx.feeBps > 10000) {
      throw new Error(`fee out of range: ${ctx.feeBps}`);
    }
    if (ctx.quantity <= 0n) throw new Error("quantity must be positive");

    if (ctx.side === "SELL") {
      if (ctx.freeYes >= ctx.quantity) {
        return { asset: "YES", amount: ctx.quantity };
      }
      // abre short YES = long NO: reserva USDC = qty * (1 - price) + fee
      const notional = (ctx.quantity * BigInt(10000 - ctx.priceBps)) / 10000n;
      const fee = (notional * BigInt(ctx.feeBps)) / 10000n;
      return { asset: "USDC", amount: notional + fee };
    }

    // BUY
    if (ctx.freeNo >= ctx.quantity) {
      return { asset: "NO", amount: ctx.quantity };
    }
    const notional = (ctx.quantity * BigInt(ctx.priceBps)) / 10000n;
    const fee = (notional * BigInt(ctx.feeBps)) / 10000n;
    return { asset: "USDC", amount: notional + fee };
  }
}
```

- [ ] **Step 8.4: Rodar — PASS**

```bash
bun x jest src/modules/trading-v2/__tests__/intent-classifier.unit.test.ts
```
Expected: 5 tests passed.

- [ ] **Step 8.5: Commit**

```bash
git add src/modules/trading-v2/services/intent-classifier.service.ts src/modules/trading-v2/__tests__/intent-classifier.unit.test.ts
git commit -m "feat(trading-v2): IntentClassifier (§5.1.1 de qual ativo reservar)"
```

---

## Task 9: ReconciliationService (detecção, sem fix)

**Files:**
- Create: `api/src/modules/trading-v2/services/reconciliation.service.ts`
- Create: `api/src/modules/trading-v2/__tests__/reconciliation.service.unit.test.ts`

Dado `MarketBalance` + leitura on-chain, retorna diff. Sem mutação. Usado pelo job diário (plano 7) e por checkers ad-hoc.

- [ ] **Step 9.1: Teste**

```typescript
import { ReconciliationService, DUST_MICRO } from "../services/reconciliation.service";
import { UNIT } from "../types";

const svc = new ReconciliationService();

test("matches within dust", () => {
  const r = svc.compare({
    free: 600n * UNIT, reserved: 400n * UNIT, onchainTotal: 1000n * UNIT + DUST_MICRO / 2n,
  });
  expect(r.inSync).toBe(true);
  expect(r.diff).toBeLessThanOrEqual(DUST_MICRO);
});

test("flags when expected > actual", () => {
  const r = svc.compare({
    free: 600n * UNIT, reserved: 400n * UNIT, onchainTotal: 900n * UNIT,
  });
  expect(r.inSync).toBe(false);
  expect(r.diff).toBe(100n * UNIT);
  expect(r.direction).toBe("db_exceeds_chain");
});

test("flags when actual > expected", () => {
  const r = svc.compare({
    free: 600n * UNIT, reserved: 400n * UNIT, onchainTotal: 1500n * UNIT,
  });
  expect(r.inSync).toBe(false);
  expect(r.direction).toBe("chain_exceeds_db");
});
```

- [ ] **Step 9.2: Rodar — FAIL**

```bash
bun x jest src/modules/trading-v2/__tests__/reconciliation.service.unit.test.ts
```

- [ ] **Step 9.3: Implementar**

```typescript
import { UNIT } from "../types/balance.types";

export const DUST_MICRO = 1000n; // 0.001 USDC de tolerância

export interface CompareInput {
  free: bigint;
  reserved: bigint;
  onchainTotal: bigint;
}

export interface CompareResult {
  inSync: boolean;
  diff: bigint;
  direction: "db_exceeds_chain" | "chain_exceeds_db" | "equal";
}

export class ReconciliationService {
  compare(input: CompareInput): CompareResult {
    const expected = input.free + input.reserved;
    const actual = input.onchainTotal;
    const diff = expected > actual ? expected - actual : actual - expected;
    const direction: CompareResult["direction"] =
      diff === 0n ? "equal"
      : expected > actual ? "db_exceeds_chain"
      : "chain_exceeds_db";

    return { inSync: diff <= DUST_MICRO, diff, direction };
  }
}
```

- [ ] **Step 9.4: Rodar — PASS**

```bash
bun x jest src/modules/trading-v2/__tests__/reconciliation.service.unit.test.ts
```
Expected: 3 passed.

- [ ] **Step 9.5: Commit**

```bash
git add src/modules/trading-v2/services/reconciliation.service.ts src/modules/trading-v2/__tests__/reconciliation.service.unit.test.ts
git commit -m "feat(trading-v2): ReconciliationService (diff puro, sem fix)"
```

---

## Task 10: Snapshot script (prepara cutover)

**Files:**
- Create: `api/src/modules/trading-v2/scripts/snapshot-onchain-balances.ts`
- Modify: `api/package.json` (script alias)

Script CLI que percorre todos os `(user, market)` com atividade, lê saldo on-chain real via RPC, popula `ob2_user_market_balances`. **Idempotente**: rodar 2x não duplica. Executado manualmente no cutover.

- [ ] **Step 10.1: Implementar**

`api/src/modules/trading-v2/scripts/snapshot-onchain-balances.ts`:

```typescript
/**
 * Snapshot on-chain balances into ob2_user_market_balances.
 *
 * Uso:
 *   bun run src/modules/trading-v2/scripts/snapshot-onchain-balances.ts
 *   bun run src/modules/trading-v2/scripts/snapshot-onchain-balances.ts --user <uuid>
 *   bun run src/modules/trading-v2/scripts/snapshot-onchain-balances.ts --market <pda>
 *
 * Idempotente: UPSERT por (user_id, market_pda, asset).
 *              Primeiro snapshot popula free=total, reserved=0.
 *              Snapshots seguintes só atualizam onchain_total e onchain_slot.
 */
import { PrismaClient } from "../../../generated/prisma/client";
import { BalanceRepository } from "../repositories/balance.repository";

// NOTA: o leitor on-chain concreto vem do módulo prediction-market antigo.
// Encapsulamos pra não acoplar a assinatura dele aqui.
interface OnchainBalanceReader {
  listUserMarkets(filter: { userId?: string; marketPda?: string }): Promise<Array<{
    userId: string; marketPda: string;
  }>>;
  read(userId: string, marketPda: string): Promise<{
    usdc: bigint; yes: bigint; no: bigint; slot: bigint;
  }>;
}

export async function runSnapshot(
  prisma: PrismaClient,
  reader: OnchainBalanceReader,
  filter: { userId?: string; marketPda?: string } = {},
): Promise<{ total: number; errors: Array<{ userId: string; marketPda: string; err: string }> }> {
  const repo = new BalanceRepository(prisma);
  const pairs = await reader.listUserMarkets(filter);
  const errors: Array<{ userId: string; marketPda: string; err: string }> = [];
  let total = 0;

  for (const { userId, marketPda } of pairs) {
    try {
      const { usdc, yes, no, slot } = await reader.read(userId, marketPda);
      await repo.upsertOnchain(userId, marketPda, "USDC", usdc, slot);
      await repo.upsertOnchain(userId, marketPda, "YES",  yes,  slot);
      await repo.upsertOnchain(userId, marketPda, "NO",   no,   slot);
      total++;
    } catch (e) {
      errors.push({ userId, marketPda, err: e instanceof Error ? e.message : String(e) });
    }
  }

  return { total, errors };
}

// CLI entry
if (import.meta.main) {
  const args = process.argv.slice(2);
  const userIdx = args.indexOf("--user");
  const marketIdx = args.indexOf("--market");
  const filter = {
    userId: userIdx >= 0 ? args[userIdx + 1] : undefined,
    marketPda: marketIdx >= 0 ? args[marketIdx + 1] : undefined,
  };

  const prisma = new PrismaClient();
  // O reader concreto será injetado quando formos integrar na cutover.
  // Por ora, o script falha explicitamente se chamado sem reader.
  throw new Error(
    "OnchainBalanceReader não implementado neste plano. " +
    "Será injetado no Plano 7 (cutover) a partir do módulo Solana existente. " +
    `Filter recebido: ${JSON.stringify(filter)}`,
  );
}
```

Justificativa do throw: o leitor on-chain real depende do módulo atual de Solana, que ainda não queremos acoplar. O script fica pronto, testável com fake, e ganha reader de verdade no Plano 7.

- [ ] **Step 10.2: Testar com fake reader**

Create `api/src/modules/trading-v2/__tests__/snapshot-onchain-balances.integration.test.ts`:

```typescript
import { PrismaClient } from "../../../generated/prisma/client";
import { runSnapshot } from "../scripts/snapshot-onchain-balances";
import { BalanceRepository } from "../repositories/balance.repository";
import { UNIT } from "../types";

const prisma = new PrismaClient();
const repo = new BalanceRepository(prisma);

const USER = "00000000-0000-0000-0000-000000000001";
const MARKET = "Market1111111111111111111111111111111111111";

beforeEach(async () => {
  await prisma.ob2Reservation.deleteMany({});
  await prisma.ob2UserMarketBalance.deleteMany({});
});
afterAll(async () => { await prisma.$disconnect(); });

const fakeReader = {
  async listUserMarkets() { return [{ userId: USER, marketPda: MARKET }]; },
  async read() {
    return { usdc: 500n * UNIT, yes: 100n * UNIT, no: 0n, slot: 999n };
  },
};

test("snapshot populates all 3 assets with free=total, reserved=0", async () => {
  const { total, errors } = await runSnapshot(prisma, fakeReader);
  expect(total).toBe(1);
  expect(errors).toHaveLength(0);

  const usdc = await repo.get(USER, MARKET, "USDC");
  expect(usdc!.free).toBe(500n * UNIT);
  expect(usdc!.reserved).toBe(0n);
  expect(usdc!.onchainSlot).toBe(999n);

  const yes = await repo.get(USER, MARKET, "YES");
  expect(yes!.free).toBe(100n * UNIT);

  const no = await repo.get(USER, MARKET, "NO");
  expect(no!.free).toBe(0n);
});

test("re-running snapshot preserves free/reserved if balance state diverged", async () => {
  await runSnapshot(prisma, fakeReader);
  // Simula reservas
  await prisma.ob2UserMarketBalance.update({
    where: { userId_marketPda_asset: { userId: USER, marketPda: MARKET, asset: "USDC" } },
    data: { free: "300.000000", reserved: "200.000000" },
  });
  await runSnapshot(prisma, fakeReader);
  const usdc = await repo.get(USER, MARKET, "USDC");
  expect(usdc!.free).toBe(300n * UNIT);
  expect(usdc!.reserved).toBe(200n * UNIT);
  expect(usdc!.onchainTotal).toBe(500n * UNIT);
});
```

- [ ] **Step 10.3: Rodar — PASS**

```bash
bun x jest src/modules/trading-v2/__tests__/snapshot-onchain-balances.integration.test.ts
```
Expected: 2 passed.

- [ ] **Step 10.4: Adicionar alias no package.json**

Edit `api/package.json`, na section `scripts`, **após** a linha `"script:drain-vault": ...,` acrescentar:

```json
    "script:ob2-snapshot": "bun run src/modules/trading-v2/scripts/snapshot-onchain-balances.ts",
```

(Mantém consistência com os outros `script:*` existentes.)

- [ ] **Step 10.5: Commit**

```bash
git add src/modules/trading-v2/scripts/ src/modules/trading-v2/__tests__/snapshot-onchain-balances.integration.test.ts package.json
git commit -m "feat(trading-v2): snapshot script (reader injetável, ready pro cutover)"
```

---

## Task 11: Suíte completa + documentação curta

**Files:**
- Modify: `api/src/modules/trading-v2/index.ts`
- Create: `api/src/modules/trading-v2/README.md`

- [ ] **Step 11.1: Reexportar services no index**

Substituir `index.ts`:

```typescript
export * from "./types/balance.types";
export * from "./types/reservation.types";
export * from "./types/intent.types";

export { BalanceRepository } from "./repositories/balance.repository";
export { BalanceService } from "./services/balance.service";
export { ReservationService } from "./services/reservation.service";
export { IntentClassifier } from "./services/intent-classifier.service";
export { ReconciliationService, DUST_MICRO } from "./services/reconciliation.service";
```

- [ ] **Step 11.2: README curto pro módulo**

`api/src/modules/trading-v2/README.md`:

```markdown
# trading-v2

Módulo do novo orderbook. Ainda **não** processa ordens; este plano entrega
só a fundação (balances + reservations + intent classifier).

Spec: `docs/superpowers/specs/2026-04-15-orderbook-rewrite-design.md`

## Invariantes verificadas aqui

- **I1** (reserva obrigatória): `reservation.service.invariant-i1.test.ts`
- **I2** (conservação): `balance.service.invariant-i2.test.ts`

## Próximos planos

- Order lifecycle + matching engine
- Settlement sync/async + listener
- Programa Solana `settle_fill`
- WebSocket v2
- MM bot externo
- Cutover

## Rodar os testes

```bash
bun x jest src/modules/trading-v2 --runInBand
```

Precisa de Postgres local com schema aplicado (`bun x prisma migrate dev`).
```

- [ ] **Step 11.3: Rodar suíte inteira**

```bash
cd /Users/rafael/Documents/GitHub/Oraculo/api
bun x jest src/modules/trading-v2 --runInBand
```

Expected:
- `balance.repository.integration.test.ts` — 3 passed
- `reservation.service.integration.test.ts` — 8 passed
- `reservation.service.invariant-i1.test.ts` — 2 passed
- `balance.service.invariant-i2.test.ts` — 2 passed
- `intent-classifier.unit.test.ts` — 5 passed
- `reconciliation.service.unit.test.ts` — 3 passed
- `snapshot-onchain-balances.integration.test.ts` — 2 passed

**Total: 25 tests passed.**

- [ ] **Step 11.4: Type-check final**

```bash
bun x tsc --noEmit 2>&1 | grep "src/modules/trading-v2" || echo "clean"
```
Expected: `clean`.

- [ ] **Step 11.5: Commit final**

```bash
git add src/modules/trading-v2/README.md src/modules/trading-v2/index.ts
git commit -m "docs(trading-v2): README + barrel export da fundação"
```

---

## Critérios de aceitação do plano

Ao final, no branch deste plano:

1. ✅ 7 novos arquivos de teste, 25 testes verdes, zero testes vermelhos em `trading-v2`.
2. ✅ Schema `ob2_*` aplicado; schema antigo intocado.
3. ✅ `BalanceService`, `ReservationService`, `IntentClassifier`, `ReconciliationService` implementados e usáveis como módulo isolado.
4. ✅ Invariante I1 (reserva atômica) demonstrada sob concorrência de 50 threads.
5. ✅ Invariante I2 (conservação) demonstrada em sequência aleatória de 100 operações.
6. ✅ Script de snapshot pronto, aguardando injeção do reader on-chain no Plano 7.
7. ✅ Zero linha de código em `prediction-market/trading/` foi modificada.
8. ✅ Zero import de `trading-v2` em outros módulos (ainda isolado; integração nos próximos planos).

---

## Riscos e observações operacionais

- **Migration em produção**: as tabelas `ob2_*` são aditivas. `bun x prisma migrate deploy` em prod é seguro — não derruba nada.
- **Pool do Prisma**: testes rodam `--runInBand` porque usam o mesmo DB local. CI pode paralelizar com databases separados via `DATABASE_URL` override.
- **Decimal vs bigint**: escolhemos bigint em micro-units em todo lugar fora do DB. Risco: esquecer conversão no boundary. Mitigação: funções `toDecimal`/`fromDecimal` isoladas, e testes cobrem os caminhos.
- **`ob2_` prefixo**: feio mas intencional. Garante que nenhuma query acidentalmente bate na tabela errada durante a coexistência.
