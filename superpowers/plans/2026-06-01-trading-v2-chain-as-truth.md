# Trading V2 — Rebuild chain-as-truth: Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Substituir a camada de ledger/settlement do trading-v2 pelo modelo chain-as-truth: saldo derivado (`available = confirmedOnchain + Σ pendingDeltas − Σ openOrderCommitments`), sem `free`/`reserved` mutáveis, sem reservas, sem apply-otimista-e-reverte.

**Architecture:** O DB deixa de ser autoridade. `confirmedOnchain` espelha a chain (escrito só no fold de confirmação + reconciliação). Ordens abertas carregam seu compromisso; trades em voo geram `Ob2SettlementDelta` assinados. Settlement escreve a verdade on-chain (tx atômica, inalterada) e dobra o delta conhecido no espelho. Falha reabre ordens, sem desfazer saldo. Mantém intactos: contrato on-chain, `SolanaOnchainCaller`, lógica de matching, rotas, eventos.

**Tech Stack:** Bun, TypeScript, Prisma 7 (PostgreSQL), Jest (ts-jest), Solana/Anchor, `@/` path alias.

**Spec:** `docs/superpowers/specs/2026-06-01-trading-v2-chain-as-truth-rebuild-design.md`

**Convenções deste plano:**
- Branch já criada: `feat/trading-v2-chain-as-truth`.
- Testes unitários puros: nome `*.unit.test.ts`. Rodar com guarda de prod: prefixar `DATABASE_URL='postgresql://invalid:5432/none' BLOCK_EXTERNAL_SERVICES=true`.
- Testes de integração: `*.integration.test.ts`, **só contra DB de teste/testcontainer — NUNCA prod** (o `.env` aponta pra prod e fazem `deleteMany`). Não rodar integração nesta máquina sem DB de teste configurado.
- Micro-unidades: `bigint` de 6 casas (helpers `toMicro`/`fromMicro` em `@/modules/trading-v2/types/decimal-helpers`).
- Commits frequentes, sem `Co-Authored-By`.

---

## File Structure

**Criar:**
- `api/src/modules/trading-v2/services/settlement-delta.ts` — pura: deltas assinados por primitive a partir de um fill.
- `api/src/modules/trading-v2/services/available-balance.ts` — pura: `computeAvailable` + `commitmentFor`.
- `api/src/modules/trading-v2/repositories/balance-mirror.repository.ts` — espelho `confirmedOnchain` (read + fold com `FOR UPDATE`).
- `api/src/modules/trading-v2/repositories/settlement-delta.repository.ts` — CRUD de deltas + soma de pendentes.
- `api/src/modules/trading-v2/services/order-admission.service.ts` — calcula `available` sob trava e admite/rejeita.
- `api/src/modules/trading-v2/services/settlement-worker.service.ts` — máquina de estados `SETTLING→SETTLED|FAILED` + recuperação.
- (já existe) `api/src/modules/trading-v2/services/reconcile-alert.ts` — alerta da reconciliação.

**Modificar:**
- `api/prisma/schema.prisma` — add `Ob2SettlementDelta`, add `Ob2Order.commitAsset/commitAmount`, add `FAILED` em `Ob2TradeStatus`; (cutover) drop `Ob2Reservation` + `free`/`reserved`.
- `api/src/modules/trading-v2/use-cases/place-order.use-case.ts` — admissão via `available`, insere ordem com compromisso.
- `api/src/modules/trading-v2/use-cases/cancel-order.use-case.ts` — só marca CANCELLED (compromisso some sozinho; sem release).
- `api/src/modules/trading-v2/services/matching-engine.service.ts` — bordas: lê `available`, escreve `Trade + SettlementDelta`, reduz compromisso no fill, reabre na falha.
- `api/src/modules/trading-v2/services/daily-reconciliation.service.ts` — sobrescreve espelho; agendar em `src/shared/jobs/index.ts`.

**Remover (na fase de limpeza):**
- `api/src/modules/trading-v2/services/reservation.service.ts`, `stub-settler.service.ts` (applyDeltas), `settlement-reverter.service.ts`.

---

## FASE 1 — Núcleo puro (sem DB)

### Task 1: Tipos e deltas de settlement (pura)

**Files:**
- Create: `api/src/modules/trading-v2/services/settlement-delta.ts`
- Test: `api/src/modules/trading-v2/__tests__/settlement-delta.unit.test.ts`

Reusa a matemática já verificada de `FeeComputationService` (`fee-computation.service.ts`) como fonte única, mapeando o resultado para deltas assinados. `marketPda=null` para USDC (global); setado para YES/NO.

- [ ] **Step 1: Escrever o teste falhando**

```typescript
// api/src/modules/trading-v2/__tests__/settlement-delta.unit.test.ts
import { computeSettlementDeltas, type SettlementDelta } from "../services/settlement-delta";

const UNIT = 1_000_000n;
const base = {
  marketPda: "MKT", buyerUserId: "B", sellerUserId: "S",
  quantityMicro: 10n * UNIT, priceBps: 6000, feeBps: 0,
};

function find(ds: SettlementDelta[], userId: string, asset: string) {
  return ds.find(d => d.userId === userId && d.asset === asset);
}

describe("computeSettlementDeltas", () => {
  test("TRADE (buyer USDC→YES): comprador −USDC/+YES, vendedor +USDC/−YES", () => {
    const ds = computeSettlementDeltas({ ...base, primitive: "TRADE", buyerReservationAsset: "USDC", sellerReservationAsset: "YES" });
    expect(find(ds, "B", "YES")!.amountMicro).toBe(10n * UNIT);
    expect(find(ds, "B", "USDC")!.amountMicro).toBe(-6n * UNIT); // 10 * 0.60
    expect(find(ds, "S", "YES")!.amountMicro).toBe(-10n * UNIT);
    expect(find(ds, "S", "USDC")!.amountMicro).toBe(6n * UNIT);
    // USDC marketPda é null (global); YES/NO setado
    expect(find(ds, "B", "USDC")!.marketPda).toBeNull();
    expect(find(ds, "B", "YES")!.marketPda).toBe("MKT");
  });

  test("conservação sem fee: soma de USDC = 0 e soma de cada token = 0", () => {
    const ds = computeSettlementDeltas({ ...base, primitive: "TRADE", buyerReservationAsset: "USDC", sellerReservationAsset: "YES" });
    const usdc = ds.filter(d => d.asset === "USDC").reduce((a, d) => a + d.amountMicro, 0n);
    const yes = ds.filter(d => d.asset === "YES").reduce((a, d) => a + d.amountMicro, 0n);
    expect(usdc).toBe(0n);
    expect(yes).toBe(0n);
  });

  test("MINT: comprador +YES/−USDC, vendedor +NO/−USDC; tokens criados, USDC sai dos dois", () => {
    const ds = computeSettlementDeltas({ ...base, primitive: "MINT", buyerReservationAsset: "USDC", sellerReservationAsset: "USDC" });
    expect(find(ds, "B", "YES")!.amountMicro).toBe(10n * UNIT);
    expect(find(ds, "S", "NO")!.amountMicro).toBe(10n * UNIT);
    expect(find(ds, "B", "USDC")!.amountMicro).toBe(-6n * UNIT);
    expect(find(ds, "S", "USDC")!.amountMicro).toBe(-4n * UNIT); // 10 * 0.40
  });
});
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `DATABASE_URL='postgresql://invalid:5432/none' BLOCK_EXTERNAL_SERVICES=true npx jest src/modules/trading-v2/__tests__/settlement-delta.unit.test.ts`
Expected: FAIL — `Cannot find module '../services/settlement-delta'`.

- [ ] **Step 3: Implementar mínimo**

```typescript
// api/src/modules/trading-v2/services/settlement-delta.ts
import { FeeComputationService } from "./fee-computation.service";
import type { Ob2Asset, Ob2Primitive } from "../../../generated/prisma/client";

export interface SettlementDelta {
  userId: string;
  asset: "USDC" | "YES" | "NO";
  marketPda: string | null; // null para USDC (global)
  amountMicro: bigint;       // assinado
}

