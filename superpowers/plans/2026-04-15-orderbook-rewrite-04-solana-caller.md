# Orderbook Rewrite — Plano 4: SolanaOnchainCaller (reuso do contrato atual)

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Implementar `SolanaOnchainCaller` concreto que cumpre o contrato `IOnchainCaller` definido no Plano 3, **reusando as instruções existentes do programa Solana** (`settle_clob` e `settle_clob_sell`) sem tocar em Rust. O caller compõe 2 instructions num único `Transaction` (atomicidade garantida pela Solana) e mapeia os 3 primitives (TRADE / MINT / MERGE) para as combinações certas.

**Architecture:** Refatoramos os serviços legacy (`clobSettleService`, `clobSettleSellService`) para expor um método `buildInstruction(params): Promise<TransactionInstruction>` puro — **sem enviar ao RPC**. O `SolanaOnchainCaller` novo, no módulo `trading-v2`, compõe duas chamadas de `buildInstruction` num único `Transaction`, assina com a platform wallet e envia via `sendAndConfirm`. O resultado (signature ou erro) é devolvido no shape `SettleFillResult`. Os métodos legacy `settle()` continuam funcionando inalterados (eles internamente chamam `buildInstruction` + `sendAndConfirm` de um leg só).

**Tech Stack:** Bun, TypeScript, `@coral-xyz/anchor`, `@solana/web3.js`, `@solana/spl-token`, Jest.

**Spec:** `docs/superpowers/specs/2026-04-15-orderbook-rewrite-design.md` (§7 settlement)
**Planos anteriores:**
- Plano 1 (merged `c891f0b`) — fundação
- Plano 2 (merged `c591534`) — orders + matching + stub settler
- Plano 3 (merged `fe88950`) — SolanaSettler + reverter + deadline worker

**Próximos planos (após este):**
- Plano 5: Listener on-chain (observar eventos do programa, idempotência via `ob2_onchain_events_processed`); fee ledger real integration.
- Plano 6: WebSocket v2 • Plano 7: MM bot externo • Plano 8: Cutover

**Notas do review do Plano 3 endereçadas aqui:**
1. Hardcoded `30_000` no `MatchingEngine.tryMatch` → refatora pra usar `TradeRepository.create` (Task 10 opcional).
2. `RevertResult.reopened` cosmético → não endereçado neste plano (pode virar Plano 5).

**Não-escopo deste plano:**
- **Listener on-chain** (observar signatures emitidas, idempotência via `ob2_onchain_events_processed`) fica pro Plano 5. Idempotência no Plano 4 é garantida pelo status check do trade (SETTLING → SETTLED) feito pelo `SolanaSettlerService` existente.
- **Divergência stub NO(BUY)×USDC(SELL) TRADE**: a composição on-chain descrita aqui cobre esse caso corretamente (o seller recebe USDC via `settle_clob_sell(BuyNo)` — não NO minted do nada como no stub). O cutover ainda precisa reconciliar quem usou a forma stub.

---

## File Structure

```
api/src/modules/trading-v2/
  services/
    solana-onchain-caller.service.ts            # CREATE: IOnchainCaller concreto
    legacy-clob-instruction-builder.ts          # CREATE: thin wrapper extraindo build* dos legacy
    __tests__/
      solana-onchain-caller.unit.test.ts        # CREATE: unit tests com stubs dos builders
      solana-onchain-caller.devnet.test.ts      # CREATE: integration end-to-end em devnet (skipped em CI)
  routes/
    orders.routes.ts                            # MODIFY: swap MockOnchainCaller → SolanaOnchainCaller
  index.ts                                      # MODIFY: export SolanaOnchainCaller

api/src/modules/prediction-market/trading/services/order/
  clob-settle.service.ts                        # MODIFY: extrair buildSettleClobInstruction() público
  clob-settle-sell.service.ts                   # MODIFY: extrair buildSettleClobSellInstruction() público
```

**Responsabilidades (novos):**

- `legacy-clob-instruction-builder.ts` — **thin wrapper** ao redor dos dois legacy services. Exporta `buildBuyLegInstruction(params)` e `buildSellLegInstruction(params)` que chamam o builder correspondente nos serviços legacy. Isolamos aqui o acoplamento com o código legacy; o resto do trading-v2 importa daqui e fica desacoplado.
- `solana-onchain-caller.service.ts` — `SolanaOnchainCaller implements IOnchainCaller`. Mapeia primitive → 2 builder calls → compõe `Transaction` → assina → `sendAndConfirm`. Retorna `SettleFillResult` (ok+signature OU ok=false+reason+retryable).
- `solana-onchain-caller.unit.test.ts` — testes unit com `MockConnection` e builders stubados, validam a composição de instruções e a classificação de erros.
- `solana-onchain-caller.devnet.test.ts` — teste integração opcional em devnet. Marcado com `describe.skip` por default; rodado manualmente via `SKIP_DEVNET_TESTS=false bun x jest`. Valida que a tx de fato é aceita pelo programa on-chain.

---

## Prerequisite check

- [ ] **Step 0: Baseline.**

```bash
cd /Users/rafael/Documents/GitHub/Oraculo/api
bun x jest src/modules/trading-v2 --runInBand
```

Expected: `Tests: 96 passed`.

- [ ] **Step 0.1: Worktree:**

```bash
git worktree add .claude/worktrees/orderbook-rewrite-04-solana-caller -b worktree-orderbook-rewrite-04-solana-caller
cp api/.env .claude/worktrees/orderbook-rewrite-04-solana-caller/api/.env
cd .claude/worktrees/orderbook-rewrite-04-solana-caller/api
bun install
bun x prisma generate
bun x jest src/modules/trading-v2 --runInBand    # 96 passed no worktree
```

---

## Task 1: Investigação — mapear os serviços legacy

**Goal:** Antes de mexer em código, entender a interface exata dos serviços `clobSettleService` e `clobSettleSellService` (legacy). Sem esse mapeamento, refatorar pra expor `buildInstruction` é chute.

**Files:**
- Create: `api/src/modules/trading-v2/services/INVESTIGATION-plano4.md` (doc de referência, será deletado no final do plano)

- [ ] **Step 1.1: Ler os 2 serviços legacy e documentar:**

```bash
cd /Users/rafael/Documents/GitHub/Oraculo/.claude/worktrees/orderbook-rewrite-04-solana-caller/api
cat src/modules/prediction-market/trading/services/order/clob-settle.service.ts
cat src/modules/prediction-market/trading/services/order/clob-settle-sell.service.ts
```

Pra cada um, identifique e anote em `INVESTIGATION-plano4.md`:

