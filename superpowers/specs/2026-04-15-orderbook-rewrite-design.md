# Orderbook, Matching, Settlement & Orders — Rewrite do Zero

**Data:** 2026-04-15
**Autor:** Rafael + Claude
**Status:** Design aprovado em brainstorming, aguardando revisão escrita
**Branch prevista:** `feat/orderbook-rewrite`

---

## 1. Contexto e motivação

Depois da migração do orderbook duplo (YES/NO) para o "single orderbook", o sistema acumulou uma sequência crônica de bugs construídos em cima da estrutura antiga. Evidência no histórico recente (`git log`):

- `fix: sincroniza user_market_positions após TRADE match`
- `fix(api): TRADE com seller sem tokens cai no fallback MINT`
- `fix(api): ghost-orders-detector — cancela sell orders sem tokens on-chain`
- `fix(api): MM asks registram como buy-NO pra forçar MINT no match`
- `fix: limpeza de ordens fantasma do Redis no MM replenish`
- `fix: phantom cleanup compara TODOS os Redis IDs com DB`
- `fix(api): cron horário de reconciliação balance DB ↔ on-chain`
- `feat(api): self-healing auto-revert pra trades com settlement FAILED`

**Sintoma mais gritante:** cliente `7tVJwHGesCLGhMtsjydMfjJwYjUunBfiheewiGMNJNuZ` vendeu tokens que não possuía on-chain. Isso é possível hoje porque não há reserva atômica no caminho de `place-order SELL`: a ordem entra no book antes de confirmar que o vendedor tem o token.

**Diagnóstico arquitetural:** o sistema foi remendado, não redesenhado. Três fontes de verdade (Redis, Postgres, Solana) com drift constante; estado MM acoplado ao engine; primitivas on-chain (MINT/MERGE/TRADE) vazando pro matching off-chain via hacks ("MM ask vira buy-NO pra forçar MINT"); vocabulário ambíguo (`pending` usado pra ordem-no-book e pra trade-aguardando-settlement).

**Escopo do rewrite:** refazer do zero as 4 camadas acopladas — orders lifecycle, book, matching, settlement — respeitando invariantes fortes por construção, não por reconciliação posterior.

**Não-escopo:** markets (catálogo, liquidez inicial, payout de resolução) ficam como estão; só integram com o novo engine pela borda.

---

## 2. Decisões arquiteturais (7 pilares, travados no brainstorming)

| # | Decisão | Consequência principal |
|---|---|---|
| 1 | **Híbrido com lock autoritativo**: pré-reserva de tokens/USDC no DB com lock atômico antes da ordem entrar no book. Settlement on-chain é obrigado a bater; divergência reverte ordem. | Elimina por construção "vendeu sem ter". |
| 2 | **On-chain negociável**: aceitamos adicionar 1–2 instruções novas se simplificarem o off-chain. | Permite retirar hacks de mapeamento match→primitiva. |
| 3 | **Book único, YES-normalizado, só BUY/SELL**: a categoria "NO" não existe no engine. UI traduz na entrada ("buy NO @ X" → "sell YES @ 1−X"). | Colapsa matriz de 4 estados pra 2; mata categorias inteiras de bug. |
| 4 | **Postgres é o book.** Redis some como storage; fica no máximo como pub/sub de WS (stateless, descartável). | Drift Redis↔DB impossível. Matching concorrente via `SELECT FOR UPDATE SKIP LOCKED`. |
| 5 | **MM é só mais um usuário.** Zero código MM-específico no engine. Replenish/cotação vira bot externo via API pública. | Mata hacks MM (`buy-NO pra forçar MINT`, `seed auto-limpa asks legacy`, `phantom cleanup`). |
| 6 | **Hard cutover com freeze curto.** Usuários atuais são testers; justificável. Reconcilia contra on-chain (on-chain wins), cancela ordens abertas, migra schema, sobe novo engine. | Começa do estado limpo. |
| 7 | **Settlement híbrido sync/async**: taker (match imediato) é síncrono no HTTP request; maker (ordem resting que casa depois) é async com SLA duro. Em ambos, falha além de SLA = revert atômico. | Sem `SETTLING` eterno. Sem `pending` órfão. |

---

## 3. Invariantes do sistema

