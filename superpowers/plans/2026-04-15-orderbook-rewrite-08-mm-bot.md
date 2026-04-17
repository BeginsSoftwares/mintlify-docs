# Orderbook Rewrite — Plano 8: MM Bot Externo

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Extrair a lógica de Market Maker pra um bot standalone que opera exclusivamente pela API HTTP v2 — zero imports internos, zero acesso direto a DB/Redis/orderbook. O engine não sabe que o MM existe. Se o bot cai, o book continua funcionando sem liquidez sintética.

**Architecture:** Novo diretório `api/src/services/mm-bot/` com um CLI executável via `bun run`. O bot usa um `TradingApiClient` (thin HTTP wrapper) pra ler o book e postar/cancelar ordens. `PricingStrategy` computa mid-price (da última trade via API), spread e levels. `ReplenishLoop` é o core: a cada intervalo lê o estado, compara com target, posta novas ordens e cancela obsoletas. Config via env vars.

**Tech Stack:** Bun, TypeScript, `fetch` nativo (sem deps extras). Jest pra testes com mock de HTTP.

**Spec:** `docs/superpowers/specs/2026-04-15-orderbook-rewrite-design.md` (§9 MM externo)
**Planos anteriores:** Planos 1-7 merged. 149 testes verdes.

**Próximos planos:**
- Plano 9: Cutover

**Não-escopo:**
- **Migração de dados MM legados**: o bot novo começa do zero; ordens do MM antigo são canceladas no cutover (Plano 9).
- **AMM on-chain**: descartado no brainstorming (decisão pilar 5).
- **Multi-market simultâneo no mesmo processo**: suportado via env `MM_MARKET_PDAS=pda1,pda2,...` mas cada market roda sequencialmente no loop. Paralelismo fica pro futuro se necessário.

---

## File Structure

```
api/src/services/mm-bot/
  types.ts                        # CREATE: config, API shapes, target order
  trading-api-client.ts           # CREATE: HTTP wrapper (GET book, POST order, DELETE order, GET trades)
  pricing-strategy.ts             # CREATE: mid-price, spread, level generation
  replenish-loop.ts               # CREATE: diff current vs target, place/cancel
  index.ts                        # CREATE: CLI entry (reads env, runs loop)
  __tests__/
    pricing-strategy.unit.test.ts
    replenish-loop.unit.test.ts
    trading-api-client.unit.test.ts

api/package.json                  # MODIFY: add script:mm-bot alias
```

**Zero imports de `modules/trading-v2/` ou `modules/prediction-market/`**. O bot é um consumidor externo da API HTTP.

---

## Prerequisite check

- [ ] **Step 0: Baseline.**

```bash
cd /Users/rafael/Documents/GitHub/Oraculo/api
bun x jest src/modules/trading-v2 --runInBand
```

Expected: 149 passed + 1 skipped.

- [ ] **Step 0.1: Worktree.**

```bash
git worktree add .claude/worktrees/orderbook-rewrite-08-mm-bot -b worktree-orderbook-rewrite-08-mm-bot
cp api/.env .claude/worktrees/orderbook-rewrite-08-mm-bot/api/.env
cd .claude/worktrees/orderbook-rewrite-08-mm-bot/api
bun install
```

---

## Task 1: Types + config

**Files:**
- Create: `api/src/services/mm-bot/types.ts`

- [ ] **Step 1.1: Create types file.**