1. **Assinatura de `settle(params)`**: campos de entrada (`marketPda`, `walletAddress`, `side: "yes"|"no"`, `tokenAmount: number`, `executionPrice: number`, `programId?`, metadados `tradeId`/`buyOrderId`/`sellOrderId`/`userId`).
2. **PDAs derivados** (qual seed, qual programId). Anote em que linha cada PDA é calculado. Ex.: `calculateVaultPDA(marketPda)`.
3. **Token accounts criados/resolvidos**: ATA do wallet + vault ATA. Identifique se `createAssociatedTokenAccountInstruction` é prepend ao tx principal (isso tem que vir junto no nosso tx composto).
4. **`question_hash`**: como é derivado? (Provavelmente `keccak256` de alguma string do market.) Anote.
5. **Instrução Anchor concreta**: que método do `program.methods` é chamado — `.settleClob(...)` ou `.settleClobSell(...)`, com que args, que `.accounts({...})`, que `.signers([...])`.
6. **Onde o `.rpc()` é chamado** — essa é a linha que vamos trocar por `.instruction()` no `buildInstruction`.
7. **Side effects fora do tx**: atualização de `market_trades`, `user_positions`, cache, notifications. Qualquer coisa feita DEPOIS do `.rpc()` success — anotar mas **não extrair** pro builder. O builder só monta a instruction, os side effects ficam no `settle()` legacy.
8. **Error handling**: que exceções são lançadas? Como distinguir retryable vs não-retryable?

Produza `INVESTIGATION-plano4.md` com seções "buy-leg (settle_clob)" e "sell-leg (settle_clob_sell)" espelhadas. Tamanho alvo: 100-200 linhas.

- [ ] **Step 1.2: Commit:**

```bash
git add api/src/modules/trading-v2/services/INVESTIGATION-plano4.md
git commit -m "docs(trading-v2): investigação dos legacy clob settle services (plano 4)"
```

Este arquivo é descartável — deletado no Task 11.

---

## Task 2: Extrair `buildSettleClobInstruction` do serviço legacy buy-leg

**Files:**
- Modify: `api/src/modules/prediction-market/trading/services/order/clob-settle.service.ts`

**Goal:** Partir o método `settle()` em dois: `buildInstruction()` (puro — devolve `TransactionInstruction` + lista de ATAs-setup instructions pré-required) e `settle()` (legacy — chama `buildInstruction` + envia tx + faz side effects).

- [ ] **Step 2.1: Adicionar interfaces públicas.** No topo do arquivo, depois das interfaces existentes `ClobSettleParams`/`ClobSettleResult`, adicionar:

```typescript
import type { TransactionInstruction } from "@solana/web3.js";

/**
 * Resultado de buildInstruction: a instruction principal + instructions de
 * setup (create ATA) que DEVEM vir antes na transaction.
 */
export interface BuiltInstruction {
  setupInstructions: TransactionInstruction[];   // may be empty
  mainInstruction: TransactionInstruction;
  /** Signers adicionais exigidos pela instruction (além do platform/fee payer). */
  signers: Array<import("@solana/web3.js").Keypair>;
}
```

- [ ] **Step 2.2: Ler o método `settle()` existente e identificar:**

  - **Prólogo puro** — chamadas que produzem valores (leitura do market, validação, derivação de PDAs, cálculo de amounts, resolução/criação de ATAs).
  - **Montagem da instruction principal** — `.methods.settleClob(...).accounts({...}).instruction()` (hoje provavelmente é `.rpc()` em vez de `.instruction()`).
  - **Envio** — `connection.sendAndConfirmTransaction` ou equivalente.
  - **Side effects pós-envio** — atualização do DB via `marketRepository`, `userPositionService`, etc.

- [ ] **Step 2.3: Criar `buildInstruction(params): Promise<BuiltInstruction>` como método público da classe `ClobSettleService`.**

Essa função DEVE fazer exatamente o **prólogo puro + montagem da instruction + retorno** — mas NÃO enviar a tx e NÃO fazer side effects.

Estrutura esperada:

```typescript
  async buildInstruction(params: ClobSettleParams): Promise<BuiltInstruction> {
    // [reaproveitar o código do settle() atual ATÉ o ponto em que ele chama .rpc() ou sendAndConfirm]
    // 1. Ler market do DB/cache
    // 2. Derivar PDAs (vault, config, user_account, market)
    // 3. Resolver ATAs (buyer USDC ATA, buyer token ATA)
    //    Se ATA do buyer não existe, adicionar create-ATA ao setupInstructions.
    // 4. Calcular tokenAmount em 9-decimals e executionPrice em cents
    // 5. Montar a instruction:
    //      const mainInstruction = await program.methods
    //        .settleClob(side, new BN(tokenAmountSmallestUnits), new BN(executionPrice), questionHash)
    //        .accounts({ ... })
    //        .instruction();
    // 6. Return { setupInstructions, mainInstruction, signers }
  }
```

**IMPORTANTE:** reescreva o MÍNIMO necessário. Se o `settle()` original tem 280 linhas e 250 delas são prólogo, a maior parte vai pro `buildInstruction`. O `settle()` passa a ser um wrapper curto:

```typescript
  async settle(params: ClobSettleParams): Promise<ClobSettleResult> {
    const built = await this.buildInstruction(params);

    // Monta tx e envia (código existente adapted)
    const tx = new Transaction();
    built.setupInstructions.forEach(ix => tx.add(ix));
    tx.add(built.mainInstruction);

    const signature = await sendAndConfirmTransaction(connection, tx, [
      platformSigner, ...built.signers,
    ]);

    // Side effects — atualização DB, etc. (código existente)
    // [preservar bloco de DB updates tal como estava]

    return { transactionSignature: signature, tokensReceived: /* ... */ };
  }
```

- [ ] **Step 2.4: Validar que o build não quebra.** Run:

```bash
cd /Users/rafael/Documents/GitHub/Oraculo/.claude/worktrees/orderbook-rewrite-04-solana-caller/api
bun x tsc --noEmit 2>&1 | grep clob-settle
```

Expected: nenhum erro novo em `clob-settle.service.ts`. (Podem existir erros pré-existentes em outros módulos — ignorar.)

- [ ] **Step 2.5: Rodar tests existentes desse serviço pra garantir que não quebramos nada.** Se houver um `clob-settle.service.test.ts` no `prediction-market/trading/__tests__/`, rode-o:

```bash
bun x jest src/modules/prediction-market/trading/__tests__/clob-settle 2>&1 | tail -10
```

Se não passar, investigar. Se passar, seguir.

- [ ] **Step 2.6: Commit:**

```bash
git add api/src/modules/prediction-market/trading/services/order/clob-settle.service.ts
git commit -m "refactor(prediction-market): extrai buildInstruction do ClobSettleService"
```

---

## Task 3: Extrair `buildSettleClobSellInstruction` do serviço legacy sell-leg