export interface DeltaInput {
  marketPda: string;
  primitive: "TRADE" | "MINT" | "MERGE";
  buyerUserId: string;
  sellerUserId: string;
  buyerReservationAsset: Ob2Asset;
  sellerReservationAsset: Ob2Asset;
  quantityMicro: bigint;
  priceBps: number;
  feeBps: number;
}

const fees = new FeeComputationService();

/** Fonte única de deltas de settlement. Reusa FeeComputationService (verificado). */
export function computeSettlementDeltas(input: DeltaInput): SettlementDelta[] {
  const f = fees.compute({
    primitive: input.primitive,
    priceBps: input.priceBps,
    quantityMicro: input.quantityMicro,
    feeBps: input.feeBps,
    buyerReservationAsset: input.buyerReservationAsset,
    sellerReservationAsset: input.sellerReservationAsset,
  });
  const q = input.quantityMicro;
  const mkt = input.marketPda;
  const usdc = (userId: string, amt: bigint): SettlementDelta => ({ userId, asset: "USDC", marketPda: null, amountMicro: amt });
  const tok = (userId: string, asset: "YES" | "NO", amt: bigint): SettlementDelta =>
    ({ userId, asset, marketPda: mkt, amountMicro: amt });

  const out: SettlementDelta[] = [];
  switch (input.primitive) {
    case "TRADE":
      if (input.buyerReservationAsset === "USDC" && input.sellerReservationAsset === "YES") {
        out.push(usdc(input.buyerUserId, -f.buyerUsdcConsumedMicro), tok(input.buyerUserId, "YES", q));
        out.push(usdc(input.sellerUserId, f.sellerUsdcReceivedMicro), tok(input.sellerUserId, "YES", -q));
      } else { // buyer NO / seller USDC
        out.push(usdc(input.buyerUserId, f.buyerUsdcReceivedMicro), tok(input.buyerUserId, "NO", -q));
        out.push(usdc(input.sellerUserId, -f.sellerUsdcConsumedMicro), tok(input.sellerUserId, "NO", q));
      }
      break;
    case "MINT":
      out.push(usdc(input.buyerUserId, -f.buyerUsdcConsumedMicro), tok(input.buyerUserId, "YES", q));
      out.push(usdc(input.sellerUserId, -f.sellerUsdcConsumedMicro), tok(input.sellerUserId, "NO", q));
      break;
    case "MERGE":
      out.push(usdc(input.buyerUserId, f.buyerUsdcReceivedMicro), tok(input.buyerUserId, "NO", -q));
      out.push(usdc(input.sellerUserId, f.sellerUsdcReceivedMicro), tok(input.sellerUserId, "YES", -q));
      break;
  }
  return out;
}
```

> **Nota ao implementador:** confirme contra `fee-computation.service.ts` os nomes exatos dos campos retornados (`buyerUsdcConsumedMicro`, `sellerUsdcReceivedMicro`, etc.) e os sinais. Se o teste de conservação falhar, é sinal de divergência — ajuste o mapeamento (NÃO o teste).

- [ ] **Step 4: Rodar e ver passar**

Run: `DATABASE_URL='postgresql://invalid:5432/none' BLOCK_EXTERNAL_SERVICES=true npx jest src/modules/trading-v2/__tests__/settlement-delta.unit.test.ts`
Expected: PASS (3 testes).

- [ ] **Step 5: Commit**

```bash
git add api/src/modules/trading-v2/services/settlement-delta.ts api/src/modules/trading-v2/__tests__/settlement-delta.unit.test.ts
git commit -m "feat(trading-v2): computeSettlementDeltas puro (deltas assinados por primitive)"
```

### Task 2: `computeAvailable` e `commitmentFor` (puras)

**Files:**
- Create: `api/src/modules/trading-v2/services/available-balance.ts`
- Test: `api/src/modules/trading-v2/__tests__/available-balance.unit.test.ts`

- [ ] **Step 1: Escrever o teste falhando**

```typescript
// api/src/modules/trading-v2/__tests__/available-balance.unit.test.ts
import { computeAvailable, commitmentFor } from "../services/available-balance";

const UNIT = 1_000_000n;

describe("computeAvailable", () => {
  test("available = confirmado + pendentes − compromissos", () => {
    expect(computeAvailable({ confirmedMicro: 100n * UNIT, pendingDeltaMicro: -10n * UNIT, openCommitmentMicro: 30n * UNIT }))
      .toBe(60n * UNIT);
  });
  test("nunca depende de coluna mutável — função pura dos 3 termos", () => {
    expect(computeAvailable({ confirmedMicro: 0n, pendingDeltaMicro: 5n * UNIT, openCommitmentMicro: 0n })).toBe(5n * UNIT);
  });
});

describe("commitmentFor", () => {
  test("BUY USDC: compromisso = notional + fee da parte não preenchida", () => {
    // qty 10 @ 6000bps, fee 100bps → notional 6 USDC, fee 0.06 → 6.06
    expect(commitmentFor({ side: "BUY", asset: "USDC", priceBps: 6000, feeBps: 100, unfilledMicro: 10n * UNIT }))
      .toBe(6_060_000n);
  });
  test("SELL token: compromisso = a própria quantidade de token", () => {
    expect(commitmentFor({ side: "SELL", asset: "YES", priceBps: 6000, feeBps: 100, unfilledMicro: 10n * UNIT }))
      .toBe(10n * UNIT);
  });
});
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `DATABASE_URL='postgresql://invalid:5432/none' BLOCK_EXTERNAL_SERVICES=true npx jest src/modules/trading-v2/__tests__/available-balance.unit.test.ts`
Expected: FAIL — módulo não existe.

- [ ] **Step 3: Implementar mínimo**

```typescript
// api/src/modules/trading-v2/services/available-balance.ts
import type { Ob2Asset, Ob2Side } from "../../../generated/prisma/client";

export interface AvailableInput {
  confirmedMicro: bigint;       // espelho da chain
  pendingDeltaMicro: bigint;    // Σ deltas de trades SETTLING (assinado)
  openCommitmentMicro: bigint;  // Σ compromissos de ordens abertas
}

export function computeAvailable(i: AvailableInput): bigint {
  return i.confirmedMicro + i.pendingDeltaMicro - i.openCommitmentMicro;
}

export interface CommitmentInput {
  side: Ob2Side;
  asset: Ob2Asset;        // asset reservado pela ordem (USDC|YES|NO)
  priceBps: number;
  feeBps: number;
  unfilledMicro: bigint;  // quantidade ainda não preenchida
}

/** Compromisso da parte não preenchida de uma ordem. Espelha IntentClassifier. */
export function commitmentFor(i: CommitmentInput): bigint {
  if (i.asset !== "USDC") return i.unfilledMicro; // SELL de token trava a própria qty
  // USDC: notional + fee. BUY usa priceBps; SELL (colateral MINT) usa (10000 - priceBps).
  const priceComponent = i.side === "BUY" ? BigInt(i.priceBps) : BigInt(10000 - i.priceBps);
  const notional = (i.unfilledMicro * priceComponent) / 10000n;
  const fee = (notional * BigInt(i.feeBps)) / 10000n;
  return notional + fee;
}
```

> **Nota:** o `commitAsset` de uma ordem é decidido pelo `IntentClassifier` existente (BUY→USDC ou NO; SELL→YES ou USDC). Reuse-o na admissão (Task 6) — `commitmentFor` só calcula o valor dado o asset já classificado.

- [ ] **Step 4: Rodar e ver passar**

Run: `DATABASE_URL='postgresql://invalid:5432/none' BLOCK_EXTERNAL_SERVICES=true npx jest src/modules/trading-v2/__tests__/available-balance.unit.test.ts`
Expected: PASS (4 testes).

- [ ] **Step 5: Commit**

```bash
git add api/src/modules/trading-v2/services/available-balance.ts api/src/modules/trading-v2/__tests__/available-balance.unit.test.ts
git commit -m "feat(trading-v2): computeAvailable + commitmentFor puros"
```

