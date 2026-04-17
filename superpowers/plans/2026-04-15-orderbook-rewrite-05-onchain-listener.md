# Orderbook Rewrite — Plano 5: On-chain listener + idempotência via memo

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fechar a janela de race entre `caller.sendSettleFill` retornando com signature e o `SolanaSettler` conseguir marcar `trade.status=SETTLED`. Se o processo morre nesse intervalo, hoje o deadline worker (P3) reverteria um trade que JÁ SE LIQUIDOU on-chain, criando inconsistência DB↔on-chain. Plano 5 entrega:
1. Memo instruction anexada ao tx composto (carrega `trade_id`).
2. `OnchainEventListener` que scanna `getSignaturesForAddress(PROGRAM_ID)`, extrai memo, marca trade SETTLED.
3. `ReconcileSettlingTradesService` consulta idempotency table antes de reverter.

**Architecture:** `SolanaOnchainCaller` injeta uma `createMemoInstruction(trade_id)` como terceira instruction na composta. `OnchainEventRepository` envolve a tabela `ob2_onchain_events_processed` (criada no Plano 2). `OnchainEventListener` é um serviço stateless: recebe uma "last cursor" (último slot ou signature processado), busca signatures novas do programa, para cada signature extrai memo → trade_id → insere em `ob2_onchain_events_processed` e marca trade SETTLED se ainda estiver SETTLING. `ReconcileSettlingTradesService.scanAndRevert` passa a consultar `OnchainEventRepository.hasEventForTrade(tradeId)` antes de chamar revert — se já processado, apenas marca SETTLED.

**Tech Stack:** Bun, TypeScript, `@solana/web3.js`, `@solana/spl-memo` (ou memo inline via raw instruction), Prisma 7, Jest.

**Spec:** `docs/superpowers/specs/2026-04-15-orderbook-rewrite-design.md` (§7.4 listener, §I7 idempotência)
**Planos anteriores:**
- Plano 1 (`c891f0b`) — fundação
- Plano 2 (`c591534`) — orders + matching (+ tabela `ob2_onchain_events_processed` já criada no schema)
- Plano 3 (`fe88950`) — SolanaSettler + reverter + deadline worker
- Plano 4 (`9674e09`) — SolanaOnchainCaller real via reuso legacy

**Próximos planos:**
- Plano 6: fee ledger real integration (hoje fees on-chain são debitadas pelo Rust mas nossa contabilidade off-chain não reflete isso)
- Plano 7: WebSocket v2 • Plano 8: MM bot externo • Plano 9: Cutover

**Notas do review do Plano 4 endereçadas aqui:**
1. `classifySolanaError` ampliar regex (429, socket hang up, node is behind, 503) — Task 8 opcional.
2. Compute budget 400k CUs — validar em devnet com tx agora levando 3 instructions (memo + 2 legs). Task 1 aumenta pra 500k preventivamente.
3. `userLookup` int-parse estrito — não endereçado neste plano (ortogonal).

**Não-escopo:**
- **Fee ledger real**: Plano 6. Event listener neste plano NÃO extrai fees; só marca SETTLED baseado em presença de evento.
- **Cron wiring do listener no app entry**: integração com scheduler do projeto (cron, node-schedule, BullMQ — TBD) fica pro Plano 9 (cutover) junto da reconciliação diária.
- **Listener em mainnet com RPC resiliente**: Plano 5 entrega o listener + um CLI runnable; produção-robusta (rate limiting, RPC fallback, alerting) é follow-up.

---

## File Structure

```
api/
  prisma/schema.prisma                          # NO CHANGE — ob2_onchain_events_processed já existe
  src/modules/trading-v2/
    types/
      onchain-event.types.ts                    # CREATE: OnchainEventRecord, EventKind
    repositories/
      onchain-event.repository.ts               # CREATE: CRUD sobre ob2_onchain_events_processed
    services/
      solana-onchain-caller.service.ts          # MODIFY: adiciona memo instruction + compute budget bump
      solana-settler.service.ts                 # MODIFY: atomic write event + SETTLED
      reconcile-settling-trades.service.ts      # MODIFY: checa event antes de revert
      onchain-event-listener.service.ts         # CREATE: scan + process
    scripts/
      run-onchain-listener.ts                   # CREATE: CLI pra rodar scan manual
    __tests__/
      onchain-event.repository.integration.test.ts
      onchain-event-listener.unit.test.ts
      solana-onchain-caller.memo.unit.test.ts
      solana-settler-idempotency.integration.test.ts
      reconcile-settling-trades-with-event.integration.test.ts
    index.ts                                    # MODIFY: export new types/services
    README.md                                   # MODIFY: Plano 5 section
```

**Responsabilidades (novos):**

- `onchain-event.types.ts` — `OnchainEventRecord { signature, instructionIndex, kind, tradeId?, processedAt }` e `EventKind = "SETTLE_FILL_BUY" | "SETTLE_FILL_SELL"` (usado pra distinguir qual instruction within a tx gerou o evento).
- `onchain-event.repository.ts` — métodos: `recordProcessed(signature, instructionIndex, kind, tradeId?)` (insert idempotente via `onConflict` no PK composto), `hasEventForTrade(tradeId): boolean`, `hasSignature(sig): boolean`, `listRecent(afterProcessedAt, limit)`.
- `onchain-event-listener.service.ts` — serviço com `scan()` method: busca signatures do `PROGRAM_ID` desde um cursor, para cada signature busca a transaction (`getTransaction`), extrai memo via parse dos logs/instructions, valida que é UUID, verifica `!hasSignature`, insere event + marca trade SETTLED. Retorna `{ scanned, processed, errors }`.
- `run-onchain-listener.ts` — script CLI (similar ao snapshot-onchain-balances do Plano 1) pra rodar `scan()` manualmente ou via cron externo.
- `SolanaSettler` modificado — success path vira tx atômico: `applyDeltas` já rodou; depois tx contendo `insert event + mark SETTLED`.
- `ReconcileSettlingTradesService` modificado — guard "if trade has matching event, mark SETTLED e pula revert".

---

## Prerequisite check

- [ ] **Step 0: Baseline.**

```bash
cd /Users/rafael/Documents/GitHub/Oraculo/api
bun x jest src/modules/trading-v2 --runInBand
```

Expected: `Tests: 103 passed, 1 skipped` (pós-Plano 4).

- [ ] **Step 0.1: Worktree:**

```bash
git worktree add .claude/worktrees/orderbook-rewrite-05-onchain-listener -b worktree-orderbook-rewrite-05-onchain-listener
cp api/.env .claude/worktrees/orderbook-rewrite-05-onchain-listener/api/.env
cd .claude/worktrees/orderbook-rewrite-05-onchain-listener/api
bun install
bun x prisma generate
bun x jest src/modules/trading-v2 --runInBand   # confirmar 103 no worktree
```

---

## Task 1: Memo instruction no SolanaOnchainCaller + compute budget bump

**Files:**
- Modify: `api/src/modules/trading-v2/services/solana-onchain-caller.service.ts`
- Modify: `api/src/modules/trading-v2/__tests__/solana-onchain-caller.unit.test.ts`

**Design:** Adicionar **antes** das legs uma memo instruction com o `tradeId` em texto (UUID ASCII). O memo fica visível nos logs de qualquer tx que chame o programa memo. Listener scanna logs, extrai memo, tem o trade_id diretamente — sem precisar parsear account keys ou argumentos da settle_clob.