**Mesma estrutura de Task 2**, aplicada a `clob-settle-sell.service.ts`. Repete o processo de:
1. Adicionar `BuiltInstruction` interface (se ainda não compartilhada — alternativa: importar do arquivo do buy-leg).
2. Criar `buildInstruction(params): Promise<BuiltInstruction>`.
3. Reduzir `settle()` a wrapper curto.

**Files:**
- Modify: `api/src/modules/prediction-market/trading/services/order/clob-settle-sell.service.ts`

- [ ] **Step 3.1: Reuse the BuiltInstruction interface defined in Task 2.** Import from `./clob-settle.service`:

```typescript
import type { BuiltInstruction } from "./clob-settle.service";
```

- [ ] **Step 3.2: Criar `async buildInstruction(params: ClobSettleSellParams): Promise<BuiltInstruction>` na classe `ClobSettleSellService`.**

Mesma estrutura da Task 2 mas chamando `.methods.settleClobSell(...)` (verifique o nome exato na investigação do Task 1). Retornar `BuiltInstruction`.

- [ ] **Step 3.3: Reduzir `settle()` a wrapper** — envia a tx composta de `setupInstructions + mainInstruction`.

- [ ] **Step 3.4: tsc + test check:**

```bash
bun x tsc --noEmit 2>&1 | grep clob-settle-sell
bun x jest src/modules/prediction-market/trading/__tests__/clob-settle-sell 2>&1 | tail -10 || true
```

- [ ] **Step 3.5: Commit:**

```bash
git add api/src/modules/prediction-market/trading/services/order/clob-settle-sell.service.ts
git commit -m "refactor(prediction-market): extrai buildInstruction do ClobSettleSellService"
```

---

## Task 4: Thin wrapper `legacy-clob-instruction-builder.ts`

**Goal:** Isolar o acoplamento com o código legacy. O `trading-v2` só importa daqui.

**Files:**
- Create: `api/src/modules/trading-v2/services/legacy-clob-instruction-builder.ts`

- [ ] **Step 4.1: Create the wrapper:**

```typescript
/**
 * Thin wrapper ao redor dos legacy ClobSettle services. Isola o import do código
 * legacy para manter `trading-v2` desacoplado.
 *
 * Quando (se) Plano N substituir o contrato on-chain, trocamos APENAS este arquivo.
 */
import type { TransactionInstruction, Keypair } from "@solana/web3.js";
import { clobSettleService, type BuiltInstruction } from "@/modules/prediction-market/trading/services/order/clob-settle.service";
import { clobSettleSellService } from "@/modules/prediction-market/trading/services/order/clob-settle-sell.service";

export type { BuiltInstruction };

export interface LegParams {
  marketPda: string;
  walletAddress: string;
  side: "yes" | "no";
  /** Quantidade de tokens em unidades HUMANAS (ex.: 1.81). Os builders legacy convertem pra 9-decimals. */
  tokenAmount: number;
  /** Preço em cents (1..99). */
  executionPrice: number;
  /** Metadados de tracking — passados só pra logs; não afetam a instruction. */
  tradeId: string;
}

export async function buildBuyLegInstruction(p: LegParams): Promise<BuiltInstruction> {
  return clobSettleService.buildInstruction({
    marketPda: p.marketPda,
    walletAddress: p.walletAddress,
    side: p.side,
    tokenAmount: p.tokenAmount,
    executionPrice: p.executionPrice,
    tradeId: p.tradeId,
  });
}

export async function buildSellLegInstruction(p: LegParams): Promise<BuiltInstruction> {
  return clobSettleSellService.buildInstruction({
    marketPda: p.marketPda,
    walletAddress: p.walletAddress,
    side: p.side,
    tokenAmount: p.tokenAmount,
    executionPrice: p.executionPrice,
    tradeId: p.tradeId,
  });
}
```

**Nota:** o shape de `ClobSettleParams`/`ClobSettleSellParams` pode ter campos obrigatórios adicionais que não estão em `LegParams` (descobertos na investigação). Ajustar `LegParams` pra cobrir tudo necessário. Se aparecer um campo que `trading-v2` não conhece (ex.: `buyOrderId`/`sellOrderId` do legacy), passar `undefined` e validar que o builder legacy aceita.

- [ ] **Step 4.2: tsc clean:**

```bash
bun x tsc --noEmit 2>&1 | grep trading-v2 || echo "clean"
```

- [ ] **Step 4.3: Commit:**

```bash
git add api/src/modules/trading-v2/services/legacy-clob-instruction-builder.ts
git commit -m "feat(trading-v2): thin wrapper pros legacy clob builders"
```

---

## Task 5: SolanaOnchainCaller — estrutura e TRADE primitive

**Files:**
- Create: `api/src/modules/trading-v2/services/solana-onchain-caller.service.ts`
- Create: `api/src/modules/trading-v2/__tests__/solana-onchain-caller.unit.test.ts`

**Design decision — wallet address mapping:** o `SettleFillParams` carrega `buyerUserId` e `sellerUserId` (UUIDs). Os legacy builders exigem `walletAddress` (Solana pubkey). Precisamos mapear UUID → wallet. O projeto tem `user.repository` com `findById` que retorna o `walletAddress`. O `SolanaOnchainCaller` recebe `userRepo` via injection.

- [ ] **Step 5.1: Failing unit test pra TRADE primitive.**

