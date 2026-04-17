# Orderbook Rewrite — Plano 10: Split USDC global vs YES/NO per-market

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Corrigir bug semântico descoberto no Tier 1 dry-run do cutover (Plano 9): USDC vive na ATA do wallet do usuário (per-user, global), mas o schema atual `ob2_user_market_balances` registra USDC per-market. Resultado: snapshot de prod replicaria o saldo USDC em cada market, criando ilusão de fundos múltiplos. Plano 10 separa USDC numa nova tabela `ob2_user_balances (user_id, asset)` e mantém `ob2_user_market_balances` apenas pra YES/NO (que SÃO per-market on-chain).

**Architecture:** Nova tabela `ob2_user_balances` com PK `(user_id, asset)` pra balance global do USDC. `ob2_user_market_balances` permanece pra YES/NO. `ReservationService`, `StubSettler` e `SettlementReverter` ganham roteamento por asset: USDC → global, YES/NO → per-market. Reservations em `ob2_reservations` continuam per-market (cada reservation pertence a uma ordem de um market). `SolanaOnchainBalanceReader` é refatorado pra escrever USDC ONCE per user no global, e YES/NO per market.

**Tech Stack:** Bun, TypeScript, Prisma 7, Jest.

**Origem:** discoberto no dry-run de Plano 9 contra dump de prod. User 408 tinha 2.205,68 USDC replicado em 51 markets — drift garantido em qualquer cutover real.

**Não-escopo:**
- **Migration de dados existentes**: a tabela `ob2_user_market_balances` está vazia em prod (não fez cutover ainda). Drop os USDC rows do schema antigo é zero-impact.
- **Rebalancing on-chain entre markets**: USDC é fungível, sem rebalanceamento.

---

## File Structure

```
api/
  prisma/schema.prisma                                   # MODIFY: nova model Ob2UserBalance
  prisma/scripts/trading-v2-user-balances.sql            # CREATE: CHECKs + cleanup USDC rows
  src/modules/trading-v2/
    types/balance.types.ts                               # MODIFY: UserBalance type (sem marketPda)
    repositories/
      balance.repository.ts                              # MODIFY: throws if asset=USDC, only YES/NO
      user-balance.repository.ts                         # CREATE: USDC global ops
    services/
      reservation.service.ts                             # MODIFY: route por asset
      balance.service.ts                                 # MODIFY: lookup combinado
      stub-settler.service.ts                            # MODIFY: route consume/credit
      settlement-reverter.service.ts                     # MODIFY: route restore
      solana-onchain-balance-reader.ts                   # MODIFY: USDC global, YES/NO per-market
      daily-reconciliation.service.ts                    # MODIFY: USDC compared globally
    __tests__/
      [vários]                                           # MODIFY: ajustar setUp pra novo split
      user-balance.repository.integration.test.ts        # CREATE
```

**Princípio**: USDC é per-user (matches on-chain ATA). YES/NO é per-market (matches market mints). Reservations continuam per-market (cada ordem é de um market).

---

## Prerequisite check

- [ ] **Step 0: Baseline.**

```bash
cd /Users/rafael/Documents/GitHub/Oraculo/api
bun x jest src/modules/trading-v2 src/services/mm-bot --runInBand
```

Expected: 174 passed + 1 skipped.

- [ ] **Step 0.1: Worktree.**

```bash
git worktree add .claude/worktrees/orderbook-rewrite-10-split-usdc -b worktree-orderbook-rewrite-10-split-usdc
cp api/.env .claude/worktrees/orderbook-rewrite-10-split-usdc/api/.env
cd .claude/worktrees/orderbook-rewrite-10-split-usdc/api && bun install && bun x prisma generate
```

---

## Task 1: Schema — `ob2_user_balances` table

**Files:**
- Modify: `api/prisma/schema.prisma`
- Create: `api/prisma/scripts/trading-v2-user-balances.sql`

- [ ] **Step 1.1: Append model to schema.prisma** (após `Ob2UserMarketBalance`):

```prisma
model Ob2UserBalance {
  userId        String    @map("user_id")
  asset         Ob2Asset
  free          Decimal   @default(0) @db.Decimal(20, 6)
  reserved      Decimal   @default(0) @db.Decimal(20, 6)
  onchainTotal  Decimal   @default(0) @map("onchain_total") @db.Decimal(20, 6)
  onchainSlot   BigInt?   @map("onchain_slot")
  updatedAt     DateTime  @default(now()) @map("updated_at")

  @@id([userId, asset])
  @@map("ob2_user_balances")
}
```

- [ ] **Step 1.2: SQL constraints + cleanup** (`api/prisma/scripts/trading-v2-user-balances.sql`):

```sql
DO $$ BEGIN
  ALTER TABLE ob2_user_balances ADD CONSTRAINT ob2_user_balance_free_nonneg CHECK (free >= 0);
EXCEPTION WHEN duplicate_object THEN NULL; END $$;

DO $$ BEGIN
  ALTER TABLE ob2_user_balances ADD CONSTRAINT ob2_user_balance_reserved_nonneg CHECK (reserved >= 0);
EXCEPTION WHEN duplicate_object THEN NULL; END $$;

-- Limpa USDC rows do per-market balances (USDC só vive na nova tabela).
-- Idempotente: se já tiver sido limpo, não afeta.
DELETE FROM ob2_user_market_balances WHERE asset = 'USDC';
```

- [ ] **Step 1.3: Apply.**

```bash
cd /Users/rafael/Documents/GitHub/Oraculo/.claude/worktrees/orderbook-rewrite-10-split-usdc/api
bun x prisma db push
bun x prisma generate
DATABASE_URL_PSQL=$(grep "^DATABASE_URL=" .env | sed 's/DATABASE_URL=//; s/"//g')
psql "$DATABASE_URL_PSQL" -f prisma/scripts/trading-v2-user-balances.sql
```

- [ ] **Step 1.4: Verify.**

```bash
psql "$DATABASE_URL_PSQL" -c "\d ob2_user_balances"
psql "$DATABASE_URL_PSQL" -c "SELECT count(*) FROM ob2_user_market_balances WHERE asset='USDC';"
```