---

## FASE 2 — Schema (aditivo, não-quebra)

### Task 3: Migration aditiva (novas estruturas)

**Files:**
- Modify: `api/prisma/schema.prisma`

Adições **não-quebram** o código atual (free/reserved/Ob2Reservation continuam existindo até o cutover na Fase 7).

- [ ] **Step 1: Adicionar `FAILED` ao enum e campos novos**

Em `api/prisma/schema.prisma`, no enum `Ob2TradeStatus` adicione `FAILED`:

```prisma
enum Ob2TradeStatus {
  SETTLING
  SETTLED
  REVERTED
  FAILED
}
```

Em `model Ob2Order`, adicione após `reservationId`:

```prisma
  commitAsset   Ob2Asset?       @map("commit_asset")
  commitAmount  Decimal         @default(0) @db.Decimal(20, 6) @map("commit_amount")
```

Adicione o novo modelo:

```prisma
model Ob2SettlementDelta {
  id        String   @id @default(uuid())
  tradeId   String   @map("trade_id")
  userId    String   @map("user_id")
  asset     Ob2Asset
  marketPda String?  @map("market_pda") // null = USDC global
  amount    Decimal  @db.Decimal(20, 6) // assinado
  createdAt DateTime @default(now()) @map("created_at")

  @@index([tradeId], map: "ob2_settlement_deltas_trade_idx")
  @@index([userId, asset], map: "ob2_settlement_deltas_user_asset_idx")
  @@map("ob2_settlement_deltas")
}
```

- [ ] **Step 2: Aplicar no DB + regenerar client (db push — o projeto NÃO usa migrations)**

Run: `cd api && npx prisma db push`
Expected: `Your database is now in sync with your Prisma schema` + client regenerado. (Só contra o DB local `localhost:5434`, NUNCA prod.)

- [ ] **Step 3: Verificar que o client compila com os novos tipos**

Run: `cd api && npx tsc --noEmit -p tsconfig.json 2>&1 | head -20`
Expected: sem erros novos relacionados a `Ob2SettlementDelta`/`commitAsset`.

- [ ] **Step 4: Commit**

```bash
git add api/prisma/schema.prisma api/prisma/migrations
git commit -m "feat(trading-v2): schema aditivo — Ob2SettlementDelta, commit fields, status FAILED"
```

---

## FASE 3 — Repositórios

### Task 4: `BalanceMirrorRepository` (espelho confirmado + fold com trava)

**Files:**
- Create: `api/src/modules/trading-v2/repositories/balance-mirror.repository.ts`
- Test: `api/src/modules/trading-v2/__tests__/balance-mirror.repository.integration.test.ts` (DB de teste)

- [ ] **Step 1: Escrever o teste de integração falhando**

```typescript
// api/src/modules/trading-v2/__tests__/balance-mirror.repository.integration.test.ts
import "dotenv/config";
import { prisma } from "../../../shared/database/config/prisma-client";
import { BalanceMirrorRepository } from "../repositories/balance-mirror.repository";

const repo = new BalanceMirrorRepository(prisma);
const U = "11111111-1111-1111-1111-111111111111";
const M = "MKTbalancemirror1111111111111111111111111111";

beforeEach(async () => {
  await prisma.ob2UserBalance.deleteMany({});
  await prisma.ob2UserMarketBalance.deleteMany({});
});
afterAll(async () => { await prisma.$disconnect(); });

test("getConfirmedMicro retorna 0 quando não há linha", async () => {
  expect(await repo.getConfirmedMicro(U, "USDC", null)).toBe(0n);
});

test("foldDelta soma no confirmado (USDC global)", async () => {
  await repo.foldDelta(prisma, U, "USDC", null, 5_000_000n, 100n);
  expect(await repo.getConfirmedMicro(U, "USDC", null)).toBe(5_000_000n);
  await repo.foldDelta(prisma, U, "USDC", null, -2_000_000n, 101n);
  expect(await repo.getConfirmedMicro(U, "USDC", null)).toBe(3_000_000n);
});

test("foldDelta YES/NO usa marketPda", async () => {
  await repo.foldDelta(prisma, U, "YES", M, 10_000_000n, 100n);
  expect(await repo.getConfirmedMicro(U, "YES", M)).toBe(10_000_000n);
});
```

- [ ] **Step 2: Rodar e ver falhar** (só com DB de teste)

Run: `cd api && npx jest src/modules/trading-v2/__tests__/balance-mirror.repository.integration.test.ts`
Expected: FAIL — módulo não existe. (Se não houver DB de teste nesta máquina, pular execução e seguir — marcar para rodar em CI.)

- [ ] **Step 3: Implementar**

```typescript
// api/src/modules/trading-v2/repositories/balance-mirror.repository.ts
import type { PrismaClient } from "../../../generated/prisma/client";
import { toMicro, fromMicro } from "../types/decimal-helpers";

type Asset = "USDC" | "YES" | "NO";

export class BalanceMirrorRepository {
  constructor(private readonly prisma: PrismaClient) {}

  async getConfirmedMicro(userId: string, asset: Asset, marketPda: string | null): Promise<bigint> {
    if (asset === "USDC") {
      const r = await this.prisma.ob2UserBalance.findUnique({
        where: { userId_asset: { userId, asset } },
        select: { onchainTotal: true },
      });
      return r ? toMicro(r.onchainTotal) : 0n;
    }
    const r = await this.prisma.ob2UserMarketBalance.findUnique({
      where: { userId_marketPda_asset: { userId, marketPda: marketPda!, asset } },
      select: { onchainTotal: true },
    });
    return r ? toMicro(r.onchainTotal) : 0n;
  }

  /** Trava a linha-espelho (mutex por user/asset) dentro da tx tx. Garante a linha. */
  async lockMirrorRow(tx: any, userId: string, asset: Asset, marketPda: string | null): Promise<void> {
    if (asset === "USDC") {
      await tx.ob2UserBalance.upsert({
        where: { userId_asset: { userId, asset } },
        create: { userId, asset, free: "0", reserved: "0", onchainTotal: "0" },
        update: {},
      });
      await tx.$executeRawUnsafe(
        `SELECT 1 FROM ob2_user_balances WHERE user_id = $1 AND asset = $2::"Ob2Asset" FOR UPDATE`,
        userId, asset,
      );
    } else {
      await tx.ob2UserMarketBalance.upsert({
        where: { userId_marketPda_asset: { userId, marketPda: marketPda!, asset } },
        create: { userId, marketPda: marketPda!, asset, free: "0", reserved: "0", onchainTotal: "0" },
        update: {},
      });
      await tx.$executeRawUnsafe(
        `SELECT 1 FROM ob2_user_market_balances WHERE user_id = $1 AND market_pda = $2 AND asset = $3::"Ob2Asset" FOR UPDATE`,
        userId, marketPda, asset,
      );
    }
  }

  /** Soma `deltaMicro` (assinado) no onchainTotal. Deve rodar após lockMirrorRow na mesma tx. */
  async foldDelta(tx: any, userId: string, asset: Asset, marketPda: string | null, deltaMicro: bigint, slot: bigint): Promise<void> {
    await this.lockMirrorRow(tx, userId, asset, marketPda);
    const amountStr = fromMicro(deltaMicro);
    if (asset === "USDC") {
      await tx.$executeRawUnsafe(
        `UPDATE ob2_user_balances SET onchain_total = onchain_total + $3::numeric, onchain_slot = $4, updated_at = now()
          WHERE user_id = $1 AND asset = $2::"Ob2Asset"`,
        userId, asset, amountStr, slot.toString(),
      );
    } else {
      await tx.$executeRawUnsafe(
        `UPDATE ob2_user_market_balances SET onchain_total = onchain_total + $4::numeric, onchain_slot = $5, updated_at = now()
          WHERE user_id = $1 AND market_pda = $2 AND asset = $3::"Ob2Asset"`,
        userId, marketPda, asset, amountStr, slot.toString(),
      );
    }
  }

  /** Sobrescreve o espelho com a verdade on-chain (reconciliação). */
  async setConfirmed(tx: any, userId: string, asset: Asset, marketPda: string | null, absoluteMicro: bigint, slot: bigint): Promise<void> {
    await this.lockMirrorRow(tx, userId, asset, marketPda);
    const amountStr = fromMicro(absoluteMicro);
    if (asset === "USDC") {
      await tx.$executeRawUnsafe(
        `UPDATE ob2_user_balances SET onchain_total = $3::numeric, onchain_slot = $4, updated_at = now()
          WHERE user_id = $1 AND asset = $2::"Ob2Asset"`,
        userId, asset, amountStr, slot.toString(),
      );
    } else {
      await tx.$executeRawUnsafe(
        `UPDATE ob2_user_market_balances SET onchain_total = $4::numeric, onchain_slot = $5, updated_at = now()
          WHERE user_id = $1 AND market_pda = $2 AND asset = $3::"Ob2Asset"`,
        userId, marketPda, asset, amountStr, slot.toString(),
      );
    }
  }
}
```

