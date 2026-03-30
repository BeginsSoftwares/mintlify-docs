# Plano de Remediação de Segurança — Oraculo Platform

> **Criado em:** 2026-03-23
> **Baseado em:** `docs/security-audit.md`
> **Total de vulnerabilidades:** 14
> **Prazo estimado:** 3–4 semanas

---

## Visão Geral das Fases

```
Fase 1 — Crítica     (dias 1–3)   : V-14, V-07, V-05, V-02
Fase 2 — Alta        (dias 3–7)   : V-04, V-03, V-09, V-13
Fase 3 — Média       (semana 2)   : V-10, V-11, V-08, V-06
Fase 4 — Baixa       (semana 3–4) : V-01, V-12
```

> **Por que V-01 (WebSocket) está na Fase 4 mesmo sendo crítica?**
> É a mudança de maior blast radius — requer coordenação backend + frontend + configuração Clerk. As fases 1–3 eliminam vulnerabilidades exploráveis com zero breaking changes. V-01 precisa de um sprint dedicado para não derrubar produção.

---

## Tabela de Esforço e Risco

| ID | Vulnerabilidade | Severidade | Esforço | Risco Regressão | Fase |
|----|----------------|-----------|---------|-----------------|------|
| V-14 | parseInt NaN no payoutId | Baixa | Trivial | Nenhum | 1 |
| V-07 | IDOR: execute payout sem userId no WHERE | Média | Trivial | Nenhum | 1 |
| V-05 | RabbitMQ credentials hardcoded | Alta | Pequeno | Baixo | 1 |
| V-02 | Webhook 3xChange HMAC opcional | Crítica | Pequeno | Baixo | 1 |
| V-04 | Redis cache envenenável | Alta | Pequeno | Muito Baixo | 2 |
| V-03 | CORS wildcard | Alta | Pequeno | Médio | 2 |
| V-09 | /health/* público | Média | Pequeno | Baixo | 2 |
| V-13 | /metrics público | Baixa | Trivial | Baixo | 2 |
| V-10 | QR code sem rate limit | Média | Pequeno | Nenhum | 3 |
| V-11 | WS messages sem schema | Média | Pequeno | Nenhum | 3 |
| V-08 | Deposit walletAddress não verificado | Média | Pequeno | Baixo | 3 |
| V-06 | Signature middleware desabilitado | Alta | Médio | Baixo | 3 |
| V-01 | WebSocket sem autenticação | Crítica | Grande | Alto | 4 |
| V-12 | Webhook alias typo permanente | Baixa | Trivial | Nenhum | 4 |

---

## Fase 1 — Crítica (Dias 1–3)

### ✅ V-14 — `parseInt(payoutId)` sem validação de NaN

**Arquivo:** `src/modules/payments/routes/withdraw.routes.ts` linha 197
**Esforço:** 5 min
**Risco de regressão:** Nenhum

**Problema:** `parseInt("abc")` = `NaN`. Prisma lança erro interno → response 500 com `error.message` potencialmente expondo detalhes da query.

**Fix:**
```typescript
// Linha 197 — substituir:
where: { id: parseInt(input.payoutId) },

// Por:
const payoutIdNum = parseInt(input.payoutId, 10);
if (isNaN(payoutIdNum) || payoutIdNum <= 0) {
  return errorResponse(c, "payoutId inválido", null, 400);
}
where: { id: payoutIdNum },
```

**Teste:** `POST /execute` com `payoutId: "abc"` → deve retornar 400, não 500.

---

### ✅ V-07 — IDOR: execute payout busca provider sem userId

**Arquivo:** `src/modules/payments/routes/withdraw.routes.ts` linhas 207–210
**Esforço:** 5 min
**Risco de regressão:** Nenhum

**Problema:** Qualquer usuário autenticado descobre o `provider` de payouts de outros usuários passando IDs sequenciais.

**Fix:**
```typescript
// Substituir:
const payout = await prisma.payout.findUnique({
  where: { id: parseInt(input.payoutId) },
  select: { provider: true },
});

// Por:
const payout = await prisma.payout.findFirst({
  where: {
    id: parseInt(input.payoutId, 10),
    userId: Number(user.userId), // escopo ao usuário autenticado
  },
  select: { provider: true },
});

if (!payout) {
  return errorResponse(c, "Payout não encontrado", null, 404);
}
```

**Teste:** Usuário A tenta `/execute` com `payoutId` do usuário B → deve retornar 404 (não 200 ou 403 com info do provider).

---

### ✅ V-05 — Credenciais RabbitMQ hardcoded como fallback

**Arquivos:**
- `src/shared/utils/env.ts` — adicionar ao schema Zod
- `index.ts` linha 216 — remover fallback hardcoded
- `src/shared/utils/rabbitmq-connection-pool.ts` linhas 4–5 — remover fallback hardcoded

**Esforço:** 20 min + tarefa DevOps
**Risco de regressão:** Baixo (apenas ambientes sem `RABBITMQ_URL` configurada vão falhar no boot — intencional)

**Fix em `env.ts`:**
```typescript
// Adicionar ao schema Zod (obrigatório):
RABBITMQ_URL: z.string().url(),
```

**Fix em `index.ts` linha 216:**
```typescript
// Remover:
const RABBITMQ_URL = process.env.RABBITMQ_URL || "amqp://user:password@localhost:5672";

// Substituir por:
import { env } from "./src/shared/utils/env";
const RABBITMQ_URL = env.RABBITMQ_URL;
```

**Fix em `rabbitmq-connection-pool.ts`:**
```typescript
// Remover fallback e usar env:
import { env } from "@/shared/utils/env";
const RABBITMQ_URL = env.RABBITMQ_URL;
```

**Tarefa DevOps:** Verificar que `RABBITMQ_URL` está configurada em todos os ambientes (staging, produção, CI) **antes** de fazer deploy desta mudança.

**Teste:** Iniciar servidor sem `RABBITMQ_URL` → deve falhar no boot com erro claro, não conectar silenciosamente com credenciais padrão.

---

### ✅ V-02 — Webhook 3xChange sem HMAC quando secret não configurado

**Arquivos:**
- `src/shared/utils/env.ts` linha 110
- `src/modules/payments/routes/webhooks.routes.ts` linhas 91–116

**Esforço:** 15 min + tarefa DevOps
**Risco de regressão:** Baixo (requer secret configurado no painel da 3xChange)

**Problema:** `THREEXCHANGE_WEBHOOK_SECRET: z.string().optional()` → se não configurado, linha 91 `if (secret)` é `false` e o webhook processa **sem validação de assinatura**.

**Fix em `env.ts`:**
```typescript
// Linha 110 — mudar de:
THREEXCHANGE_WEBHOOK_SECRET: z.string().optional(),

// Para:
THREEXCHANGE_WEBHOOK_SECRET: z.string().min(32),
```

**Fix em `webhooks.routes.ts`:**
```typescript
// Remover o `if (secret)` condicional:
const secret = env.THREEXCHANGE_WEBHOOK_SECRET; // agora sempre definido

// Tornar a validação HMAC incondicional:
const isValidHmac = await validateHmac(rawBody, signature, secret);
if (!isValidHmac) {
  return c.json({ error: "Invalid signature" }, 401);
}
```

**Tarefa DevOps:**
1. Gerar um secret com ≥32 chars: `openssl rand -hex 32`
2. Configurar `THREEXCHANGE_WEBHOOK_SECRET=<secret>` em produção e staging
3. Registrar o mesmo secret no painel da 3xChange como webhook signing secret
4. Testar com um webhook manual antes do deploy

**Teste:** POST webhook sem header de assinatura → 401. POST com assinatura inválida → 401. POST com assinatura válida → 200.

---

## Fase 2 — Alta (Dias 3–7)

### ✅ V-04 — Redis cache: userId sem validação cruzada com Clerk

**Arquivo:** `src/shared/middleware/clerk-auth.middleware.ts` linhas 46–62
**Esforço:** 1h (incluindo testes)
**Risco de regressão:** Muito baixo (apenas rejeita entradas inválidas)

**Fix:**
```typescript
if (cachedUser) {
  const userData = typeof cachedUser === 'string' ? JSON.parse(cachedUser) : cachedUser;

  // Validação cruzada: sessionId no cache DEVE bater com o userId do Clerk token
  if (userData.sessionId !== auth.userId) {
    loggers.auth.error(
      { cached: userData.sessionId, token: auth.userId },
      "Possível cache poisoning detectado — invalidando entrada"
    );
    await redisService.del(cacheKey).catch(() => {});
    // Cair no path de leitura do banco de dados (sem retornar aqui)
  } else {
    return {
      userId: userData.id,
      sessionId: userData.sessionId,
      username: userData.username,
      wallet: userData.walletAddress,
    };
  }
}
```

**Teste:** Inserir manualmente no Redis uma entrada com `sessionId` diferente do `auth.userId` → deve invalidar cache e buscar do banco, não retornar o dado adulterado.

---

### ✅ V-03 — CORS wildcard sem origin allowlist

**Arquivo:** `index.ts` linha 93
**Esforço:** 30 min + mapeamento de origens
**Risco de regressão:** Médio (qualquer origem legítima não listada vai quebrar)

**Antes de implementar:** Mapear todas as origens legítimas:
- [ ] App web principal
- [ ] Domínio de staging
- [ ] App mobile (se usar webview)
- [ ] Dashboard admin
- [ ] Scripts internos (CI/ferramentas internas)

**Fix em `env.ts`:**
```typescript
// Adicionar:
CORS_ALLOWED_ORIGINS: z.string().default("https://app.oraculo.gg,https://oraculo.gg"),
```

**Fix em `index.ts`:**
```typescript
// Substituir:
app.use("/api/*", cors());

// Por:
app.use("/api/*", cors({
  origin: env.CORS_ALLOWED_ORIGINS.split(",").map(s => s.trim()),
  allowHeaders: ["Authorization", "Content-Type", "X-Idempotency-Key"],
  allowMethods: ["GET", "POST", "PUT", "DELETE", "OPTIONS"],
  credentials: true,
  maxAge: 86400,
}));
```

**Teste:** Request de `http://evil.com` para qualquer rota `/api/*` → header `Access-Control-Allow-Origin` não deve estar presente na resposta. Request de `https://app.oraculo.gg` → deve estar presente.

---

### ✅ V-09 — /health/* e V-13 — /metrics sem autenticação

**Arquivo:** `index.ts` linhas 198–258
**Esforço:** 30 min + configurar token no sistema de monitoring
**Risco de regressão:** Baixo (se token for opcional em dev)

**Fix em `env.ts`:**
```typescript
HEALTH_CHECK_TOKEN: z.string().optional(), // opcional para facilitar dev local
```

**Fix em `index.ts` — adicionar middleware antes dos endpoints:**
```typescript
const healthAuthMiddleware = async (c: Context, next: Next) => {
  const token = env.HEALTH_CHECK_TOKEN;
  if (!token) return next(); // dev: sem token configurado = sem proteção (aceitável)
  const auth = c.req.header("Authorization");
  if (auth !== `Bearer ${token}`) {
    return c.json({ error: "Unauthorized" }, 401);
  }
  return next();
};

// Aplicar nos endpoints sensíveis (manter /health raiz público para load balancer):
app.use("/health/redis", healthAuthMiddleware);
app.use("/health/rabbitmq", healthAuthMiddleware);
app.use("/health/matching", healthAuthMiddleware);
app.use("/metrics", healthAuthMiddleware);
```

**Tarefa DevOps:**
1. Gerar token: `openssl rand -hex 16`
2. Configurar `HEALTH_CHECK_TOKEN=<token>` em produção
3. Atualizar Prometheus scrape config para incluir `bearer_token: <token>`
4. Atualizar qualquer outra ferramenta de monitoring que chame esses endpoints

**Teste:** GET `/health/redis` sem token → 401 em produção (onde `HEALTH_CHECK_TOKEN` está configurado). GET `/health` (raiz) → 200 sempre.

---

## Fase 3 — Média (Semana 2)

### ✅ V-10 — QR code público sem rate limit

**Arquivo:** `src/modules/payments/routes/deposit.routes.ts` linha 202
**Esforço:** 15 min
**Risco de regressão:** Nenhum

**Fix:**
```typescript
// Adicionar em rate-limiter.ts:
export const qrCodeRateLimit = rateLimiter({
  windowMs: 60 * 1000,    // janela de 1 minuto
  maxRequests: 20,         // máximo 20 QR codes por minuto por IP
  message: "Muitas requisições. Aguarde.",
  keyGenerator: (c) => `qrcode:${c.req.header("x-forwarded-for") || "unknown"}`,
});

// Em deposit.routes.ts linha 202:
depositRoutes.get("/qrcode/:externalId", qrCodeRateLimit, async (c) => { ... });
```

**Verificar também:** Se `externalId` é UUID v4 aleatório (bom) ou ID sequencial (ruim — rate limit sozinho não é suficiente).

---

### ✅ V-11 — WebSocket: mensagens sem validação de schema

**Arquivo:** `index.ts` linhas 368–474
**Esforço:** 30 min
**Risco de regressão:** Nenhum para uso legítimo

**Fix — adicionar limite no `lookbackSeconds`:**
```typescript
// Linha 450 — adicionar:
const MAX_LOOKBACK_SECONDS = 3600; // máximo 1 hora de histórico

const lookbackSeconds = typeof data.lookbackSeconds === "number" && data.lookbackSeconds > 0
  ? Math.min(data.lookbackSeconds, MAX_LOOKBACK_SECONDS)
  : 600;
```

**Fix — validar `tokenAddress` e `resolution`:**
```typescript
// Antes de passar para subscribeToToken:
const tokenAddress = typeof data.tokenAddress === "string" && data.tokenAddress.length <= 44
  ? data.tokenAddress
  : null;

if (!tokenAddress) {
  ws.send(JSON.stringify({ type: "error", message: "tokenAddress inválido" }));
  return;
}

const VALID_RESOLUTIONS = ["1m", "5m", "15m", "1h", "4h", "1d"] as const;
const resolution = VALID_RESOLUTIONS.includes(data.resolution) ? data.resolution : "1m";
```

---

### ✅ V-08 — Deposit: walletAddress do path não verificado contra sessão

**Arquivo:** `src/modules/payments/routes/deposit.routes.ts` linhas 66–100
**Esforço:** 30 min (incluindo verificação dos callers no frontend)
**Risco de regressão:** Baixo

**Antes de implementar:** Verificar no frontend (`useBuyFlow.ts`, `deposit-modal.tsx`) se o `walletAddress` passado na URL já é sempre o da carteira autenticada. Se sim, a validação só confirma o comportamento atual.

**Fix:**
```typescript
const walletAddress = c.req.param("walletAddress");

// Validar ownership — carteira deve pertencer ao usuário autenticado
if (user.wallet && walletAddress.toLowerCase() !== user.wallet.toLowerCase()) {
  return errorResponse(c, "Carteira não pertence ao usuário autenticado", null, 403);
}
```

**Alternativa mais segura:** Derivar `walletAddress` da sessão, não do path:
```typescript
// Remover do path e usar diretamente da sessão:
const walletAddress = user.wallet;
if (!walletAddress) {
  return errorResponse(c, "Carteira não configurada para este usuário", null, 400);
}
```

---

### ✅ V-06 — Signature Middleware desabilitado

**Arquivos:**
- `index.ts` linha 114
- `src/shared/middleware/signature.middleware.ts` linhas 172–174

**Esforço:** 2–3h
**Risco de regressão:** Baixo se aplicado apenas a `/admin/*` (rotas admin já têm auth própria)

**Contexto:** O middleware está comentado porque o frontend nunca foi atualizado para enviar o header `Signature`. Reativar globalmente quebraria 100% das rotas de frontend imediatamente.

**Fix em `env.ts`:**
```typescript
// Tornar obrigatório:
SIGNATURE_SECRET_KEY: z.string().min(32),
```

**Fix em `signature.middleware.ts` linha 172–174:**
```typescript
// Remover fallback inseguro:
secretKey: env.SIGNATURE_SECRET_KEY, // sem fallback
```

**Fix em `index.ts` linha 114:**
```typescript
// Em vez de reativar globalmente, aplicar apenas em rotas admin:
// app.use("/api/*", signatureMiddleware);  ← continua comentado
app.use("/api/v1/admin/*", signatureMiddleware); // ← reativar apenas para admin
```

**Tarefa futura:** Para proteger todas as rotas, o frontend precisa enviar `Signature: <nonce>:<hmac>` em cada request. Isso é um projeto separado (fase 5 futura).

---

## Fase 4 — Baixa (Semanas 3–4)

### ✅ V-01 — WebSocket sem autenticação Clerk

> Esta é a vulnerabilidade mais crítica, mas tem o maior impacto de implementação — requer mudanças coordenadas em backend e frontend.

**Arquivos afetados:**
- `index.ts` linhas 337–539 (servidor WebSocket)
- `web/lib/services/socket.ts` linhas 63, 150 (cliente)
- `web/components/withdraw/withdraw-modal.tsx` linha 121
- `web/lib/hooks/useBuyFlow.ts` linhas 38, 337

**Estratégia:** Usar JWT Clerk via query param `?token=<jwt>` na URL do WebSocket. Browsers não suportam headers customizados no upgrade WebSocket, mas suportam query params. O servidor valida o JWT via `createClerkClient().verifyToken(token)`.

**Fix no servidor (`index.ts`):**
```typescript
import { createClerkClient } from "@clerk/backend";

const clerkClient = createClerkClient({ secretKey: env.CLERK_SECRET_KEY });

// Dentro do callback upgradeWebSocket, antes de registrar a conexão:
onOpen: async (evt, ws) => {
  try {
    const url = new URL(requestUrl, "http://localhost");
    const token = url.searchParams.get("token");

    if (!token) {
      ws.close(4001, "Missing authentication token");
      return;
    }

    // Verificar JWT Clerk
    const payload = await clerkClient.verifyToken(token).catch(() => null);
    if (!payload) {
      ws.close(4001, "Invalid or expired token");
      return;
    }

    // Buscar o usuário no banco para obter walletAddress verificado
    const dbUser = await prisma.user.findFirst({
      where: { clerkId: payload.sub },
      select: { walletAddress: true },
    });

    if (!dbUser?.walletAddress) {
      ws.close(4003, "User wallet not found");
      return;
    }

    // Registrar com walletAddress do banco (não do query param do cliente)
    registeredUserId = dbUser.walletAddress;
    paymentSocketService.registerConnection(dbUser.walletAddress, ws as any);
  } catch (error) {
    ws.close(4000, "Authentication error");
  }
}
```

**Fix no frontend (`socket.ts`):**
```typescript
// connectPixUsdcPayment precisa receber o getToken além do walletAddress
async connectPixUsdcPayment(walletAddress: string, getToken: () => Promise<string | null>) {
  const token = await getToken();
  if (!token) { /* handle erro */ return; }

  // Incluir token na URL do WebSocket (não walletAddress — servidor deriva do token)
  const wsUrl = `${this.baseWsUrl}/ws?token=${encodeURIComponent(token)}`;
  // ... resto da lógica
}
```

**Fix nos componentes:** Propagar `getToken` (de `useAuth()` do Clerk) para onde `connectPixUsdcPayment` é chamado.

**Testes obrigatórios:**
1. Conectar sem `?token` → WS fecha com code 4001
2. Conectar com JWT expirado → WS fecha com code 4001
3. Conectar com JWT válido de usuário A mas espiar eventos de usuário B → não recebe nada
4. Fluxo completo de payout → eventos chegam corretamente ao usuário correto

---

### ✅ V-12 — Webhook alias com typo registrado

**Arquivo:** `src/modules/payments/routes/webhooks.routes.ts` linhas 147–149
**Esforço:** 2 min (após período de observação)
**Risco de regressão:** Nenhum (após confirmação que a URL corrigida está ativa)

**Processo:**
1. Verificar nos logs por 7 dias se os endpoints com typo ainda recebem tráfego real
2. Se nenhum tráfego: deletar as linhas 147–149
3. Se ainda há tráfego: confirmar com a 3xChange que a URL foi corrigida no painel deles

```typescript
// Remover (após confirmação):
webhookRoutes.post("/3xc%20%20hange", handle3xChangeWebhook);
webhookRoutes.post("/3xc  hange", handle3xChangeWebhook);
```

---

## Checklist de Deploy

### Variáveis de Ambiente a Configurar Antes do Deploy da Fase 1

```bash
# Obrigatórias novas (falha no boot se não configuradas):
THREEXCHANGE_WEBHOOK_SECRET=<openssl rand -hex 32>
RABBITMQ_URL=amqp://<user>:<password>@<host>:<port>