Usar `MemoProgram` from `@solana/spl-memo`. Se não estiver instalado, substitui por raw instruction — o programa memo tem ID `MemoSq4gqABAXKb96qnH8TysNcWxMyWCqXgDLGmfcHr` (memo v2). Verificar se `@solana/spl-memo` já é dep do projeto:

```bash
cd /Users/rafael/Documents/GitHub/Oraculo/.claude/worktrees/orderbook-rewrite-05-onchain-listener/api
grep -E '"@solana/spl-memo"' package.json || echo "NOT INSTALLED"
```

Se NOT INSTALLED, usar raw instruction (não adicionar dep nova):

```typescript
import { PublicKey, TransactionInstruction } from "@solana/web3.js";
const MEMO_PROGRAM_ID = new PublicKey("MemoSq4gqABAXKb96qnH8TysNcWxMyWCqXgDLGmfcHr");
function buildMemoInstruction(text: string): TransactionInstruction {
  return new TransactionInstruction({
    keys: [],
    programId: MEMO_PROGRAM_ID,
    data: Buffer.from(text, "utf8"),
  });
}
```

- [ ] **Step 1.1: Ler o arquivo atual.**

```bash
cat src/modules/trading-v2/services/solana-onchain-caller.service.ts | head -100
```

Identificar onde `buildLegs` retorna o array de instructions. A modificação é em `sendSettleFill` — depois de montar o array `instructions` (buy + sell), prepend memo.

- [ ] **Step 1.2: Failing test (append ao `solana-onchain-caller.unit.test.ts` como novo arquivo separado pra melhor organização):**

Create `api/src/modules/trading-v2/__tests__/solana-onchain-caller.memo.unit.test.ts`:

```typescript
import { SolanaOnchainCaller } from "../services/solana-onchain-caller.service";
import type { InstructionBuilder, TransactionSender, UserWalletLookup } from "../services/solana-onchain-caller.service";
import type { SettleFillParams } from "../types/onchain-caller.types";
import type { LegParams, BuiltInstruction } from "../services/legacy-clob-instruction-builder";
import { PublicKey, TransactionInstruction } from "@solana/web3.js";
import { UNIT } from "../types/balance.types";

const MEMO_PROGRAM_ID = new PublicKey("MemoSq4gqABAXKb96qnH8TysNcWxMyWCqXgDLGmfcHr");

const fakeBuiltIx = (marker: string): BuiltInstruction => ({
  setupInstructions: [],
  mainInstruction: new TransactionInstruction({
    keys: [], programId: new PublicKey("11111111111111111111111111111111"), data: Buffer.from(marker),
  }),
  signers: [],
});

const fakeUserRepo: UserWalletLookup = {
  async getWalletAddress(userId: string): Promise<string> {
    return `${userId}Walletxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`.slice(0, 44);
  },
};

const baseParams: SettleFillParams = {
  tradeId: "00000000-0000-0000-0000-0000000000aa",
  marketPda: "Market1111111111111111111111111111111111111",
  buyerUserId: "u-buyer", sellerUserId: "u-seller",
  buyerReservationAsset: "USDC",
  sellerReservationAsset: "YES",
  priceBps: 5000,
  quantityMicro: 100n * UNIT,
  primitive: "TRADE",
};

test("memo instruction is prepended to the tx carrying the tradeId", async () => {
  let capturedInstructions: TransactionInstruction[] = [];
  const builder: InstructionBuilder = {
    async buildBuyLegInstruction()  { return fakeBuiltIx("buy");  },
    async buildSellLegInstruction() { return fakeBuiltIx("sell"); },
  };
  const sender: TransactionSender = {
    async sendTransaction(args) {
      capturedInstructions = args.instructions;
      return { ok: true as const, signature: "sig1" };
    },
  };

  const caller = new SolanaOnchainCaller(builder, sender, fakeUserRepo);
  await caller.sendSettleFill(baseParams);

  expect(capturedInstructions).toHaveLength(3);   // memo + 2 legs
  const memoIx = capturedInstructions[0];
  expect(memoIx.programId.equals(MEMO_PROGRAM_ID)).toBe(true);
  expect(memoIx.data.toString("utf8")).toBe(baseParams.tradeId);
  expect(memoIx.keys).toHaveLength(0);
});

test("memo instruction precedes setup instructions as well (they come between memo and main)", async () => {
  const builderWithSetup: InstructionBuilder = {
    async buildBuyLegInstruction() {
      const setup = new TransactionInstruction({
        keys: [], programId: new PublicKey("11111111111111111111111111111111"), data: Buffer.from("setup"),
      });
      return { setupInstructions: [setup], mainInstruction: fakeBuiltIx("buy").mainInstruction, signers: [] };
    },
    async buildSellLegInstruction() { return fakeBuiltIx("sell"); },
  };
  let capturedInstructions: TransactionInstruction[] = [];
  const sender: TransactionSender = {
    async sendTransaction(args) {
      capturedInstructions = args.instructions;
      return { ok: true as const, signature: "sig2" };
    },
  };

  const caller = new SolanaOnchainCaller(builderWithSetup, sender, fakeUserRepo);
  await caller.sendSettleFill(baseParams);

  // memo + buy-setup + buy-main + sell-main = 4
  expect(capturedInstructions).toHaveLength(4);
  expect(capturedInstructions[0].programId.equals(MEMO_PROGRAM_ID)).toBe(true);
  expect(capturedInstructions[1].data.toString("utf8")).toBe("setup");
  expect(capturedInstructions[2].data.toString("utf8")).toBe("buy");
  expect(capturedInstructions[3].data.toString("utf8")).toBe("sell");
});
```

- [ ] **Step 1.3: Run, fail.**

```bash
bun x jest src/modules/trading-v2/__tests__/solana-onchain-caller.memo.unit.test.ts --runInBand
```

Expected: 2 tests fail (memo not yet added).

- [ ] **Step 1.4: Modify `sendSettleFill` in `solana-onchain-caller.service.ts`.**

Add the helper function + memo import near the top of the file:

```typescript
import { PublicKey } from "@solana/web3.js";

/** SPL Memo program (v2). Used by the on-chain event listener to correlate signatures → tradeIds. */
const MEMO_PROGRAM_ID = new PublicKey("MemoSq4gqABAXKb96qnH8TysNcWxMyWCqXgDLGmfcHr");

function buildMemoInstruction(text: string): TransactionInstruction {
  return new TransactionInstruction({
    keys: [],
    programId: MEMO_PROGRAM_ID,
    data: Buffer.from(text, "utf8"),
  });
}
```

Inside `sendSettleFill`, after accumulating `instructions` from legs but BEFORE `sender.sendTransaction`, prepend the memo:

```typescript
  async sendSettleFill(params: SettleFillParams): Promise<SettleFillResult> {
    try {
      const legs = await this.buildLegs(params);
      const instructions: TransactionInstruction[] = [buildMemoInstruction(params.tradeId)];
      const extraSigners: BuiltInstruction["signers"] = [];
      for (const leg of legs) {
        instructions.push(...leg.setupInstructions, leg.mainInstruction);
        extraSigners.push(...leg.signers);
      }
      return await this.sender.sendTransaction({ instructions, extraSigners });
    } catch (e) {
      const msg = e instanceof Error ? e.message : String(e);
      return { ok: false, reason: `caller_exception: ${msg}`, retryable: false };
    }
  }
```

- [ ] **Step 1.5: Run, 2 memo tests pass.**

Also run the pre-existing unit tests to confirm not broken:

```bash
bun x jest src/modules/trading-v2/__tests__/solana-onchain-caller.unit.test.ts --runInBand
```

**Issue:** pre-existing tests check `expect(builderCalls).toHaveLength(2)` but now the SENDER receives 3 instructions. The builder call count is unchanged (still 2), so these pass. **But** if any pre-existing test inspects `capturedInstructions.length === 2`, it'll now be 3 (memo+buy+sell). Scan the existing tests; if such an assertion exists, update to `>= 2` or `=== 3`.

Expected after fix: all caller tests green.

- [ ] **Step 1.6: Bump compute budget in `RealTransactionSender`.**

Find the `ComputeBudgetProgram.setComputeUnitLimit({ units: 400_000 })` line. Change to:

```typescript
      tx.add(ComputeBudgetProgram.setComputeUnitLimit({ units: 500_000 }));
```

Rationale: memo instruction consumes minimal CUs (~1k), but we now have 3 program invocations in the tx. 400k was already tight; 500k provides safety margin.

- [ ] **Step 1.7: Full trading-v2 suite green.**

```bash
bun x jest src/modules/trading-v2 --runInBand
```

Expected: 105 passed + 1 skipped (103 baseline + 2 memo tests).

- [ ] **Step 1.8: Commit.**

```bash
cd /Users/rafael/Documents/GitHub/Oraculo/.claude/worktrees/orderbook-rewrite-05-onchain-listener
git add api/src/modules/trading-v2/services/solana-onchain-caller.service.ts \
        api/src/modules/trading-v2/__tests__/solana-onchain-caller.memo.unit.test.ts \
        api/src/modules/trading-v2/__tests__/solana-onchain-caller.unit.test.ts
git commit -m "feat(trading-v2): memo instruction com tradeId + compute budget 500k"
```

---

## Task 2: Types + OnchainEventRepository

**Files:**
- Create: `api/src/modules/trading-v2/types/onchain-event.types.ts`
- Create: `api/src/modules/trading-v2/repositories/onchain-event.repository.ts`
- Create: `api/src/modules/trading-v2/__tests__/onchain-event.repository.integration.test.ts`

- [ ] **Step 2.1: Types:**

```typescript
export const EventKind = {
  SettleFill: "SETTLE_FILL",
} as const;

export type EventKind = typeof EventKind[keyof typeof EventKind];

export interface OnchainEventRecord {
  signature: string;
  instructionIndex: number;
  kind: EventKind;
  tradeId: string | null;
  processedAt: Date;
}

export interface RecordEventInput {
  signature: string;
  instructionIndex: number;
  kind: EventKind;
  tradeId?: string;
}
```

- [ ] **Step 2.2: Failing test:**

```typescript
import "dotenv/config";
import { prisma } from "../../../shared/database/config/prisma-client";
import { OnchainEventRepository } from "../repositories/onchain-event.repository";
import { EventKind } from "../types/onchain-event.types";

const repo = new OnchainEventRepository(prisma);

const TRADE = "00000000-0000-0000-0000-000000000aaa";
const SIG = "SigAaa111111111111111111111111111111111111111111111111111111111111111111111111111111";

beforeEach(async () => {
  await prisma.ob2OnchainEvent.deleteMany({});
});
afterAll(async () => { await prisma.$disconnect(); });

test("recordProcessed inserts a new event row", async () => {
  await repo.recordProcessed({ signature: SIG, instructionIndex: 0, kind: EventKind.SettleFill, tradeId: TRADE });
  const row = await prisma.ob2OnchainEvent.findUnique({
    where: { signature_instructionIndex: { signature: SIG, instructionIndex: 0 } },
  });
  expect(row).not.toBeNull();
  expect(row!.tradeId).toBe(TRADE);
  expect(row!.kind).toBe("SETTLE_FILL");
});

test("recordProcessed is idempotent: same (signature, instructionIndex) is no-op", async () => {
  await repo.recordProcessed({ signature: SIG, instructionIndex: 0, kind: EventKind.SettleFill, tradeId: TRADE });
  await repo.recordProcessed({ signature: SIG, instructionIndex: 0, kind: EventKind.SettleFill, tradeId: TRADE });
  const count = await prisma.ob2OnchainEvent.count({ where: { signature: SIG } });
  expect(count).toBe(1);
});

test("hasSignature returns true if at least one event for that signature exists", async () => {
  expect(await repo.hasSignature(SIG)).toBe(false);
  await repo.recordProcessed({ signature: SIG, instructionIndex: 0, kind: EventKind.SettleFill, tradeId: TRADE });
  expect(await repo.hasSignature(SIG)).toBe(true);
});

test("hasEventForTrade returns true when any event references the trade", async () => {
  expect(await repo.hasEventForTrade(TRADE)).toBe(false);
  await repo.recordProcessed({ signature: SIG, instructionIndex: 0, kind: EventKind.SettleFill, tradeId: TRADE });
  expect(await repo.hasEventForTrade(TRADE)).toBe(true);
});

test("different instructionIndex under same signature is allowed (composite PK)", async () => {
  await repo.recordProcessed({ signature: SIG, instructionIndex: 0, kind: EventKind.SettleFill, tradeId: TRADE });
  await repo.recordProcessed({ signature: SIG, instructionIndex: 1, kind: EventKind.SettleFill, tradeId: TRADE });
  const count = await prisma.ob2OnchainEvent.count({ where: { signature: SIG } });
  expect(count).toBe(2);
});

test("listRecent returns events newer than cursor ordered by processedAt asc", async () => {
  await repo.recordProcessed({ signature: SIG + "a", instructionIndex: 0, kind: EventKind.SettleFill, tradeId: TRADE });
  await new Promise(r => setTimeout(r, 5));
  const cursor = new Date();
  await new Promise(r => setTimeout(r, 5));
  await repo.recordProcessed({ signature: SIG + "b", instructionIndex: 0, kind: EventKind.SettleFill, tradeId: TRADE });

  const rows = await repo.listRecent(cursor, 10);
  expect(rows).toHaveLength(1);
  expect(rows[0].signature).toBe(SIG + "b");
});
```

- [ ] **Step 2.3: Run, fail.**

```bash
bun x jest src/modules/trading-v2/__tests__/onchain-event.repository.integration.test.ts --runInBand
```

- [ ] **Step 2.4: Implement `onchain-event.repository.ts`:**

```typescript
import type { PrismaClient } from "../../../generated/prisma/client";
import type { OnchainEventRecord, RecordEventInput } from "../types/onchain-event.types";

export class OnchainEventRepository {
  constructor(private readonly prisma: PrismaClient) {}

  /**
   * Insere o evento; se (signature, instructionIndex) já existe, é no-op.
   * Garante idempotência do listener + do settler.
   */
  async recordProcessed(input: RecordEventInput): Promise<void> {
    try {
      await this.prisma.ob2OnchainEvent.create({
        data: {
          signature: input.signature,
          instructionIndex: input.instructionIndex,
          kind: input.kind,
          tradeId: input.tradeId ?? null,
        },
      });
    } catch (e) {
      // P2002 = Prisma unique constraint. Idempotente.
      if (e instanceof Error && e.message.includes("P2002")) return;
      throw e;
    }
  }

  async hasSignature(signature: string): Promise<boolean> {
    const n = await this.prisma.ob2OnchainEvent.count({ where: { signature } });
    return n > 0;
  }

  async hasEventForTrade(tradeId: string): Promise<boolean> {
    const n = await this.prisma.ob2OnchainEvent.count({ where: { tradeId } });
    return n > 0;
  }

  async listRecent(after: Date, limit: number): Promise<OnchainEventRecord[]> {
    const rows = await this.prisma.ob2OnchainEvent.findMany({
      where: { processedAt: { gt: after } },
      orderBy: { processedAt: "asc" },
      take: limit,
    });
    return rows.map(r => ({
      signature: r.signature,
      instructionIndex: r.instructionIndex,
      kind: r.kind as OnchainEventRecord["kind"],
      tradeId: r.tradeId,
      processedAt: r.processedAt,
    }));
  }
}
```