- [ ] **Step 4: Rodar e ver passar** (DB de teste); senão marcar p/ CI.

Run: `cd api && npx jest src/modules/trading-v2/__tests__/balance-mirror.repository.integration.test.ts`
Expected: PASS (3 testes).

- [ ] **Step 5: Commit**

```bash
git add api/src/modules/trading-v2/repositories/balance-mirror.repository.ts api/src/modules/trading-v2/__tests__/balance-mirror.repository.integration.test.ts
git commit -m "feat(trading-v2): BalanceMirrorRepository (espelho confirmado + fold/lock)"
```

### Task 5: `SettlementDeltaRepository` (CRUD + soma de pendentes)

**Files:**
- Create: `api/src/modules/trading-v2/repositories/settlement-delta.repository.ts`
- Test: `api/src/modules/trading-v2/__tests__/settlement-delta.repository.integration.test.ts`

- [ ] **Step 1: Teste de integração falhando**

```typescript
// api/src/modules/trading-v2/__tests__/settlement-delta.repository.integration.test.ts
import "dotenv/config";
import { prisma } from "../../../shared/database/config/prisma-client";
import { SettlementDeltaRepository } from "../repositories/settlement-delta.repository";

const repo = new SettlementDeltaRepository(prisma);
const U = "22222222-2222-2222-2222-222222222222";

beforeEach(async () => { await prisma.ob2SettlementDelta.deleteMany({}); await prisma.ob2Trade.deleteMany({}); });
afterAll(async () => { await prisma.$disconnect(); });

test("sumPendingMicro soma só deltas de trades SETTLING", async () => {
  // helper insere uma trade SETTLING e seus deltas
  const tradeId = await seedSettlingTradeWithDelta(U, "USDC", null, -3_000_000n);
  expect(await repo.sumPendingMicro(U, "USDC", null)).toBe(-3_000_000n);
  // marcar SETTLED → deixa de contar
  await prisma.ob2Trade.update({ where: { id: tradeId }, data: { status: "SETTLED" } });
  expect(await repo.sumPendingMicro(U, "USDC", null)).toBe(0n);
});

async function seedSettlingTradeWithDelta(userId: string, asset: any, marketPda: string | null, amountMicro: bigint) {
  const trade = await prisma.ob2Trade.create({ data: {
    marketPda: "MKT", makerOrderId: "m", takerOrderId: "t", priceBps: 6000, quantity: "10",
    primitive: "TRADE", status: "SETTLING", sync: true, settlingDeadline: new Date(Date.now() + 30000),
  }});
  await repo.create(prisma, trade.id, [{ userId, asset, marketPda, amountMicro }]);
  return trade.id;
}
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `cd api && npx jest src/modules/trading-v2/__tests__/settlement-delta.repository.integration.test.ts`
Expected: FAIL — módulo não existe.

- [ ] **Step 3: Implementar**

```typescript
// api/src/modules/trading-v2/repositories/settlement-delta.repository.ts
import type { PrismaClient } from "../../../generated/prisma/client";
import { toMicro, fromMicro } from "../types/decimal-helpers";
import type { SettlementDelta } from "../services/settlement-delta";

export class SettlementDeltaRepository {
  constructor(private readonly prisma: PrismaClient) {}

  async create(tx: any, tradeId: string, deltas: SettlementDelta[]): Promise<void> {
    if (deltas.length === 0) return;
    await tx.ob2SettlementDelta.createMany({
      data: deltas.map(d => ({
        tradeId, userId: d.userId, asset: d.asset, marketPda: d.marketPda, amount: fromMicro(d.amountMicro),
      })),
    });
  }

  /** Σ deltas de trades ainda SETTLING para (user, asset[, marketPda]). */
  async sumPendingMicro(userId: string, asset: "USDC" | "YES" | "NO", marketPda: string | null): Promise<bigint> {
    const rows = await this.prisma.$queryRawUnsafe<Array<{ total: string }>>(
      `SELECT COALESCE(SUM(d.amount), 0)::text AS total
         FROM ob2_settlement_deltas d
         JOIN ob2_trades t ON t.id = d.trade_id
        WHERE d.user_id = $1 AND d.asset = $2::"Ob2Asset"
          AND ($3::text IS NULL AND d.market_pda IS NULL OR d.market_pda = $3)
          AND t.status = 'SETTLING'::"Ob2TradeStatus"`,
      userId, asset, marketPda,
    );
    return toMicro(rows[0]?.total ?? "0");
  }

  async listForTrade(tradeId: string): Promise<SettlementDelta[]> {
    const rows = await this.prisma.ob2SettlementDelta.findMany({ where: { tradeId } });
    return rows.map(r => ({ userId: r.userId, asset: r.asset as any, marketPda: r.marketPda, amountMicro: toMicro(r.amount) }));
  }
}
```

- [ ] **Step 4: Rodar e ver passar**

Run: `cd api && npx jest src/modules/trading-v2/__tests__/settlement-delta.repository.integration.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add api/src/modules/trading-v2/repositories/settlement-delta.repository.ts api/src/modules/trading-v2/__tests__/settlement-delta.repository.integration.test.ts
git commit -m "feat(trading-v2): SettlementDeltaRepository (create + sumPending + listForTrade)"
```

---

## FASE 4 — Admissão de ordem

### Task 6: `OrderAdmissionService` (available sob trava + insere com compromisso)

**Files:**
- Create: `api/src/modules/trading-v2/services/order-admission.service.ts`
- Test: `api/src/modules/trading-v2/__tests__/order-admission.integration.test.ts`

- [ ] **Step 1: Teste de integração falhando (admite até available; rejeita acima; concorrência não estoura)**

```typescript
// api/src/modules/trading-v2/__tests__/order-admission.integration.test.ts
import "dotenv/config";
import { prisma } from "../../../shared/database/config/prisma-client";
import { OrderAdmissionService } from "../services/order-admission.service";
import { BalanceMirrorRepository } from "../repositories/balance-mirror.repository";
import { SettlementDeltaRepository } from "../repositories/settlement-delta.repository";

const svc = new OrderAdmissionService(prisma, new BalanceMirrorRepository(prisma), new SettlementDeltaRepository(prisma));
const U = "33333333-3333-3333-3333-333333333333";
const M = "MKTadmission111111111111111111111111111111111";

beforeEach(async () => {
  await prisma.ob2Order.deleteMany({});
  await prisma.ob2UserBalance.deleteMany({});
  await prisma.ob2UserMarketBalance.deleteMany({});
  // semeia 10 USDC confirmados
  await prisma.ob2UserBalance.create({ data: { userId: U, asset: "USDC", free: "0", reserved: "0", onchainTotal: "10" } });
});
afterAll(async () => { await prisma.$disconnect(); });