O código deve **recusar** qualquer operação que viole alguma destas. Testes automatizados verificam cada uma.

1. **I1 — Reserva obrigatória.** Nenhuma ordem entra no book sem reserva atômica confirmada (USDC pra BUY, tokens pra SELL). Reserva é um UPDATE condicional com `WHERE available >= amount`; afetar 0 linhas aborta a transação.
2. **I2 — Conservação de saldo.** Pra cada `(user, market)`: `total_on_chain = free + reserved` (sempre, observável a qualquer momento).
3. **I3 — Estados terminais únicos.** Uma ordem termina exatamente em `FILLED | CANCELLED | REJECTED`. Um trade termina em `SETTLED | REVERTED`. Nunca em estado intermediário.
4. **I4 — SLA de settlement.** Trade fica em `SETTLING` por no máximo 30s (sync/taker) ou 120s (async/maker). Expirou → revert automático. Sem cron de "self-healing" necessário; SLA é garantido pelo caminho feliz + worker de expiração determinístico.
5. **I5 — YES-normalização.** No engine, `side ∈ {BUY, SELL}` apenas. Tradução NO↔YES acontece exclusivamente em 2 pontos: schema de entrada HTTP e serializer de saída HTTP/WS.
6. **I6 — Uma fonte de verdade por domínio.** Book = Postgres. Saldos livres e reservados = Postgres. Posições finais e tokens circulantes = Solana. Não há "cache que é a verdade".
7. **I7 — Idempotência de webhook/event.** Todo evento on-chain processado carrega `(signature, instructionIndex)` como chave; reprocessar é no-op.

---

## 4. Modelo de dados (Postgres)

**Princípio:** uma tabela = um conceito. Sem campos sobrecarregados por tipo (como o `originalType` atual).

### 4.1 Tabelas novas (substituem as atuais)

```sql
-- Saldo por usuário × mercado × token side
-- "free" + "reserved" = total on-chain (invariante I2)
CREATE TABLE user_market_balances (
  user_id        UUID        NOT NULL,
  market_pda     TEXT        NOT NULL,
  asset          asset_enum  NOT NULL,       -- 'USDC' | 'YES' | 'NO'
  free           NUMERIC(20,6) NOT NULL DEFAULT 0 CHECK (free >= 0),
  reserved       NUMERIC(20,6) NOT NULL DEFAULT 0 CHECK (reserved >= 0),
  onchain_total  NUMERIC(20,6) NOT NULL DEFAULT 0,  -- última leitura on-chain
  onchain_slot   BIGINT,
  updated_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (user_id, market_pda, asset)
);

-- Reservas ativas (1 linha por reserva, referenciada pela ordem que a criou)
CREATE TABLE reservations (
  id           UUID PRIMARY KEY,
  user_id      UUID NOT NULL,
  market_pda   TEXT NOT NULL,
  asset        asset_enum NOT NULL,
  amount       NUMERIC(20,6) NOT NULL CHECK (amount > 0),
  order_id     UUID NOT NULL,                  -- FK lógica pra orders
  released_at  TIMESTAMPTZ,                    -- NULL = ativa
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ON reservations (order_id) WHERE released_at IS NULL;

-- Book: uma linha = uma ordem viva ou encerrada
CREATE TABLE orders (
  id              UUID PRIMARY KEY,
  user_id         UUID NOT NULL,
  market_pda      TEXT NOT NULL,
  side            side_enum NOT NULL,          -- 'BUY' | 'SELL' (sempre YES-normalizado)
  price           NUMERIC(6,4) NOT NULL CHECK (price > 0 AND price < 1),
  quantity        NUMERIC(20,6) NOT NULL CHECK (quantity > 0),
  filled          NUMERIC(20,6) NOT NULL DEFAULT 0 CHECK (filled <= quantity),
  status          order_status_enum NOT NULL,  -- 'OPEN' | 'FILLED' | 'CANCELLED' | 'REJECTED'
  reject_reason   TEXT,
  client_order_id TEXT,                         -- idempotência na criação
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  closed_at       TIMESTAMPTZ,
  UNIQUE (user_id, client_order_id)
);
-- Índice do book: busca por melhor preço, status aberto
CREATE INDEX orders_book_idx ON orders (market_pda, side, price, created_at)
  WHERE status = 'OPEN';

-- Trade: criado no momento do match, uma linha por fill
CREATE TABLE trades (
  id                UUID PRIMARY KEY,
  market_pda        TEXT NOT NULL,
  maker_order_id    UUID NOT NULL REFERENCES orders(id),
  taker_order_id    UUID NOT NULL REFERENCES orders(id),
  price             NUMERIC(6,4) NOT NULL,
  quantity          NUMERIC(20,6) NOT NULL,
  primitive         primitive_enum NOT NULL,   -- 'TRADE' | 'MINT' | 'MERGE'
  status            trade_status_enum NOT NULL,-- 'SETTLING' | 'SETTLED' | 'REVERTED'
  sync              BOOLEAN NOT NULL,          -- true = taker síncrono, false = async
  settling_deadline TIMESTAMPTZ NOT NULL,
  tx_signature      TEXT,
  revert_reason     TEXT,
  created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
  settled_at        TIMESTAMPTZ,
  UNIQUE (tx_signature)
);

-- Log append-only de eventos on-chain processados (idempotência I7)
CREATE TABLE onchain_events_processed (
  signature         TEXT NOT NULL,
  instruction_index INT  NOT NULL,
  kind              TEXT NOT NULL,             -- 'SETTLE_FILL' | 'MINT' | 'MERGE' | ...
  trade_id          UUID,
  processed_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  PRIMARY KEY (signature, instruction_index)
);
```