Expected: tabela criada, USDC count = 0.

- [ ] **Step 1.5: Commit.**

```bash
cd /Users/rafael/Documents/GitHub/Oraculo/.claude/worktrees/orderbook-rewrite-10-split-usdc
git add api/prisma/schema.prisma api/prisma/scripts/trading-v2-user-balances.sql
git commit -m "feat(trading-v2): split USDC global em ob2_user_balances"
```

---

## Task 2: UserBalanceRepository (USDC global)

**Files:**
- Create: `api/src/modules/trading-v2/repositories/user-balance.repository.ts`
- Create: `api/src/modules/trading-v2/__tests__/user-balance.repository.integration.test.ts`
- Modify: `api/src/modules/trading-v2/types/balance.types.ts`

- [ ] **Step 2.1: Add `UserBalance` type to `types/balance.types.ts`** (append):

```typescript
export interface UserBalance {
  userId: string;
  asset: Ob2Asset;
  free: bigint;
  reserved: bigint;
  onchainTotal: bigint;
  onchainSlot: bigint | null;
}
```

- [ ] **Step 2.2: Failing test:**

```typescript
import "dotenv/config";
import { prisma } from "../../../shared/database/config/prisma-client";
import { UserBalanceRepository } from "../repositories/user-balance.repository";
import { UNIT } from "../types/balance.types";

const repo = new UserBalanceRepository(prisma);
const USER = "00000000-0000-0000-0000-000000000001";

beforeEach(async () => {
  await prisma.ob2UserBalance.deleteMany({});
});
afterAll(async () => { await prisma.$disconnect(); });

test("upsertOnchain creates row with free=total, reserved=0", async () => {
  await repo.upsertOnchain(USER, "USDC", 1000n * UNIT, 12345n);
  const b = await repo.get(USER, "USDC");
  expect(b!.free).toBe(1000n * UNIT);
  expect(b!.reserved).toBe(0n);
  expect(b!.onchainTotal).toBe(1000n * UNIT);
  expect(b!.onchainSlot).toBe(12345n);
});

test("upsertOnchain on existing row preserves free/reserved", async () => {
  await repo.upsertOnchain(USER, "USDC", 1000n * UNIT, 1n);
  await prisma.ob2UserBalance.update({
    where: { userId_asset: { userId: USER, asset: "USDC" } },
    data: { free: "600", reserved: "400" },
  });
  await repo.upsertOnchain(USER, "USDC", 1000n * UNIT, 99n);
  const b = await repo.get(USER, "USDC");
  expect(b!.free).toBe(600n * UNIT);
  expect(b!.reserved).toBe(400n * UNIT);
  expect(b!.onchainSlot).toBe(99n);
});

test("get returns null when missing", async () => {
  expect(await repo.get(USER, "USDC")).toBeNull();
});
```

- [ ] **Step 2.3: Implement `user-balance.repository.ts`:**

```typescript
import type { PrismaClient, Ob2Asset } from "../../../generated/prisma/client";
import type { UserBalance } from "../types/balance.types";
import { toMicro, fromMicro } from "../types/decimal-helpers";

export class UserBalanceRepository {
  constructor(private readonly prisma: PrismaClient) {}

  async get(userId: string, asset: Ob2Asset): Promise<UserBalance | null> {
    const row = await this.prisma.ob2UserBalance.findUnique({
      where: { userId_asset: { userId, asset } },
    });
    if (!row) return null;
    return {
      userId: row.userId,
      asset: row.asset,
      free: toMicro(row.free),
      reserved: toMicro(row.reserved),
      onchainTotal: toMicro(row.onchainTotal),
      onchainSlot: row.onchainSlot,
    };
  }

  async upsertOnchain(
    userId: string, asset: Ob2Asset,
    onchainTotal: bigint, onchainSlot: bigint,
  ): Promise<void> {
    await this.prisma.ob2UserBalance.upsert({
      where: { userId_asset: { userId, asset } },
      create: {
        userId, asset,
        free: fromMicro(onchainTotal),
        reserved: "0",
        onchainTotal: fromMicro(onchainTotal),
        onchainSlot,
      },
      update: {
        onchainTotal: fromMicro(onchainTotal),
        onchainSlot,
      },
    });
  }
}
```

- [ ] **Step 2.4: Run, 3 pass.**

- [ ] **Step 2.5: Commit.**

```bash
git add api/src/modules/trading-v2/repositories/user-balance.repository.ts \
        api/src/modules/trading-v2/types/balance.types.ts \
        api/src/modules/trading-v2/__tests__/user-balance.repository.integration.test.ts
git commit -m "feat(trading-v2): UserBalanceRepository (USDC global)"
```

---

## Task 3: ReservationService — route por asset

`ReservationService` precisa rotear: se asset=USDC → mexe em `ob2_user_balances`; se YES/NO → `ob2_user_market_balances`. As reservations em `ob2_reservations` continuam idênticas (per-market, com asset).

**Files:**
- Modify: `api/src/modules/trading-v2/services/reservation.service.ts`

- [ ] **Step 3.1: Refactor `reserve()`.**

Read the current file, then replace the raw SQL UPDATE call with a branch:

```typescript
  async reserve(input: ReservationInput): Promise<Reservation> {
    if (input.amount <= 0n) throw new Error("amount must be positive");

    const amountStr = fromMicro(input.amount);
    const id = randomUUID();

    return this.prisma.$transaction(async (tx) => {
      let result: number;
      if (input.asset === "USDC") {
        // USDC global: upsert ensures row exists with zero, then conditional UPDATE
        await tx.ob2UserBalance.upsert({
          where: { userId_asset: { userId: input.userId, asset: "USDC" } },
          create: { userId: input.userId, asset: "USDC", free: "0", reserved: "0", onchainTotal: "0" },
          update: {},
        });
        result = await tx.$executeRawUnsafe(
          `UPDATE ob2_user_balances
              SET free = free - $3::numeric,
                  reserved = reserved + $3::numeric,
                  updated_at = now()
            WHERE user_id = $1 AND asset = $2::"Ob2Asset"
              AND free >= $3::numeric`,
          input.userId, input.asset, amountStr,
        );
      } else {
        // YES/NO per-market
        result = await tx.$executeRawUnsafe(
          `UPDATE ob2_user_market_balances
              SET free = free - $4::numeric,
                  reserved = reserved + $4::numeric,
                  updated_at = now()
            WHERE user_id = $1 AND market_pda = $2 AND asset = $3::"Ob2Asset"
              AND free >= $4::numeric`,
          input.userId, input.marketPda, input.asset, amountStr,
        );
      }

      if (result === 0) {
        const have = await this.fetchFree(tx, input.userId, input.marketPda, input.asset);
        throw new InsufficientBalanceError(input.asset, input.amount, have);
      }

      const created = await tx.ob2Reservation.create({
        data: {
          id, userId: input.userId, marketPda: input.marketPda,
          asset: input.asset, amount: amountStr, orderId: input.orderId,
        },
      });

      return {
        id: created.id, userId: created.userId, marketPda: created.marketPda,
        asset: created.asset, amount: toMicro(String(created.amount)),
        orderId: created.orderId, releasedAt: created.releasedAt,
        createdAt: created.createdAt,
      };
    });
  }

  private async fetchFree(tx: any, userId: string, marketPda: string, asset: Ob2Asset): Promise<bigint> {
    if (asset === "USDC") {
      const row = await tx.ob2UserBalance.findUnique({ where: { userId_asset: { userId, asset } } });
      return row ? toMicro(String(row.free)) : 0n;
    }
    const row = await tx.ob2UserMarketBalance.findUnique({
      where: { userId_marketPda_asset: { userId, marketPda, asset } },
    });
    return row ? toMicro(String(row.free)) : 0n;
  }
```

- [ ] **Step 3.2: Refactor `release()`.**

Same pattern — branch on `res.asset`. Replace the inner UPDATE block:

```typescript
      if (res.asset === "USDC") {
        await tx.$executeRawUnsafe(
          `UPDATE ob2_user_balances
              SET reserved = reserved - $3::numeric,
                  free = free + $3::numeric,
                  updated_at = now()
            WHERE user_id = $1 AND asset = $2::"Ob2Asset"`,
          res.userId, res.asset, amountStr,
        );
      } else {
        await tx.$executeRawUnsafe(
          `UPDATE ob2_user_market_balances
              SET reserved = reserved - $4::numeric,
                  free = free + $4::numeric,
                  updated_at = now()
            WHERE user_id = $1 AND market_pda = $2 AND asset = $3::"Ob2Asset"`,
          res.userId, res.marketPda, res.asset, amountStr,
        );
      }
```

- [ ] **Step 3.3: Refactor `releasePartial()` and `consumePartial()`.** Same branching pattern. `consumePartial` only debits reserved (no free credit).

For `releasePartial`:

```typescript
      if (res.asset === "USDC") {
        await tx.$executeRawUnsafe(
          `UPDATE ob2_user_balances
              SET reserved = reserved - $3::numeric,
                  free = free + $3::numeric,
                  updated_at = now()
            WHERE user_id = $1 AND asset = $2::"Ob2Asset"`,
          res.userId, res.asset, amountStr,
        );
      } else {
        await tx.$executeRawUnsafe(
          `UPDATE ob2_user_market_balances
              SET reserved = reserved - $4::numeric,
                  free = free + $4::numeric,
                  updated_at = now()
            WHERE user_id = $1 AND market_pda = $2 AND asset = $3::"Ob2Asset"`,
          res.userId, res.marketPda, res.asset, amountStr,
        );
      }
```

For `consumePartial`:

```typescript
      if (res.asset === "USDC") {
        await tx.$executeRawUnsafe(
          `UPDATE ob2_user_balances
              SET reserved = reserved - $3::numeric,
                  updated_at = now()
            WHERE user_id = $1 AND asset = $2::"Ob2Asset"`,
          res.userId, res.asset, amountStr,
        );
      } else {
        await tx.$executeRawUnsafe(
          `UPDATE ob2_user_market_balances
              SET reserved = reserved - $4::numeric,
                  updated_at = now()
            WHERE user_id = $1 AND market_pda = $2 AND asset = $3::"Ob2Asset"`,
          res.userId, res.marketPda, res.asset, amountStr,
        );
      }
```

- [ ] **Step 3.4: tsc clean.**

```bash
bun x tsc --noEmit 2>&1 | grep "reservation\\.service" || echo "clean"
```

- [ ] **Step 3.5: Run reservation tests.** Existing tests use USDC reservations; must update them to first ensure USDC row exists in `ob2_user_balances`.

```bash
bun x jest src/modules/trading-v2/__tests__/reservation.service --runInBand 2>&1 | tail -20
```

Expected: tests fail because they call `balanceRepo.upsertOnchain(USER, MARKET, "USDC", ...)` which is now invalid for USDC. Refactor the test setup to use `userBalanceRepo.upsertOnchain(USER, "USDC", ...)` for USDC seeding.

In the test file, add at top:

```typescript
import { UserBalanceRepository } from "../repositories/user-balance.repository";
const userBalanceRepo = new UserBalanceRepository(prisma);
```

Replace every `balanceRepo.upsertOnchain(USER, MARKET, "USDC", ...)` with `userBalanceRepo.upsertOnchain(USER, "USDC", ...)`. Add `await prisma.ob2UserBalance.deleteMany({});` to `beforeEach`.

- [ ] **Step 3.6: Run, all reservation tests pass.**

- [ ] **Step 3.7: Commit.**

```bash
git add api/src/modules/trading-v2/services/reservation.service.ts \
        api/src/modules/trading-v2/__tests__/reservation.service.integration.test.ts \
        api/src/modules/trading-v2/__tests__/reservation.service.invariant-i1.test.ts
git commit -m "feat(trading-v2): ReservationService roteia USDC→global, YES/NO→per-market"
```

---

## Task 4: BalanceService refactor

`BalanceService` é a fachada de leitura. `get` precisa rotear, `getAll` precisa combinar. `checkInvariantI2` precisa nova lógica.

