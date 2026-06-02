# Task 16 — Runbook do cutover chain-as-truth (PROD)

**Operação de produção que toca saldo.** Downtime curto (scale→0). Tudo reversível via 2 pontos de rollback.

## Pontos de rollback (já preparados / a preparar)
- **Código:** tag git **`prod-pre-chain-as-truth`** → `208dfcf` (código de prod ANTES do rebuild). Rollback de código = redeploy deste commit.
- **Schema + dados:** **snapshot RDS** tirado no passo 2 abaixo. Restaurá-lo traz de volta `free`/`reserved`/`Ob2Reservation`/`reservationId` + os dados.
- Rollback COMPLETO = restore do snapshot RDS **E** redeploy da tag `prod-pre-chain-as-truth` (os dois juntos — código antigo precisa das colunas antigas).

## O que NÃO é perdido
- Dinheiro real (USDC no vault, tokens YES/NO, USDC nas wallets) — vive **on-chain**, não é tocado.
- `userAccount.balance` (V1, PIX/saque) — não é tocado.
- Histórico de trades (`ob2_trades`) e ordens já fechadas — **preservados** (inertes no motor novo).

## O que é descartado (de propósito)
- Colunas `free`/`reserved` (bookkeeping off-chain, stale/bugado) + tabela `Ob2Reservation` + `Ob2Order.reservationId`.
- Ordens **OPEN** atuais → canceladas (não têm `commitAsset`, não migram).
- Efeito esperado: usuários com drift off-chain têm o saldo **reconciliado pra verdade on-chain** (pode subir/descer). É o conserto.

---

## Pré-requisitos
- [ ] PR #177 revisado e **mergeado** em `main`.
- [ ] Código novo **deployado no ECS** (Pulumi/pipeline padrão). NÃO rodar o cutover antes do código novo estar no ar — o código antigo lê `free`/`reserved` e quebraria com o drop.
- [ ] `SOLANA_RPC_URL` configurado no ambiente do job de cutover (o seed lê on-chain via `SolanaOnchainBalanceReader`).
- [ ] `DATABASE_URL` do job apontando pro **cluster de prod** (writer).
- [ ] Janela de manutenção combinada (downtime de poucos minutos).

## Passos do cutover

**1. Backup (antes de QUALQUER coisa destrutiva)**
- [ ] Snapshot do cluster Aurora (RDS console/CLI): `aws rds create-db-cluster-snapshot --db-cluster-identifier <cluster> --db-cluster-snapshot-identifier pre-chain-as-truth-$(date +%Y%m%d%H%M)`. Aguardar `available`.
- [ ] `pg_dump` das tabelas ob2 (restore pontual rápido):
  `pg_dump "$DATABASE_URL" -t ob2_user_balances -t ob2_user_market_balances -t ob2_reservations -t ob2_orders -t ob2_trades -t ob2_settlement_deltas > ob2_pre_cutover_$(date +%Y%m%d%H%M).sql` (salvar fora do repo).

**2. Parar tráfego (downtime começa)**
- [ ] Scale do serviço ECS de trading → **0 tasks** (Pulumi ou `aws ecs update-service --desired-count 0`). Confirmar 0 running. (Com 0 tasks, o freeze in-process é irrelevante — nada serve.)

**3. Drop do schema (com o código novo já deployado)**
- [ ] Rodar `npx prisma db push --accept-data-loss` apontando pro DATABASE_URL de prod. Dropa `free`/`reserved`/`Ob2Reservation`/`reservationId`. (Idempotente: se o schema já bate, é no-op.)
  - ⚠️ Confirmar que o `schema.prisma` deployado é o do PR #177 (1548→ versão sem free/reserved) antes de rodar.

**4. Cancelar ordens OPEN + seed do espelho (one-off job com o código novo)**
- [ ] Cancelar ordens abertas (não migram): SQL `UPDATE ob2_orders SET status='CANCELLED', closed_at=now() WHERE status='OPEN';` (o compromisso some com o status; sem reservas a liberar).
- [ ] Tratar SETTLING travadas (se houver): com tráfego parado, ou aguardar drain antes do scale-0, ou marcar FAILED as antigas: `UPDATE ob2_trades SET status='FAILED', revert_reason='cutover' WHERE status='SETTLING';` (não há saldo a desfazer no modelo novo).
- [ ] **Seed do `onchain_total` a partir da chain**: rodar a reconciliação do trading-v2 (lê on-chain real e sobrescreve o espelho). Via um script one-off que instancia `DailyReconciliationService(prisma, new SolanaOnchainBalanceReader(prisma))` e chama `reconcile()` (ou `runWithAlerts`). Reaproveitar/adaptar `src/modules/trading-v2/scripts/run-cutover.ts` (que já faz o snapshotOnchain) — **adicionar** a ele os 2 passos de cancelamento acima se quiser tudo num comando. Confirmar `imported/total` no log e `errors: 0`.

**5. Smoke check (antes de subir)**
- [ ] `bun scripts/trading-v2-drift-audit.ts` contra prod (read-only): **0 available negativo**, **0 SETTLING travada**, **0 onchain_total negativo**.
- [ ] Spot-check: pegar 1-2 users de teste conhecidos e conferir que `onchain_total` bate com o saldo on-chain real (vault/wallet) — via explorer ou um read pontual.

**6. Subir tráfego (downtime acaba)**
- [ ] Scale ECS de volta (desired-count original). Confirmar tasks healthy.
- [ ] Teste manual end-to-end: depósito reflete em `available`; colocar/cancelar ordem; um trade casando → liquida (SETTLED) e saldo atualiza.

## Rollback (se algo der errado)
1. Scale ECS → 0.
2. Restaurar o **snapshot RDS** (passo 1) — recria as colunas/tabela + dados. (Restore Aurora cria novo cluster; repontar o `DATABASE_URL`/endpoint conforme seu processo.)
3. Redeploy do código da tag **`prod-pre-chain-as-truth`** (`208dfcf`).
4. Scale ECS de volta. Validar trading no modelo antigo.

## Notas
- O `TradingFreezeGuard` é in-process — com scale-0 não é necessário. Se algum dia o cutover for sem downtime, precisará de um freeze compartilhado (Redis/DB/env), não o in-process.
- Base de prod hoje é pequena (~14 users de teste) → seed e verificação são rápidos.
- Manter o snapshot RDS + pg_dump por alguns dias após o cutover antes de descartar.