### 4.2 Tabelas que desaparecem

- `user_market_positions` (será derivada por `VIEW` somando trades SETTLED + mints + merges)
- Todo schema Redis de orderbook (keys `orderbook:*`, `levels:*`, `orders:*`)
- Tabelas/campos do MM especial (flag + tabelas separadas): MM usa `orders` como qualquer um

### 4.3 Migração (cutover)

1. Freeze: bloquear endpoints de trading; WS informa banner.
2. Snapshot on-chain por `(user, market)` → popular `user_market_balances.onchain_total` e `free = onchain_total`.
3. Cancelar todas as ordens antigas em memória; USDC reservado volta via reconciliação direta do vault.
4. Drop das tabelas/Redis antigos; apply do schema novo.
5. Deploy do novo engine; unfreeze.

---

## 5. Order lifecycle

### 5.1 Fluxo de criação (place-order)

```
HTTP POST /orders
  → use-case.place-order(userId, market, side, price, qty, clientOrderId)
    → BEGIN TX
      → validate market open, price ∈ (0,1), qty > 0
      → classify intent (ver §5.1.1 abaixo) → (asset, amount)
      → UPDATE user_market_balances
           SET free=free-amount, reserved=reserved+amount
           WHERE user_id=$1 AND market_pda=$2 AND asset=$3 AND free >= amount
         if affected=0 → REJECT 'insufficient_<asset>'
      → INSERT reservation row (asset, amount, order_id)
      → INSERT order row (status=OPEN)
    → COMMIT
    → MatchingEngine.tryMatch(order)   -- ver §6
    → return order (+ eventual trades)
```

#### 5.1.1 Classificação de intent e asset reservado

O book é YES-normalizado (§3 I5), mas o ativo realmente reservado depende do estado atual de posição do usuário. A classificação é **determinística e roda dentro da mesma transação da reserva**, então não há race entre "decidir o que reservar" e "reservar":

```
classify(userId, market, side, qty, price):
  SELECT free FROM user_market_balances FOR SHARE
    WHERE user_id=? AND market_pda=? AND asset IN ('YES','NO','USDC')

  if side == SELL:
    if free.YES >= qty:
      return (asset=YES,  amount=qty)              # vende YES que tem → TRADE ou MERGE
    else:
      return (asset=USDC, amount=qty*(1-price) + fee)  # abre short YES = long NO → MINT

  if side == BUY:
    if free.NO >= qty:
      return (asset=NO,   amount=qty)              # fecha short YES → MERGE
    else:
      return (asset=USDC, amount=qty*price + fee)      # abre long YES → TRADE ou MINT
```

A matriz acima cobre os 4 caminhos possíveis. **Em nenhum deles o engine aceita a ordem sem reserva do ativo certo**. O caso `7tVJwH...` (SELL sem ter YES) hoje entra no book; no novo, cai no branch `free.YES < qty` e reserva USDC — o que exige que o usuário tenha USDC livre suficiente pra abrir a short. Se não tiver, REJECT.