test("admite ordem cujo compromisso cabe no available", async () => {
  const res = await svc.admit({ userId: U, marketPda: M, side: "BUY", priceBps: 5000, feeBps: 0, quantityMicro: 10_000_000n, commitAsset: "USDC" });
  expect(res.admitted).toBe(true);
  const orders = await prisma.ob2Order.findMany({ where: { userId: U } });
  expect(orders).toHaveLength(1);
  expect(orders[0].commitAsset).toBe("USDC");
});

test("rejeita quando compromisso excede available", async () => {
  const res = await svc.admit({ userId: U, marketPda: M, side: "BUY", priceBps: 5000, feeBps: 0, quantityMicro: 30_000_000n, commitAsset: "USDC" });
  expect(res.admitted).toBe(false);
});

test("duas admissões concorrentes não estouram o saldo", async () => {
  // cada uma quer 6 USDC (qty 12 @ 5000bps); só uma cabe em 10
  const mk = () => svc.admit({ userId: U, marketPda: M, side: "BUY", priceBps: 5000, feeBps: 0, quantityMicro: 12_000_000n, commitAsset: "USDC" });
  const [a, b] = await Promise.all([mk(), mk()]);
  expect([a.admitted, b.admitted].filter(Boolean)).toHaveLength(1);
});
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `cd api && npx jest src/modules/trading-v2/__tests__/order-admission.integration.test.ts`
Expected: FAIL — módulo não existe.

- [ ] **Step 3: Implementar**

```typescript
// api/src/modules/trading-v2/services/order-admission.service.ts
import { randomUUID } from "crypto";
import type { PrismaClient, Ob2Asset, Ob2Side } from "../../../generated/prisma/client";
import { BalanceMirrorRepository } from "../repositories/balance-mirror.repository";
import { SettlementDeltaRepository } from "../repositories/settlement-delta.repository";
import { computeAvailable, commitmentFor } from "./available-balance";
import { toMicro, fromMicro } from "../types/decimal-helpers";

export interface AdmitInput {
  userId: string; marketPda: string; side: Ob2Side;
  priceBps: number; feeBps: number; quantityMicro: bigint;
  commitAsset: Ob2Asset; clientOrderId?: string;
}
export interface AdmitResult { admitted: boolean; orderId?: string; reason?: string }

export class OrderAdmissionService {
  constructor(
    private readonly prisma: PrismaClient,
    private readonly mirror: BalanceMirrorRepository,
    private readonly deltas: SettlementDeltaRepository,
  ) {}

  async admit(input: AdmitInput): Promise<AdmitResult> {
    const marketScope = input.commitAsset === "USDC" ? null : input.marketPda;
    const commitMicro = commitmentFor({
      side: input.side, asset: input.commitAsset, priceBps: input.priceBps,
      feeBps: input.feeBps, unfilledMicro: input.quantityMicro,
    });

    return this.prisma.$transaction(async (tx) => {
      // 1. trava o mutex (user, commitAsset[, market])
      await this.mirror.lockMirrorRow(tx, input.userId, input.commitAsset as any, marketScope);

      // 2. available = confirmado + Σpendentes − Σcompromissos abertos
      const confirmed = await this.mirror.getConfirmedMicro(input.userId, input.commitAsset as any, marketScope);
      const pending = await this.deltas.sumPendingMicro(input.userId, input.commitAsset as any, marketScope);
      const openCommit = await this.sumOpenCommitmentsMicro(tx, input.userId, input.commitAsset, marketScope);
      const available = computeAvailable({ confirmedMicro: confirmed, pendingDeltaMicro: pending, openCommitmentMicro: openCommit });

      if (available < commitMicro) {
        return { admitted: false, reason: "insufficient_available" };
      }

      // 3. insere a ordem (a ordem É a trava do compromisso)
      const orderId = randomUUID();
      await tx.ob2Order.create({ data: {
        id: orderId, userId: input.userId, marketPda: input.marketPda, side: input.side,
        priceBps: input.priceBps, feeBps: input.feeBps, quantity: fromMicro(input.quantityMicro),
        filled: "0", status: "OPEN", clientOrderId: input.clientOrderId,
        commitAsset: input.commitAsset, commitAmount: fromMicro(commitMicro),
      }});
      return { admitted: true, orderId };
    });
  }

  private async sumOpenCommitmentsMicro(tx: any, userId: string, asset: Ob2Asset, marketPda: string | null): Promise<bigint> {
    const rows = await tx.$queryRawUnsafe<Array<{ total: string }>>(
      `SELECT COALESCE(SUM(commit_amount), 0)::text AS total
         FROM ob2_orders
        WHERE user_id = $1 AND status = 'OPEN'::"Ob2OrderStatus" AND commit_asset = $2::"Ob2Asset"
          AND ($3::text IS NULL OR market_pda = $3)`,
      userId, asset, marketPda,
    );
    return toMicro(rows[0]?.total ?? "0");
  }
}
```

- [ ] **Step 4: Rodar e ver passar**

Run: `cd api && npx jest src/modules/trading-v2/__tests__/order-admission.integration.test.ts`
Expected: PASS (3 testes, incluindo o de concorrência).

- [ ] **Step 5: Commit**

```bash
git add api/src/modules/trading-v2/services/order-admission.service.ts api/src/modules/trading-v2/__tests__/order-admission.integration.test.ts
git commit -m "feat(trading-v2): OrderAdmissionService (available sob trava, anti gasto-duplo)"
```

### Task 7: Integrar admissão no `place-order.use-case.ts`

**Files:**
- Modify: `api/src/modules/trading-v2/use-cases/place-order.use-case.ts`

- [ ] **Step 1: Ler o use-case atual e mapear pontos de troca**

Run: `sed -n '1,260p' api/src/modules/trading-v2/use-cases/place-order.use-case.ts`
Observe: hoje usa `reservations.reserve(...)`, `classifier.classify(...)`, e tem rede-de-segurança de release. Mantém: validações de mercado, classificação de `commitAsset` via `IntentClassifier`, `maxSpend`. Troca: `reservations.reserve` → `admission.admit`; remove releases (não há reserva).

- [ ] **Step 2: Escrever teste de integração do fluxo novo (admite via available, sem reserva)**

```typescript
// api/src/modules/trading-v2/__tests__/place-order-admission.integration.test.ts
import "dotenv/config";
import { prisma } from "../../../shared/database/config/prisma-client";
// ... montar PlaceOrderUseCase com OrderAdmissionService injetado ...
// Asserções: ordem criada com commitAsset/commitAmount; NENHUMA linha em ob2_reservations.
test("place-order admite sem criar reserva", async () => {
  // arrange: 10 USDC confirmados; act: place BUY que cabe; assert:
  //   - ob2_orders tem 1 OPEN com commit_amount > 0
  //   - ob2_reservations permanece vazia
  expect(true).toBe(true); // substituir pela montagem real seguindo o wiring do módulo (ver index.ts)
});
```

> **Nota ao implementador:** monte o `PlaceOrderUseCase` seguindo o wiring atual em `api/src/modules/trading-v2/index.ts`. O teste real deve: semear `ob2_user_balances`, chamar `execute`, e asserir ordem criada + `ob2_reservations` vazia. Veja `place-order.use-case.ts` para o shape de `PlaceOrderInput`.

- [ ] **Step 3: Rodar e ver falhar**

Run: `cd api && npx jest src/modules/trading-v2/__tests__/place-order-admission.integration.test.ts`
Expected: FAIL (até a integração estar feita).

- [ ] **Step 4: Editar o use-case**

Substituir o bloco de reserva por admissão. Pseudo-diff dos pontos-chave (manter validações e maxSpend existentes):

```typescript
// ANTES (remover):
//   const reservation = await this.reservations.reserve({ userId, marketPda, asset, amount: reserveAmount, orderId });
//   order = await this.orders.create({ ... reservationId: reservation.id ... });
//   ...todas as chamadas this.reservations.release(...) (IOC, safety net, catch)

// DEPOIS:
const { asset: commitAsset } = this.classifier.classify({ /* ...mesma classificação atual... */ });
const admit = await this.admission.admit({
  userId: input.userId, marketPda: input.marketPda, side: input.side,
  priceBps: effectivePriceBps, feeBps: input.feeBps, quantityMicro: effectiveQuantity,
  commitAsset, clientOrderId: input.clientOrderId,
});
if (!admit.admitted) throw new OrderRejectedError("insufficient_available", "saldo disponível insuficiente");
const order = await this.orders.getById(admit.orderId!);
// matching continua igual (engine.tryMatch). IOC: se sobrar OPEN, apenas markCancelled
// (o compromisso some sozinho — NÃO há release).
```