```typescript
/**
 * MM Bot types. No imports from trading-v2 or prediction-market modules.
 * All shapes are API-surface contracts (what the HTTP endpoints return/accept).
 */

/** Config loaded from env vars. */
export interface MmBotConfig {
  /** Base URL of the trading v2 API. E.g., "http://localhost:3000" */
  apiBaseUrl: string;
  /** Auth token (Clerk JWT or service account token) for placing orders. */
  authToken: string;
  /** The MM's userId (used in API calls). */
  userId: string;
  /** Comma-separated list of market PDAs to provide liquidity for. */
  marketPdas: string[];
  /** Fee bps to include in orders. */
  feeBps: number;
  /** Number of price levels on each side (bid + ask). */
  levels: number;
  /** Total spread in basis points (e.g., 200 = 2 cent spread at priceBps scale). */
  spreadBps: number;
  /** Max USDC budget per market (micro-units). */
  maxBudgetMicro: bigint;
  /** Replenish interval in milliseconds. */
  intervalMs: number;
}

/** Shape returned by GET /api/v2/trading/orders?marketPda=X&status=OPEN */
export interface ApiOrder {
  id: string;
  userId: string;
  marketPda: string;
  side: "BUY" | "SELL";
  priceBps: number;
  quantity: string;       // bigint serialized
  filled: string;
  status: string;
  clientOrderId: string | null;
  createdAt: string;
  closedAt: string | null;
}

/** Shape returned by POST /api/v2/trading/orders */
export interface ApiPlaceResult {
  order: ApiOrder;
  trades: Array<{
    id: string;
    priceBps: number;
    quantity: string;
    primitive: string;
    status: string;
  }>;
}

/** Shape of a book level from BookSnapshotService (if exposed via HTTP). */
export interface BookLevel {
  priceBps: number;
  quantity: string;
}

/** Shape for the book snapshot (may come from WS or a future HTTP endpoint). */
export interface BookSnapshot {
  bids: BookLevel[];
  asks: BookLevel[];
}

/** A target order the MM wants to have resting in the book. */
export interface TargetOrder {
  side: "BUY" | "SELL";
  priceBps: number;
  quantityMicro: bigint;
}

/** Diff result: what needs to change to match the target. */
export interface ReplenishPlan {
  toCancel: string[];        // order IDs to cancel
  toPlace: TargetOrder[];    // new orders to place
}

export function loadConfig(): MmBotConfig {
  const required = (key: string): string => {
    const v = process.env[key];
    if (!v) throw new Error(`missing env var: ${key}`);
    return v;
  };
  return {
    apiBaseUrl: required("MM_API_BASE_URL"),
    authToken: required("MM_AUTH_TOKEN"),
    userId: required("MM_USER_ID"),
    marketPdas: required("MM_MARKET_PDAS").split(",").map(s => s.trim()).filter(Boolean),
    feeBps: parseInt(process.env.MM_FEE_BPS ?? "50", 10),
    levels: parseInt(process.env.MM_LEVELS ?? "5", 10),
    spreadBps: parseInt(process.env.MM_SPREAD_BPS ?? "200", 10),
    maxBudgetMicro: BigInt(process.env.MM_MAX_BUDGET_MICRO ?? "10000000000"),  // 10k USDC default
    intervalMs: parseInt(process.env.MM_INTERVAL_MS ?? "10000", 10),
  };
}
```

- [ ] **Step 1.2: Commit.**

```bash
cd /Users/rafael/Documents/GitHub/Oraculo/.claude/worktrees/orderbook-rewrite-08-mm-bot
git add api/src/services/mm-bot/types.ts
git commit -m "feat(mm-bot): types + config loader"
```

---

## Task 2: TradingApiClient

**Files:**
- Create: `api/src/services/mm-bot/trading-api-client.ts`
- Create: `api/src/services/mm-bot/__tests__/trading-api-client.unit.test.ts`

Thin HTTP wrapper. Uses native `fetch`. All methods return typed results or throw on non-2xx.

- [ ] **Step 2.1: Failing tests.**

```typescript
import { TradingApiClient } from "../trading-api-client";
import type { ApiOrder, ApiPlaceResult } from "../types";

// Mock global fetch
const mockFetch = jest.fn();
global.fetch = mockFetch as any;

const client = new TradingApiClient({
  baseUrl: "http://localhost:3000",
  authToken: "test-token",
});

beforeEach(() => { mockFetch.mockReset(); });

test("getOpenOrders calls GET /api/v2/trading/orders with correct params and auth", async () => {
  const fakeOrders: ApiOrder[] = [
    { id: "o1", userId: "u1", marketPda: "M1", side: "BUY", priceBps: 5000,
      quantity: "100", filled: "0", status: "OPEN", clientOrderId: null,
      createdAt: "2026-01-01T00:00:00Z", closedAt: null },
  ];
  mockFetch.mockResolvedValueOnce({
    ok: true,
    json: async () => ({ success: true, data: fakeOrders }),
  });

  const result = await client.getOpenOrders("M1");

  expect(mockFetch).toHaveBeenCalledTimes(1);
  const [url, opts] = mockFetch.mock.calls[0];
  expect(url).toBe("http://localhost:3000/api/v2/trading/orders?marketPda=M1&status=OPEN");
  expect(opts.headers.Authorization).toBe("Bearer test-token");
  expect(result).toEqual(fakeOrders);
});

test("placeOrder calls POST /api/v2/trading/orders with body and auth", async () => {
  const fakeResult: ApiPlaceResult = {
    order: { id: "o2", userId: "u1", marketPda: "M1", side: "BUY", priceBps: 5000,
      quantity: "100", filled: "0", status: "OPEN", clientOrderId: "mm-1",
      createdAt: "2026-01-01T00:00:00Z", closedAt: null },
    trades: [],
  };
  mockFetch.mockResolvedValueOnce({
    ok: true,
    json: async () => ({ success: true, data: fakeResult }),
  });

  const result = await client.placeOrder({
    marketPda: "M1", side: "BUY", priceBps: 5000,
    quantity: "100000000", feeBps: 50,
    clientOrderId: "mm-1",
  });

  expect(mockFetch).toHaveBeenCalledTimes(1);
  const [url, opts] = mockFetch.mock.calls[0];
  expect(url).toBe("http://localhost:3000/api/v2/trading/orders");
  expect(opts.method).toBe("POST");
  expect(JSON.parse(opts.body)).toEqual({
    marketPda: "M1", side: "BUY", priceBps: 5000,
    quantity: "100000000", feeBps: 50,
    clientOrderId: "mm-1",
  });
  expect(result).toEqual(fakeResult);
});

test("cancelOrder calls DELETE /api/v2/trading/orders/:id with auth", async () => {
  mockFetch.mockResolvedValueOnce({
    ok: true,
    json: async () => ({ success: true, data: {} }),
  });

  await client.cancelOrder("o3");

  expect(mockFetch).toHaveBeenCalledTimes(1);
  const [url, opts] = mockFetch.mock.calls[0];
  expect(url).toBe("http://localhost:3000/api/v2/trading/orders/o3");
  expect(opts.method).toBe("DELETE");
});

test("throws on non-2xx response", async () => {
  mockFetch.mockResolvedValueOnce({
    ok: false,
    status: 400,
    text: async () => "Bad Request",
  });

  await expect(client.getOpenOrders("M1")).rejects.toThrow(/400/);
});
```

