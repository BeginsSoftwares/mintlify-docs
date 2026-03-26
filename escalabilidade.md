# Plano de Escalabilidade — 5k Usuários Simultâneos

## Diagnóstico atual

| Componente | Situação | Aguenta 5k? |
|---|---|---|
| BTC price WebSocket | OK — cada ECS task tem sua própria conexão Kraken | ✅ Sim |
| Car count WebSocket | Usuários conectam direto no Python service | ❌ Não |
| HTTP polling ao expirar mercado | 1k req/s idênticos sem cache | ⚠️ Risco |
| Orderbook / Payments WS | Redis pub/sub implementado | ✅ Sim |
| Endpoints de mercado (GET) | Sem cache Redis, bate direto no Postgres | ⚠️ Risco |
| ECS horizontal scaling | Sem pub/sub para BTC price WS entre tasks | ⚠️ Parcial |

---

## Item 1 — Car Count WS: Proxy no Oraculo API + Redis Pub/Sub

**Prioridade: Crítica**

### Problema
O `wsUrl` retornado pelo endpoint `/markets/car-count/active` aponta diretamente para o serviço Python (`oraculo-cars-count`). Com 5k usuários num mercado ativo, o Python recebe 5k conexões WebSocket simultâneas — um único processo uvicorn com fan-out de 5k `ws.send()` por update.

### Solução
Criar um proxy WebSocket no Oraculo API. O Python service mantém **1 conexão de saída** para a API; a API distribui para todos os usuários via Redis pub/sub entre tasks ECS.

```
Antes:
  Python (YOLO) → ws.send() × 5.000 clientes

Depois:
  Python (YOLO) → 1 WS → Oraculo API Task A
                              ↓ redis.publish("car-count:{roundId}", update)
  ECS Task A → ws.send() × 1.667 clientes
  ECS Task B → ws.send() × 1.667 clientes  (via Redis subscriber)
  ECS Task C → ws.send() × 1.667 clientes  (via Redis subscriber)
```

### O que implementar

**No Python (`oraculo-cars-count`):**
- Adicionar endpoint `POST /rounds/{roundId}/push-ws-url` ou simplesmente trocar: ao invés de aceitar conexões WS de clientes, o Python conecta como **cliente** num endpoint do Oraculo API
- Alternativa mais simples: Python continua aceitando 1 conexão de "relay" vinda do Oraculo API

**No Oraculo API:**
- Novo endpoint WS: `GET /ws/car-count/{roundId}` (clientes conectam aqui)
- Ao subir, API conecta ao Python como relay: `Python WS → recebe update → redis.publish`
- Redis subscriber em cada task: ao receber mensagem, faz broadcast local para clientes do `roundId`
- Reusar o padrão já existente em `websocket-orderbook.service.ts` e `socket.service.ts`

**No `cars-count.service.ts`:**
- `getWsUrl(roundId)` passa a retornar `wss://api.oraculo.fun/ws/car-count/{roundId}` em vez da URL do Python

**Estimativa:** 1-2 dias de implementação

---

## Item 2 — Cache Redis para Endpoints de Mercado

**Prioridade: Alta**

### Problema
Quando um mercado de 5min expira, o frontend faz `invalidateQueries` a cada 5s até o cron resolver (~30s). Com 5k usuários simultâneos: **1.000 req/s idênticos** todos buscando `GET /prediction-market/markets/{pda}` — sem cache, cada request bate no Postgres.

O mesmo ocorre na home: `GET /markets?page=1` sem cache com `staleTime: 30s`.

### Solução
Adicionar camada de cache Redis com TTL curto nos endpoints quentes.

```typescript
// Exemplo no handler de GET /prediction-market/markets/:pda
const cacheKey = `market:${pda}`;
const cached = await redis.get(cacheKey);
if (cached) return c.json(JSON.parse(cached));

const market = await marketRepository.findByPda(pda);
await redis.setex(cacheKey, 5, JSON.stringify(market)); // TTL: 5s
return c.json(market);
```

### Endpoints a cachear

| Endpoint | TTL sugerido | Motivo |
|---|---|---|
| `GET /prediction-market/markets/:pda` | 5s | Hit em massa ao expirar mercado |
| `GET /markets?page=1` (home) | 10s | Todos os usuários na home buscam o mesmo |
| `GET /markets/car-count/active` | 10s | Polling a cada 30s de todos os usuários |
| `GET /markets/btc-price/active` | 10s | Idem |

### Invalidação
Ao resolver um mercado (cron), invalidar o cache:
```typescript
await redis.del(`market:${pda}`);
```

