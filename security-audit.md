# Relatório de Segurança — Oraculo Platform

> **Data:** 2026-03-23
> **Escopo:** API (`/api/src`) + Web (`/web`)
> **Metodologia:** Revisão de código estático (white-box), threat modeling financeiro
> **Arquivos analisados:** `index.ts`, rotas de payments, middlewares, serviços, frontend

---

## Índice de Severidade

| ID | Título | Severidade | Arquivo Principal |
|----|--------|-----------|-------------------|
| V-01 | WebSocket sem autenticação — receber notificações de qualquer usuário | **CRÍTICA** | `index.ts:494` |
| V-02 | Webhook 3xChange sem HMAC quando variável não configurada | **CRÍTICA** | `webhooks.routes.ts:92` |
| V-03 | CORS wildcard em todas as rotas `/api/*` | **ALTA** | `index.ts:92` |
| V-04 | Cache Redis envenenável como vetor de escalada de privilégio | **ALTA** | `clerk-auth.middleware.ts:46` |
| V-05 | Credenciais padrão de RabbitMQ hardcoded | **ALTA** | `index.ts:212` |
| V-06 | Signature Middleware desabilitado | **ALTA** | `index.ts:114` |
| V-07 | Execute Payout vaza `provider` de payouts alheios (IDOR parcial) | **MÉDIA** | `withdraw.routes.ts:196` |
| V-08 | Deposit — walletAddress controlado pelo cliente, não pela sessão | **MÉDIA** | `deposit.routes.ts:81` |
| V-09 | Endpoints de health públicos expõem infraestrutura | **MÉDIA** | `index.ts:198` |
| V-10 | QR Code endpoint público sem rate limit — enumeração de pagamentos | **MÉDIA** | `deposit.routes.ts:202` |
| V-11 | Mensagens WebSocket sem validação de schema | **MÉDIA** | `index.ts:368` |
| V-12 | Webhook alias com typo permanentemente registrado | **BAIXA** | `webhooks.routes.ts:148` |
| V-13 | Métricas Prometheus públicas | **BAIXA** | `index.ts:249` |
| V-14 | `parseInt(NaN)` em payoutId pode causar comportamento inesperado | **BAIXA** | `withdraw.routes.ts:197` |

---

## Vulnerabilidades Críticas

### V-01 — WebSocket sem autenticação: qualquer atacante pode receber notificações de outro usuário

**Arquivo:** `index.ts`
**Linhas:** `477–525`
**CVSS estimado:** 9.1 (AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N)

#### Código Vulnerável

```typescript
// index.ts:494
userId = url.searchParams.get("userId");

// index.ts:503–505
if (userId) {
  registeredUserId = userId;
  paymentSocketService.registerConnection(userId, ws as any);
```

#### Problema

O endpoint `/ws` **não exige autenticação Clerk**. O `userId` é simplesmente lido da query string `?userId=<walletAddress>` sem qualquer verificação de que esse endereço de carteira pertence à sessão do solicitante.

#### Ataque

```bash
# Atacante sabe o endereço de carteira da vítima (endereços Solana são públicos on-chain)
wscat -c "wss://api.oraculo.com/ws?userId=HkFEQExLczMGHyBKVzE8P1K9...VICTIM_WALLET"
```

O atacante recebe em tempo real:
- Todos os eventos `pix_usdc_payment_status` da vítima
- Eventos `pix_usdc_amount_credited` com valor e timestamp
- Eventos `payout_status_update` com `payoutId`, stage, e mensagens
- Qualquer evento futuro registrado no payment socket

#### Impacto

- **Espionagem financeira completa** em tempo real de qualquer usuário
- `payoutId` exposto permite ataques secundários de timing (saber exatamente quando o USDC foi enviado)
- Endereços Solana são públicos — a vítima não precisa ser conhecida previamente; basta monitorar transações on-chain para obter o walletAddress

#### Correção

Exigir autenticação Clerk no upgrade WebSocket:

```typescript
// Antes de registrar a conexão, verificar o token:
const authHeader = c.req.header("Authorization");
const token = authHeader?.replace("Bearer ", "");
// Verificar JWT Clerk e comparar userId do token com o userId da query
const auth = getAuth(c); // Clerk middleware deve estar aplicado em /ws
if (!auth?.userId) { ws.close(4001, "Unauthorized"); return; }

// Verificar que o walletAddress pertence ao userId autenticado
const user = await prisma.user.findFirst({ where: { sessionId: auth.userId } });
if (user?.walletAddress?.toLowerCase() !== userId?.toLowerCase()) {
  ws.close(4003, "Forbidden");
  return;
}
```