- [ ] **Step 2.2: Run, fail.**

```bash
cd /Users/rafael/Documents/GitHub/Oraculo/.claude/worktrees/orderbook-rewrite-08-mm-bot/api
bun x jest src/services/mm-bot/__tests__/trading-api-client.unit.test.ts --runInBand
```

- [ ] **Step 2.3: Implement.**

```typescript
import type { ApiOrder, ApiPlaceResult } from "./types";

export interface TradingApiClientConfig {
  baseUrl: string;
  authToken: string;
}

export interface PlaceOrderInput {
  marketPda: string;
  side: "BUY" | "SELL";
  priceBps: number;
  quantity: string;        // bigint as string (micro-units)
  feeBps: number;
  clientOrderId?: string;
}

export class TradingApiClient {
  private readonly base: string;
  private readonly headers: Record<string, string>;

  constructor(config: TradingApiClientConfig) {
    this.base = config.baseUrl.replace(/\/$/, "");
    this.headers = {
      "Content-Type": "application/json",
      "Authorization": `Bearer ${config.authToken}`,
    };
  }

  async getOpenOrders(marketPda: string): Promise<ApiOrder[]> {
    const url = `${this.base}/api/v2/trading/orders?marketPda=${encodeURIComponent(marketPda)}&status=OPEN`;
    const res = await fetch(url, { headers: this.headers });
    if (!res.ok) throw new Error(`GET orders failed: ${res.status} ${await res.text()}`);
    const body = await res.json();
    return body.data ?? body;
  }

  async placeOrder(input: PlaceOrderInput): Promise<ApiPlaceResult> {
    const url = `${this.base}/api/v2/trading/orders`;
    const res = await fetch(url, {
      method: "POST",
      headers: this.headers,
      body: JSON.stringify(input),
    });
    if (!res.ok) throw new Error(`POST order failed: ${res.status} ${await res.text()}`);
    const body = await res.json();
    return body.data ?? body;
  }

  async cancelOrder(orderId: string): Promise<void> {
    const url = `${this.base}/api/v2/trading/orders/${orderId}`;
    const res = await fetch(url, {
      method: "DELETE",
      headers: this.headers,
    });
    if (!res.ok) throw new Error(`DELETE order failed: ${res.status} ${await res.text()}`);
  }
}
```

- [ ] **Step 2.4: Run, 4 tests pass.**

- [ ] **Step 2.5: Commit.**

```bash
git add api/src/services/mm-bot/trading-api-client.ts \
        api/src/services/mm-bot/__tests__/trading-api-client.unit.test.ts
git commit -m "feat(mm-bot): TradingApiClient (HTTP wrapper)"
```

---

## Task 3: PricingStrategy

**Files:**
- Create: `api/src/services/mm-bot/pricing-strategy.ts`
- Create: `api/src/services/mm-bot/__tests__/pricing-strategy.unit.test.ts`

Pure function. Given mid-price (bps), spread (bps), levels count, max budget, computes an array of `TargetOrder[]` — symmetric bids and asks around mid.

- [ ] **Step 3.1: Failing tests.**

