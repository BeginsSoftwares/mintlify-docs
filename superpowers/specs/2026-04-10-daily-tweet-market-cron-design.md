# Daily Tweet Market Cron + History Badges

## Goal

Criar mercados `tweet_count` diários (00:00 BRT, janela de 24h) automaticamente para uma lista de contas configurada em `adminSettings`, começando com `@elonmusk`. Mostrar na tela `/tweet-tracker/[marketPda]` badges dos últimos mercados do mesmo usuário (igual ao padrão `CarCountMobileControls`), mas com count numérico em vez de winner.

## Decisões

- **Ranges**: dinâmicos, 5 ranges proporcionais baseados no `resolvedTweetCount` do último mercado do usuário. Multiplicadores: `0.6 / 0.85 / 1.15 / 1.4`. Fallback quando não há histórico: `defaultFinalCount` por conta (25 para Elon).
- **Horário**: pre-create ~5 min antes das 00:00 BRT (BRT = UTC-3, sem DST). Cron roda a cada 1 min.
- **Scoping**: lista de contas em `adminSettings.tweet_count_market_cron_enabled`, iterável.
- **Badges**: count numérico puro (ex: "27", "31"), 7 dias, 2-3 pills recentes + dropdown "Past".

## Backend

### `TweetCountMarketCronService`
Novo arquivo: `api/src/modules/markets/services/tweet-count-market-cron.service.ts`.

- Interval: 60s. Distributed lock `cron:tweet-count-market-creation`.
- Lê `adminSettings.tweet_count_market_cron_enabled`:
  ```ts
  { enabled: boolean, accounts: Array<{
      handle: string, userId: string, displayName?: string,
      image?: string, defaultFinalCount?: number
  }>, seedConfig?: { enabled, totalUsdc, levels, spreadCents } }
  ```
- Lógica por conta:
  1. Calcular `mostRecentMidnightBRT` (início do dia atual BRT) e `nextMidnightBRT`.
  2. Se não existe mercado `tweet_count` com `countStartDate = mostRecentMidnightBRT` **E** `twitterAccountId = X`, cria em modo **recovery** (janela atual, `countStartDate` no passado).
  3. Senão, se `now >= nextMidnightBRT - 5min` e não existe mercado para `nextMidnightBRT`, cria em modo **pre-create**.
  4. Antes de criar: busca último `tweet_count` resolvido para esse `twitterAccountId` com `resolvedTweetCount != null`; computa ranges a partir dele (ou `defaultFinalCount`).
- Chama `new CreateTweetCountMarketUseCase().execute(input, adminWallet)` (context omitido).
- Usa `env.PRICE_MARKET_AUTHORITY_WALLET` como wallet autoritativa (mesmo padrão do car-count cron).

### Refactor `CreateTweetCountMarketUseCase`
Tornar `c: Context` opcional. Pular chamada do `auditLogService` quando ausente.

### Range utility
```ts
function computeProportionalRanges(finalCount: number): TweetRangeInput[] {
  const f = Math.max(1, Math.round(finalCount));
  const mults = [0.6, 0.85, 1.15, 1.4];
  const b = mults.map((m) => Math.round(f * m));
  // enforce strict monotonic +1
  for (let i = 1; i < b.length; i++) if (b[i] <= b[i - 1]) b[i] = b[i - 1] + 1;
  if (b[0] < 1) b[0] = 1;
  for (let i = 1; i < b.length; i++) if (b[i] <= b[i - 1]) b[i] = b[i - 1] + 1;
  return [
    { min: 0, max: b[0], label: "" },
    { min: b[0] + 1, max: b[1], label: "" },
    { min: b[1] + 1, max: b[2], label: "" },
    { min: b[2] + 1, max: b[3], label: "" },
    { min: b[3] + 1, max: null, label: "" },
  ];
}
```

### Endpoint `GET /markets/tweet-count/past`
Novo endpoint em `markets/routes.ts`. Query: `twitterUserId`, `limit` (default 7, max 20).
Retorna lista de parent markets `tweet_count` resolvidos do usuário, ordenados por `endDate` desc, com `resolvedTweetCount`, `countStartDate`, `countEndDate`, `winner`.

### worker.ts
Importar e iniciar `tweetCountMarketCronService.start()`.

## Frontend

### Componente `TweetCountHistoryBadges`
`web/components/market/tweet-count-history-badges.tsx`. Estrutura similar a `CarCountMobileControls`, mas:
- Sem `ResultCircle` (sem winner). Em vez disso, pill mostra o `resolvedTweetCount` (número grande).
- Current day = pill branca "Hoje" (live indicator).
- Past = botão dropdown + 2 pills das datas anteriores com o count.
- Clique navega para `/tweet-tracker/[marketPda]`.

### Integração na tela `/tweet-tracker/[marketPda]/page-client.tsx`
Abaixo do bloco de stats, renderizar `<TweetCountHistoryBadges twitterUserId={data.userId} currentMarketPda={marketPda} />`.

## Out of scope
- Mudanças no resolver (`tweet-count-market.resolver.ts`) — não precisam.
- Cron de criação via UI admin — fica na rota HTTP existente.
- Privy/3xChange migrations (não relacionado).