# Opcional (fase 2):
HEALTH_CHECK_TOKEN=<openssl rand -hex 16>
CORS_ALLOWED_ORIGINS=https://app.oraculo.gg,https://oraculo.gg

# Fase 3:
SIGNATURE_SECRET_KEY=<openssl rand -hex 32>
```

### Ordem de Deploy Recomendada

```
[ ] 1. Configurar env vars (DevOps) — ANTES de qualquer deploy
[ ] 2. Testar em staging com as novas env vars
[ ] 3. Deploy Fase 1 (V-14, V-07, V-05, V-02)
[ ] 4. Validar webhooks 3xChange estão funcionando em staging
[ ] 5. Deploy Fase 2 (V-04, V-03, V-09, V-13)
[ ] 6. Validar CORS com o frontend em staging
[ ] 7. Deploy Fase 3 (V-10, V-11, V-08, V-06)
[ ] 8. Deploy Fase 4 — V-01 requer RC separado com frontend + backend
```

---

## Matriz de Teste de Regressão

| Fluxo | Afetado por | Como testar após o fix |
|-------|------------|------------------------|
| Depósito PIX completo | V-03, V-08 | Criar depósito PIX do zero em staging |
| Saque 3xChange completo | V-02, V-07, V-14 | Executar saque de $1 em staging |
| WebSocket de notificações | V-01, V-11 | Abrir modal de saque e verificar eventos |
| Webhooks 3xChange | V-02 | Disparo manual via cURL com HMAC correto |
| Dashboard admin | V-06, V-03 | Verificar todas as ações admin |
| Health checks de infra | V-09, V-13 | Verificar Prometheus/Datadog ainda coletam |

---

## Bônus: O que Está Bem Implementado (não mexer)

- ✅ CAS lock atômico no execute-payout-3x — previne double-spend
- ✅ Live verification na 3xChange antes de enviar USDC
- ✅ `timingSafeEqual` para HMAC do Blindpay webhook
- ✅ Deduplicação de webhooks por status
- ✅ Redis Pub/Sub para WebSocket multi-instância
- ✅ Idempotência no execute-payout com `TERMINAL_OR_INFLIGHT`
- ✅ Security alert se depositAddress divergir do banco

---

*Documento mantido junto ao `security-audit.md`. Marcar itens como `[x]` conforme implementados.*