```typescript
import { PricingStrategy } from "../pricing-strategy";
import type { TargetOrder } from "../types";

const strategy = new PricingStrategy();

test("generates symmetric bids and asks around midBps", () => {
  const targets = strategy.computeTargets({
    midBps: 5000,          // 50 cents
    spreadBps: 200,        // 2 cent spread → half = 1 cent each side
    levels: 3,
    maxBudgetMicro: 100_000_000_000n,  // 100k USDC — should not be limiting
    feeBps: 50,
  });

  // 3 bids: 4900, 4800, 4700 (descending from mid - halfSpread)
  // 3 asks: 5100, 5200, 5300 (ascending from mid + halfSpread)
  const bidPrices = targets.filter(t => t.side === "BUY").map(t => t.priceBps);
  const askPrices = targets.filter(t => t.side === "SELL").map(t => t.priceBps);

  expect(bidPrices).toEqual([4900, 4800, 4700]);
  expect(askPrices).toEqual([5100, 5200, 5300]);
  expect(targets).toHaveLength(6);
});

test("each level has equal USDC allocation", () => {
  const targets = strategy.computeTargets({
    midBps: 5000,
    spreadBps: 200,
    levels: 2,
    maxBudgetMicro: 10_000_000n,   // 10 USDC
    feeBps: 0,
  });

  // Budget per level = 10 USDC / (2 levels × 2 sides) = 2.5 USDC per level
  // All targets should have quantity * price/10000 ≈ 2.5 USDC
  for (const t of targets) {
    const cost = (t.quantityMicro * BigInt(t.priceBps)) / 10000n;
    expect(cost).toBeLessThanOrEqual(2_500_000n + 1n);  // ≤ 2.5 USDC + 1 micro rounding
    expect(cost).toBeGreaterThan(0n);
  }
});

test("prices clamped to (0, 10000) range", () => {
  // Mid at 200 bps, spread 400 → would go below 0 without clamp
  const targets = strategy.computeTargets({
    midBps: 200,
    spreadBps: 400,
    levels: 3,
    maxBudgetMicro: 100_000_000_000n,
    feeBps: 0,
  });

  for (const t of targets) {
    expect(t.priceBps).toBeGreaterThan(0);
    expect(t.priceBps).toBeLessThan(10000);
  }
});

test("returns empty array when midBps is 0 or ≥ 10000", () => {
  expect(strategy.computeTargets({ midBps: 0, spreadBps: 200, levels: 3, maxBudgetMicro: 1n, feeBps: 0 })).toEqual([]);
  expect(strategy.computeTargets({ midBps: 10000, spreadBps: 200, levels: 3, maxBudgetMicro: 1n, feeBps: 0 })).toEqual([]);
});

test("feeBps is included in the per-level budget", () => {
  const withFee = strategy.computeTargets({
    midBps: 5000, spreadBps: 200, levels: 1,
    maxBudgetMicro: 10_000_000n, feeBps: 500,  // 5% fee
  });
  const noFee = strategy.computeTargets({
    midBps: 5000, spreadBps: 200, levels: 1,
    maxBudgetMicro: 10_000_000n, feeBps: 0,
  });
  // With fee, quantity should be smaller (budget includes fee)
  const qtyWithFee = withFee.find(t => t.side === "BUY")!.quantityMicro;
  const qtyNoFee   = noFee.find(t => t.side === "BUY")!.quantityMicro;
  expect(qtyWithFee).toBeLessThan(qtyNoFee);
});
```

- [ ] **Step 3.2: Run, fail.**

- [ ] **Step 3.3: Implement.**

```typescript
import type { TargetOrder } from "./types";

export interface PricingInput {
  midBps: number;            // 1..9999
  spreadBps: number;         // total spread in bps
  levels: number;            // levels per side
  maxBudgetMicro: bigint;    // total USDC budget for all levels both sides
  feeBps: number;            // fee included in per-level cost
}

export class PricingStrategy {
  computeTargets(input: PricingInput): TargetOrder[] {
    if (input.midBps <= 0 || input.midBps >= 10000) return [];
    if (input.levels <= 0) return [];

    const halfSpread = Math.ceil(input.spreadBps / 2);
    const totalSlots = input.levels * 2;
    const budgetPerSlot = input.maxBudgetMicro / BigInt(totalSlots);
    const targets: TargetOrder[] = [];

    for (let i = 0; i < input.levels; i++) {
      const bidPrice = input.midBps - halfSpread - i * 100;
      const askPrice = input.midBps + halfSpread + i * 100;

      if (bidPrice > 0 && bidPrice < 10000) {
        const qty = this.qtyFromBudget(budgetPerSlot, bidPrice, input.feeBps);
        if (qty > 0n) targets.push({ side: "BUY", priceBps: bidPrice, quantityMicro: qty });
      }
      if (askPrice > 0 && askPrice < 10000) {
        const qty = this.qtyFromBudget(budgetPerSlot, askPrice, input.feeBps);
        if (qty > 0n) targets.push({ side: "SELL", priceBps: askPrice, quantityMicro: qty });
      }
    }

    // Sort bids descending, asks ascending (for readability)
    targets.sort((a, b) => {
      if (a.side !== b.side) return a.side === "BUY" ? -1 : 1;
      return a.side === "BUY" ? b.priceBps - a.priceBps : a.priceBps - b.priceBps;
    });

    return targets;
  }

  /**
   * Given a USDC budget (micro) and a price (bps), compute how many tokens
   * (micro-units) we can afford including fee.
   *
   * For BUY: cost = qty * price/10000 + fee. Invert: qty = budget / (price/10000 * (1 + feeBps/10000))
   * For SELL: similar (the "cost" is in NO-side collateral, but we simplify to same formula
   * since the MM uses USDC for both sides via the IntentClassifier in the API).
   */
  private qtyFromBudget(budgetMicro: bigint, priceBps: number, feeBps: number): bigint {
    // effectivePriceBps = priceBps * (1 + feeBps/10000) = priceBps + priceBps*feeBps/10000
    const effectivePrice = BigInt(priceBps) + (BigInt(priceBps) * BigInt(feeBps)) / 10000n;
    if (effectivePrice <= 0n) return 0n;
    // qty = budget * 10000 / effectivePrice
    return (budgetMicro * 10000n) / effectivePrice;
  }
}
```