**Files:**
- Modify: `api/src/modules/trading-v2/services/balance.service.ts`

- [ ] **Step 4.1: Refactor `BalanceService`.**

```typescript
import type { PrismaClient, Ob2Asset } from "../../../generated/prisma/client";
import { BalanceRepository } from "../repositories/balance.repository";
import { UserBalanceRepository } from "../repositories/user-balance.repository";
import type { MarketBalance, UserBalance } from "../types/balance.types";
import { toMicro } from "../types/decimal-helpers";

export interface InvariantI2Result {
  holds: boolean;
  free: bigint;
  reserved: bigint;
  sumOfActiveReservations: bigint;
  onchainTotal: bigint;
}

export class BalanceService {
  private readonly marketRepo: BalanceRepository;
  private readonly userRepo: UserBalanceRepository;

  constructor(private readonly prisma: PrismaClient) {
    this.marketRepo = new BalanceRepository(prisma);
    this.userRepo = new UserBalanceRepository(prisma);
  }

  /** USDC retorna UserBalance (sem marketPda); YES/NO retorna MarketBalance. */
  async get(userId: string, marketPda: string, asset: Ob2Asset): Promise<MarketBalance | UserBalance | null> {
    if (asset === "USDC") return this.userRepo.get(userId, asset);
    return this.marketRepo.get(userId, marketPda, asset);
  }

  async getAll(userId: string, marketPda: string): Promise<{
    usdc: UserBalance | null;
    yes: MarketBalance | null;
    no: MarketBalance | null;
  }> {
    const [usdc, yes, no] = await Promise.all([
      this.userRepo.get(userId, "USDC"),
      this.marketRepo.get(userId, marketPda, "YES"),
      this.marketRepo.get(userId, marketPda, "NO"),
    ]);
    return { usdc, yes, no };
  }

  /**
   * Para USDC: a invariante é GLOBAL — soma de todas reservations USDC ativas
   * (across markets) == reserved no ob2_user_balances. Para YES/NO: per-market.
   */
  async checkInvariantI2(
    userId: string, marketPda: string, asset: Ob2Asset,
  ): Promise<InvariantI2Result> {
    if (asset === "USDC") {
      const bal = await this.userRepo.get(userId, "USDC");
      const resSum = await this.prisma.ob2Reservation.aggregate({
        _sum: { amount: true },
        where: { userId, asset: "USDC", releasedAt: null },
      });
      const sumOfActive = resSum._sum.amount ? toMicro(String(resSum._sum.amount)) : 0n;
      const free = bal?.free ?? 0n;
      const reserved = bal?.reserved ?? 0n;
      const onchainTotal = bal?.onchainTotal ?? 0n;
      return { holds: reserved === sumOfActive, free, reserved, sumOfActiveReservations: sumOfActive, onchainTotal };
    }
    // YES/NO per-market
    const bal = await this.marketRepo.get(userId, marketPda, asset);
    const resSum = await this.prisma.ob2Reservation.aggregate({
      _sum: { amount: true },
      where: { userId, marketPda, asset, releasedAt: null },
    });
    const sumOfActive = resSum._sum.amount ? toMicro(String(resSum._sum.amount)) : 0n;
    const free = bal?.free ?? 0n;
    const reserved = bal?.reserved ?? 0n;
    const onchainTotal = bal?.onchainTotal ?? 0n;
    return { holds: reserved === sumOfActive, free, reserved, sumOfActiveReservations: sumOfActive, onchainTotal };
  }
}
```

- [ ] **Step 4.2: Update `balance.service.invariant-i2.test.ts`.** Tests probably use USDC heavily. The test now needs `userBalanceRepo.upsertOnchain` for seeding instead of `balanceRepo.upsertOnchain(..., "USDC", ...)`.

Walk through each test, replace USDC seeding with the new repo.

- [ ] **Step 4.3: Run tests.**

```bash
bun x jest src/modules/trading-v2/__tests__/balance.service.invariant-i2 --runInBand
```

Expected: all pass.

- [ ] **Step 4.4: Commit.**

```bash
git add api/src/modules/trading-v2/services/balance.service.ts \
        api/src/modules/trading-v2/__tests__/balance.service.invariant-i2.test.ts
git commit -m "feat(trading-v2): BalanceService rotea USDC global vs YES/NO per-market"
```

---

## Task 5: BalanceRepository — narrow to YES/NO

**Files:**
- Modify: `api/src/modules/trading-v2/repositories/balance.repository.ts`
- Modify: `api/src/modules/trading-v2/__tests__/balance.repository.integration.test.ts`

`BalanceRepository.upsertOnchain` continua aceitando todos os assets pra retrocompat de outros tests, mas opcionalmente pode validar que asset != USDC. Pragmático: mantém o método aceitando qualquer asset (raw SQL acessa `ob2_user_market_balances`); se asset=USDC, a row vai ser criada na tabela errada (ainda funciona mas é sintoma de bug).

Cleaner: bloquear USDC explicitamente. Adicionar guard:

```typescript
  async upsertOnchain(
    userId: string, marketPda: string, asset: Ob2Asset,
    onchainTotal: bigint, onchainSlot: bigint,
  ): Promise<void> {
    if (asset === "USDC") {
      throw new Error("USDC balances are global — use UserBalanceRepository instead");
    }
    // ... existing impl ...
  }

  async get(userId: string, marketPda: string, asset: Ob2Asset): Promise<MarketBalance | null> {
    if (asset === "USDC") {
      throw new Error("USDC balances are global — use UserBalanceRepository.get instead");
    }
    // ... existing impl ...
  }
```

- [ ] **Step 5.1: Add guards** to `BalanceRepository.upsertOnchain` and `BalanceRepository.get`.

- [ ] **Step 5.2: Update existing test file `balance.repository.integration.test.ts`.** Replace USDC tests with NO/YES tests (since USDC is now invalid):