---

### V-02 — Webhook 3xChange sem HMAC quando secret não está configurado

**Arquivo:** `webhooks.routes.ts` + `env.ts`
**Linhas:** `webhooks.routes.ts:91–116`, `env.ts:110`
**CVSS estimado:** 9.8 (AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N)

#### Código Vulnerável

```typescript
// env.ts:110
THREEXCHANGE_WEBHOOK_SECRET: z.string().optional(),  // ← Optional!

// webhooks.routes.ts:91–92
const secret = env.THREEXCHANGE_WEBHOOK_SECRET;
if (secret) {  // ← Se não configurado, executa SEM validação de assinatura
  // ... HMAC check
}
// Se secret=undefined: cai direto para processamento
const useCase = new ThreeXChangeWebhookUseCase();
const result = await useCase.execute(payload as any);
```

#### Ataque

Se `THREEXCHANGE_WEBHOOK_SECRET` não estiver configurado em produção:

```bash
# Atacante forja evento de payout completed para um payout específico
curl -X POST https://api.oraculo.com/api/v1/payments/webhook/3xchange \
  -H "Content-Type: application/json" \
  -d '{
    "type": "PAYOUT",
    "event": "order.completed",
    "status": "COMPLETED",
    "orderId": "ANY_ORDER_ID_FROM_ENUMERATION",
    "pixEndToEndId": "FAKE_E2E_ID",
    "timestamp": "2026-03-23T00:00:00Z"
  }'
```

Resultado:
1. `payout.status` atualizado para `payout_completed`
2. Evento `pix_usdc_amount_credited` enviado ao usuário via WebSocket
3. **Sem um único centavo ter sido transferido**

Combinando com V-01 (WebSocket sem auth), o atacante pode monitorar payoutIds ativos e acionar completions falsas.

#### Impacto

- Confirmação de saques que nunca aconteceram
- Notificações falsas ao usuário indicando que o PIX foi recebido
- Possível bypass de qualquer lógica downstream que dependa do status `payout_completed`

#### Correção

```typescript
// env.ts — tornar obrigatório
THREEXCHANGE_WEBHOOK_SECRET: z.string().min(32),

// webhooks.routes.ts — falhar explicitamente se não configurado
if (!secret) {
  loggers.app.error("THREEXCHANGE_WEBHOOK_SECRET não configurado — rejeitando webhook");
  return c.json({ error: "Webhook validation not configured" }, 500);
}
```

---

## Vulnerabilidades Altas

### V-03 — CORS wildcard em todas as rotas da API

**Arquivo:** `index.ts`
**Linha:** `92`

#### Código Vulnerável

```typescript
app.use("/api/*", cors()); // sem opções = Access-Control-Allow-Origin: *
```

#### Problema

`cors()` sem configuração retorna `Access-Control-Allow-Origin: *`. Isso significa que qualquer site malicioso pode fazer requisições autenticadas às rotas `/api/*` usando as credenciais Clerk do usuário logado (cookies/tokens armazenados no browser).

#### Ataque (CSRF via CORS)

```html
<!-- site malicioso embebido em phishing -->
<script>
fetch("https://api.oraculo.com/api/v1/payments/withdraw/execute", {
  method: "POST",
  credentials: "include",   // envia cookies/JWT armazenados
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify({ payoutId: "DISCOVERED_ID" })
});
</script>
```

Se o usuário acessar o site malicioso enquanto logado, o saque pode ser disparado.

#### Correção

```typescript
app.use("/api/*", cors({
  origin: ["https://app.oraculo.com", "https://oraculo.com"],
  allowMethods: ["GET", "POST", "PUT", "DELETE", "OPTIONS"],
  allowHeaders: ["Content-Type", "Authorization"],
  credentials: true,
}));
```

---

### V-04 — Cache Redis envenenável para escalada de privilégio

**Arquivo:** `src/shared/middleware/clerk-auth.middleware.ts`
**Linhas:** `46–54`

#### Código Vulnerável

```typescript
if (cachedUser) {
  try {
    const userData = typeof cachedUser === 'string' ? JSON.parse(cachedUser) : cachedUser;
    return {
      userId: userData.id,       // ← Vem do Redis, não do Clerk
      sessionId: userData.sessionId,
      username: userData.username,
      wallet: userData.walletAddress,
    };
  }
```