Nota: a primitiva on-chain (§6.2) é decidida no match, não na criação. A classificação aqui só serve pra saber **o que travar**; se o match acabar sendo TRADE ou MINT, a reserva já está no ativo correto pra ser liberada corretamente pelo settlement.

### 5.2 Cancelamento

```
HTTP DELETE /orders/:id
  → use-case.cancel-order(userId, orderId)
    → BEGIN TX
      → SELECT order FOR UPDATE
      → if status != OPEN → 409
      → remaining = quantity - filled
      → UPDATE order SET status=CANCELLED, closed_at=now()
      → release reservation of remaining
      → UPDATE balances SET reserved=reserved-x, free=free+x
    → COMMIT
    → emit WS cancel event
```

### 5.3 Estados

```
         REJECTED ←──── (falha na reserva ou validação)
                      ╱
place-order ── OPEN ──┼── FILLED       (filled == quantity)
                      │
                      ├── CANCELLED    (cancel-order ou expiry)
```

Sem `PARTIALLY_FILLED` como estado separado: uma ordem com `filled > 0 AND filled < quantity` está em `OPEN`. A distinção vive no campo `filled`, não numa enum.

---

## 6. Matching engine

### 6.1 Algoritmo

**Price-time priority** clássico. `SELECT FOR UPDATE SKIP LOCKED` no book garante matching concorrente sem serialização global.

```
tryMatch(takerOrder):
  loop:
    SELECT * FROM orders
      WHERE market_pda = $1
        AND status = 'OPEN'
        AND side = opposite(takerOrder.side)
        AND (takerOrder.side = BUY  ? price <= takerOrder.price
                                    : price >= takerOrder.price)
      ORDER BY
        CASE takerOrder.side WHEN BUY THEN price END ASC,
        CASE takerOrder.side WHEN SELL THEN price END DESC,
        created_at ASC
      LIMIT 1
      FOR UPDATE SKIP LOCKED;

    if no row or takerOrder.filled == quantity → break

    fillQty = min(makerRemaining, takerRemaining)
    primitive = decidePrimitive(maker, taker)   -- §6.2
    createTrade(maker, taker, fillQty, makerPrice, primitive, sync=true)
    updateBothOrders(fillQty)
    settleSync(trade)   -- §7
```

### 6.2 Decisão de primitiva (MINT / TRADE / MERGE)

A primitiva é **derivada do ativo reservado de cada lado** (armazenado em `reservations.asset`, decidido na criação — §5.1.1). O engine lê a reserva, não o balance atual:

| Taker asset reservado | Maker asset reservado | Primitiva | Semântica |
|---|---|---|---|
| USDC (BUY) | YES (SELL) | **TRADE** | Token existente muda de dono. |
| YES (SELL) | USDC (BUY) | **TRADE** | Idem, simétrico. |
| USDC (BUY) | USDC (SELL) | **MINT** | Vault cria par YES+NO: BUY recebe YES, SELL recebe NO. |
| NO (BUY) | YES (SELL) | **MERGE** | Vault destrói par: BUY e SELL recebem USDC proporcional. |
| YES (SELL) | NO (BUY) | **MERGE** | Idem, simétrico. |
| NO (BUY) | USDC (SELL) | **TRADE** | Taker fecha short, maker abre short → NO muda de dono. |

As outras 2 combinações (NO × NO, USDC(BUY) × NO(BUY)) são logicamente impossíveis sob as regras de §5.1.1 e o engine trata como bug assertivo (panic + alert), não como caso a tratar.

**Crítico:** nenhuma dessas decisões depende de consultar saldo on-chain ou balance atual no momento do match. Toda informação necessária já está travada na reserva. Isso elimina a classe de bug "maker mudou posição entre match e settle".