```typescript
import { SolanaOnchainCaller } from "../services/solana-onchain-caller.service";
import type { SettleFillParams } from "../types/onchain-caller.types";
import type { LegParams, BuiltInstruction } from "../services/legacy-clob-instruction-builder";
import { Keypair, PublicKey, TransactionInstruction } from "@solana/web3.js";
import { UNIT } from "../types/balance.types";

// Fakes ──────────────────────────────────────────────────────────
const fakeBuiltIx = (marker: string): BuiltInstruction => ({
  setupInstructions: [],
  mainInstruction: new TransactionInstruction({
    keys: [], programId: new PublicKey("11111111111111111111111111111111"), data: Buffer.from(marker),
  }),
  signers: [],
});

const fakeUserRepo = {
  async getWalletAddress(userId: string): Promise<string> {
    if (userId === "u-buyer")  return "BuyerWalletxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx";
    if (userId === "u-seller") return "SellerWalletxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx";
    throw new Error(`unknown user ${userId}`);
  },
};

const baseParams: SettleFillParams = {
  tradeId: "t-1",
  marketPda: "Market1111111111111111111111111111111111111",
  buyerUserId: "u-buyer", sellerUserId: "u-seller",
  buyerReservationAsset: "USDC",
  sellerReservationAsset: "YES",
  priceBps: 5000,
  quantityMicro: 100n * UNIT,
  primitive: "TRADE",
};

test("TRADE: compõe buy-leg (buyer, YES) + sell-leg (seller, YES) no mesmo tx", async () => {
  const builderCalls: Array<{ kind: "buy" | "sell"; params: LegParams }> = [];
  const builder = {
    async buildBuyLegInstruction(p: LegParams)  { builderCalls.push({ kind: "buy", params: p });  return fakeBuiltIx("buy");  },
    async buildSellLegInstruction(p: LegParams) { builderCalls.push({ kind: "sell", params: p }); return fakeBuiltIx("sell"); },
  };

  const sender = {
    async sendTransaction(tx: Array<TransactionInstruction>) {
      return { ok: true as const, signature: "sig-fake-1" };
    },
  };

  const caller = new SolanaOnchainCaller(builder, sender, fakeUserRepo);
  const result = await caller.sendSettleFill(baseParams);

  expect(result.ok).toBe(true);
  if (result.ok) expect(result.signature).toBe("sig-fake-1");
  expect(builderCalls).toHaveLength(2);
  expect(builderCalls[0].kind).toBe("buy");
  expect(builderCalls[0].params.walletAddress).toBe("BuyerWalletxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx");
  expect(builderCalls[0].params.side).toBe("yes");
  expect(builderCalls[0].params.executionPrice).toBe(50);    // 5000bps → 50 cents
  expect(builderCalls[1].kind).toBe("sell");
  expect(builderCalls[1].params.walletAddress).toBe("SellerWalletxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx");
  expect(builderCalls[1].params.side).toBe("yes");
});

test("TRADE: NO(BUY) × USDC(SELL) variant → buy-leg(buyer,NO) + sell-leg(seller,NO)", async () => {
  const builderCalls: Array<{ kind: "buy" | "sell"; params: LegParams }> = [];
  const builder = {
    async buildBuyLegInstruction(p: LegParams)  { builderCalls.push({ kind: "buy", params: p });  return fakeBuiltIx("buy");  },
    async buildSellLegInstruction(p: LegParams) { builderCalls.push({ kind: "sell", params: p }); return fakeBuiltIx("sell"); },
  };
  const sender = {
    async sendTransaction() { return { ok: true as const, signature: "sig-fake-2" }; },
  };

  const caller = new SolanaOnchainCaller(builder, sender, fakeUserRepo);
  const result = await caller.sendSettleFill({
    ...baseParams,
    buyerReservationAsset: "NO",
    sellerReservationAsset: "USDC",
  });

  expect(result.ok).toBe(true);
  expect(builderCalls[0].kind).toBe("buy");
  expect(builderCalls[0].params.side).toBe("no");
  expect(builderCalls[1].kind).toBe("sell");
  expect(builderCalls[1].params.side).toBe("no");
});
```

- [ ] **Step 5.2: Run, fail.**

```bash
cd /Users/rafael/Documents/GitHub/Oraculo/.claude/worktrees/orderbook-rewrite-04-solana-caller/api
bun x jest src/modules/trading-v2/__tests__/solana-onchain-caller.unit.test.ts --runInBand
```

- [ ] **Step 5.3: Implement the caller** (`solana-onchain-caller.service.ts`) with ONLY the TRADE path for now:

```typescript
import type { TransactionInstruction } from "@solana/web3.js";
import type { IOnchainCaller, SettleFillParams, SettleFillResult } from "../types/onchain-caller.types";
import type { LegParams, BuiltInstruction } from "./legacy-clob-instruction-builder";

/**
 * Resolve a user's Solana wallet address. Injected by caller so we don't
 * hard-couple to the user repository (and tests can use a fake).
 */
export interface UserWalletLookup {
  getWalletAddress(userId: string): Promise<string>;
}

/**
 * Builds a main + setup instructions per leg. Injected for testability; in prod
 * the implementation just delegates to legacy-clob-instruction-builder functions.
 */
export interface InstructionBuilder {
  buildBuyLegInstruction(params: LegParams): Promise<BuiltInstruction>;
  buildSellLegInstruction(params: LegParams): Promise<BuiltInstruction>;
}

/**
 * Sends a list of instructions as a single atomic Solana transaction.
 * Returns ok+signature on confirmed success, or a classified failure.
 */
export interface TransactionSender {
  sendTransaction(
    instructions: TransactionInstruction[],
  ): Promise<{ ok: true; signature: string } | { ok: false; reason: string; retryable: boolean }>;
}

export class SolanaOnchainCaller implements IOnchainCaller {
  constructor(
    private readonly builder: InstructionBuilder,
    private readonly sender: TransactionSender,
    private readonly users: UserWalletLookup,
  ) {}

  async sendSettleFill(params: SettleFillParams): Promise<SettleFillResult> {
    try {
      const legs = await this.buildLegs(params);
      const allInstructions: TransactionInstruction[] = [];
      legs.forEach(l => allInstructions.push(...l.setupInstructions, l.mainInstruction));
      const result = await this.sender.sendTransaction(allInstructions);
      return result;
    } catch (e) {
      const msg = e instanceof Error ? e.message : String(e);
      return { ok: false, reason: `caller_exception: ${msg}`, retryable: false };
    }
  }

  /** Retorna os BuiltInstructions na ordem em que devem entrar no tx (buy-leg primeiro). */
  private async buildLegs(params: SettleFillParams): Promise<BuiltInstruction[]> {
    const buyerWallet  = await this.users.getWalletAddress(params.buyerUserId);
    const sellerWallet = await this.users.getWalletAddress(params.sellerUserId);
    const tokenAmount = this.microToHuman(params.quantityMicro);
    const executionPrice = Math.floor(params.priceBps / 100);   // bps → cents

    const buyerSide  = this.sideFor(params.buyerReservationAsset,  params.sellerReservationAsset);
    const sellerSide = buyerSide;  // TRADE/MINT/MERGE all use the same side for both legs (see mapping)

    switch (params.primitive) {
      case "TRADE": {
        // Buyer side depends on reservation: USDC(BUY)×YES(SELL) → side="yes"
        //                                     NO(BUY)×USDC(SELL)  → side="no"
        const side: "yes" | "no" = params.buyerReservationAsset === "USDC" ? "yes" : "no";
        const buy = await this.builder.buildBuyLegInstruction({
          marketPda: params.marketPda, walletAddress: buyerWallet,
          side, tokenAmount, executionPrice, tradeId: params.tradeId,
        });
        const sell = await this.builder.buildSellLegInstruction({
          marketPda: params.marketPda, walletAddress: sellerWallet,
          side, tokenAmount, executionPrice, tradeId: params.tradeId,
        });
        return [buy, sell];
      }
      case "MINT": {
        // To implement in Task 6
        throw new Error("MINT not implemented yet");
      }
      case "MERGE": {
        // To implement in Task 7
        throw new Error("MERGE not implemented yet");
      }
    }
  }

  // Placeholder helper — see §§ below for real mapping
  private sideFor(_b: string, _s: string): "yes" | "no" { return "yes"; }

  private microToHuman(v: bigint): number {
    // micro-units (10^6) to human decimal number. Tokens have 9 decimals on-chain
    // but the legacy builder takes a human number and multiplies by 10^9 internally.
    // Our micro-units are 10^6. Divide by 10^6 to get human number.
    // Avoid Number overflow for large bigint: convert via string.
    const s = v.toString();
    if (v === 0n) return 0;
    if (v < 0n) throw new Error("negative quantity");
    // Pad to at least 7 digits, split last 6 as fraction
    const padded = s.padStart(7, "0");
    const intPart = padded.slice(0, -6);
    const fracPart = padded.slice(-6);
    return Number(`${intPart}.${fracPart}`);
  }
}
```