#### Problema

O `userId` retornado ao sistema **vem do cache Redis**, não do token Clerk. O token Clerk é validado apenas na primeira vez (quando o cache está frio). Nas próximas 300 segundos, qualquer que seja o conteúdo do Redis para a key `user:<clerk_session_id>`, ele será usado como identidade.

#### Vetor de Ataque (Redis SSRF / MITM)

1. Atacante obtém SSRF para o Redis (via header injection, ou Redis exposto sem auth)
2. `SET "astron:user:clerk_session_do_admin" '{"id":1,"sessionId":"clerk_session_do_admin","walletAddress":"hacker_wallet"}'`
3. Na próxima request do admin, `userId=1` mas `walletAddress=hacker_wallet` — acesso de admin com wallet do atacante

#### Impacto

- Sem SSRF: risco médio (Redis tipicamente não exposto)
- Com SSRF ou Redis sem autenticação: **bypass completo de autenticação** por 5 minutos

#### Correção

```typescript
// Adicionar validação cruzada: o sessionId do cache deve bater com o do Clerk
if (userData.sessionId !== auth.userId) {
  // Cache inválido/envenenado, invalidar e buscar do banco
  await redisService.del(cacheKey);
  // ... buscar do banco
}
```

---

### V-05 — Credenciais padrão de RabbitMQ hardcoded

**Arquivo:** `index.ts`
**Linha:** `212`

#### Código Vulnerável

```typescript
const RABBITMQ_URL = process.env.RABBITMQ_URL || "amqp://user:password@localhost:5672";
```

#### Problema

1. **Credenciais padrão `user:password`** no código-fonte — se o env var não for definido em qualquer ambiente, RabbitMQ usa essas credenciais fracas
2. O endpoint `/health/rabbitmq` é **público, sem autenticação** e retorna `error.message` se a conexão falhar, podendo vazar a URL completa (incluindo credenciais) em mensagens de erro
3. O código-fonte está em um repositório privado, mas é um risco de supply chain se houver leak

#### Correção

```typescript
const RABBITMQ_URL = process.env.RABBITMQ_URL;
if (!RABBITMQ_URL) throw new Error("RABBITMQ_URL not configured");
```

E proteger o endpoint `/health/rabbitmq` com autenticação interna ou removê-lo de produção.

---

### V-06 — Signature Middleware desabilitado

**Arquivo:** `index.ts`
**Linha:** `114`

#### Código Vulnerável

```typescript
// Signature validation middleware
// app.use("/api/*", signatureMiddleware);  ← COMENTADO
```

#### Problema

O `signatureMiddleware` foi implementado (existe em `src/shared/middleware/signature.middleware.ts`) mas está comentado em produção. Esse middleware validaria assinaturas de request para prevenir replay attacks e garantir a integridade das requisições. Sem ele:

- Requests capturados podem ser repetidos indefinidamente (replay attacks)
- Man-in-the-middle pode modificar requisições em trânsito sem detecção

---

## Vulnerabilidades Médias

### V-07 — Execute Payout: vaza provider de payouts de outros usuários (IDOR parcial)

**Arquivo:** `src/modules/payments/routes/withdraw.routes.ts`
**Linhas:** `196–205`

#### Código Vulnerável

```typescript
// Linha 196–199: busca por ID sem verificar userId
const payout = await prisma.payout.findUnique({
  where: { id: parseInt(input.payoutId) },
  select: { provider: true },  // ← qualquer payoutId funciona
});

// Linha 201–205: responde de forma diferente baseado no provider
if (payout?.provider === "3xchange") {
  // ... fork de provider
}
```

#### Problema

Qualquer usuário autenticado pode enviar `POST /execute` com o `payoutId` de outro usuário e descobrir qual provider foi usado (`3xchange` vs `blindpay`). O ownership check acontece dentro do use-case, mas **o response de erro é diferente** dependendo do provider (401 "não pertence ao usuário" vs 400 "saque não está pronto"), vazando informação.

Adicionalmente, IDs são sequenciais (`parseInt`), facilitando enumeração de todos os payouts do sistema.

#### Impacto

- Enumeração de todos os payoutIds e providers do sistema
- Timing side-channel: diferença de latência entre "payout não encontrado" e "não pertence ao usuário"

#### Correção