- [ ] **Step 3.4: Run, 5 tests pass.**

- [ ] **Step 3.5: Commit.**

```bash
git add api/src/services/mm-bot/pricing-strategy.ts \
        api/src/services/mm-bot/__tests__/pricing-strategy.unit.test.ts
git commit -m "feat(mm-bot): PricingStrategy (symmetric bids/asks + budget allocation)"
```

---

## Task 4: ReplenishLoop (core diff + place/cancel)

**Files:**
- Create: `api/src/services/mm-bot/replenish-loop.ts`
- Create: `api/src/services/mm-bot/__tests__/replenish-loop.unit.test.ts`

Core logic: read current open orders → compute target orders → diff → cancel stale → place new.

- [ ] **Step 4.1: Failing tests.**

```typescript
import { ReplenishLoop } from "../replenish-loop";
import type { ApiOrder, TargetOrder, ReplenishPlan } from "../types";
import type { TradingApiClient, PlaceOrderInput } from "../trading-api-client";

const mkOrder = (overrides: Partial<ApiOrder> = {}): ApiOrder => ({
  id: "o1", userId: "mm", marketPda: "M1", side: "BUY", priceBps: 5000,
  quantity: "1000000", filled: "0", status: "OPEN", clientOrderId: null,
  createdAt: "2026-01-01T00:00:00Z", closedAt: null,
  ...overrides,
});

test("computePlan: no existing orders + targets → place all", () => {
  const loop = new ReplenishLoop();
  const targets: TargetOrder[] = [
    { side: "BUY", priceBps: 4900, quantityMicro: 100_000_000n },
    { side: "SELL", priceBps: 5100, quantityMicro: 100_000_000n },
  ];
  const plan = loop.computePlan([], targets);
  expect(plan.toCancel).toEqual([]);
  expect(plan.toPlace).toEqual(targets);
});

test("computePlan: existing order matches target → keep (no cancel, no place)", () => {
  const loop = new ReplenishLoop();
  const existing: ApiOrder[] = [mkOrder({ side: "BUY", priceBps: 4900, quantity: "100000000" })];
  const targets: TargetOrder[] = [{ side: "BUY", priceBps: 4900, quantityMicro: 100_000_000n }];
  const plan = loop.computePlan(existing, targets);
  expect(plan.toCancel).toEqual([]);
  expect(plan.toPlace).toEqual([]);
});

test("computePlan: existing order at wrong price → cancel + re-place", () => {
  const loop = new ReplenishLoop();
  const existing: ApiOrder[] = [mkOrder({ id: "stale", side: "BUY", priceBps: 4800, quantity: "100000000" })];
  const targets: TargetOrder[] = [{ side: "BUY", priceBps: 4900, quantityMicro: 100_000_000n }];
  const plan = loop.computePlan(existing, targets);
  expect(plan.toCancel).toEqual(["stale"]);
  expect(plan.toPlace).toEqual(targets);
});

test("computePlan: extra existing orders not in targets → cancel them", () => {
  const loop = new ReplenishLoop();
  const existing: ApiOrder[] = [
    mkOrder({ id: "keep", side: "BUY", priceBps: 4900, quantity: "100000000" }),
    mkOrder({ id: "extra", side: "SELL", priceBps: 7000, quantity: "100000000" }),
  ];
  const targets: TargetOrder[] = [{ side: "BUY", priceBps: 4900, quantityMicro: 100_000_000n }];
  const plan = loop.computePlan(existing, targets);
  expect(plan.toCancel).toEqual(["extra"]);
  expect(plan.toPlace).toEqual([]);
});

test("executePlan: calls cancelOrder for each cancel + placeOrder for each place", async () => {
  const loop = new ReplenishLoop();
  const cancelled: string[] = [];
  const placed: PlaceOrderInput[] = [];

  const mockClient = {
    cancelOrder: async (id: string) => { cancelled.push(id); },
    placeOrder: async (input: PlaceOrderInput) => { placed.push(input); return { order: {} as any, trades: [] }; },
    getOpenOrders: async () => [],
  };

  const plan: ReplenishPlan = {
    toCancel: ["o1", "o2"],
    toPlace: [{ side: "BUY", priceBps: 4900, quantityMicro: 100_000_000n }],
  };

  await loop.executePlan(mockClient as any, "M1", plan, 50);

  expect(cancelled).toEqual(["o1", "o2"]);
  expect(placed).toHaveLength(1);
  expect(placed[0].priceBps).toBe(4900);
  expect(placed[0].quantity).toBe("100000000");
  expect(placed[0].feeBps).toBe(50);
  expect(placed[0].side).toBe("BUY");
  expect(placed[0].marketPda).toBe("M1");
});

test("executePlan: cancel failure doesn't block remaining operations", async () => {
  const loop = new ReplenishLoop();
  const placed: PlaceOrderInput[] = [];

  const mockClient = {
    cancelOrder: async () => { throw new Error("not found"); },
    placeOrder: async (input: PlaceOrderInput) => { placed.push(input); return { order: {} as any, trades: [] }; },
    getOpenOrders: async () => [],
  };

  const plan: ReplenishPlan = {
    toCancel: ["stale"],
    toPlace: [{ side: "SELL", priceBps: 5100, quantityMicro: 50_000_000n }],
  };

  await loop.executePlan(mockClient as any, "M1", plan, 0);

  expect(placed).toHaveLength(1); // placed despite cancel failure
});
```