```typescript
test("upsertOnchain (YES) sets free=total and reserved=0 for new row", async () => {
  await repo.upsertOnchain(USER, MARKET, "YES", 1000n * UNIT, 12345n);
  const b = await repo.get(USER, MARKET, "YES");
  expect(b!.free).toBe(1000n * UNIT);
  expect(b!.reserved).toBe(0n);
  expect(b!.onchainTotal).toBe(1000n * UNIT);
});

test("upsertOnchain on existing row only updates onchain snapshot (not free/reserved)", async () => {
  await repo.upsertOnchain(USER, MARKET, "YES", 1000n * UNIT, 12345n);
  await prisma.ob2UserMarketBalance.update({
    where: { userId_marketPda_asset: { userId: USER, marketPda: MARKET, asset: "YES" } },
    data: { free: "600", reserved: "400" },
  });
  await repo.upsertOnchain(USER, MARKET, "YES", 1000n * UNIT, 99999n);
  const b = await repo.get(USER, MARKET, "YES");
  expect(b!.free).toBe(600n * UNIT);
  expect(b!.reserved).toBe(400n * UNIT);
  expect(b!.onchainSlot).toBe(99999n);
});

test("get returns null when row missing", async () => {
  const b = await repo.get(USER, MARKET, "NO");
  expect(b).toBeNull();
});

test("upsertOnchain throws when asset=USDC", async () => {
  await expect(
    repo.upsertOnchain(USER, MARKET, "USDC", 1000n * UNIT, 1n)
  ).rejects.toThrow(/USDC balances are global/);
});
```

- [ ] **Step 5.3: Run, 4 tests pass.**

- [ ] **Step 5.4: Commit.**

```bash
git add api/src/modules/trading-v2/repositories/balance.repository.ts \
        api/src/modules/trading-v2/__tests__/balance.repository.integration.test.ts
git commit -m "feat(trading-v2): BalanceRepository rejeita USDC explicitamente"
```

---

## Task 6: StubSettler — route por asset

**Files:**
- Modify: `api/src/modules/trading-v2/services/stub-settler.service.ts`
- Modify: `api/src/modules/trading-v2/__tests__/stub-settler.integration.test.ts` and `stub-settler-with-fees.integration.test.ts`

Os helpers `debitReservedAndShrink` e `creditFree` precisam rotear pela asset.

- [ ] **Step 6.1: Modify `debitReservedAndShrink`** in `stub-settler.service.ts`:

```typescript
  private async debitReservedAndShrink(
    tx: any, userId: string, marketPda: string, asset: Ob2Asset, amount: bigint, reservationId: string,
  ): Promise<void> {
    const amountStr = fromMicro(amount);
    if (asset === "USDC") {
      await tx.$executeRawUnsafe(
        `UPDATE ob2_user_balances
            SET reserved = reserved - $3::numeric, updated_at = now()
          WHERE user_id = $1 AND asset = $2::"Ob2Asset"`,
        userId, asset, amountStr,
      );
    } else {
      await tx.$executeRawUnsafe(
        `UPDATE ob2_user_market_balances
            SET reserved = reserved - $4::numeric, updated_at = now()
          WHERE user_id = $1 AND market_pda = $2 AND asset = $3::"Ob2Asset"`,
        userId, marketPda, asset, amountStr,
      );
    }
    // Shrink reservation row (unchanged)
    const res = await tx.ob2Reservation.findUnique({ where: { id: reservationId } });
    if (!res) return;
    const current = toMicro(String(res.amount));
    const remaining = current - amount;
    if (remaining <= 0n) {
      await tx.ob2Reservation.update({ where: { id: reservationId }, data: { releasedAt: new Date() } });
    } else {
      await tx.ob2Reservation.update({ where: { id: reservationId }, data: { amount: fromMicro(remaining) } });
    }
  }
```

- [ ] **Step 6.2: Modify `creditFree`**:

```typescript
  private async creditFree(
    tx: any, userId: string, marketPda: string, asset: Ob2Asset, amount: bigint,
  ): Promise<void> {
    const amountStr = fromMicro(amount);
    if (asset === "USDC") {
      await tx.ob2UserBalance.upsert({
        where: { userId_asset: { userId, asset } },
        create: { userId, asset, free: "0", reserved: "0", onchainTotal: "0" },
        update: {},
      });
      await tx.$executeRawUnsafe(
        `UPDATE ob2_user_balances
            SET free = free + $3::numeric, updated_at = now()
          WHERE user_id = $1 AND asset = $2::"Ob2Asset"`,
        userId, asset, amountStr,
      );
    } else {
      await tx.ob2UserMarketBalance.upsert({
        where: { userId_marketPda_asset: { userId, marketPda, asset } },
        create: { userId, marketPda, asset, free: "0", reserved: "0", onchainTotal: "0" },
        update: {},
      });
      await tx.$executeRawUnsafe(
        `UPDATE ob2_user_market_balances
            SET free = free + $4::numeric, updated_at = now()
          WHERE user_id = $1 AND market_pda = $2 AND asset = $3::"Ob2Asset"`,
        userId, marketPda, asset, amountStr,
      );
    }
  }
```

- [ ] **Step 6.3: Update stub-settler tests.** All test setup that does `balanceRepo.upsertOnchain(USER, MARKET, "USDC", ...)` must use `userBalanceRepo.upsertOnchain(USER, "USDC", ...)`. All assertions reading USDC balance must use the new repo. Add `await prisma.ob2UserBalance.deleteMany({})` to `beforeEach`.

- [ ] **Step 6.4: Run.**

```bash
bun x jest src/modules/trading-v2/__tests__/stub-settler --runInBand
```

Expected: all pass.

- [ ] **Step 6.5: Commit.**

```bash
git add api/src/modules/trading-v2/services/stub-settler.service.ts \
        api/src/modules/trading-v2/__tests__/stub-settler.integration.test.ts \
        api/src/modules/trading-v2/__tests__/stub-settler-with-fees.integration.test.ts
git commit -m "feat(trading-v2): StubSettler rotea consume/credit por asset"
```

---

## Task 7: SettlementReverter — route por asset

**Files:**
- Modify: `api/src/modules/trading-v2/services/settlement-reverter.service.ts`
- Modify: `api/src/modules/trading-v2/__tests__/settlement-reverter.integration.test.ts`