Injetar `OrderAdmissionService` no construtor (e no wiring em `index.ts`). Remover a dependência de `ReservationService` deste use-case.

- [ ] **Step 5: Rodar e ver passar + suite do módulo**

Run: `cd api && npx jest src/modules/trading-v2/__tests__/place-order-admission.integration.test.ts`
Expected: PASS.

- [ ] **Step 6: Commit**

```bash
git add api/src/modules/trading-v2/use-cases/place-order.use-case.ts api/src/modules/trading-v2/index.ts api/src/modules/trading-v2/__tests__/place-order-admission.integration.test.ts
git commit -m "feat(trading-v2): place-order admite via available (sem reservas)"
```

### Task 8: Simplificar `cancel-order.use-case.ts`

**Files:**
- Modify: `api/src/modules/trading-v2/use-cases/cancel-order.use-case.ts`

- [ ] **Step 1: Teste — cancelar marca CANCELLED e NÃO mexe em saldo/reserva**

```typescript
// api/src/modules/trading-v2/__tests__/cancel-order-no-release.integration.test.ts
import "dotenv/config";
import { prisma } from "../../../shared/database/config/prisma-client";
// montar CancelOrderUseCase (sem ReservationService)
test("cancela marca CANCELLED; compromisso some via status; nada em ob2_reservations", async () => {
  // arrange: ordem OPEN com commit_amount; act: cancel; assert: status CANCELLED, ob2_reservations vazia
  expect(true).toBe(true); // substituir pela montagem real
});
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `cd api && npx jest src/modules/trading-v2/__tests__/cancel-order-no-release.integration.test.ts`
Expected: FAIL.

- [ ] **Step 3: Editar — remover release**

```typescript
// cancel-order.use-case.ts: execute()
async execute(input: CancelOrderInput): Promise<OrderView> {
  const order = await this.orders.getById(input.orderId);
  if (!order) throw new OrderNotFoundError(input.orderId);
  if (order.userId !== input.userId) throw new OrderNotFoundError(input.orderId);
  if (order.status !== "OPEN") throw new OrderNotCancellableError(order.status);
  const result = await this.orders.markCancelled(order.id);
  // SEM release: ao sair de OPEN, o compromisso deixa de contar no available automaticamente.
  if (this.publisher) {
    const event = orderToEvent(result);
    await Promise.all([this.publisher.publishMarket(result.marketPda, event), this.publisher.publishUser(result.userId, event)]);
  }
  return result;
}
```

Remover `ReservationService` do construtor e do wiring.

- [ ] **Step 4: Rodar e ver passar**

Run: `cd api && npx jest src/modules/trading-v2/__tests__/cancel-order-no-release.integration.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add api/src/modules/trading-v2/use-cases/cancel-order.use-case.ts api/src/modules/trading-v2/index.ts api/src/modules/trading-v2/__tests__/cancel-order-no-release.integration.test.ts
git commit -m "feat(trading-v2): cancel-order sem release (compromisso some via status)"
```

---

## FASE 5 — Matching edges + worker de settlement

### Task 9: Bordas do matching engine (escreve Trade+Deltas, reduz compromisso)

**Files:**
- Modify: `api/src/modules/trading-v2/services/matching-engine.service.ts`

- [ ] **Step 1: Teste de integração — um fill cria Trade SETTLING + deltas e reduz commit das ordens**

```typescript
// api/src/modules/trading-v2/__tests__/matching-deltas.integration.test.ts
import "dotenv/config";
import { prisma } from "../../../shared/database/config/prisma-client";
// montar MatchingEngine com as novas dependências (SettlementDeltaRepository)
test("fill cria Ob2Trade SETTLING + 4 deltas e reduz commitAmount das ordens", async () => {
  // arrange: 2 ordens cruzando (BUY USDC vs SELL YES) com saldos semeados;
  // act: engine.tryMatch(taker);
  // assert: existe Ob2Trade status SETTLING; ob2_settlement_deltas tem 4 linhas; commit_amount das ordens reduziu pela qty preenchida
  expect(true).toBe(true); // substituir pela montagem real seguindo index.ts
});
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `cd api && npx jest src/modules/trading-v2/__tests__/matching-deltas.integration.test.ts`
Expected: FAIL.

- [ ] **Step 3: Editar o engine — trocar applyDeltas por escrita de deltas**

No ponto onde hoje cria `Ob2Trade` + chama `settler.settle`/`stub.applyDeltas`:
- Manter `decidePrimitive` e o cálculo de `fillQty` (inalterados).
- Dentro da mesma transação do fill:
  1. `applyFillRaw` nas duas ordens (já existe) **e** reduzir `commit_amount` proporcional à qty preenchida:
     ```typescript
     // novo commit = commitmentFor(side, commitAsset, priceBps, feeBps, unfilled = quantity - filled)
     await tx.$executeRawUnsafe(
       `UPDATE ob2_orders SET commit_amount = $2::numeric, updated_at = now() WHERE id = $1`,
       orderId, fromMicro(newCommitMicro),
     );
     ```
  2. Criar `Ob2Trade` com `status: "SETTLING"`.
  3. `const deltas = computeSettlementDeltas({ marketPda, primitive, buyerUserId, sellerUserId, buyerReservationAsset, sellerReservationAsset, quantityMicro: fillQty, priceBps: makerPriceBps, feeBps })`
  4. `await this.deltaRepo.create(tx, trade.id, deltas)`
- **Remover** a chamada a `settler.settle`/`stub.applyDeltas` daqui. Após o commit, enfileirar a trade pro worker (Task 10) — ex.: `this.settlementWorker.enqueue(trade.id)`.
- **Remover** `refundPriceImprovement` e `releaseResidualIfOrderFilled` (não há reserva; a melhoria já está no available via redução de commit).

- [ ] **Step 4: Rodar e ver passar + suite do engine**

Run: `cd api && npx jest src/modules/trading-v2/__tests__/matching-deltas.integration.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add api/src/modules/trading-v2/services/matching-engine.service.ts api/src/modules/trading-v2/__tests__/matching-deltas.integration.test.ts
git commit -m "feat(trading-v2): matching escreve Trade+SettlementDelta e reduz compromisso (sem applyDeltas/refund)"
```

### Task 10: `SettlementWorkerService` (SETTLING→SETTLED|FAILED + recuperação)

**Files:**
- Create: `api/src/modules/trading-v2/services/settlement-worker.service.ts`
- Test: `api/src/modules/trading-v2/__tests__/settlement-worker.integration.test.ts`

- [ ] **Step 1: Teste — confirmação dobra deltas no espelho e marca SETTLED; falha reabre ordens**

```typescript
// api/src/modules/trading-v2/__tests__/settlement-worker.integration.test.ts
import "dotenv/config";
import { prisma } from "../../../shared/database/config/prisma-client";
import { SettlementWorkerService } from "../services/settlement-worker.service";
import { BalanceMirrorRepository } from "../repositories/balance-mirror.repository";
import { SettlementDeltaRepository } from "../repositories/settlement-delta.repository";

// caller fake: ok=true → confirma; ok=false → falha definitiva
function fakeCaller(ok: boolean) {
  return { async sendSettleFill() { return ok ? { ok: true, signature: "sig123" } : { ok: false, reason: "x", retryable: false }; },
           async getSignatureStatuses() { return []; } };
}

beforeEach(async () => {
  await prisma.ob2SettlementDelta.deleteMany({}); await prisma.ob2Trade.deleteMany({});
  await prisma.ob2Order.deleteMany({}); await prisma.ob2UserBalance.deleteMany({});
});
afterAll(async () => { await prisma.$disconnect(); });

test("confirmação: dobra deltas no espelho, marca SETTLED", async () => {
  // arrange: trade SETTLING + delta -3 USDC pro user U; espelho 10 USDC
  // act: worker.process(tradeId) com fakeCaller(true)
  // assert: ob2_trades.status === SETTLED; onchain_total === 7
  expect(true).toBe(true); // substituir pela montagem real
});

test("falha definitiva: marca FAILED e reabre as ordens (OPEN, filled revertido), sem mexer no espelho", async () => {
  expect(true).toBe(true); // substituir pela montagem real
});
```