**Estimativa:** 1 dia

---

## Item 3 — BTC Price WS: Redis Pub/Sub entre ECS Tasks

**Prioridade: Média** (só se o número de tasks ECS escalar acima de 1)

### Problema
Hoje cada ECS task conecta independentemente ao Kraken e faz broadcast local. Com 1 task isso é OK. Se o ECS escalar para 3+ tasks e uma task perder a conexão Kraken (flap de rede), os usuários daquela task ficam sem atualizações — as outras tasks recebem normalmente mas não compartilham.

### Solução
Task "líder" conecta ao Kraken → publica no Redis → todas as tasks consomem e fazem broadcast local.

```
Kraken WS → ECS Task A (líder) → redis.publish("btc-price", { ts, price })
                                      ↓
                          ECS Task A: broadcast local
                          ECS Task B: broadcast local (via redis.subscribe)
                          ECS Task C: broadcast local (via redis.subscribe)
```

O padrão já existe em `socket.service.ts` e `websocket-orderbook.service.ts` — replicar para `btc-price-websocket.service.ts`.

**Estimativa:** 1 dia

---

## Item 4 — Redis: Aumentar Limite de Memória

**Prioridade: Média**

### Problema
Redis configurado com `maxmemory 256mb` e política `allkeys-lru`. Com cache de mercados + pub/sub de WS + sessions + rate limiting, 256MB pode ser insuficiente e causar eviction de chaves de sessão/cache prematuramente.

### Solução
- Aumentar para **512MB–1GB** dependendo do número de mercados ativos
- Considerar separar instâncias Redis por responsabilidade:
  - `redis-cache`: dados efêmeros (mercados, preços) — pode usar `allkeys-lru`
  - `redis-session`: sessões, blacklist de tokens — usar `noeviction` ou `volatile-lru`
- Na ECS, usar ElastiCache (Redis gerenciado) ao invés de container Redis para HA e failover automático

---

## Item 5 — ECS: Task Sizing e Auto Scaling

**Prioridade: Média**

### Configuração atual
Sem CPU/memory limits explícitos no docker-compose. Em ECS, sem limits definidos a task pode ser mal alocada.

### Recomendações

**Task sizing para 1.700 usuários por task (assumindo 3 tasks para 5k):**
- CPU: `1 vCPU` (1024 units)
- Memory: `2GB`
- WebSocket connections por task: ~1.700 (cada ~50KB de overhead = ~85MB de WS)

**Auto Scaling:**
```
Métrica principal: número de conexões WebSocket ativas (já tem Prometheus)
Scale out: > 1.200 conexões por task
Scale in: < 600 conexões por task
Min tasks: 2 (HA)
Max tasks: 10
```

**ALB (Application Load Balancer):**
- Habilitar **sticky sessions** para WebSocket (ou confiar no Redis pub/sub para cross-task)
- Timeout do ALB para WS: aumentar para 1 hora (padrão é 60s)

---

## Item 6 — Rate Limiting nos Endpoints de Polling

**Prioridade: Baixa** (coberto parcialmente pelo cache)

### Solução
Adicionar rate limit por IP nos endpoints mais buscados:

```typescript
// 60 req/min por IP em GET /markets/*
rateLimiter({ windowMs: 60_000, max: 60 })
```

O Redis já tem infraestrutura de rate limiting em `redis.service.ts` — só aplicar nos routes de mercado.

---

## Ordem de implementação sugerida

```
Sprint 1 (urgente):
  [1] Car Count WS Proxy + Redis pub/sub
  [2] Cache Redis nos endpoints quentes

Sprint 2 (antes de crescer):
  [3] BTC Price WS Redis pub/sub entre tasks
  [4] Redis: aumentar memória / migrar para ElastiCache

Sprint 3 (otimização):
  [5] ECS Auto Scaling configurado com métricas de WS
  [6] Rate limiting nos endpoints de polling
```

---

## Impacto esperado pós-implementação

| Cenário | Antes | Depois |
|---|---|---|
| 5k usuários no mercado de carro | Python recebe 5k WS → colapsa | Python recebe 1 relay WS → OK |
| Mercado expira com 5k usuários | 1k req/s no Postgres por 30s | ~5 req/s no Postgres (cache 5s) |
| ECS task perde conexão Kraken | Usuários daquela task sem preço | Redis pub/sub garante continuidade |
| BTC price com 3 tasks ECS | 3 conexões Kraken independentes | 1 conexão Kraken, fan-out via Redis |