Os helpers `creditReserved` e `debitFree` precisam mesmo roteamento.

- [ ] **Step 7.1: Modify `creditReserved`** (used in revert to restore reservation balance):

```typescript
  private async creditReserved(
    tx: any, userId: string, marketPda: string, asset: Ob2Asset, amount: bigint,
  ): Promise<void> {
    if (amount === 0n) return;
    const amountStr = fromMicro(amount);
    if (asset === "USDC") {
      await tx.ob2UserBalance.upsert({
        where: { userId_asset: { userId, asset } },
        create: { userId, asset, free: "0", reserved: "0", onchainTotal: "0" },
        update: {},
      });
      await tx.$executeRawUnsafe(
        `UPDATE ob2_user_balances
            SET reserved = reserved + $3::numeric, updated_at = now()
          WHERE user_id = $1 AND asset = $2::"Ob2Asset"`,
        userId, asset, amountStr,
      );
    } else {
      await tx.ob2UserMarketBalance.upsert({
        where: { userId_marketPda_asset: { userId, marketPda, asset } },
        create: { userId, marketPda, asset, free: "0", reserved: "0", onchainTotal: "0" },
        update: {},
      });
      await tx.$executeRawUnsafe(
        `UPDATE ob2_user_market_balances
            SET reserved = reserved + $4::numeric, updated_at = now()
          WHERE user_id = $1 AND market_pda = $2 AND asset = $3::"Ob2Asset"`,
        userId, marketPda, asset, amountStr,
      );
    }
  }
```

- [ ] **Step 7.2: Modify `debitFree`** (used in revert to undo a credit):

```typescript
  private async debitFree(
    tx: any, userId: string, marketPda: string, asset: Ob2Asset, amount: bigint,
  ): Promise<void> {
    if (amount === 0n) return;
    const amountStr = fromMicro(amount);
    if (asset === "USDC") {
      await tx.$executeRawUnsafe(
        `UPDATE ob2_user_balances
            SET free = free - $3::numeric, updated_at = now()
          WHERE user_id = $1 AND asset = $2::"Ob2Asset"
            AND free >= $3::numeric`,
        userId, asset, amountStr,
      );
    } else {
      await tx.$executeRawUnsafe(
        `UPDATE ob2_user_market_balances
            SET free = free - $4::numeric, updated_at = now()
          WHERE user_id = $1 AND market_pda = $2 AND asset = $3::"Ob2Asset"
            AND free >= $4::numeric`,
        userId, marketPda, asset, amountStr,
      );
    }
  }
```

- [ ] **Step 7.3: Update revert tests.** Same pattern: replace USDC seeding/assertions.

- [ ] **Step 7.4: Run, all pass.**

- [ ] **Step 7.5: Commit.**

```bash
git add api/src/modules/trading-v2/services/settlement-reverter.service.ts \
        api/src/modules/trading-v2/__tests__/settlement-reverter.integration.test.ts
git commit -m "feat(trading-v2): SettlementReverter rotea por asset"
```

---

## Task 8: Refactor remaining test files

Remaining tests still seed USDC via the old repo. Walk through each and adjust:

**Files that need updating:**
- `place-order.use-case.integration.test.ts`
- `cancel-order.use-case.integration.test.ts`
- `place-order-window.integration.test.ts`
- `matching-engine.integration.test.ts`
- `matching-scenarios.e2e.test.ts`
- `solana-settler.integration.test.ts`
- `solana-settler-idempotency.integration.test.ts`
- `reconcile-settling-trades.integration.test.ts`
- `reconcile-settling-trades-with-event.integration.test.ts`
- `onchain-event-listener.unit.test.ts`
- `event-emission.integration.test.ts`
- `fee-ledger-integration.integration.test.ts`
- `snapshot-onchain-balances.integration.test.ts`

For each:
1. Add import: `import { UserBalanceRepository } from "../repositories/user-balance.repository";`
2. Add instance: `const userBalanceRepo = new UserBalanceRepository(prisma);`
3. Add to beforeEach: `await prisma.ob2UserBalance.deleteMany({});`
4. Replace `balanceRepo.upsertOnchain(USER, MARKET, "USDC", ...)` with `userBalanceRepo.upsertOnchain(USER, "USDC", ...)`
5. Replace `balanceRepo.get(USER, MARKET, "USDC")` with `userBalanceRepo.get(USER, "USDC")`

- [ ] **Step 8.1: Refactor `place-order.use-case.integration.test.ts`** following the pattern.

- [ ] **Step 8.2: Refactor `cancel-order.use-case.integration.test.ts`.**

- [ ] **Step 8.3: Refactor `place-order-window.integration.test.ts`.**

- [ ] **Step 8.4: Refactor `matching-engine.integration.test.ts`.**

- [ ] **Step 8.5: Refactor `matching-scenarios.e2e.test.ts`.**

- [ ] **Step 8.6: Refactor `solana-settler.integration.test.ts` + `solana-settler-idempotency.integration.test.ts`.**

- [ ] **Step 8.7: Refactor `reconcile-settling-trades.integration.test.ts` + `reconcile-settling-trades-with-event.integration.test.ts`.**

- [ ] **Step 8.8: Refactor `onchain-event-listener.unit.test.ts` + `event-emission.integration.test.ts` + `fee-ledger-integration.integration.test.ts`.**

- [ ] **Step 8.9: Refactor `snapshot-onchain-balances.integration.test.ts`** — the snapshot script's `runSnapshot` will change behavior: USDC writes go to `ob2_user_balances`. Update test fakes and assertions accordingly.

- [ ] **Step 8.10: Run full suite.**

```bash
bun x jest src/modules/trading-v2 --runInBand 2>&1 | tail -10
```

Expected: all pass (~174 tests, ±some adjustments).

- [ ] **Step 8.11: Commit.**

```bash
git add api/src/modules/trading-v2/__tests__/
git commit -m "refactor(trading-v2): testes ajustam USDC pra UserBalanceRepository"
```

---