- [ ] **Step 2: Rodar e ver falhar**

Run: `cd api && npx jest src/modules/trading-v2/__tests__/settlement-worker.integration.test.ts`
Expected: FAIL — módulo não existe.

- [ ] **Step 3: Implementar o worker**

```typescript
// api/src/modules/trading-v2/services/settlement-worker.service.ts
import type { PrismaClient } from "../../../generated/prisma/client";
import type { IOnchainCaller, SettleFillParams } from "../types/onchain-caller.types";
import { BalanceMirrorRepository } from "../repositories/balance-mirror.repository";
import { SettlementDeltaRepository } from "../repositories/settlement-delta.repository";
import { toMicro } from "../types/decimal-helpers";

export class SettlementWorkerService {
  constructor(
    private readonly prisma: PrismaClient,
    private readonly caller: IOnchainCaller,
    private readonly mirror: BalanceMirrorRepository,
    private readonly deltas: SettlementDeltaRepository,
    private readonly buildParams: (tradeId: string) => Promise<SettleFillParams>,
  ) {}

  /** Processa uma trade SETTLING: envia tx atômica, dobra-na-confirmação ou falha-e-reabre. */
  async process(tradeId: string): Promise<void> {
    const trade = await this.prisma.ob2Trade.findUnique({ where: { id: tradeId } });
    if (!trade || trade.status !== "SETTLING") return; // idempotente

    const params = await this.buildParams(tradeId);
    const res = await this.caller.sendSettleFill(params);

    if (res.ok) {
      await this.confirm(tradeId, res.signature);
      return;
    }
    if (!res.retryable) {
      await this.fail(tradeId, res.reason);
    }
    // retryable → deixa SETTLING; a varredura de recuperação re-tenta/consulta status.
  }

  /** Dobra os deltas no espelho e marca SETTLED — reivindicado por UPDATE condicional (idempotente). */
  async confirm(tradeId: string, signature: string): Promise<void> {
    await this.prisma.$transaction(async (tx) => {
      const claimed: number = await tx.$executeRawUnsafe(
        `UPDATE ob2_trades SET status = 'SETTLED'::"Ob2TradeStatus", settled_at = now(), tx_signature = $2
          WHERE id = $1 AND status = 'SETTLING'::"Ob2TradeStatus"`,
        tradeId, signature,
      );
      if (claimed === 0) return; // já processado
      const deltas = await this.deltas.listForTrade(tradeId);
      // ordem determinística pra evitar deadlock
      deltas.sort((a, b) => (a.userId + a.asset).localeCompare(b.userId + b.asset));
      const slot = 0n; // slot real pode vir de getSignatureStatuses; 0 é placeholder seguro p/ fold incremental
      for (const d of deltas) {
        await this.mirror.foldDelta(tx, d.userId, d.asset, d.marketPda, d.amountMicro, slot);
      }
    });
  }

  /** Marca FAILED e reabre as ordens (status OPEN, filled revertido pela qty da trade). NÃO mexe no espelho. */
  async fail(tradeId: string, reason: string): Promise<void> {
    await this.prisma.$transaction(async (tx) => {
      const claimed: number = await tx.$executeRawUnsafe(
        `UPDATE ob2_trades SET status = 'FAILED'::"Ob2TradeStatus", revert_reason = $2
          WHERE id = $1 AND status = 'SETTLING'::"Ob2TradeStatus"`,
        tradeId, reason,
      );
      if (claimed === 0) return;
      const trade = await tx.ob2Trade.findUnique({ where: { id: tradeId } });
      if (!trade) return;
      // reabre maker e taker: filled -= quantity, status volta a OPEN, recomputa commit
      for (const orderId of [trade.makerOrderId, trade.takerOrderId]) {
        await tx.$executeRawUnsafe(
          `UPDATE ob2_orders SET filled = GREATEST(filled - $2::numeric, 0),
                  status = 'OPEN'::"Ob2OrderStatus", closed_at = NULL, updated_at = now()
            WHERE id = $1`,
          orderId, trade.quantity.toString(),
        );
        // recomputa commit_amount pela parte não preenchida (ver Task 9 helper)
      }
    });
  }

  /** Varredura de recuperação: trades SETTLING além do prazo → consulta status → confirma/falha. */
  async recoverStale(): Promise<void> {
    const stale = await this.prisma.ob2Trade.findMany({
      where: { status: "SETTLING", settlingDeadline: { lt: new Date() } },
      select: { id: true, txSignature: true },
    });
    for (const t of stale) {
      if (!t.txSignature) { await this.fail(t.id, "no_signature_past_deadline"); continue; }
      const statuses = await this.caller.getSignatureStatuses([t.txSignature]);
      const s = statuses[0]?.status;
      if (s === "confirmed" || s === "finalized") await this.confirm(t.id, t.txSignature);
      else if (s === null) { /* unknown → NÃO falha; re-tenta na próxima varredura */ }
    }
  }
}
```

> **Nota:** `buildParams(tradeId)` monta o `SettleFillParams` a partir da trade (buyer/seller, primitive, qty, price) — replicar o shape que o `solana-settler` antigo passava ao caller. O `slot` real pode ser lido de `getSignatureStatuses`; para o fold incremental, o valor exato do slot não afeta o saldo (só o campo de auditoria), então 0 é aceitável e a reconciliação corrige `onchain_slot`.

- [ ] **Step 4: Rodar e ver passar**

Run: `cd api && npx jest src/modules/trading-v2/__tests__/settlement-worker.integration.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add api/src/modules/trading-v2/services/settlement-worker.service.ts api/src/modules/trading-v2/__tests__/settlement-worker.integration.test.ts
git commit -m "feat(trading-v2): SettlementWorkerService (confirm dobra deltas; fail reabre ordens; recover)"
```

### Task 11: Conservação ponta-a-ponta (property test)

**Files:**
- Test: `api/src/modules/trading-v2/__tests__/conservation.integration.test.ts`

- [ ] **Step 1: Escrever o teste-mestre de conservação**

```typescript
// api/src/modules/trading-v2/__tests__/conservation.integration.test.ts
import "dotenv/config";
import { prisma } from "../../../shared/database/config/prisma-client";
// Cenário: 2 users, semear USDC; place + match + confirm (caller fake ok). 
// Invariante: Σ (onchain_total USDC de todos) + Σ pendentes − fees == total inicial.
test("match→settle→confirm conserva valor (descontadas fees explícitas)", async () => {
  // arrange total inicial conhecido; act fluxo completo; assert conservação
  expect(true).toBe(true); // substituir pela montagem real reutilizando helpers das tasks 6/9/10
});
```

- [ ] **Step 2: Rodar e ver falhar** (montagem incompleta) → **Step 3: completar a montagem** reutilizando os serviços já construídos → **Step 4: ver passar**.

Run: `cd api && npx jest src/modules/trading-v2/__tests__/conservation.integration.test.ts`
Expected: PASS.

- [ ] **Step 5: Commit**

```bash
git add api/src/modules/trading-v2/__tests__/conservation.integration.test.ts
git commit -m "test(trading-v2): invariante de conservação ponta-a-ponta"
```

---

## FASE 6 — Reconciliação como rede de segurança

### Task 12: Reconciliação sobrescreve o espelho + alerta

**Files:**
- Modify: `api/src/modules/trading-v2/services/daily-reconciliation.service.ts`
- Modify: `api/src/shared/jobs/index.ts`