- [ ] **Step 4.2: Run, fail.**

- [ ] **Step 4.3: Implement.**

```typescript
import type { ApiOrder, TargetOrder, ReplenishPlan } from "./types";
import type { TradingApiClient, PlaceOrderInput } from "./trading-api-client";

/**
 * Core MM logic: diff current orders vs targets, produce cancel/place plan,
 * execute via API client. Zero internal imports — only uses public API types.
 */
export class ReplenishLoop {
  /**
   * Compare existing OPEN orders (from API) against desired target orders.
   * An existing order "matches" a target if same side AND same priceBps.
   * Quantity differences are tolerated (we don't re-place for qty drift).
   */
  computePlan(existing: ApiOrder[], targets: TargetOrder[]): ReplenishPlan {
    const matched = new Set<string>(); // IDs of existing orders that match a target
    const targetKeys = new Set<string>(); // "BUY:4900", "SELL:5100"

    for (const t of targets) {
      targetKeys.add(`${t.side}:${t.priceBps}`);
    }

    const toCancel: string[] = [];
    for (const e of existing) {
      const key = `${e.side}:${e.priceBps}`;
      if (targetKeys.has(key)) {
        matched.add(key);
      } else {
        toCancel.push(e.id);
      }
    }

    const toPlace: TargetOrder[] = targets.filter(t => {
      const key = `${t.side}:${t.priceBps}`;
      return !matched.has(key);
    });

    return { toCancel, toPlace };
  }

  async executePlan(
    client: TradingApiClient,
    marketPda: string,
    plan: ReplenishPlan,
    feeBps: number,
  ): Promise<{ cancelled: number; placed: number; errors: string[] }> {
    const errors: string[] = [];
    let cancelled = 0;
    let placed = 0;

    // Cancel first — free up budget
    for (const id of plan.toCancel) {
      try {
        await client.cancelOrder(id);
        cancelled++;
      } catch (e) {
        errors.push(`cancel ${id}: ${e instanceof Error ? e.message : String(e)}`);
      }
    }

    // Then place
    for (const t of plan.toPlace) {
      try {
        const input: PlaceOrderInput = {
          marketPda,
          side: t.side,
          priceBps: t.priceBps,
          quantity: t.quantityMicro.toString(),
          feeBps,
          clientOrderId: `mm-${t.side.toLowerCase()}-${t.priceBps}-${Date.now()}`,
        };
        await client.placeOrder(input);
        placed++;
      } catch (e) {
        errors.push(`place ${t.side}@${t.priceBps}: ${e instanceof Error ? e.message : String(e)}`);
      }
    }

    return { cancelled, placed, errors };
  }
}
```

- [ ] **Step 4.4: Run, 6 tests pass.**

- [ ] **Step 4.5: Commit.**

```bash
git add api/src/services/mm-bot/replenish-loop.ts \
        api/src/services/mm-bot/__tests__/replenish-loop.unit.test.ts
git commit -m "feat(mm-bot): ReplenishLoop (diff + execute via API client)"
```

---

## Task 5: CLI entry point + package.json alias

**Files:**
- Create: `api/src/services/mm-bot/index.ts`
- Modify: `api/package.json`

- [ ] **Step 5.1: Create CLI entry.**