## Task 9: SolanaOnchainBalanceReader + snapshot script — USDC global

**Files:**
- Modify: `api/src/modules/trading-v2/services/solana-onchain-balance-reader.ts`
- Modify: `api/src/modules/trading-v2/scripts/snapshot-onchain-balances.ts`

The reader currently returns `{ usdc, yes, no, slot }` per (user, market). The snapshot script writes ALL into `ob2_user_market_balances` per market — incluindo USDC, que agora é wrong.

Two approaches:
- **(a) Keep reader API**, change snapshot logic to deduplicate USDC writes.
- **(b) Split reader API** into `readUserUsdc(userId)` + `readMarketTokens(userId, marketPda)`.

(b) is cleaner but more disruption. (a) is pragmatic. Choose (a).

- [ ] **Step 9.1: Modify `runSnapshot` in `snapshot-onchain-balances.ts`** to write USDC once per user:

```typescript
import { UserBalanceRepository } from "../repositories/user-balance.repository";

export async function runSnapshot(
  prisma: PrismaClient,
  reader: OnchainBalanceReader,
  filter: { userId?: string; marketPda?: string } = {},
): Promise<{ total: number; errors: Array<{ userId: string; marketPda: string; err: string }> }> {
  const repo = new BalanceRepository(prisma);
  const userRepo = new UserBalanceRepository(prisma);
  const pairs = await reader.listUserMarkets(filter);
  const errors: Array<{ userId: string; marketPda: string; err: string }> = [];
  let total = 0;
  const usdcSeenForUser = new Set<string>();

  for (const { userId, marketPda } of pairs) {
    try {
      const { usdc, yes, no, slot } = await reader.read(userId, marketPda);
      // USDC: global per user. Write only once per userId.
      if (!usdcSeenForUser.has(userId)) {
        await userRepo.upsertOnchain(userId, "USDC", usdc, slot);
        usdcSeenForUser.add(userId);
      }
      // YES/NO: per market.
      await repo.upsertOnchain(userId, marketPda, "YES", yes, slot);
      await repo.upsertOnchain(userId, marketPda, "NO", no, slot);
      total++;
    } catch (e) {
      errors.push({ userId, marketPda, err: e instanceof Error ? e.message : String(e) });
    }
  }

  return { total, errors };
}
```

- [ ] **Step 9.2: Update `snapshot-onchain-balances.integration.test.ts`** assertions: USDC now in `ob2_user_balances`, not per-market.

```typescript
import { UserBalanceRepository } from "../repositories/user-balance.repository";
const userRepo = new UserBalanceRepository(prisma);

beforeEach(async () => {
  await prisma.ob2UserBalance.deleteMany({});
  // ... existing cleanups
});

test("snapshot populates USDC globally and YES/NO per-market", async () => {
  await runSnapshot(prisma, fakeReader);
  const usdc = await userRepo.get(USER, "USDC");
  expect(usdc!.free).toBe(500n * UNIT);
  const yes = await repo.get(USER, MARKET, "YES");
  expect(yes!.free).toBe(100n * UNIT);
  const no = await repo.get(USER, MARKET, "NO");
  expect(no!.free).toBe(0n);
});
```

- [ ] **Step 9.3: Run.**

```bash
bun x jest src/modules/trading-v2/__tests__/snapshot-onchain-balances --runInBand
```

Expected: pass.

- [ ] **Step 9.4: Commit.**

```bash
git add api/src/modules/trading-v2/scripts/snapshot-onchain-balances.ts \
        api/src/modules/trading-v2/__tests__/snapshot-onchain-balances.integration.test.ts
git commit -m "feat(trading-v2): snapshot escreve USDC global once-per-user"
```

---

## Task 10: DailyReconciliationService — USDC global

**Files:**
- Modify: `api/src/modules/trading-v2/services/daily-reconciliation.service.ts`
- Modify: `api/src/modules/trading-v2/__tests__/daily-reconciliation.integration.test.ts`

Currently `reconcile()` calls `repo.upsertOnchain(userId, marketPda, "USDC", ...)` and compares per-market. Move USDC to global.

- [ ] **Step 10.1: Refactor `reconcile()`:**

```typescript
import { BalanceRepository } from "../repositories/balance.repository";
import { UserBalanceRepository } from "../repositories/user-balance.repository";

export class DailyReconciliationService {
  private readonly repo: BalanceRepository;
  private readonly userRepo: UserBalanceRepository;
  private readonly reconciler = new ReconciliationService();

  constructor(
    private readonly prisma: PrismaClient,
    private readonly reader: OnchainBalanceReader,
  ) {
    this.repo = new BalanceRepository(prisma);
    this.userRepo = new UserBalanceRepository(prisma);
  }

  async reconcile(filter?: { userId?: string; marketPda?: string }): Promise<ReconcileResult> {
    const pairs = await this.reader.listUserMarkets(filter);
    const drifts: DriftEntry[] = [];
    const errors: ReconcileResult["errors"] = [];
    const usdcSeenForUser = new Set<string>();
    let total = 0;

    for (const { userId, marketPda } of pairs) {
      try {
        const { usdc, yes, no, slot } = await this.reader.read(userId, marketPda);

        // USDC global — once per user
        if (!usdcSeenForUser.has(userId)) {
          await this.userRepo.upsertOnchain(userId, "USDC", usdc, slot);
          const bal = await this.userRepo.get(userId, "USDC");
          if (bal) {
            const r = this.reconciler.compare({
              free: bal.free, reserved: bal.reserved, onchainTotal: usdc,
            });
            if (!r.inSync) {
              drifts.push({
                userId, marketPda: "(global)", asset: "USDC",
                expected: bal.free + bal.reserved,
                actual: usdc, diff: r.diff, direction: r.direction,
              });
            }
          }
          usdcSeenForUser.add(userId);
        }

        // YES/NO per-market
        await this.repo.upsertOnchain(userId, marketPda, "YES", yes, slot);
        await this.repo.upsertOnchain(userId, marketPda, "NO", no, slot);

        for (const [asset, onchainAmount] of [["YES", yes], ["NO", no]] as const) {
          const bal = await this.repo.get(userId, marketPda, asset);
          if (!bal) continue;
          const r = this.reconciler.compare({
            free: bal.free, reserved: bal.reserved, onchainTotal: onchainAmount,
          });
          if (!r.inSync) {
            drifts.push({
              userId, marketPda, asset,
              expected: bal.free + bal.reserved,
              actual: onchainAmount, diff: r.diff, direction: r.direction,
            });
          }
        }
        total++;
      } catch (e) {
        errors.push({ userId, marketPda, err: e instanceof Error ? e.message : String(e) });
      }
    }

    return { total, drifts, errors };
  }
}
```