- [ ] **Step 2.5: Run, 6 tests pass.**

- [ ] **Step 2.6: Commit.**

```bash
git add api/src/modules/trading-v2/types/onchain-event.types.ts \
        api/src/modules/trading-v2/repositories/onchain-event.repository.ts \
        api/src/modules/trading-v2/__tests__/onchain-event.repository.integration.test.ts
git commit -m "feat(trading-v2): OnchainEventRepository + types"
```

---

## Task 3: SolanaSettler registra event atomic com SETTLED mark

**Problem:** hoje SolanaSettler.settle faz applyDeltas (tx #1), sendSettleFill (on-chain), mark SETTLED (tx #2). Se o processo morrer entre sendSettleFill ok e mark SETTLED, on-chain liquidou mas DB fica em SETTLING.

**Fix:** mark SETTLED vira atomic com INSERT em `ob2_onchain_events_processed`. Se o processo morre no meio, next time listener OU deadline worker consulta o event table: se existe, o trade tá SETTLED on-chain, só falta marcar; se não existe, não processou ainda — deadline worker reverte.

**Files:**
- Modify: `api/src/modules/trading-v2/services/solana-settler.service.ts`
- Create: `api/src/modules/trading-v2/__tests__/solana-settler-idempotency.integration.test.ts`

- [ ] **Step 3.1: Failing test:**

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

const BUYER = "00000000-0000-0000-0000-000000000001";
const SELLER = "00000000-0000-0000-0000-000000000002";
const MARKET = "Market1111111111111111111111111111111111111";

async function setupTRADE() {
  await prisma.ob2OnchainEvent.deleteMany({});
  await prisma.ob2Trade.deleteMany({});
  await prisma.ob2Order.deleteMany({});
  await prisma.ob2Reservation.deleteMany({});
  await prisma.ob2UserMarketBalance.deleteMany({});
  await balanceRepo.upsertOnchain(BUYER, MARKET, "USDC", 1000n * UNIT, 1n);
  await balanceRepo.upsertOnchain(SELLER, MARKET, "YES", 100n * UNIT, 1n);
  const bo = await orderRepo.create({ userId: BUYER, marketPda: MARKET, side: "BUY", priceBps: 6000, quantity: 100n * UNIT, reservationId: "p" });
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
  return { tradeId: t.id };
}

afterAll(async () => { await prisma.$disconnect(); });

test("on success: event row inserted atomically with SETTLED mark", async () => {
  const { tradeId } = await setupTRADE();
  const caller = new MockOnchainCaller({ mode: "success" });
  const settler = new SolanaSettlerService(prisma, tradeRepo, stub, reverter, caller, { eventRepo: events });

  await settler.settle(tradeId);

  const trade = await tradeRepo.getById(tradeId);
  expect(trade!.status).toBe("SETTLED");
  expect(trade!.txSignature).toMatch(/^mock:/);
  expect(await events.hasEventForTrade(tradeId)).toBe(true);
  expect(await events.hasSignature(trade!.txSignature!)).toBe(true);
});

test("replay settle on SETTLED trade is no-op (event already recorded)", async () => {
  const { tradeId } = await setupTRADE();
  const caller = new MockOnchainCaller({ mode: "success" });
  const settler = new SolanaSettlerService(prisma, tradeRepo, stub, reverter, caller, { eventRepo: events });

  await settler.settle(tradeId);
  await settler.settle(tradeId);   // 2nd call must no-op (status != SETTLING)

  expect(caller.history).toHaveLength(1);  // single on-chain send
  const eventsCount = await prisma.ob2OnchainEvent.count({ where: { tradeId } });
  expect(eventsCount).toBe(1);
});

test("on non-retryable failure: no event is recorded and trade reverts", async () => {
  const { tradeId } = await setupTRADE();
  const caller = new MockOnchainCaller({ mode: "failure", reason: "program_error", retryable: false });
  const settler = new SolanaSettlerService(prisma, tradeRepo, stub, reverter, caller, { eventRepo: events });

  await settler.settle(tradeId);

  const trade = await tradeRepo.getById(tradeId);
  expect(trade!.status).toBe("REVERTED");
  expect(await events.hasEventForTrade(tradeId)).toBe(false);
});
```

- [ ] **Step 3.2: Run, fail.**

```bash
bun x jest src/modules/trading-v2/__tests__/solana-settler-idempotency.integration.test.ts --runInBand
```

- [ ] **Step 3.3: Modify `SolanaSettlerService`:**

1. Add optional `eventRepo` in the config:

```typescript
import { OnchainEventRepository } from "../repositories/onchain-event.repository";
import { EventKind } from "../types/onchain-event.types";

export interface SolanaSettlerConfig {
  maxRetries?: number;
  retryDelayMs?: number;
  eventRepo?: OnchainEventRepository;   // NEW
}

export class SolanaSettlerService implements ISettler {
  private readonly maxRetries: number;
  private readonly retryDelayMs: number;
  private readonly eventRepo: OnchainEventRepository | null;

  constructor(
    // ... existing params ...
    config: SolanaSettlerConfig = {},
  ) {
    this.maxRetries = config.maxRetries ?? 3;
    this.retryDelayMs = config.retryDelayMs ?? 200;
    this.eventRepo = config.eventRepo ?? null;
  }
  // ...
}
```

2. In the success branch of the retry loop, the existing `$transaction` marks SETTLED. Replace it with:

```typescript
      if (res.ok) {
        await this.prisma.$transaction(async (tx) => {
          const current = await tx.ob2Trade.findUnique({ where: { id: tradeId } });
          if (!current || current.status !== "SETTLING") return;
          await this.stub.applyDeltas(tx, tradeId);

          // Record on-chain event first (idempotent). If this tx crashes AFTER this
          // INSERT but BEFORE the UPDATE, the event is in the DB and the deadline worker
          // (Task 4) will find it and mark SETTLED without reverting.
          if (this.eventRepo) {
            try {
              await tx.ob2OnchainEvent.create({
                data: {
                  signature: res.signature,
                  instructionIndex: 0,
                  kind: EventKind.SettleFill,
                  tradeId,
                },
              });
            } catch (e) {
              // Unique-violation means the listener got to it first — proceed.
              if (!(e instanceof Error && e.message.includes("P2002"))) throw e;
            }
          }

          await tx.ob2Trade.update({
            where: { id: tradeId },
            data: { status: "SETTLED", settledAt: new Date(), txSignature: res.signature },
          });
        });
        return;
      }
```

**Important:** `applyDeltas` was already moved in P3 to run before the on-chain call (DB-first). Keep that. The new addition is just the event INSERT within the same tx as the SETTLED mark.

- [ ] **Step 3.4: Run, 3 tests pass.** Confirm no regression in existing `solana-settler.integration.test.ts`.

```bash
bun x jest src/modules/trading-v2/__tests__/solana-settler --runInBand
```

Expected: 5 pre-existing + 3 new = 8 passed.

- [ ] **Step 3.5: Full suite.**

```bash
bun x jest src/modules/trading-v2 --runInBand
```

Expected: 105 + 6 + 3 = 114 passed + 1 skipped.

- [ ] **Step 3.6: Commit.**

```bash
git add api/src/modules/trading-v2/services/solana-settler.service.ts \
        api/src/modules/trading-v2/__tests__/solana-settler-idempotency.integration.test.ts
git commit -m "feat(trading-v2): SolanaSettler registra event atomic com SETTLED"
```

---

## Task 4: ReconcileSettlingTradesService consulta event antes de revert

**Files:**
- Modify: `api/src/modules/trading-v2/services/reconcile-settling-trades.service.ts`
- Create: `api/src/modules/trading-v2/__tests__/reconcile-settling-trades-with-event.integration.test.ts`

**Design:** quando o worker scan um trade expirado (SETTLING > deadline), consultar `OnchainEventRepository.hasEventForTrade(tradeId)`:
- Se YES: on-chain liquidou; mark SETTLED (via settler path) sem revert.
- Se NO: revert como antes.

- [ ] **Step 4.1: Failing test:**

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
import { OnchainEventRepository } from "../repositories/onchain-event.repository";
import { EventKind } from "../types/onchain-event.types";
import { UNIT } from "../types/balance.types";

const balanceRepo = new BalanceRepository(prisma);
const resSvc = new ReservationService(prisma);
const orderRepo = new OrderRepository(prisma);
const tradeRepo = new TradeRepository(prisma);
const stub = new StubSettlerService(prisma);
const reverter = new SettlementReverter(prisma, tradeRepo, orderRepo);
const events = new OnchainEventRepository(prisma);
const worker = new ReconcileSettlingTradesService(tradeRepo, reverter, { eventRepo: events, stubSettler: stub, prisma });

const BUYER = "00000000-0000-0000-0000-000000000001";
const SELLER = "00000000-0000-0000-0000-000000000002";
const MARKET = "Market1111111111111111111111111111111111111";

async function makeExpiredSettlingTrade(): Promise<string> {
  await prisma.ob2OnchainEvent.deleteMany({});
  await prisma.ob2Trade.deleteMany({});
  await prisma.ob2Order.deleteMany({});
  await prisma.ob2Reservation.deleteMany({});
  await prisma.ob2UserMarketBalance.deleteMany({});
  await balanceRepo.upsertOnchain(BUYER, MARKET, "USDC", 1000n * UNIT, 1n);
  await balanceRepo.upsertOnchain(SELLER, MARKET, "YES", 100n * UNIT, 1n);
  const bo = await orderRepo.create({ userId: BUYER, marketPda: MARKET, side: "BUY", priceBps: 6000, quantity: 100n * UNIT, reservationId: "p" });
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
  await prisma.ob2Trade.update({
    where: { id: t.id },
    data: { settlingDeadline: new Date(Date.now() - 10_000) },
  });
  return t.id;
}

afterAll(async () => { await prisma.$disconnect(); });

test("expired trade WITH event → marked SETTLED, not REVERTED", async () => {
  const tradeId = await makeExpiredSettlingTrade();
  // Simulate the crash-recovery scenario: on-chain event was recorded by listener
  // but trade is still SETTLING because the settler crashed before marking it.
  await events.recordProcessed({
    signature: "sig-from-listener", instructionIndex: 0,
    kind: EventKind.SettleFill, tradeId,
  });
  // Also need deltas applied (in real crash the deltas ARE applied — DB-first flow).
  await stub.applyDeltas.call({ prisma } as any, prisma as any, tradeId);  // optional — see note below
  // NOTE: if applyDeltas requires tx context, the worker code must call it appropriately.

  await worker.scanAndRevert();

  const t = await tradeRepo.getById(tradeId);
  expect(t!.status).toBe("SETTLED");
  expect(t!.txSignature).toBe("sig-from-listener");
  expect(t!.revertReason).toBeNull();
});

test("expired trade WITHOUT event → REVERTED as before", async () => {
  const tradeId = await makeExpiredSettlingTrade();
  const result = await worker.scanAndRevert();
  expect(result.revertedCount).toBe(1);
  const t = await tradeRepo.getById(tradeId);
  expect(t!.status).toBe("REVERTED");
});
```

**Note on `applyDeltas` in the "with event" test:** the real flow is DB-first (deltas applied BEFORE on-chain call). So when the listener records the event, the deltas are already applied to DB. The test's `stub.applyDeltas.call(...)` simulates that prior state. If the signature of `applyDeltas` doesn't accept being called outside `$transaction`, use `await prisma.$transaction(async (tx) => stub.applyDeltas(tx, tradeId))` in the test setup instead.

- [ ] **Step 4.2: Run, fail.**

- [ ] **Step 4.3: Modify `reconcile-settling-trades.service.ts`:**

```typescript
import type { PrismaClient } from "../../../generated/prisma/client";
import type { TradeRepository } from "../repositories/trade.repository";
import type { SettlementReverter } from "./settlement-reverter.service";
import type { OnchainEventRepository } from "../repositories/onchain-event.repository";
import type { StubSettlerService } from "./stub-settler.service";
import { RevertReason } from "../types/revert.types";

export interface ScanResult {
  scannedCount: number;
  revertedCount: number;
  settledFromEventCount: number;
  errors: Array<{ tradeId: string; err: string }>;
}

export interface ReconcileConfig {
  eventRepo?: OnchainEventRepository;
  stubSettler?: StubSettlerService;   // pra aplicar deltas quando recuperando via evento (noop se já aplicados)
  prisma?: PrismaClient;              // pra abrir tx quando settling-from-event
}

export class ReconcileSettlingTradesService {
  private readonly eventRepo: OnchainEventRepository | null;
  private readonly stubSettler: StubSettlerService | null;
  private readonly prisma: PrismaClient | null;

  constructor(
    private readonly trades: TradeRepository,
    private readonly reverter: SettlementReverter,
    config: ReconcileConfig = {},
  ) {
    this.eventRepo = config.eventRepo ?? null;
    this.stubSettler = config.stubSettler ?? null;
    this.prisma = config.prisma ?? null;
  }

  async scanAndRevert(): Promise<ScanResult> {
    const expired = await this.trades.listExpiredSettling();
    const errors: ScanResult["errors"] = [];
    let revertedCount = 0;
    let settledFromEventCount = 0;

    for (const trade of expired) {
      try {
        // Guard: if on-chain event exists, the settlement happened — mark SETTLED instead of reverting.
        if (this.eventRepo && await this.eventRepo.hasEventForTrade(trade.id)) {
          await this.settleFromEvent(trade.id);
          settledFromEventCount++;
          continue;
        }
        const r = await this.reverter.revert(trade.id, RevertReason.DeadlineExceeded);
        if (r) revertedCount++;
      } catch (e) {
        errors.push({ tradeId: trade.id, err: e instanceof Error ? e.message : String(e) });
      }
    }

    return { scannedCount: expired.length, revertedCount, settledFromEventCount, errors };
  }

  /**
   * Trade já liquidou on-chain (evento existe) mas ficou preso em SETTLING no DB.
   * Busca a signature do evento, marca SETTLED. NOTE: deltas são aplicados
   * anteriormente pelo fluxo DB-first do SolanaSettler; aqui não re-aplicamos.
   */
  private async settleFromEvent(tradeId: string): Promise<void> {
    if (!this.eventRepo || !this.prisma) {
      throw new Error("settleFromEvent requires eventRepo and prisma injection");
    }
    const events = await this.prisma.ob2OnchainEvent.findMany({
      where: { tradeId },
      orderBy: { processedAt: "asc" },
      take: 1,
    });
    if (events.length === 0) return;

    const sig = events[0].signature;
    await this.prisma.ob2Trade.update({
      where: { id: tradeId },
      data: { status: "SETTLED", settledAt: new Date(), txSignature: sig },
    });
  }
}
```

- [ ] **Step 4.4: Run, 2 new tests pass.**

- [ ] **Step 4.5: Existing reconcile tests.** Open `reconcile-settling-trades.integration.test.ts` — some may need the new `ReconcileConfig` injection. Pre-existing test that doesn't configure eventRepo still works (falls through to revert path) because the guard checks `this.eventRepo && ...`.

```bash
bun x jest src/modules/trading-v2/__tests__/reconcile-settling-trades --runInBand
```

Expected: 4 pre-existing + 2 new = 6 passed.

- [ ] **Step 4.6: Full suite.**

```bash
bun x jest src/modules/trading-v2 --runInBand
```

Expected: 114 + 2 = 116 passed + 1 skipped.

- [ ] **Step 4.7: Commit.**

```bash
git add api/src/modules/trading-v2/services/reconcile-settling-trades.service.ts \
        api/src/modules/trading-v2/__tests__/reconcile-settling-trades-with-event.integration.test.ts
git commit -m "feat(trading-v2): deadline worker consulta event antes de revert"
```

---

## Task 5: OnchainEventListener — serviço de scan + process

**Files:**
- Create: `api/src/modules/trading-v2/services/onchain-event-listener.service.ts`
- Create: `api/src/modules/trading-v2/__tests__/onchain-event-listener.unit.test.ts`

**Design:** listener injetado com:
- `Connection` (Solana) — abstrato via interface `ISolanaConnection` pra testabilidade.
- `programId`
- `OnchainEventRepository`
- `TradeRepository` (pra marcar SETTLED)
- `StubSettlerService` + `PrismaClient` (pra aplicar deltas se evento encontrado antes do settler marcar)

Métodos principais:
- `scan({ untilSignature?: string, limit: number }): Promise<ScanResult>` — busca signatures recentes do program (`getSignaturesForAddress`), para cada, checa se já processado via `eventRepo.hasSignature`, se não: fetch tx → extract memo → validate trade_id UUID → match com SETTLING trade no DB → record event + mark SETTLED.
- Retorna `{ scanned, processed, skipped, errors }`.

- [ ] **Step 5.1: Failing test.** Usar mocks pra evitar chamar Solana real.

```typescript
import "dotenv/config";
import { prisma } from "../../../shared/database/config/prisma-client";
import { OnchainEventListener, type ISolanaConnection } from "../services/onchain-event-listener.service";
import { OnchainEventRepository } from "../repositories/onchain-event.repository";
import { TradeRepository } from "../repositories/trade.repository";
import { OrderRepository } from "../repositories/order.repository";
import { BalanceRepository } from "../repositories/balance.repository";
import { ReservationService } from "../services/reservation.service";
import { StubSettlerService } from "../services/stub-settler.service";
import { PublicKey } from "@solana/web3.js";
import { UNIT } from "../types/balance.types";

const PROGRAM_ID = new PublicKey("AHiRBEXouJnoVsvQ37KEjV6AP62r6Yi9YkkbNBtsqBaW");

const eventRepo = new OnchainEventRepository(prisma);
const tradeRepo = new TradeRepository(prisma);
const orderRepo = new OrderRepository(prisma);
const balanceRepo = new BalanceRepository(prisma);
const resSvc = new ReservationService(prisma);
const stub = new StubSettlerService(prisma);

const BUYER = "00000000-0000-0000-0000-000000000001";
const SELLER = "00000000-0000-0000-0000-000000000002";
const MARKET = "Market1111111111111111111111111111111111111";

async function setupSettlingTrade(): Promise<string> {
  await prisma.ob2OnchainEvent.deleteMany({});
  await prisma.ob2Trade.deleteMany({});
  await prisma.ob2Order.deleteMany({});
  await prisma.ob2Reservation.deleteMany({});
  await prisma.ob2UserMarketBalance.deleteMany({});
  await balanceRepo.upsertOnchain(BUYER, MARKET, "USDC", 1000n * UNIT, 1n);
  await balanceRepo.upsertOnchain(SELLER, MARKET, "YES", 100n * UNIT, 1n);
  const bo = await orderRepo.create({ userId: BUYER, marketPda: MARKET, side: "BUY", priceBps: 6000, quantity: 100n * UNIT, reservationId: "p" });
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
  return t.id;
}

afterAll(async () => { await prisma.$disconnect(); });

test("scan: sig with valid memo → record event + mark trade SETTLED", async () => {
  const tradeId = await setupSettlingTrade();
  // NOTE: in real DB-first flow, stub.applyDeltas was already applied before the hypothetical crash.
  await prisma.$transaction(async (tx) => { await stub.applyDeltas(tx, tradeId); });

  const fakeConnection: ISolanaConnection = {
    async getSignaturesForAddress() {
      return [{ signature: "sig-aaa", slot: 100, blockTime: null, err: null }];
    },
    async getTransaction() {
      return {
        meta: { logMessages: [`Program log: Memo (len ${tradeId.length}): "${tradeId}"`], err: null },
        transaction: { message: { instructions: [] }, signatures: ["sig-aaa"] },
      };
    },
  };

  const listener = new OnchainEventListener({
    connection: fakeConnection, programId: PROGRAM_ID,
    eventRepo, tradeRepo, prisma,
  });

  const result = await listener.scan({ limit: 10 });
  expect(result.processed).toBe(1);

  const trade = await tradeRepo.getById(tradeId);
  expect(trade!.status).toBe("SETTLED");
  expect(trade!.txSignature).toBe("sig-aaa");
  expect(await eventRepo.hasSignature("sig-aaa")).toBe(true);
});

test("scan: sig already processed (hasSignature=true) → skip", async () => {
  const tradeId = await setupSettlingTrade();
  await eventRepo.recordProcessed({ signature: "sig-dup", instructionIndex: 0, kind: "SETTLE_FILL" as const, tradeId });

  const fakeConnection: ISolanaConnection = {
    async getSignaturesForAddress() {
      return [{ signature: "sig-dup", slot: 100, blockTime: null, err: null }];
    },
    async getTransaction() {
      throw new Error("should not be called — already processed");
    },
  };

  const listener = new OnchainEventListener({
    connection: fakeConnection, programId: PROGRAM_ID,
    eventRepo, tradeRepo, prisma,
  });

  const result = await listener.scan({ limit: 10 });
  expect(result.skipped).toBe(1);
  expect(result.processed).toBe(0);
});

test("scan: tx with no valid memo → skip silently (foreign tx to our program)", async () => {
  const fakeConnection: ISolanaConnection = {
    async getSignaturesForAddress() {
      return [{ signature: "sig-foreign", slot: 100, blockTime: null, err: null }];
    },
    async getTransaction() {
      return {
        meta: { logMessages: ["Program log: some other operation"], err: null },
        transaction: { message: { instructions: [] }, signatures: ["sig-foreign"] },
      };
    },
  };

  const listener = new OnchainEventListener({
    connection: fakeConnection, programId: PROGRAM_ID,
    eventRepo, tradeRepo, prisma,
  });

  const result = await listener.scan({ limit: 10 });
  expect(result.skipped).toBe(1);
  expect(result.processed).toBe(0);
});

test("scan: tx with memo but trade not found (already REVERTED or wrong tradeId) → record event, don't mark", async () => {
  const tradeId = "ff000000-0000-0000-0000-00000000ffff";  // não existe no DB
  const fakeConnection: ISolanaConnection = {
    async getSignaturesForAddress() {
      return [{ signature: "sig-orphan", slot: 100, blockTime: null, err: null }];
    },
    async getTransaction() {
      return {
        meta: { logMessages: [`Program log: Memo (len ${tradeId.length}): "${tradeId}"`], err: null },
        transaction: { message: { instructions: [] }, signatures: ["sig-orphan"] },
      };
    },
  };

  const listener = new OnchainEventListener({
    connection: fakeConnection, programId: PROGRAM_ID,
    eventRepo, tradeRepo, prisma,
  });

  const result = await listener.scan({ limit: 10 });
  expect(result.processed).toBe(0);
  expect(result.orphaned).toBe(1);   // evento ficou registrado mas não tinha trade
  expect(await eventRepo.hasSignature("sig-orphan")).toBe(true);
});
```

- [ ] **Step 5.2: Run, fail.**

- [ ] **Step 5.3: Implement `onchain-event-listener.service.ts`:**

```typescript
import { PublicKey } from "@solana/web3.js";
import type { PrismaClient } from "../../../generated/prisma/client";
import type { OnchainEventRepository } from "../repositories/onchain-event.repository";
import type { TradeRepository } from "../repositories/trade.repository";
import { EventKind } from "../types/onchain-event.types";

/**
 * Minimal shape of Solana Connection methods we need. Abstracted for testability.
 * Production: inject a real `@solana/web3.js` Connection.
 */
export interface ISolanaConnection {
  getSignaturesForAddress(
    address: PublicKey,
    options?: { limit?: number; before?: string; until?: string },
  ): Promise<Array<{ signature: string; slot: number; blockTime: number | null; err: any }>>;
  getTransaction(signature: string, options?: any): Promise<any | null>;
}

export interface OnchainEventListenerConfig {
  connection: ISolanaConnection;
  programId: PublicKey;
  eventRepo: OnchainEventRepository;
  tradeRepo: TradeRepository;
  prisma: PrismaClient;
}

export interface ListenerScanOptions {
  /** Max signatures to fetch per scan. Default 100. */
  limit?: number;
  /** Don't fetch signatures older than this one. */
  before?: string;
  /** Stop when we hit this signature (inclusive). */
  until?: string;
}

export interface ListenerScanResult {
  scanned: number;
  processed: number;       // marked SETTLED
  skipped: number;         // already processed OR no memo match
  orphaned: number;        // memo had a tradeId but DB has no matching SETTLING trade
  errors: Array<{ signature: string; err: string }>;
}

// Matches "Program log: Memo (len N): "<text>"" — standard SPL Memo log format.
const MEMO_LOG_PATTERN = /Memo \(len \d+\): "([^"]+)"/;
const UUID_PATTERN = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;

export class OnchainEventListener {
  constructor(private readonly cfg: OnchainEventListenerConfig) {}

  async scan(options: ListenerScanOptions = {}): Promise<ListenerScanResult> {
    const limit = options.limit ?? 100;
    const result: ListenerScanResult = { scanned: 0, processed: 0, skipped: 0, orphaned: 0, errors: [] };

    const sigs = await this.cfg.connection.getSignaturesForAddress(this.cfg.programId, {
      limit, before: options.before, until: options.until,
    });
    result.scanned = sigs.length;

    for (const { signature } of sigs) {
      try {
        if (await this.cfg.eventRepo.hasSignature(signature)) {
          result.skipped++;
          continue;
        }
        const tx = await this.cfg.connection.getTransaction(signature, { commitment: "confirmed", maxSupportedTransactionVersion: 0 });
        if (!tx) { result.skipped++; continue; }

        const tradeId = extractTradeIdFromTx(tx);
        if (!tradeId) { result.skipped++; continue; }

        // Record event (idempotent).
        await this.cfg.eventRepo.recordProcessed({
          signature, instructionIndex: 0, kind: EventKind.SettleFill, tradeId,
        });

        // Try to mark SETTLED if the trade is SETTLING in our DB.
        const trade = await this.cfg.tradeRepo.getById(tradeId);
        if (!trade) {
          result.orphaned++;
          continue;
        }
        if (trade.status === "SETTLING") {
          await this.cfg.prisma.ob2Trade.update({
            where: { id: tradeId },
            data: { status: "SETTLED", settledAt: new Date(), txSignature: signature },
          });
          result.processed++;
        } else {
          // Trade já SETTLED ou REVERTED — evento recorded mas não muda status.
          result.skipped++;
        }
      } catch (e) {
        result.errors.push({ signature, err: e instanceof Error ? e.message : String(e) });
      }
    }

    return result;
  }
}

/**
 * Extracts a UUID tradeId from the SPL Memo log in a parsed transaction response.
 * Returns null if no valid memo found.
 */
function extractTradeIdFromTx(tx: any): string | null {
  const logs: string[] = tx?.meta?.logMessages ?? [];
  for (const log of logs) {
    const match = log.match(MEMO_LOG_PATTERN);
    if (!match) continue;
    const text = match[1];
    if (UUID_PATTERN.test(text)) return text.toLowerCase();
  }
  return null;
}
```

- [ ] **Step 5.4: Run, 4 tests pass.**

- [ ] **Step 5.5: Commit.**

```bash
git add api/src/modules/trading-v2/services/onchain-event-listener.service.ts \
        api/src/modules/trading-v2/__tests__/onchain-event-listener.unit.test.ts
git commit -m "feat(trading-v2): OnchainEventListener (scan + process via memo)"
```

---

## Task 6: CLI script `run-onchain-listener.ts`

**Files:**
- Create: `api/src/modules/trading-v2/scripts/run-onchain-listener.ts`
- Modify: `api/package.json` (add script alias)

- [ ] **Step 6.1: Script:**

```typescript
/**
 * Runs one pass of the OnchainEventListener.scan against mainnet/devnet.
 *
 * Usage:
 *   bun run src/modules/trading-v2/scripts/run-onchain-listener.ts [--limit=100] [--loop-seconds=N]
 *
 * Without --loop-seconds: runs one scan and exits.
 * With --loop-seconds N: loops forever with N-second delay between scans.
 */
import "dotenv/config";
import { Connection, PublicKey } from "@solana/web3.js";
import { prisma } from "../../../shared/database/config/prisma-client";
import { OnchainEventListener, type ISolanaConnection } from "../services/onchain-event-listener.service";
import { OnchainEventRepository } from "../repositories/onchain-event.repository";
import { TradeRepository } from "../repositories/trade.repository";

function parseArgs(): { limit: number; loopSeconds: number | null } {
  const args = process.argv.slice(2);
  let limit = 100;
  let loopSeconds: number | null = null;
  for (const arg of args) {
    const [k, v] = arg.replace(/^--/, "").split("=");
    if (k === "limit") limit = parseInt(v, 10);
    if (k === "loop-seconds") loopSeconds = parseInt(v, 10);
  }
  return { limit, loopSeconds };
}

async function main() {
  const { limit, loopSeconds } = parseArgs();

  const rpcUrl = process.env.SOLANA_RPC_URL;
  const programIdStr = process.env.PREDICTION_MARKET_PROGRAM_ID;
  if (!rpcUrl || !programIdStr) {
    throw new Error("missing env: SOLANA_RPC_URL and PREDICTION_MARKET_PROGRAM_ID required");
  }

  const connection = new Connection(rpcUrl, "confirmed") as unknown as ISolanaConnection;
  const programId = new PublicKey(programIdStr);
  const eventRepo = new OnchainEventRepository(prisma);
  const tradeRepo = new TradeRepository(prisma);

  const listener = new OnchainEventListener({ connection, programId, eventRepo, tradeRepo, prisma });

  const runOnce = async () => {
    const start = Date.now();
    const result = await listener.scan({ limit });
    const elapsedMs = Date.now() - start;
    // eslint-disable-next-line no-console
    console.log(JSON.stringify({
      timestamp: new Date().toISOString(),
      elapsedMs,
      ...result,
    }));
  };

  if (loopSeconds === null) {
    await runOnce();
    await prisma.$disconnect();
    return;
  }

  // Loop mode
  while (true) {
    try { await runOnce(); }
    catch (e) { console.error("[listener] scan error:", e); }
    await new Promise(r => setTimeout(r, loopSeconds * 1000));
  }
}

main().catch(e => {
  console.error(e);
  process.exit(1);
});
```

- [ ] **Step 6.2: Add script alias to `api/package.json`.**

Find the `scripts` section. After the existing `script:ob2-snapshot` line add:

```json
    "script:ob2-listener": "bun run src/modules/trading-v2/scripts/run-onchain-listener.ts",
```

- [ ] **Step 6.3: Validate JSON.**

```bash
cd /Users/rafael/Documents/GitHub/Oraculo/.claude/worktrees/orderbook-rewrite-05-onchain-listener/api
cat package.json | python3 -c "import json,sys;json.load(sys.stdin);print('valid')"
```

- [ ] **Step 6.4: Commit.**

```bash
git add api/src/modules/trading-v2/scripts/run-onchain-listener.ts api/package.json
git commit -m "feat(trading-v2): CLI run-onchain-listener (one-shot ou loop mode)"
```

---

## Task 7: Barrel + README + final suite

**Files:**
- Modify: `api/src/modules/trading-v2/index.ts`
- Modify: `api/src/modules/trading-v2/README.md`

- [ ] **Step 7.1: Append to `index.ts`:**

```typescript
export * from "./types/onchain-event.types";
export { OnchainEventRepository } from "./repositories/onchain-event.repository";
export { OnchainEventListener } from "./services/onchain-event-listener.service";
export type { ISolanaConnection, OnchainEventListenerConfig, ListenerScanOptions, ListenerScanResult } from "./services/onchain-event-listener.service";
```

- [ ] **Step 7.2: Update `README.md`.** Adicionar seção "On-chain listener (Plano 5)" depois de "Solana caller":

```markdown
## On-chain listener (Plano 5)

Hoje existe uma janela de race: `SolanaSettler.settle` chama `caller.sendSettleFill`, recebe um signature com sucesso, e depois marca `trade.status=SETTLED` numa segunda transação DB. Se o processo morrer entre essas duas etapas, on-chain tem os tokens movidos mas DB fica em `SETTLING`. O `ReconcileSettlingTradesService` então revertaria — criando inconsistência.

**Mitigação (Plano 5):**
1. `SolanaOnchainCaller` prepende uma memo instruction ao tx composto com o `trade_id` (UUID).
2. `SolanaSettler` agora registra o event em `ob2_onchain_events_processed` atomically com `SETTLED` mark.
3. `OnchainEventListener` escanea `getSignaturesForAddress(PROGRAM_ID)` periodicamente, extrai memo dos logs, marca trades SETTLING como SETTLED quando encontra o evento on-chain.
4. `ReconcileSettlingTradesService` consulta a tabela de events antes de reverter — se existe evento pro trade, marca SETTLED em vez de reverter.

### Rodar o listener

```bash
# Scan one-shot:
SOLANA_RPC_URL=... PREDICTION_MARKET_PROGRAM_ID=AHiRBEX... \
  bun run src/modules/trading-v2/scripts/run-onchain-listener.ts --limit=100

# Loop mode (cada N segundos):
bun run src/modules/trading-v2/scripts/run-onchain-listener.ts --loop-seconds=30
```

Em produção, ligar o loop mode como serviço separado (systemd, Docker container, etc.).
Integração com cron do app fica pro Plano 9 (cutover).
```

- [ ] **Step 7.3: Rodar suíte completa.**

```bash
cd /Users/rafael/Documents/GitHub/Oraculo/.claude/worktrees/orderbook-rewrite-05-onchain-listener/api
bun x jest src/modules/trading-v2 --runInBand
```

Expected: 103 (baseline P4) + 2 (Task 1 memo) + 6 (Task 2 repo) + 3 (Task 3 idempotency) + 2 (Task 4 event-guard) + 4 (Task 5 listener) = **120 passed + 1 skipped**.

- [ ] **Step 7.4: tsc clean.**

```bash
bun x tsc --noEmit 2>&1 | grep trading-v2 || echo "clean"
```

- [ ] **Step 7.5: Commit.**

```bash
git add api/src/modules/trading-v2/index.ts api/src/modules/trading-v2/README.md
git commit -m "docs(trading-v2): README pós-plano 5 + barrel exports"
```

---

## Critérios de aceitação

1. ✅ Memo instruction com trade_id prepended ao tx composto.
2. ✅ `OnchainEventRepository` + `ob2_onchain_events_processed` integrado com (a) `SolanaSettler` no sucesso e (b) `OnchainEventListener`.
3. ✅ `OnchainEventListener.scan` extrai memo dos logs, marca trade SETTLED se em SETTLING.
4. ✅ `ReconcileSettlingTradesService` consulta event antes de reverter — fecha janela de race.
5. ✅ CLI `run-onchain-listener.ts` executável em one-shot ou loop.
6. ✅ Full suite ≥ 120 passed + 1 skipped; tsc clean.

---

## Riscos e mitigações

| Risco | Mitigação |
|---|---|
| Memo program ID diferente em devnet/mainnet | `MemoSq4gqABAXKb96qnH8TysNcWxMyWCqXgDLGmfcHr` é o canonical Memo v2, igual nos dois clusters. |
| Compute budget 500k CU estoura | Validar em devnet; subir pra 700k se necessário. |
| `getSignaturesForAddress` paginação | Plano 5 faz scan simples com cursor opcional; produção pode precisar catch-up loop com `before` cursor. |
| Memo pode ser alterado por MEV / tampering | Program memo é assinado como parte da tx; alterar memo = nova signature = listener trata como outra tx. |
| Listener processa tx que não é nossa (outro caller) | UUID regex + trade lookup garantem: sem UUID match → skipped. Com UUID match mas sem trade → orphaned (event recorded pra debug, status não muda). |
| Event table cresce indefinidamente | Sem TTL neste plano. Cleanup fica pro Plano 9 (reconciliação diária) — só purga events > 90 dias. |

---

## O que NÃO está neste plano

- **Fee ledger integration**: Plano 6.
- **WebSocket emit de trade_settled**: Plano 7.
- **Cron wiring do listener no app entry**: Plano 9 (cutover).
- **Partial signatures recovery**: se o tx foi enviado mas nunca confirmou e depois perdemos a referência, ninguém recupera. Plano 5 trata apenas o caso "tx confirmou mas mark SETTLED crashou". Casos mais exóticos exigem tracking pré-send que ainda não temos.