```typescript
const payout = await prisma.payout.findUnique({
  where: { id: parseInt(input.payoutId), userId: Number(user.userId) }, // ← sempre com userId
  select: { provider: true },
});
if (!payout) return errorResponse(c, "Payout não encontrado", null, 404);
```

---

### V-08 — Deposit: walletAddress controlado pelo cliente

**Arquivo:** `src/modules/payments/routes/deposit.routes.ts`
**Linha:** `81`

#### Código Vulnerável

```typescript
// Rota: POST /payment/:walletAddress
const walletAddress = c.req.param("walletAddress");  // ← do path, não da sessão
const paymentUseCase = new CreateDepositPaymentUseCase(depositRepository);
const result = await paymentUseCase.execute(
  { ...input, walletAddress },   // ← passa walletAddress não-verificado
  user.userId
);
```

#### Problema

O `walletAddress` vem do path parameter, não da carteira autenticada do usuário. Um usuário pode criar um pagamento PIX associado à carteira de outro usuário. Se a lógica interna não cruzar `walletAddress` com `userId`, o USDC creditado pode ir para a carteira errada.

#### Investigar

É necessário verificar se `CreateDepositPaymentUseCase` valida que `walletAddress` pertence ao `userId` da sessão.

---

### V-09 — Endpoints de health públicos expõem infraestrutura

**Arquivo:** `index.ts`
**Linhas:** `198–244`

```typescript
app.get("/health/redis", async (c) => { /* retorna redis pong */ });
app.get("/health/rabbitmq", async (c) => {
  // ...
  return c.json({ status: "error", error: error?.message ?? String(error) }, 503);
  // error.message pode conter: "connect ECONNREFUSED 10.0.1.45:5672"
  // → revela IP interno, porta, tipo de serviço
});
app.get("/health/matching", async (c) => {
  return c.json({ rabbitmq: ..., circuitBreakers: getBreakerStates(), pendingOrders: ... });
  // → revela estado interno dos circuit breakers e volume de ordens pendentes
});
```

#### Impacto

- Mapeamento da infraestrutura interna (IPs, portas, serviços)
- Estado dos circuit breakers revela comportamento do sistema sob carga
- Volume de ordens pendentes revela carga do sistema em tempo real

#### Correção

Proteger com IP allowlist ou token de autenticação:

```typescript
app.get("/health/*", (c, next) => {
  const token = c.req.header("X-Health-Token");
  if (token !== process.env.HEALTH_SECRET) return c.json({ error: "Forbidden" }, 403);
  return next();
});
```

---

### V-10 — QR Code público sem rate limit: enumeração de pagamentos

**Arquivo:** `src/modules/payments/routes/deposit.routes.ts`
**Linhas:** `202–238`

#### Código Vulnerável

```typescript
// Sem autenticação, sem rate limit
depositRoutes.get("/qrcode/:externalId", async (c) => {
  const externalId = c.req.param("externalId");
  const payment = await prisma.pixUsdcPayment.findFirst({
    where: { externalId },
  });
  // ...gera e retorna imagem PNG
});
```

#### Problema

- Endpoint completamente público (intencionalmente, para exibir QR codes)
- `externalId` é provavelmente um UUID ou ID gerado pela 3xChange/Blindpay
- Sem rate limit: permite scanning massivo de IDs
- Retorna `404` para IDs inexistentes vs `200` com imagem para IDs válidos: **oracle para enumeração de todos os pagamentos ativos do sistema**

#### Impacto

- Descoberta de todos os pagamentos PIX ativos (com QR code) do sistema
- QR codes PIX expirados mas ainda no banco são expostos
- Volume de pagamentos ativos revela métricas de negócio

---

### V-11 — Mensagens WebSocket sem validação de schema

**Arquivo:** `index.ts`
**Linhas:** `368–474`

#### Código Vulnerável

```typescript
const data = JSON.parse(event.data as string);

// Linha 386-390: tokenAddress sem sanitização
webSocketService.subscribeToToken(
  tokenAddress,   // ← direto do client, sem validação
  ws as any,
  resolution,     // ← direto do client
  callback        // ← direto do client
);
```

#### Problema

- `tokenAddress` e `resolution` são usados diretamente sem validação
- `lookbackSeconds` (linha 447): `typeof data.lookbackSeconds === "number"` mas sem limite máximo — um atacante pode solicitar `lookbackSeconds: 999999999` causando query massiva no Redis/banco
- `callback` do client passado para `subscribeToToken` — dependendo da implementação, pode ser injetável