- [ ] **Step 5.4: Run, TRADE tests pass.**

Expected: 2 TRADE tests passed. MINT/MERGE tests don't exist yet.

- [ ] **Step 5.5: Commit:**

```bash
git add api/src/modules/trading-v2/services/solana-onchain-caller.service.ts \
        api/src/modules/trading-v2/__tests__/solana-onchain-caller.unit.test.ts
git commit -m "feat(trading-v2): SolanaOnchainCaller (TRADE primitive)"
```

---

## Task 6: SolanaOnchainCaller — MINT primitive

**Files:**
- Modify: `api/src/modules/trading-v2/services/solana-onchain-caller.service.ts`
- Modify: `api/src/modules/trading-v2/__tests__/solana-onchain-caller.unit.test.ts`

MINT semantics: buyer opens long YES, seller opens long NO. Both pay USDC. On-chain:
- Buy leg for buyer: `settle_clob(side="yes", qty, price)` — buyer pays `price` cents per token, gets YES.
- Buy leg for seller: `settle_clob(side="no", qty, 100-price)` — seller pays `(100-price)` cents per token, gets NO. **execution_price é INVERTIDO** porque no `settle_clob(BuyNo)` o usuário paga pelo NO que é complement do YES.

Verificar na investigação (Task 1) se `settle_clob(BuyNo, qty, executionPrice)` trata `executionPrice` como preço do NO (em cents, 1..99) ou como preço do YES. Se for preço do NO, passamos `100 - priceYesCents`. Se for sempre preço do YES (e o programa calcula o NO internamente), passamos `priceYesCents`. **Esta é a checagem mais crítica desta task** — errar aqui introduz drift de centavo por trade.

- [ ] **Step 6.1: Acrescentar test pra MINT:**

```typescript
test("MINT: buyer gets YES, seller gets NO via 2× buy-leg with opposite sides", async () => {
  const builderCalls: Array<{ kind: "buy" | "sell"; params: LegParams }> = [];
  const builder = {
    async buildBuyLegInstruction(p: LegParams)  { builderCalls.push({ kind: "buy", params: p });  return fakeBuiltIx("buy");  },
    async buildSellLegInstruction(p: LegParams) { builderCalls.push({ kind: "sell", params: p }); return fakeBuiltIx("sell"); },
  };
  const sender = {
    async sendTransaction() { return { ok: true as const, signature: "sig-mint" }; },
  };

  const caller = new SolanaOnchainCaller(builder, sender, fakeUserRepo);
  const result = await caller.sendSettleFill({
    ...baseParams,
    primitive: "MINT",
    buyerReservationAsset: "USDC",
    sellerReservationAsset: "USDC",
    priceBps: 6000,    // 60 cents
  });

  expect(result.ok).toBe(true);
  expect(builderCalls).toHaveLength(2);
  expect(builderCalls[0].kind).toBe("buy");
  expect(builderCalls[0].params.side).toBe("yes");
  expect(builderCalls[0].params.executionPrice).toBe(60);
  expect(builderCalls[0].params.walletAddress).toBe("BuyerWalletxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx");
  expect(builderCalls[1].kind).toBe("buy");     // NOT sell — this is the key difference from TRADE
  expect(builderCalls[1].params.side).toBe("no");
  // executionPrice for NO leg: verify against the investigation doc.
  // If settle_clob(BuyNo) interprets executionPrice as "price of NO", expect 40 (100 - 60).
  // If it interprets as "price of YES", expect 60.
  // Default assumption here is "price of NO" — MUST match investigation finding.
  expect(builderCalls[1].params.executionPrice).toBe(40);
  expect(builderCalls[1].params.walletAddress).toBe("SellerWalletxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx");
});
```

**If the investigation in Task 1 established that `settle_clob(BuyNo, _, execPrice)` expects the YES price (not NO), change the expectation to `60` and adjust the implementation accordingly.**

- [ ] **Step 6.2: Run, fail.**

- [ ] **Step 6.3: Replace the `case "MINT":` branch in `buildLegs` with:**

```typescript
      case "MINT": {
        // Buyer gets YES → buy-leg side=yes, price = priceBps in cents
        // Seller gets NO → buy-leg side=no, price = complement (or YES price, per §6.2 investigation)
        const priceYesCents = Math.floor(params.priceBps / 100);
        const priceNoCents  = 100 - priceYesCents;   // adjust if investigation says otherwise
        const buy = await this.builder.buildBuyLegInstruction({
          marketPda: params.marketPda, walletAddress: buyerWallet,
          side: "yes", tokenAmount, executionPrice: priceYesCents, tradeId: params.tradeId,
        });
        const sellerOpensShort = await this.builder.buildBuyLegInstruction({
          marketPda: params.marketPda, walletAddress: sellerWallet,
          side: "no",  tokenAmount, executionPrice: priceNoCents,  tradeId: params.tradeId,
        });
        return [buy, sellerOpensShort];
      }
```

- [ ] **Step 6.4: Run, 3 tests pass (2 TRADE + 1 MINT).**

- [ ] **Step 6.5: Commit:**

```bash
git add api/src/modules/trading-v2/services/solana-onchain-caller.service.ts \
        api/src/modules/trading-v2/__tests__/solana-onchain-caller.unit.test.ts
git commit -m "feat(trading-v2): SolanaOnchainCaller MINT primitive (2× buy-leg)"
```

---

## Task 7: SolanaOnchainCaller — MERGE primitive

MERGE semantics: buyer is closing NO short (has NO, wants USDC), seller is closing YES long (has YES, wants USDC). On-chain:
- Sell leg for buyer: `settle_clob_sell(side="no", qty, price_no_cents)` — buyer returns NO, gets USDC at NO price (which is `100-priceYesCents`).
- Sell leg for seller: `settle_clob_sell(side="yes", qty, price_yes_cents)` — seller returns YES, gets USDC at YES price.

Vault burns both tokens.

**Investigation check:** confirm `settle_clob_sell(BuyYes|BuyNo, qty, execPrice)` expects `execPrice` in the same scheme as `settle_clob` for symmetric price handling. If there's a divergence (e.g., sell-side always uses YES price regardless of side), adjust.

- [ ] **Step 7.1: Test:**