**Nova instrução on-chain proposta (pilar #2):** `settle_fill(maker, taker, side, qty, price, primitive_hint)`. O programa valida o hint e executa MINT/TRADE/MERGE internamente, removendo a responsabilidade do off-chain decidir perfeitamente. Se hint divergir do estado on-chain, a instrução falha e o trade reverte normalmente via SLA.

### 6.3 Makers que entram no book

Se `tryMatch` não consome a ordem inteira, o restante fica `OPEN` no book. Quando outra ordem casar com ela depois, o trade gerado é `sync=false` e roda pelo caminho async (§7.2).

---

## 7. Settlement

### 7.1 Taker síncrono

O request HTTP do taker não retorna até o trade estar `SETTLED` ou `REVERTED`.

```
settleSync(trade):
  signature = solana.sendAndConfirm(settle_fill(...), commitment='confirmed')
  if success:
    UPDATE trades SET status='SETTLED', tx_signature=?, settled_at=now()
    applyBalanceDeltas(trade)   -- libera reservas, cria posição
    emit ws trade_settled
  else:
    revertTrade(trade, reason)  -- §7.3
  return
```

Timeout do request HTTP: 30s (I4). Se o RPC não confirmar em 30s, a lógica do revert roda e retorna 503 pro taker com mensagem clara.

### 7.2 Maker assíncrono

Quando uma ordem resting é consumida por um taker, o trade é criado `sync=false`. O próprio fluxo do taker dispara o settle on-chain (é a mesma tx, mesmo `settle_fill`). Diferença: o resultado volta via:

1. **Evento on-chain** (listener): event processor marca `SETTLED` e libera reservas.
2. **Worker de expiração** (polling): scan `trades WHERE status='SETTLING' AND settling_deadline < now()` → `revertTrade`.

Isso significa: maker não depende de nenhum cron de "self-healing". Ou o evento chega (caminho feliz), ou o worker de deadline reverte em ≤120s (SLA duro).

### 7.3 Revert

```
revertTrade(trade, reason):
  BEGIN TX
    UPDATE trades SET status='REVERTED', revert_reason=?
    -- desfazer o fill nas ordens
    UPDATE orders SET filled = filled - trade.qty,
                      status = CASE WHEN filled - trade.qty = 0 AND closed THEN reopen
                                    ELSE 'OPEN' END
                 WHERE id IN (maker_order_id, taker_order_id)
    -- restaurar reservas proporcionalmente
    applyBalanceDeltas(INVERSE of trade)
  COMMIT
  emit ws trade_reverted
```

Se a ordem já estava `FILLED` (fill total) antes do revert, ela reabre como `OPEN` com a quantidade revertida — comportamento configurável por flag, default = **cancel** em vez de reabrir, pra não surpreender usuário.

### 7.4 Listener on-chain

Serviço único que consome o stream de transações do programa e, pra cada instrução relevante (`settle_fill`, `cancel`, etc.):

1. Valida `(signature, idx)` via `onchain_events_processed` (I7).
2. Resolve `trade_id` pela signature.
3. Marca `SETTLED` ou dispara reconciliação se divergente.
4. `INSERT INTO onchain_events_processed`.

Reconexão e backfill: ao subir, lê último slot processado e faz catch-up. Sem janela cega.

---

## 8. Fees

Mantém o fee ledger atual (commit `feat: fee ledger + fix ghost positions no perfil`). Integração: no `place-order BUY`, a reserva inclui `qty*price + fee`. No `settle_fill`, o programa debita fee pra wallet de fee configurada. Fee é campo de config do mercado.

---

## 9. Market Maker (externo)

MM deixa de viver dentro do engine. Vira **bot separado** (`services/mm-bot/`) que:

- Lê orderbook via endpoint público (`GET /markets/:pda/book`).
- Consulta mid-price via endpoint público (`GET /markets/:pda/mid`).
- Posta ordens via `POST /orders` como qualquer usuário, usando wallet própria.
- Cancela ordens via `DELETE /orders/:id`.

Nenhum privilégio. O engine não sabe que existe MM. Replenish, cotação, spread, limites — tudo vive no bot. Se o bot cai, o book continua funcionando sem liquidez sintética, mas sem drift.

Migração: código atual do MM vira bot. Tabelas `mm_*` são deletadas.

---

## 10. WebSocket

Serviço stateless que observa mudanças em `orders` e `trades` (via `LISTEN/NOTIFY` do Postgres ou pub/sub com Redis efêmero) e propaga:

- `order_update` (created, filled, cancelled)
- `trade_settled`
- `trade_reverted`
- `book_snapshot` (top N levels, projeção do SELECT)

Zero estado no WS service. Pode cair e voltar sem consequência.

---

## 11. Reconciliação (defesa em profundidade)

Mesmo com as invariantes, roda um job diário que compara por `(user, market, asset)`:

```
expected_onchain = free + reserved
actual_onchain   = RPC fetch

if |expected - actual| > dust:
  LOG critical
  alert oncall
  freeze user (flag)
```

**Esse job é puramente defensivo.** No sistema atual, reconciliação é obrigatória pro funcionamento. No novo, ela existe só pra detectar bug — não pra consertar estado.

---

## 12. Testing strategy

### 12.1 Invariantes (property-based + integration)

Um teste por invariante (I1–I7). Rodam em CI.

- **I1:** concurrent SELLs com saldo insuficiente — 0 sucessos além do saldo.
- **I2:** sequência aleatória de place/cancel/fill — soma free+reserved == total em todo snapshot.
- **I3:** fuzz de ordens — nenhuma termina fora de {FILLED, CANCELLED, REJECTED}.
- **I4:** trade com settle mockado travado — reverter em ≤30s/120s.
- **I5:** input NO vira SELL YES corretamente; output SELL YES vira NO na serialização; grep do código garante "NO" não aparece em services/.
- **I6:** desligar Redis — trading continua 100% funcional.
- **I7:** replay do mesmo evento on-chain 100x — no-op, delta zero.

### 12.2 Cenários de regressão (replicam bugs atuais)

- `7tVJwH` case: vender sem tokens retorna 422 em place-order, não executa.
- "MM ask vira buy-NO": teste que grep `buy-NO` em código e falha se encontrar.
- Ghost orders: matar engine no meio de match → restart → estado consistente.
- Drift DB↔on-chain: simular settle_fill com sucesso on-chain mas listener perde evento → worker de deadline reverte; reconciliação acusa divergência e corrige na próxima leitura.

### 12.3 E2E

Playwright full-flow: 2 usuários, um compra, outro vende, settle real na devnet, verificar posições corretas em ambos.

---

## 13. Plano de entrega (alto nível, plano detalhado virá no próximo passo)

1. **Fase 0 — on-chain**: adicionar `settle_fill` ao programa Solana; manter antigas pra backfill.
2. **Fase 1 — schema novo**: migrations, modelos Prisma, repositórios.
3. **Fase 2 — reservation service + balance service**: com testes I1/I2.
4. **Fase 3 — order lifecycle**: place + cancel + estados, sem matching ainda.
5. **Fase 4 — matching engine sync**: taker only, sem async.
6. **Fase 5 — settlement sync + listener + revert**: fecha ciclo taker.
7. **Fase 6 — maker async + worker de deadline**.
8. **Fase 7 — WS sobre novo modelo**.
9. **Fase 8 — MM bot externo**: extrair código MM atual.
10. **Fase 9 — migração/cutover**: script de snapshot, freeze, switch.
11. **Fase 10 — reconciliação diária + alerting**.

Cada fase entrega testada e mergeada antes da próxima. Fase 9 é o único momento de downtime.

---

## 14. Riscos e mitigações

| Risco | Mitigação |
|---|---|
| Mudança no programa Solana (Fase 0) introduz bug em prod | Deploy primeiro em devnet, mantém instruções antigas; `settle_fill` é additive. |
| MM bot externo reduz liquidez durante migração | Bot é entregue na Fase 8 e testado em staging antes do cutover. |
| Postgres vira gargalo de matching em volume alto | Volume atual é trivial. Se crescer, migramos pra Redis event-sourced (modelo (b) da Pergunta 4) — mas só com evidência. |
| Revert reabrir ordem já `FILLED` surpreende usuário | Default = cancelar em vez de reabrir; configurável. WS notifica explicitamente. |
| Cutover deixa saldo órfão | Snapshot on-chain é a fonte; qualquer valor residual no DB é descartado. |

---

## 15. O que NÃO está neste spec (e por quê)

- **Markets/resolução/payout**: fora de escopo, integra via borda.
- **Chat, notificações, identity**: intocados.
- **AMM/liquidez automática**: decidimos MM externo; AMM on-chain descartado (opção (c) da Pergunta 5).
- **Order types além de limit**: market/IOC/FOK podem vir depois; spec atual = limit GTC com cancel explícito.
- **Dark pool, hidden orders, iceberg**: YAGNI.