#### Ataque

```json
{"type": "subscribe_btc_price", "startTimestamp": 1, "lookbackSeconds": 2147483647}
```

Potencial: query enorme ou loop infinito no servidor.

---

## Vulnerabilidades Baixas

### V-12 — Webhook alias com typo permanentemente registrado

**Arquivo:** `src/modules/payments/routes/webhooks.routes.ts`
**Linhas:** `148–149`

```typescript
// "Alias para URL com typo cadastrada no painel da 3xChange"
webhookRoutes.post("/3xc%20%20hange", handle3xChangeWebhook);
webhookRoutes.post("/3xc  hange", handle3xChangeWebhook);
```

O comentário diz "Remover após corrigir a URL no painel". Se nunca removido, cria uma superfície adicional de ataque — qualquer request para essas URLs bypassa qualquer WAF que filtre por path, pois o path com espaços/encoding é incomum.

---

### V-13 — Métricas Prometheus públicas

**Arquivo:** `index.ts`
**Linhas:** `249–254`

```typescript
app.get("/metrics", async (c) => {
  const { contentType, body } = await getMetricsContent();
  return c.text(body, 200, { "Content-Type": contentType });
});
```

Endpoint `/metrics` sem autenticação expõe:
- Latência de endpoints (revela quais rotas estão sendo mais usadas)
- Contadores de erros por rota
- Dados de performance que ajudam a planejar ataques DoS

---

### V-14 — parseInt com NaN em payoutId

**Arquivo:** `src/modules/payments/routes/withdraw.routes.ts`
**Linha:** `197`

```typescript
const payout = await prisma.payout.findUnique({
  where: { id: parseInt(input.payoutId) },  // parseInt("abc") = NaN
```

`parseInt("abc")` retorna `NaN`. Prisma provavelmente lança um erro, mas o tratamento de erro retorna `error.message` na response HTTP, potencialmente expondo detalhes internos da query.

Usar Zod para validar que `payoutId` é numérico antes de usar.

---

## Resumo Executivo e Priorização

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ AÇÃO IMEDIATA (produção em risco agora)                                     │
├─────────────────────────────────────────────────────────────────────────────┤
│ 1. V-01: Adicionar auth Clerk no /ws endpoint                               │
│ 2. V-02: Tornar THREEXCHANGE_WEBHOOK_SECRET obrigatório                     │
│ 3. V-05: Remover credenciais default de RabbitMQ do código                  │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ ESTA SEMANA                                                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│ 4. V-03: Configurar CORS com allowlist de domínios                          │
│ 5. V-09: Proteger /health/* com token secreto                               │
│ 6. V-07: Adicionar userId no WHERE de payout lookup em /execute             │
└─────────────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────────────┐
│ PRÓXIMO SPRINT                                                               │
├─────────────────────────────────────────────────────────────────────────────┤
│ 7. V-04: Validação cruzada do cache Redis com sessionId do Clerk            │
│ 8. V-08: Verificar que walletAddress em /deposit pertence ao usuário        │
│ 9. V-11: Adicionar Zod schema para mensagens WebSocket recebidas            │
│ 10. V-06: Reabilitar signature middleware (ou documentar por que está off)  │
│ 11. V-10: Adicionar rate limit no endpoint de QR code público               │
│ 12. V-12: Remover rotas alias de typo após corrigir URL no painel           │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Notas sobre o que está BEM implementado

Para fairness, vale registrar as proteções que estão corretas:

- ✅ **CAS lock atômico no execute-payout-3x** (`WHERE status = 'payout_order_placed' AND locked_at IS NULL`) — previne double-spend corretamente
- ✅ **Verificação ao vivo na 3xChange antes de enviar USDC** — não confia apenas no banco
- ✅ **Security alert se depositAddress divergir** entre banco e 3xChange
- ✅ **`timingSafeEqual` para comparação HMAC** — previne timing attacks na validação de webhooks Blindpay
- ✅ **Deduplicação de webhooks por status** — previne processamento duplo de eventos
- ✅ **Redis Pub/Sub para WebSocket multi-instância** — arquitetura correta para ECS
- ✅ **Idempotência no execute-payout com `TERMINAL_OR_INFLIGHT`** — retorna resultado existente sem re-executar

---

*Relatório gerado por análise estática de código. Não foram executados testes de penetração ativos contra infraestrutura de produção.*