```typescript
/**
 * MM Bot — standalone market maker.
 *
 * Usage:
 *   MM_API_BASE_URL=http://localhost:3000 \
 *   MM_AUTH_TOKEN=... \
 *   MM_USER_ID=... \
 *   MM_MARKET_PDAS=MarketPda1,MarketPda2 \
 *   bun run src/services/mm-bot/index.ts
 *
 * Optional env:
 *   MM_FEE_BPS=50           (default 50)
 *   MM_LEVELS=5             (default 5)
 *   MM_SPREAD_BPS=200       (default 200)
 *   MM_MAX_BUDGET_MICRO=... (default 10000000000 = 10k USDC)
 *   MM_INTERVAL_MS=10000    (default 10s)
 *
 * The bot runs a loop: for each market, it reads open orders via the v2 HTTP API,
 * computes a target set of symmetric bids/asks around last-trade mid-price, diffs
 * current vs target, cancels stale orders, and places new ones. Zero internal
 * imports — uses only the public trading-v2 HTTP API.
 *
 * Mid-price source: last trade price from GET /api/v2/trading/orders (sorted by
 * closedAt desc, first FILLED order's price). If no trades exist, defaults to 5000
 * (50 cents).
 */
import { loadConfig } from "./types";
import { TradingApiClient } from "./trading-api-client";
import { PricingStrategy } from "./pricing-strategy";
import { ReplenishLoop } from "./replenish-loop";

async function getMidPrice(client: TradingApiClient, marketPda: string): Promise<number> {
  // Use existing open orders to infer mid: avg of best bid and best ask.
  // If no orders, default to 5000 (50 cents).
  try {
    const orders = await client.getOpenOrders(marketPda);
    const buys = orders.filter(o => o.side === "BUY").sort((a, b) => b.priceBps - a.priceBps);
    const sells = orders.filter(o => o.side === "SELL").sort((a, b) => a.priceBps - b.priceBps);
    const bestBid = buys[0]?.priceBps;
    const bestAsk = sells[0]?.priceBps;
    if (bestBid && bestAsk) return Math.round((bestBid + bestAsk) / 2);
    if (bestBid) return bestBid;
    if (bestAsk) return bestAsk;
  } catch { /* fall through */ }
  return 5000;
}

async function main() {
  const config = loadConfig();
  const client = new TradingApiClient({ baseUrl: config.apiBaseUrl, authToken: config.authToken });
  const pricing = new PricingStrategy();
  const loop = new ReplenishLoop();

  // eslint-disable-next-line no-console
  console.log(`[mm-bot] starting. markets=${config.marketPdas.join(",")} interval=${config.intervalMs}ms levels=${config.levels} spread=${config.spreadBps}bps`);

  const tick = async () => {
    for (const marketPda of config.marketPdas) {
      try {
        const midBps = await getMidPrice(client, marketPda);
        const targets = pricing.computeTargets({
          midBps,
          spreadBps: config.spreadBps,
          levels: config.levels,
          maxBudgetMicro: config.maxBudgetMicro,
          feeBps: config.feeBps,
        });
        const existing = await client.getOpenOrders(marketPda);
        const mmOrders = existing.filter(o => o.userId === config.userId);
        const plan = loop.computePlan(mmOrders, targets);

        if (plan.toCancel.length === 0 && plan.toPlace.length === 0) {
          // eslint-disable-next-line no-console
          console.log(`[mm-bot] ${marketPda.slice(0,8)}... mid=${midBps} — no changes needed`);
          continue;
        }

        const result = await loop.executePlan(client, marketPda, plan, config.feeBps);
        // eslint-disable-next-line no-console
        console.log(JSON.stringify({
          timestamp: new Date().toISOString(),
          market: marketPda.slice(0, 8),
          mid: midBps,
          cancelled: result.cancelled,
          placed: result.placed,
          errors: result.errors.length > 0 ? result.errors : undefined,
        }));
      } catch (e) {
        // eslint-disable-next-line no-console
        console.error(`[mm-bot] error on ${marketPda.slice(0, 8)}:`, e instanceof Error ? e.message : e);
      }
    }
  };

  // Run once immediately, then loop
  await tick();

  setInterval(async () => {
    try { await tick(); }
    catch (e) { console.error("[mm-bot] tick error:", e); }
  }, config.intervalMs);
}

main().catch(e => {
  console.error("[mm-bot] fatal:", e);
  process.exit(1);
});
```

- [ ] **Step 5.2: Add alias to `api/package.json`.**

Find the `scripts` section. After `"script:ob2-listener":...` add:

```json
    "script:mm-bot": "bun run src/services/mm-bot/index.ts",
```

- [ ] **Step 5.3: Validate JSON.**

```bash
cat package.json | python3 -c "import json,sys;json.load(sys.stdin);print('valid')"
```

- [ ] **Step 5.4: Commit.**

```bash
git add api/src/services/mm-bot/index.ts api/package.json
git commit -m "feat(mm-bot): CLI entry point + package.json alias"
```

---

## Task 6: Full suite + README

**Files:**
- Modify: `api/src/modules/trading-v2/README.md`

