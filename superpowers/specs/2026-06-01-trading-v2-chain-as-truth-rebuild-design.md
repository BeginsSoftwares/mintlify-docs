# Trading V2 — Rebuild do ledger/settlement: chain-as-truth + projeção otimista

**Data:** 2026-06-01
**Status:** Design aprovado (seções validadas em brainstorm) — aguardando revisão da spec escrita
**Escopo:** Reconstruir a camada de **ledger autoritativo e settlement** do trading-v2. NÃO altera o contrato on-chain, o caller atômico, nem a lógica de matching.
**Relacionado:** [`2026-04-15-orderbook-rewrite-design.md`](./2026-04-15-orderbook-rewrite-design.md), `api/docs/trading-v2-vs-polymarket.md`

---

## 1. Contexto e motivação

O trading-v2 tem histórico de bugs de saldo (saldos errados, reservas presas, drift). Auditoria read-only em prod (2026-06-01, ~14 users) confirmou bugs reais concentrados (ex.: `user 408` / mkt `2ufUooYvp6bu`): 5 violações do invariante I2, 3 reservas órfãs (`releasedAt=null` em ordens CANCELLED/FILLED), 1 saldo negativo (`free=-1.49`). Também: `onchainTotal` congelado ~24+ dias (reconciliação do trading-v2 fora do cron).

**Causa-raiz (não são bugs isolados):** o DB mantém um **ledger mutável autoritativo** (`free`/`reserved` + `Ob2Reservation`) e aplica deltas **otimisticamente antes da confirmação on-chain**, com revert na falha, coordenado por 4 workers via o campo `status`. Cada costura entre dois números mutáveis é origem de drift.

**Verificação que delimitou o escopo:** o contrato on-chain (`AHiRBEXouJnoVsvQ37KEjV6AP62r6Yi9YkkbNBtsqBaW`) e o `SolanaOnchainCaller` estão **sólidos** — as duas pernas (`settle_clob` + `settle_clob_sell`) são empacotadas numa **única transação Solana atômica**, com `token_amount` igual e preços complementares (`P_YES + P_NO = 100`). Settlement on-chain é atômico e a solvência do vault é preservada por construção. **O problema está inteiramente na camada off-chain.**

**Decisão estratégica:** produto não está no ar — é o momento de refazer corretamente, sem gambiarras. Rebuild **cirúrgico** (Abordagem A): trocar só o miolo de ledger/settlement, manter o que foi verificado sólido.

## 2. Princípio central — inverter a fonte de verdade

Hoje o DB mutável é a autoridade e o on-chain confirma depois. **Invertemos.** Três camadas, cada uma reconciliando pra de baixo:

```
[on-chain]            ← VERDADE absoluta (vault, mints, supplies)
   ↑ reconcilia
[backend: projeção]   ← autoridade do servidor (DERIVADA, não mutável)
   ↑ espelha
[front: otimista]     ← antecipa no clique, reconcilia com o backend
```

Saldo deixa de ser número mutado e vira número **derivado**. Fórmula única, na leitura:

```
available(user, asset) =  confirmedOnchain(user, asset)        // espelho da chain
                        + Σ pendingSettlementDeltas(user,asset) // trades in-flight (otimista)
                        − Σ openOrderCommitments(user,asset)    // travado por ordens abertas
```

Os três termos são fontes derivadas/append-only — **não há dois números mutáveis que precisam ficar em sincronia**, que é a origem de todo drift. Consequências (eliminação por construção):
- **Sem `reserved`** → impossível reserva órfã. A "trava" é a ordem aberta; cancelou, o termo some.
- **Sem apply-otimista-e-reverte** → o saldo autoritativo nunca é mutado especulativamente.
- **`confirmedOnchain` espelha a chain** → não fica negativo nem inventa valor.
- **Sem dust reserva≠consumo** → a tx atômica on-chain é o consumo exato.

## 3. Modelo de dados

**Removido (ledger autoritativo podre):**
- Tabela `Ob2Reservation` — **deletada**.
- Colunas `free` e `reserved` de `Ob2UserBalance` e `Ob2UserMarketBalance` — **removidas**.

**Vira espelho da chain (termo `confirmedOnchain`):**
- `Ob2UserBalance` (USDC, global) e `Ob2UserMarketBalance` (YES/NO, por mercado) ficam com `confirmedAmount` (hoje `onchainTotal`) + `slot` (`onchainSlot`) + `updatedAt`. Escritas só por (a) fold de confirmação de settlement e (b) reconciliação.

