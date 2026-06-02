# Task 15 (re-escopada) — Migrar read-paths off `free`/`reserved`, depois dropar colunas

> **For agentic workers:** REQUIRED SUB-SKILL: superpowers:subagent-driven-development. TDD em cada sub-task. Steps com checkbox.

**Contexto:** o motor chain-as-truth (Tasks 0–14, 11–13) está pronto e NÃO usa `free`/`reserved`. Mas alguns read-paths ainda leem essas colunas — e como o motor não as escreve mais, **já mostram dado stale**. Antes de dropar `free`/`reserved`/`Ob2Reservation`, migrar esses read-paths pro modelo derivado (`available`/`confirmedOnchain`).

**Pré-requisito:** DB local de teste em `localhost:5434` (container `oraculo-pg-test`). Rodar testes serial (`--runInBand`). Não rodar contra prod.

**Helper central já pronto:** `computeAvailable({confirmedMicro, pendingDeltaMicro, openCommitmentMicro})` + os repos `BalanceMirrorRepository.getConfirmedMicro`, `SettlementDeltaRepository.sumPendingMicro`, e a soma de `commit_amount` de ordens OPEN. Considere extrair um `AvailableBalanceService.available(userId, asset, marketScope)` que encapsula os 3 termos — os 4 read-paths abaixo reusam ele (DRY).

---

## Task 15a — `AvailableBalanceService` (encapsula a fórmula para os read-paths)
**Files:** Create `api/src/modules/trading-v2/services/available-balance.service.ts` (+ `__tests__/available-balance.service.integration.test.ts`).
- [ ] Teste: seed mirror + ordem OPEN + delta SETTLING → `available()` retorna `confirmed + pending − openCommit`. E `availableAll(userId, marketPda)` retorna `{usdc, yes, no}`.
- [ ] Implementar `available(userId, asset, marketScope)` reusando `BalanceMirrorRepository`/`SettlementDeltaRepository`/`computeAvailable` + um `SELECT SUM(commit_amount) ... status='OPEN' AND commit_asset=$asset AND (marketScope null→USDC global, senão market_pda)`. `availableAll` = 3 chamadas.
- [ ] Verde + commit.

## Task 15b — Feed do classifier em place-order
**Files:** Modify `api/src/modules/trading-v2/use-cases/place-order.use-case.ts` (linhas ~124-132).
- [ ] Teste (integração): com saldo só em `onchainTotal` (sem `free`), place-order classifica corretamente o `commitAsset` (ex.: SELL com YES confirmado → commitAsset YES; sem YES → USDC).
- [ ] Trocar `this.balances.getAll(...)` (que lê `free`) por `availableBalanceService.availableAll(userId, marketPda)`; passar `freeYes/freeNo/freeUsdc` = os `available` derivados ao `classifier.classify`. Injetar `AvailableBalanceService` no construtor (e no wiring em `orders.routes.ts`).
- [ ] Verde (rodar place-order + conservation + matching-deltas) + commit.

## Task 15c — `reconciliation.service.compare` + `daily-reconciliation`
**Files:** Modify `api/src/modules/trading-v2/services/reconciliation.service.ts` e `daily-reconciliation.service.ts`.
- [ ] Teste: `compare` detecta divergência entre o `onchainTotal` ANTERIOR (espelho) e o valor fresco lido da chain (reader), sem usar `free`/`reserved`.
- [ ] Reescrever `compare({ priorOnchainMicro, chainMicro })` (em vez de `{free, reserved, onchainTotal}`). Em `daily-reconciliation.reconcile`, ler o `onchainTotal` atual ANTES de sobrescrever, comparar com o valor do reader, registrar drift, então sobrescrever. `buildReconcileAlert` continua funcionando (usa `drifts`).
- [ ] Verde (reconcile-alert-wiring + daily-reconciliation tests) + commit.

## Task 15d — Display de saldo nos endpoints
**Files:** Modify `api/src/modules/trading-v2/routes/orders.routes.ts` (linhas ~404, 557-563, 587-597, 797).
- [ ] Definir semântica nova: `totalBalance` = `confirmedOnchain` (USDC); `availableBalance` = `available()`; `lockedBalance` = `Σ openOrderCommitments` (= confirmed − available). Remover o endpoint/branch de "sync free" (linhas ~587-597) — não há mais `free` pra sincronizar; o espelho é atualizado pela reconciliação.
- [ ] Trocar `Number(b.free)+Number(b.reserved)` por leitura do `onchainTotal`/`available` via `AvailableBalanceService`. Ajustar os 4 pontos.
- [ ] Teste de rota (se houver harness de rota) ou ao menos tsc + smoke manual. Commit.

## Task 15e — Tipos + repositórios param de ler/escrever `free`/`reserved`
**Files:** `types/balance.types.ts`, `repositories/balance.repository.ts`, `repositories/user-balance.repository.ts`, `repositories/balance-mirror.repository.ts`, `services/balance.service.ts`.
- [ ] Remover `free`/`reserved` de `MarketBalance`/`UserBalance` types. Remover `checkInvariantI2` de `BalanceService` (invariante do modelo antigo). Repos: parar de selecionar/escrever `free`/`reserved`; `upsertOnchain` seta só `onchainTotal`/`onchainSlot`; `balance-mirror.create` sem `free`/`reserved`. Harness `seedConfirmedUsdc/Token` sem `free`/`reserved`.
- [ ] tsc limpo no trading-v2 + suite serial verde. Commit.

## Task 15f — Drop no schema + db push
**Files:** `prisma/schema.prisma`.
- [ ] Remover `model Ob2Reservation`, `Ob2Order.reservationId`, e as colunas `free`/`reserved` de `Ob2UserBalance`/`Ob2UserMarketBalance`. (Opcional: remover `REVERTED` de `Ob2TradeStatus` se nada usa.)
- [ ] `cd api && npx prisma db push --accept-data-loss` (só DB local).
- [ ] tsc limpo + suite serial verde + `bun scripts/trading-v2-drift-audit.ts` roda limpo. Commit.

---

## Verificação final (antes do cutover Task 16)
- [ ] `npx tsc --noEmit` sem erros novos em trading-v2.
- [ ] Suite trading-v2 serial verde (os 27 bun-nativos rodam via `bun test`).
- [ ] `grep -rniE "\.free\b|reserved|ob2_reservations|reservation" src/modules/trading-v2 --include='*.ts' | grep -v __tests__` — sem referências funcionais.
- [ ] Só então Task 16 (cutover prod: dump + db push --accept-data-loss + seed do on-chain), com tráfego parado e OK explícito.