- [ ] **Step 6.1: Append to README.md**, after the WebSocket section and before "Próximos planos":

```markdown
## MM Bot Externo (Plano 8)

Market maker como bot standalone em `api/src/services/mm-bot/`. Zero imports internos — opera exclusivamente via API HTTP v2 (`/api/v2/trading/orders`).

### Arquitetura

- `TradingApiClient` — thin HTTP wrapper (GET/POST/DELETE orders).
- `PricingStrategy` — pure: computa bids/asks simétricos em torno de mid-price.
- `ReplenishLoop` — diff current orders vs target, cancela stale, posta novas.
- `index.ts` — CLI executável via `bun run src/services/mm-bot/index.ts`.

### Config (env vars)

| Var | Default | Descrição |
|-----|---------|-----------|
| `MM_API_BASE_URL` | (required) | URL base da API v2 |
| `MM_AUTH_TOKEN` | (required) | JWT Clerk ou service token |
| `MM_USER_ID` | (required) | userId do MM (pra filtrar ordens) |
| `MM_MARKET_PDAS` | (required) | PDAs dos mercados, comma-separated |
| `MM_FEE_BPS` | 50 | Fee em bps |
| `MM_LEVELS` | 5 | Levels de preço por lado |
| `MM_SPREAD_BPS` | 200 | Spread total em bps |
| `MM_MAX_BUDGET_MICRO` | 10000000000 | Budget máximo por mercado (10k USDC) |
| `MM_INTERVAL_MS` | 10000 | Intervalo do loop em ms |

### Rodar

```bash
MM_API_BASE_URL=http://localhost:3000 \
MM_AUTH_TOKEN=... \
MM_USER_ID=... \
MM_MARKET_PDAS=MarketPda1 \
bun run src/services/mm-bot/index.ts
```

### Princípio

Se o bot cai, o book continua funcionando. Sem liquidez sintética, mas sem drift ou bugs de MM. O engine não sabe que o MM existe.
```

### Step 6.2: Run full suites.

MM bot tests:

```bash
bun x jest src/services/mm-bot --runInBand
```

Expected: 15 passed (4 client + 5 pricing + 6 replenish).

Trading-v2 suite (unchanged):

```bash
bun x jest src/modules/trading-v2 --runInBand
```

Expected: 149 passed + 1 skipped.

### Step 6.3: Commit.

```bash
git add api/src/modules/trading-v2/README.md
git commit -m "docs(trading-v2): README MM bot externo (Plano 8)"
```

---

## Critérios de aceitação

1. ✅ `TradingApiClient` — thin HTTP wrapper com tests (mock fetch).
2. ✅ `PricingStrategy` — symmetric bids/asks, budget allocation, price clamping, fee-aware.
3. ✅ `ReplenishLoop` — diff + execute, cancel-first then place, error resilience.
4. ✅ CLI executável via `bun run src/services/mm-bot/index.ts`.
5. ✅ Zero imports de `modules/trading-v2/` ou `modules/prediction-market/`.
6. ✅ Config via env vars com defaults sensatos.
7. ✅ ~15 tests novos (mm-bot) + 149 trading-v2 inalterados.

---

## Riscos e mitigações

| Risco | Mitigação |
|---|---|
| MM bot coloca order e é matchado contra si mesmo | `computePlan` filtra `existing.filter(o => o.userId === config.userId)` — bot vê apenas suas ordens. Se outro bot com mesmo user existir, ambos operam o mesmo set. Se users diferentes → orders cruzam naturalmente (é o comportamento desejado). |
| API retorna 401 por token expirado | Bot loga o erro e retenta no próximo tick. Alerting externo (systemd watchdog, etc.) detecta loops sem sucesso. |
| Mid-price instável (flip-flop) causa cancel/place contínuo | `computePlan` compara por price level: enquanto mid não mudar, target fica igual. Se mid move 1bps, apenas levels afetados são recalculados. |
| Budget esgotado on-chain mas bot tenta postar | API retorna 400 (InsufficientBalance); bot loga e continua no próximo tick. |
| Race entre cancel e place do mesmo nível | Cancel-first garante que budget está livre antes de postar. Se cancel falha (já executou), bot ignora e posta novo — worst case é um extra order que será cancelado no próximo tick. |

---

## O que NÃO está neste plano

- **Multi-process parallelismo**: cada market roda sequencialmente no loop. Fork pra paralelo é simples mas não necessário pro volume atual.
- **WS feed pro mid-price**: bot usa HTTP polling. WS subscription seria mais eficiente mas adiciona complexidade de reconexão — deferido.
- **Cleanup de ordens legacy**: o cutover (Plano 9) cancela as ordens do MM antigo. Bot novo só opera com orders v2.
- **Profit/loss tracking**: bot é provedor de liquidez puro. P&L analytics é consumer externo do fee ledger.