**Termo `openOrderCommitments` (substitui reservas):**
- `Ob2Order` ganha `commitAsset` (USDC|YES|NO) e `commitAmount` = compromisso da parte **não preenchida**. `available` é um `SUM` limpo sobre ordens abertas. Cancelar/preencher remove/reduz o termo automaticamente. (Armazenado explícito pra manter a query um agregado simples e evitar recomputar com risco de rounding.)

**Termo `pendingSettlementDeltas` (substitui apply-otimista):**
- Nova tabela `Ob2SettlementDelta`: `(tradeId, userId, asset, marketPda?, amount assinado)`. Um TRADE gera 4 deltas (comprador −USDC/+YES, vendedor +USDC/−YES). Enquanto a trade está `SETTLING`, entram na projeção. Na confirmação, cada delta é **dobrado** no espelho `confirmedAmount` e a trade vira `SETTLED`.
- `Ob2Trade`: status `SETTLING → SETTLED | FAILED`.

Resultado: zero números mutáveis em sincronia. `available` = 1 espelho + 2 somas derivadas.

## 4. Admissão de ordem (prevenir gasto-duplo sem `reserved`)

Ao colocar ordem (`user`, `asset`, compromisso `C`), numa transação:
1. `SELECT ... FOR UPDATE` na **linha-espelho do par (user,asset)** — `ob2_user_balances` quando `asset=USDC` (global), `ob2_user_market_balances` quando `asset∈{YES,NO}` (por mercado). Essa linha-espelho é o **mutex por (user,asset)** (garante a linha via upsert se não existir).
2. `available = confirmedOnchain + Σ pendingDeltas − Σ openOrderCommitments`.
3. Se `available >= C`: `INSERT ob2_orders` (com `commitAsset`, `commitAmount = C`). A ordem **é** a trava.
4. Senão: rejeita (saldo insuficiente).

**Concorrência:** toda operação que muda um termo do `available(user,asset)` — admissão **e** fold de settlement — pega `FOR UPDATE` na linha-espelho. `available` sempre num snapshot consistente; admissões concorrentes serializam.

**Anti-deadlock:** um trade toca dois usuários (e até dois assets) → travas adquiridas em **ordem determinística** (`userId`, depois `asset`).

**Fills parciais:** ao casar `q` de `Q`, o compromisso da ordem cai pra `(Q−q)` e nasce um `Ob2SettlementDelta` pra `q`, **na mesma transação** do match.

**Price improvement cai de graça:** num BUY casado abaixo do limite, o compromisso liberado (`q×P_limite`) menos o delta pendente (`−q×P_exec`) devolve a melhoria ao `available` automaticamente. O cálculo manual `refundPriceImprovement` (bug `c207cb1`) **deixa de existir**.

## 5. Fluxo de settlement (sem apply-e-reverte, sem corrida de workers)

Regra de ouro: **`confirmedOnchain` é tocado exatamente uma vez, só DEPOIS da tx confirmar.**

1. **Match** (matching engine, uma transação, travas em ordem determinística): acha maker (lógica atual inalterada), reduz compromisso das duas ordens (`filled`), cria `Ob2Trade(SETTLING)` + `Ob2SettlementDelta`. Commit → deltas aparecem otimistas no `available`. Nenhum saldo autoritativo mexido.
2. **Envio on-chain:** `SolanaOnchainCaller.sendSettleFill` (inalterado) — tx atômica de 2 pernas, envia, confirma.
3. **Confirmação** (uma transação, com travas): para cada delta, `confirmedOnchain[user,asset] += amount`; marca `SETTLED`. Delta conhecido == efeito on-chain (nós construímos a tx).
4. **Falha definitiva** (uma transação): marca `FAILED` (deltas saem da soma); **restaura compromisso e reabre as ordens** (`OPEN`, `filled -= q`). **Zero desfazimento de saldo.**

**Um worker, máquina de estados `SETTLING → SETTLED | FAILED`:**
- O `settlement-reverter` (desfazia saldo; bug `9181731`) **some**. O `stub-settler.applyDeltas` **some**. A coordenação de 4 workers vira **1 worker + sua trilha de recuperação**.
- O "deadline-worker" colapsa na recuperação: trade `SETTLING` além do prazo → `getSignatureStatuses` (assinatura salva) → dobra-se-confirmou / falha-se-não.