- [ ] **Step 1: Teste — reconcile sobrescreve onchain_total com a verdade do reader e alerta no drift**

```typescript
// api/src/modules/trading-v2/__tests__/reconcile-overwrite.integration.test.ts
import "dotenv/config";
import { prisma } from "../../../shared/database/config/prisma-client";
// reader fake retorna saldo on-chain != espelho → após reconcile, espelho == reader; alerta chamado
test("reconcile sobrescreve espelho com a verdade on-chain e alerta drift", async () => {
  expect(true).toBe(true); // substituir pela montagem real
});
```

- [ ] **Step 2: Rodar e ver falhar** → **Step 3: editar** `reconcile()` para usar `BalanceMirrorRepository.setConfirmed` (sobrescrever, não comparar contra free/reserved) e, quando `result.drifts.length`/`errors`, chamar `buildReconcileAlert` + `alertService`. → **Step 4: ver passar**.

- [ ] **Step 5: Agendar no cron (30 min)**

Em `api/src/shared/jobs/index.ts`, adicionar (espelhando os outros intervals):

```typescript
import { tradingV2ReconciliationService } from "@/modules/trading-v2/services/daily-reconciliation.service";
// ...
const TRADING_V2_RECONCILE_MS = 30 * 60 * 1000;
tradingV2ReconcileInterval = setInterval(async () => {
  try { await tradingV2ReconciliationService.run(); }
  catch (err: any) { loggers.app.error({ err: err?.message }, "tradingV2Reconcile: erro não capturado"); }
}, TRADING_V2_RECONCILE_MS);
```

(Adicionar `tradingV2ReconcileInterval` à lista de `clearInterval` no shutdown, e ao log de "Cron jobs iniciados".)

- [ ] **Step 6: Commit**

```bash
git add api/src/modules/trading-v2/services/daily-reconciliation.service.ts api/src/shared/jobs/index.ts api/src/modules/trading-v2/__tests__/reconcile-overwrite.integration.test.ts
git commit -m "feat(trading-v2): reconciliação sobrescreve espelho + alerta, agendada 30min"
```

### Task 13: Atualizar `trading-v2-drift-audit.ts` pros invariantes novos

**Files:**
- Modify: `api/scripts/trading-v2-drift-audit.ts`

- [ ] **Step 1: Trocar os checks** — remover I2 (reservas) e free/reserved; adicionar: (1) `onchain_total` vs leitura on-chain (drift real); (2) nenhuma trade `SETTLING` mais velha que X min; (3) `available` (recomputado) negativo pra algum (user,asset). Manter read-only.

- [ ] **Step 2: Rodar contra prod read-only** (após cutover) e conferir saída limpa.

Run: `cd api && bun scripts/trading-v2-drift-audit.ts`
Expected: relatório dos novos invariantes.

- [ ] **Step 3: Commit**

```bash
git add api/scripts/trading-v2-drift-audit.ts
git commit -m "chore(trading-v2): drift-audit usa invariantes do modelo chain-as-truth"
```

---

## FASE 7 — Limpeza + cutover

### Task 14: Remover código morto (reservas, stub-settler, reverter)

**Files:**
- Delete: `api/src/modules/trading-v2/services/reservation.service.ts`, `stub-settler.service.ts`, `settlement-reverter.service.ts` (e `solana-settler.service.ts` se totalmente substituído pelo worker)
- Modify: `api/src/modules/trading-v2/index.ts` (remover wiring), e quaisquer imports remanescentes.

- [ ] **Step 1: Buscar referências remanescentes**

Run: `cd api && grep -rn "ReservationService\|stub-settler\|StubSettler\|settlement-reverter\|SettlementReverter\|reservations\." src/modules/trading-v2 --include='*.ts' | grep -v __tests__`
Expected: lista do que ainda referencia — zerar.

- [ ] **Step 2: Remover arquivos + imports; remover testes obsoletos** dessas classes.

- [ ] **Step 3: Compilar**

Run: `cd api && npx tsc --noEmit 2>&1 | head -30`
Expected: sem erros.

- [ ] **Step 4: Commit**

```bash
git add -A api/src/modules/trading-v2
git commit -m "refactor(trading-v2): remove reservas, stub-settler e reverter (modelo antigo)"
```

### Task 15: Migration de remoção (drop reservas + free/reserved)

**Files:**
- Modify: `api/prisma/schema.prisma`

- [ ] **Step 1: Remover do schema** o modelo `Ob2Reservation`, o campo `Ob2Order.reservationId`, e as colunas `free`/`reserved` de `Ob2UserBalance`/`Ob2UserMarketBalance`. (Opcional: remover `REVERTED` do enum se nenhuma linha usa.)

- [ ] **Step 2: Aplicar no DB (db push, aceita data-loss do drop)**

Run: `cd api && npx prisma db push --accept-data-loss`
Expected: colunas/tabela dropadas e client regenerado. (Só DB local, NUNCA prod.)

- [ ] **Step 3: Compilar + suite unit**

Run: `cd api && npx tsc --noEmit 2>&1 | head -20 && DATABASE_URL='postgresql://invalid:5432/none' BLOCK_EXTERNAL_SERVICES=true npx jest --testPathPatterns="trading-v2/.*unit"`
Expected: compila; unit tests passam.

- [ ] **Step 4: Commit**

```bash
git add api/prisma/schema.prisma api/prisma/migrations
git commit -m "feat(trading-v2): drop Ob2Reservation + free/reserved (cutover)"
```

### Task 16: Cutover em prod (dump + wipe + seed do on-chain)

**Files:**
- Create: `api/scripts/trading-v2-cutover.ts` (orquestra dump→wipe→seed; idempotente; exige flag `--confirm`)

- [ ] **Step 1: Dump das tabelas Ob2 atuais** (segurança — clientes de teste)

Run (manual, prod): `pg_dump "$DATABASE_URL" -t ob2_user_balances -t ob2_user_market_balances -t ob2_reservations -t ob2_orders -t ob2_trades > ob2_predump_2026-06-01.sql`
Expected: arquivo de dump salvo fora do repo.

- [ ] **Step 2: Aplicar o schema novo** em prod (db push)

Run (manual, prod, tráfego parado): `cd api && npx prisma db push --accept-data-loss`
Expected: schema novo aplicado (Ob2SettlementDelta + commit fields; drop de reservas/free/reserved).

- [ ] **Step 3: Seed do espelho a partir do on-chain**

Escrever `trading-v2-cutover.ts` que: zera `ob2_orders`/`ob2_trades`/`ob2_settlement_deltas`, e roda a reconciliação (que sobrescreve `onchain_total` a partir do `SolanaOnchainBalanceReader`). Reusa `DailyReconciliationService` + `BalanceMirrorRepository.setConfirmed`.

Run (manual, prod, com tráfego parado): `cd api && bun scripts/trading-v2-cutover.ts --confirm`
Expected: `onchain_total` populado da chain; ordens/trades zeradas.

- [ ] **Step 4: Verificar com o drift-audit**

Run: `cd api && bun scripts/trading-v2-drift-audit.ts`
Expected: zero drift, zero trade SETTLING, zero available negativo.

- [ ] **Step 5: Commit do script**

```bash
git add api/scripts/trading-v2-cutover.ts
git commit -m "feat(trading-v2): script de cutover (dump+wipe+seed do on-chain)"
```

---

## Verificação final (antes de abrir PR)

- [ ] `cd api && npx tsc --noEmit` sem erros.
- [ ] `DATABASE_URL='postgresql://invalid:5432/none' BLOCK_EXTERNAL_SERVICES=true npx jest --testPathPatterns="trading-v2/.*unit"` — todos os unit passam.
- [ ] Suite de integração trading-v2 verde **em CI / DB de teste** (nunca prod).
- [ ] `grep -rn "reservation\|free\|reserved" src/modules/trading-v2 --include='*.ts' | grep -v __tests__` — sem referências autoritativas remanescentes.
- [ ] Abrir PR de `feat/trading-v2-chain-as-truth` → `main`.