```typescript
test("MERGE: buyer returns NO via sell-leg(no), seller returns YES via sell-leg(yes); vault burns both", async () => {
  const builderCalls: Array<{ kind: "buy" | "sell"; params: LegParams }> = [];
  const builder = {
    async buildBuyLegInstruction(p: LegParams)  { builderCalls.push({ kind: "buy", params: p });  return fakeBuiltIx("buy");  },
    async buildSellLegInstruction(p: LegParams) { builderCalls.push({ kind: "sell", params: p }); return fakeBuiltIx("sell"); },
  };
  const sender = {
    async sendTransaction() { return { ok: true as const, signature: "sig-merge" }; },
  };

  const caller = new SolanaOnchainCaller(builder, sender, fakeUserRepo);
  const result = await caller.sendSettleFill({
    ...baseParams,
    primitive: "MERGE",
    buyerReservationAsset: "NO",
    sellerReservationAsset: "YES",
    priceBps: 6000,
  });

  expect(result.ok).toBe(true);
  expect(builderCalls).toHaveLength(2);
  expect(builderCalls[0].kind).toBe("sell");
  expect(builderCalls[0].params.side).toBe("no");
  expect(builderCalls[0].params.executionPrice).toBe(40);  // NO complement — adjust per investigation
  expect(builderCalls[1].kind).toBe("sell");
  expect(builderCalls[1].params.side).toBe("yes");
  expect(builderCalls[1].params.executionPrice).toBe(60);
});
```

- [ ] **Step 7.2: Run, fail.**

- [ ] **Step 7.3: Replace the `case "MERGE":` branch:**

```typescript
      case "MERGE": {
        const priceYesCents = Math.floor(params.priceBps / 100);
        const priceNoCents  = 100 - priceYesCents;
        const buyerReturnsNo = await this.builder.buildSellLegInstruction({
          marketPda: params.marketPda, walletAddress: buyerWallet,
          side: "no",  tokenAmount, executionPrice: priceNoCents,  tradeId: params.tradeId,
        });
        const sellerReturnsYes = await this.builder.buildSellLegInstruction({
          marketPda: params.marketPda, walletAddress: sellerWallet,
          side: "yes", tokenAmount, executionPrice: priceYesCents, tradeId: params.tradeId,
        });
        return [buyerReturnsNo, sellerReturnsYes];
      }
```

- [ ] **Step 7.4: Run, 4 tests pass.**

- [ ] **Step 7.5: Commit:**

```bash
git add api/src/modules/trading-v2/services/solana-onchain-caller.service.ts \
        api/src/modules/trading-v2/__tests__/solana-onchain-caller.unit.test.ts
git commit -m "feat(trading-v2): SolanaOnchainCaller MERGE primitive (2× sell-leg)"
```

---

## Task 8: Error classification — retryable vs non-retryable

**Goal:** Quando `sender.sendTransaction` retorna erro, precisamos classificar pra `SolanaSettler.settle` decidir retry vs revert. Convenções:

- **Retryable:** network timeouts, RPC down, `BlockhashNotFound` (expirou antes de confirmar), `TransactionExpiredTimeoutError`, `fetch` errors.
- **Non-retryable:** program error (custom error code), `InsufficientFundsForRent`, simulation failure, assinatura inválida.

**Files:**
- Modify: `api/src/modules/trading-v2/services/solana-onchain-caller.service.ts`
- Modify: `api/src/modules/trading-v2/__tests__/solana-onchain-caller.unit.test.ts`

- [ ] **Step 8.1: Acrescentar tests:**

```typescript
test("retryable failure from sender is passed through", async () => {
  const builder = {
    async buildBuyLegInstruction()  { return fakeBuiltIx("buy");  },
    async buildSellLegInstruction() { return fakeBuiltIx("sell"); },
  };
  const sender = {
    async sendTransaction() {
      return { ok: false as const, reason: "rpc_timeout", retryable: true };
    },
  };

  const caller = new SolanaOnchainCaller(builder, sender, fakeUserRepo);
  const r = await caller.sendSettleFill(baseParams);
  expect(r.ok).toBe(false);
  if (!r.ok) {
    expect(r.reason).toBe("rpc_timeout");
    expect(r.retryable).toBe(true);
  }
});

test("non-retryable failure from sender is passed through", async () => {
  const builder = {
    async buildBuyLegInstruction()  { return fakeBuiltIx("buy");  },
    async buildSellLegInstruction() { return fakeBuiltIx("sell"); },
  };
  const sender = {
    async sendTransaction() {
      return { ok: false as const, reason: "program_error_0x1234", retryable: false };
    },
  };

  const caller = new SolanaOnchainCaller(builder, sender, fakeUserRepo);
  const r = await caller.sendSettleFill(baseParams);
  expect(r.ok).toBe(false);
  if (!r.ok) expect(r.retryable).toBe(false);
});

test("unknown user throws inside buildLegs → caller returns non-retryable", async () => {
  const builder = {
    async buildBuyLegInstruction()  { return fakeBuiltIx("buy");  },
    async buildSellLegInstruction() { return fakeBuiltIx("sell"); },
  };
  const sender = {
    async sendTransaction() { throw new Error("should not be called"); },
  };

  const caller = new SolanaOnchainCaller(builder, sender, fakeUserRepo);
  const r = await caller.sendSettleFill({ ...baseParams, buyerUserId: "unknown-user" });
  expect(r.ok).toBe(false);
  if (!r.ok) {
    expect(r.retryable).toBe(false);
    expect(r.reason).toMatch(/unknown user/);
  }
});
```