**Idempotência & crash-safety:**
- Fold reivindicado por UPDATE condicional `SETTLING→SETTLED` (como o flag `sync` hoje) → roda no máximo uma vez.
- Status `unknown` da tx → **não dobra e não falha**; re-consulta depois. Seguro porque nada é commitado até confirmar (era bug `4cabb27`).
- Crash após enviar e antes de confirmar → no restart a varredura acha `SETTLING`, consulta a assinatura, dobra-ou-falha. Idempotente, sem dinheiro pra desenrolar.

## 6. Reconciliação (rede de segurança) + regra geral

`confirmedOnchain` já é dobrado a cada confirmação → vive fresco. A reconciliação é **backstop**: re-lê o saldo on-chain real periodicamente e **sobrescreve o espelho com a verdade da chain** (auto-corretivo; não há "ajuste entre dois ledgers").

- Divergência além da poeira = **bug de verdade** → alerta com **baixo ruído**. Reaproveita `buildReconcileAlert` (já implementado + testado) e `trading-v2-drift-audit.ts` (repurposeado pros invariantes novos: `confirmedOnchain == chain`, nenhum `available` negativo, nenhuma trade `SETTLING` velha).
- A `DailyReconciliationService` do trading-v2 (achada **morta**, fora do cron) finalmente entra agendada — só atualiza o espelho.

**Regra geral:** *mexeu no saldo on-chain → atualiza o espelho.* Qualquer fluxo fora do orderbook (depósito/saque PIX/3xChange, `trade` AMM legado, `claim_payout`) dobra seu delta conhecido no espelho ou é pego pela reconciliação.

## 7. O que fica intacto (verificado sólido)

- Programa on-chain (contrato) — não toca.
- `SolanaOnchainCaller` + `RealTransactionSender` (tx atômica) — não toca.
- `legacy-clob-instruction-builder` (monta as pernas) — não toca.
- **Lógica** de matching (preço-tempo, `compatibleAssetsFor`, `decidePrimitive` TRADE/MINT/MERGE) — mantida. Mudam só as bordas: lê `available` e escreve `Trade + SettlementDelta` (em vez de `applyDeltas`).
- Rotas, APIs de leitura do book, publicação de eventos — mantidas.

## 8. Estratégia de testes

- **Unit puro (sem DB), TDD:** fórmula `available`, cálculo de compromisso, deltas por primitive, fold, reabrir-na-falha.
- **Invariantes como property tests (oráculo):**
  - **Conservação** (teste-mestre): match→settle→confirm não cria/destrói valor (só fees explícitas e depósito/saque mudam o total).
  - `available` nunca negativo se admissão respeitada.
  - Fold idempotente; reconciliar após fold não muda nada.
  - Price improvement volta exatamente ao `available`.
  - Falha reabre exatamente o compromisso preenchido.
- **Integração em DB de TESTE (NUNCA prod):** concorrência de admissão, fills parciais, transições de status, recuperação de crash. ⚠️ `.env` aponta pra prod e testes de integração fazem `deleteMany` — rodar só contra testcontainer/DB descartável.

## 9. Cutover — começar limpo (com dump)

1. **Dump** das tabelas Ob2 atuais (clientes de teste) — arquivado, não migrado.
2. Drop de `Ob2Reservation` + colunas `free`/`reserved`; zera `Ob2Order`/`Ob2Trade`.
3. **Semeia `confirmedOnchain` de uma leitura on-chain fresca** (reconciliação popula do que existe na chain). Mercados/PDAs permanecem.
4. Produto fora do ar → cutover sem tráfego.

## 10. Sequência de construção (alto nível)

Miolo puro (TDD) → peças com DB (DB de teste) → bordas do matching engine → worker de settlement → reconciliação agendada → cutover (dump + limpa + semeia da chain). Detalhamento no plano de implementação (writing-plans).

## 11. Não-objetivos (YAGNI)

- Não alterar o contrato on-chain nem migrar de modelo de PDA.
- Não introduzir oracle/janela de disputa na resolução (decisão explícita: fora de escopo).
- Não migrar dados de saldo do modelo antigo (só dump de arquivo).
- Não reescrever matching, rotas ou eventos.