- [ ] **Step 10.2: Update reconciliation tests** to use `userRepo` for USDC seeding.

- [ ] **Step 10.3: Run.**

Expected: all pass.

- [ ] **Step 10.4: Commit.**

```bash
git add api/src/modules/trading-v2/services/daily-reconciliation.service.ts \
        api/src/modules/trading-v2/__tests__/daily-reconciliation.integration.test.ts
git commit -m "feat(trading-v2): DailyReconciliationService trata USDC globalmente"
```

---

## Task 11: Barrel + final suite + dry-run validation

**Files:**
- Modify: `api/src/modules/trading-v2/index.ts`
- Modify: `api/src/modules/trading-v2/README.md`

- [ ] **Step 11.1: Append to barrel:**

```typescript
export { UserBalanceRepository } from "./repositories/user-balance.repository";
```

- [ ] **Step 11.2: README — add section.**

```markdown
## Storage de balances (Plano 10)

- **USDC** vive em `ob2_user_balances (user_id, asset)` — global, matches on-chain ATA per-wallet.
- **YES/NO** vivem em `ob2_user_market_balances (user_id, market_pda, asset)` — per-market, matches on-chain mints per-market.
- **Reservations** continuam per-market em `ob2_reservations` (cada ordem é de um market específico). USDC reservation rows ainda têm `marketPda` (do order) mas debitam balance global.

Routing automático:
- `ReservationService.reserve/release/etc` rotea por `asset`.
- `StubSettler.applyDeltas` rotea por `asset`.
- `SettlementReverter` rotea por `asset`.
- `BalanceService.get` rotea: USDC → `UserBalanceRepository`; YES/NO → `BalanceRepository`.
- `BalanceService.checkInvariantI2` (USDC): soma reservations USDC do user across todos os markets == reserved global.
- `BalanceService.checkInvariantI2` (YES/NO): soma per-market.
```

- [ ] **Step 11.3: Run full suite.**

```bash
cd /Users/rafael/Documents/GitHub/Oraculo/.claude/worktrees/orderbook-rewrite-10-split-usdc/api
bun x jest src/modules/trading-v2 src/services/mm-bot --runInBand 2>&1 | tail -5
```

Expected: ~177 passed + 1 skipped (174 baseline + 3 user-balance.repository tests + others maybe ±).

- [ ] **Step 11.4: Re-run dry-run against prod-test DB.**

```bash
DATABASE_URL="postgresql://astron:astron@localhost:5434/astron_wallet_prod_test" \
  bun run src/modules/trading-v2/scripts/cutover-dry-run.ts
```

Expected: same 91 pairs, same balances. Now confirm: if we ran Tier 2 snapshot, USDC would be written ONCE per user (12 rows in `ob2_user_balances`), not 91 times.

- [ ] **Step 11.5: tsc clean.**

```bash
bun x tsc --noEmit 2>&1 | grep trading-v2 || echo "clean"
```

- [ ] **Step 11.6: Commit.**

```bash
git add api/src/modules/trading-v2/index.ts api/src/modules/trading-v2/README.md
git commit -m "docs(trading-v2): README split USDC/YES-NO + barrel pós-plano 10"
```

---

## Critérios de aceitação

1. ✅ Tabela `ob2_user_balances` criada com PK `(user_id, asset)` + CHECKs.
2. ✅ `UserBalanceRepository` implementado.
3. ✅ `BalanceRepository.upsertOnchain/get` rejeita USDC com erro claro.
4. ✅ `ReservationService` rotea USDC↔global, YES/NO↔per-market — invariantes I1/I2 mantidas.
5. ✅ `StubSettler.applyDeltas` rotea consume/credit por asset.
6. ✅ `SettlementReverter` rotea restore por asset.
7. ✅ `BalanceService` fachada combinada.
8. ✅ Snapshot script escreve USDC ONCE per user.
9. ✅ Daily reconciliation trata USDC globalmente.
10. ✅ Suite trading-v2 verde + dry-run prod-test confirma USDC writes deduplicados.

---

## Riscos e mitigações

| Risco | Mitigação |
|---|---|
| Reservations USDC ainda têm `marketPda` populated, criando confusão | Documentar: marketPda é só pra trace (qual order originou); o balance afetado é GLOBAL. Aggregate em `checkInvariantI2(USDC)` ignora marketPda. |
| Tests legacy quebrados em massa por refactor | Walk-through em Tasks 8 + 10 cobre todos. Pattern é mecânico (replace upsertOnchain → userRepo.upsertOnchain pra USDC). |
| Reservation com USDC e seller-side opening short reservou USDC, settlement consumiu USDC global — mas IntentClassifier ainda atribui `marketPda` da order na reservation | OK — a reservation row tem marketPda (do order), mas o balance debitado é global. Settler já lê `reservation.asset` pra rotear (Task 6). |
| Dry-run pos-Plano10 mostra mesmo dado mas Tier 2 snapshot vai criar 12 USDC rows ao invés de 91 | Esse é o fix — comportamento desejado. Validar Tier 2 antes do cutover real. |

---

## O que NÃO está neste plano

- **Migration de dados existentes em prod**: prod ainda não fez cutover; o `ob2_user_market_balances` está vazio em prod, o `DELETE FROM ob2_user_market_balances WHERE asset='USDC'` é zero-impact.
- **Rollback**: nenhum plano de rollback estruturado. Se o split causar problema, drop a tabela `ob2_user_balances` + revert as alterações no código.