- [ ] **Step 8.2: Run.** The first 2 tests should already pass (caller passes through sender's result). The 3rd test validates the exception path in `sendSettleFill` — already handled by existing try/catch.

Expected: 3 new tests pass, 4 existing pass → 7 total.

If any fail: investigate the error propagation path. The existing implementation's `catch (e)` returns `retryable: false` which matches.

- [ ] **Step 8.3: Commit (even if no code change needed):**

```bash
git add api/src/modules/trading-v2/__tests__/solana-onchain-caller.unit.test.ts
git commit -m "test(trading-v2): error classification paths do SolanaOnchainCaller"
```

---

## Task 9: Implementação real do `TransactionSender`

**Goal:** Até agora o `TransactionSender` foi injetado como fake. Agora implementamos o real usando as primitivas existentes em `shared/config/solana.config.ts` (connection, getPlatformSigner).

**Files:**
- Modify: `api/src/modules/trading-v2/services/solana-onchain-caller.service.ts` — adicionar classe `RealTransactionSender`

- [ ] **Step 9.1: Append to the file:**

```typescript
import { Transaction, ComputeBudgetProgram } from "@solana/web3.js";
import { getConnection } from "@/shared/config/solana.config";
import { getPlatformFeeSigner } from "@/shared/services/wallet-signer.service";

/**
 * Envia um array de instructions como um único Transaction Solana.
 * Erros de RPC/network são retryable; program errors e rent errors são não-retryable.
 */
export class RealTransactionSender implements TransactionSender {
  async sendTransaction(
    instructions: TransactionInstruction[],
  ): Promise<{ ok: true; signature: string } | { ok: false; reason: string; retryable: boolean }> {
    try {
      const connection = getConnection();
      const platform = await getPlatformFeeSigner();

      const { blockhash, lastValidBlockHeight } = await connection.getLatestBlockhash();
      const tx = new Transaction({ feePayer: platform.publicKey, blockhash, lastValidBlockHeight });
      // Compute budget bump — per-trade tx touches many accounts
      tx.add(ComputeBudgetProgram.setComputeUnitLimit({ units: 400_000 }));
      instructions.forEach(ix => tx.add(ix));

      tx.sign(platform);
      const sig = await connection.sendRawTransaction(tx.serialize(), { skipPreflight: false });
      await connection.confirmTransaction({ signature: sig, blockhash, lastValidBlockHeight }, "confirmed");
      return { ok: true, signature: sig };
    } catch (e) {
      return classifyError(e);
    }
  }
}

function classifyError(e: unknown): { ok: false; reason: string; retryable: boolean } {
  const msg = e instanceof Error ? e.message : String(e);

  // Retryable: transient network / RPC state
  const retryablePatterns = [
    /blockhash not found/i,
    /block height exceeded/i,
    /transaction expired/i,
    /rpc .*timeout/i,
    /fetch failed/i,
    /econnreset|enotfound|etimedout/i,
  ];
  if (retryablePatterns.some(p => p.test(msg))) {
    return { ok: false, reason: msg, retryable: true };
  }

  // Non-retryable: program errors, simulation failures, etc.
  return { ok: false, reason: msg, retryable: false };
}
```

- [ ] **Step 9.2: Quick sanity — tsc clean.**

```bash
bun x tsc --noEmit 2>&1 | grep solana-onchain-caller || echo "clean"
```

- [ ] **Step 9.3: Commit:**

```bash
git add api/src/modules/trading-v2/services/solana-onchain-caller.service.ts
git commit -m "feat(trading-v2): RealTransactionSender com classificação de erro"
```

**Nota:** `RealTransactionSender` não tem teste unit — ele é um adapter fino ao redor da infra Solana. O teste real dele acontece em `solana-onchain-caller.devnet.test.ts` (Task 10).

---

## Task 10: Teste de integração em devnet (opt-in)

**Goal:** Validar end-to-end: SolanaOnchainCaller → RealTransactionSender → programa Solana real em devnet. Teste skippable por default (não bloqueia CI), executável manual com env var.

**Files:**
- Create: `api/src/modules/trading-v2/__tests__/solana-onchain-caller.devnet.test.ts`

- [ ] **Step 10.1: Criar o teste:**

```typescript
import "dotenv/config";
import { prisma } from "../../../shared/database/config/prisma-client";
import { SolanaOnchainCaller, RealTransactionSender } from "../services/solana-onchain-caller.service";
import * as legacyBuilder from "../services/legacy-clob-instruction-builder";
import { UNIT } from "../types/balance.types";

// Skip por default. Pra rodar:
//   SKIP_DEVNET_TESTS=false \
//   TEST_DEVNET_MARKET_PDA=... \
//   TEST_DEVNET_BUYER_USER_ID=... \
//   TEST_DEVNET_SELLER_USER_ID=... \
//   bun x jest src/modules/trading-v2/__tests__/solana-onchain-caller.devnet.test.ts
const skip = process.env.SKIP_DEVNET_TESTS !== "false";

const d = skip ? describe.skip : describe;

// Minimal UserWalletLookup backed by Prisma user table
const userLookup = {
  async getWalletAddress(userId: string): Promise<string> {
    const u = await prisma.user.findUnique({ where: { id: userId }, select: { walletAddress: true } });
    if (!u?.walletAddress) throw new Error(`unknown user ${userId}`);
    return u.walletAddress;
  },
};

d("SolanaOnchainCaller devnet integration", () => {
  afterAll(async () => { await prisma.$disconnect(); });

  test("TRADE primitive on devnet returns confirmed signature", async () => {
    const marketPda = process.env.TEST_DEVNET_MARKET_PDA!;
    const buyer  = process.env.TEST_DEVNET_BUYER_USER_ID!;
    const seller = process.env.TEST_DEVNET_SELLER_USER_ID!;
    expect(marketPda).toBeTruthy();
    expect(buyer).toBeTruthy();
    expect(seller).toBeTruthy();

    const caller = new SolanaOnchainCaller(legacyBuilder, new RealTransactionSender(), userLookup);
    const r = await caller.sendSettleFill({
      tradeId: `devnet-test-${Date.now()}`,
      marketPda,
      buyerUserId: buyer, sellerUserId: seller,
      buyerReservationAsset: "USDC",
      sellerReservationAsset: "YES",
      priceBps: 5000,
      quantityMicro: 1n * UNIT,     // 1 token humano
      primitive: "TRADE",
    });

    if (!r.ok) console.error("Devnet settle failed:", r);
    expect(r.ok).toBe(true);
    if (r.ok) expect(r.signature).toMatch(/^[1-9A-HJ-NP-Za-km-z]+$/);   // base58
  }, 60_000);
});
```

- [ ] **Step 10.2: Confirm the test is skipped by default:**

```bash
bun x jest src/modules/trading-v2/__tests__/solana-onchain-caller.devnet.test.ts --runInBand 2>&1 | tail -5
```

Expected: `Tests: 1 skipped`.

- [ ] **Step 10.3: Commit:**

```bash
git add api/src/modules/trading-v2/__tests__/solana-onchain-caller.devnet.test.ts
git commit -m "test(trading-v2): devnet integration test (opt-in via SKIP_DEVNET_TESTS=false)"
```

**Manual execution pre-merge:** RECOMENDADO rodar o devnet test uma vez antes de mergear Plano 4, com um market devnet válido. Se falhar, não mergear — o caller não funciona contra o programa real e o Plano é vacuo.

---

## Task 11: Wire real caller nas routes + barrel + README

**Files:**
- Modify: `api/src/modules/trading-v2/routes/orders.routes.ts`
- Modify: `api/src/modules/trading-v2/index.ts`
- Modify: `api/src/modules/trading-v2/README.md`
- Delete: `api/src/modules/trading-v2/services/INVESTIGATION-plano4.md`

- [ ] **Step 11.1: Update `routes/orders.routes.ts`:**

Substituir:
```typescript
const caller = new MockOnchainCaller({ mode: "success" });
```

Por:
```typescript
import { SolanaOnchainCaller, RealTransactionSender } from "../services/solana-onchain-caller.service";
import * as legacyBuilder from "../services/legacy-clob-instruction-builder";
import { userRepository } from "@/modules/identity/repositories/user.repository";

const userLookup = {
  async getWalletAddress(userId: string): Promise<string> {
    const u = await userRepository.findById(userId);
    if (!u?.walletAddress) throw new Error(`user ${userId} has no wallet`);
    return u.walletAddress;
  },
};

const caller = new SolanaOnchainCaller(legacyBuilder, new RealTransactionSender(), userLookup);
```

**Verificar** que `userRepository.findById` retorna um objeto com `walletAddress`. Se o campo se chamar diferente no projeto (ex.: `wallet.address` aninhado), ajustar. Ver `src/modules/identity/repositories/user.repository.ts`.

- [ ] **Step 11.2: Remover import de `MockOnchainCaller` se não usado mais** (pode ficar no barrel pra uso em testes). Confirmar tsc:

```bash
bun x tsc --noEmit 2>&1 | grep trading-v2 || echo "clean"
```

- [ ] **Step 11.3: Barrel — append ao `index.ts`:**

```typescript
export { SolanaOnchainCaller, RealTransactionSender } from "./services/solana-onchain-caller.service";
export type { InstructionBuilder, TransactionSender, UserWalletLookup } from "./services/solana-onchain-caller.service";
```

- [ ] **Step 11.4: Atualizar README.** Adicionar seção "Solana caller" depois de "Settlement":

```markdown
## Solana caller

`SolanaOnchainCaller` (Plano 4) implementa `IOnchainCaller` reusando as
instruções `settle_clob` e `settle_clob_sell` do programa atual — zero
mudanças em Rust. Compõe 2 instructions por trade num único `Transaction`,
garantindo atomicidade.

Mapping primitivo → instruções on-chain:
- **TRADE**: buy-leg(buyer, side) + sell-leg(seller, side), where side depends
  on reservation assets (USDC×YES → "yes", NO×USDC → "no").
- **MINT**: 2× buy-leg with opposite sides (buyer gets YES, seller opens NO short).
- **MERGE**: 2× sell-leg with opposite sides (vault burns pair, both get USDC).

`RealTransactionSender` classifica erros Solana em retryable (blockhash
expired, RPC timeout) vs non-retryable (program errors, simulation failures).
O retry policy vive no `SolanaSettlerService` (Plano 3).

### Teste de devnet

Integração opt-in em `__tests__/solana-onchain-caller.devnet.test.ts`:

```bash
SKIP_DEVNET_TESTS=false \
TEST_DEVNET_MARKET_PDA=... \
TEST_DEVNET_BUYER_USER_ID=... \
TEST_DEVNET_SELLER_USER_ID=... \
bun x jest src/modules/trading-v2/__tests__/solana-onchain-caller.devnet.test.ts
```
```

- [ ] **Step 11.5: Delete INVESTIGATION doc:**

```bash
rm api/src/modules/trading-v2/services/INVESTIGATION-plano4.md
```

- [ ] **Step 11.6: Rodar full suite.** Deve continuar verde — a única diferença runtime é que `orders.routes.ts` agora usa caller real em vez de mock, mas as routes não são exercitadas pelos testes de trading-v2 (que usam Stub/SolanaSettler injetados diretamente).

```bash
bun x jest src/modules/trading-v2 --runInBand
```

Expected: 96 pre-Plano-4 + 7 novos unit tests do caller (Tasks 5+6+7+8) + 1 skipped devnet = 103 passed, 1 skipped. Report actual.

- [ ] **Step 11.7: Commit:**

```bash
git add api/src/modules/trading-v2/routes/orders.routes.ts \
        api/src/modules/trading-v2/index.ts \
        api/src/modules/trading-v2/README.md
git rm api/src/modules/trading-v2/services/INVESTIGATION-plano4.md
git commit -m "feat(trading-v2): rotas usam SolanaOnchainCaller real + docs"
```

---

## Critérios de aceitação

1. ✅ `clobSettleService.buildInstruction()` e `clobSettleSellService.buildInstruction()` públicos e testados (via testes legacy pré-existentes).
2. ✅ `SolanaOnchainCaller` mapeia TRADE / MINT / MERGE corretamente em composição de 2 instructions.
3. ✅ Erros do sender propagam com `retryable` correto; exceções internas caem como non-retryable.
4. ✅ `RealTransactionSender` envia atomic tx e classifica erros comuns Solana.
5. ✅ Devnet test presente, opt-in, passou pelo menos uma execução manual antes do merge.
6. ✅ Routes HTTP usam o caller real; `MockOnchainCaller` fica disponível só pra testes.
7. ✅ Full trading-v2 suite verde (~103 passed + 1 skipped).

---

## Riscos e mitigações

| Risco | Mitigação |
|---|---|
| `settle_clob(BuyNo, _, execPrice)` semântica de preço difere do que assumimos (price do NO vs YES) | Task 1 investigação obriga resolver isso antes de escrever código. Task 6 test captura expectativa; se errada, falha explícita. |
| Legacy `settle()` faz side effects no DB (user_positions, cache) que vão acontecer DUAS vezes quando chamamos buildInstruction() de cada leg separadamente | **Não acontece**: `buildInstruction` é a parte PURA extraída; side effects ficam só no `settle()` legacy que não é chamado pelo novo caller. |
| Legacy builder espera campos (tradeId, userId) que v2 não tem | `LegParams` no wrapper Task 4 documenta o mínimo obrigatório. Campos opcionais do legacy passam como undefined e o builder tolera. Se não tolerar, adicionar default sentinel no wrapper. |
| Devnet test falha porque o ambiente local não tem keypairs de teste | Marcado como opt-in, não bloqueia CI. Deve ser rodado manualmente antes do merge. |
| Atomicidade rompe se compute budget estoura na tx composta | `ComputeBudgetProgram.setComputeUnitLimit(400_000)` no sender. Se estourar na prática, subir pra 600_000. |
| `sendAndConfirmTransaction` no sender difere do `confirmTransaction` usado no código legacy (commitment level) | Sender usa `"confirmed"` — matches o que taker request quer (30s SLA). Se o legacy usa `"finalized"` e precisa ser compatível, revisar. |

---

## O que NÃO está neste plano

- **Listener on-chain** (observador de eventos emitidos pelo programa, idempotência via `ob2_onchain_events_processed`): Plano 5.
- **Fee ledger real**: fee continua sendo ignorado pelo settler. Integração com o fee ledger existente vem depois.
- **Remoção do MockOnchainCaller**: fica disponível pros testes unit de `SolanaSettler` (que já usam).
- **Deprecar totalmente `prediction-market/trading/services/order/clob-settle*`**: o `settle()` legacy continua funcionando pro módulo legacy; removemos apenas quando o módulo inteiro for deprecado no cutover (Plano 8).
