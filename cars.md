# Mercado Contagem de Carros 5min — Plano de Implementação

## Visão Geral

Novo tipo de mercado binário onde usuários apostam se uma câmera de rodovia vai registrar **Mais** ou **Menos** do que um limiar de veículos em 5 minutos. O mercado dura 5 minutos, mas apostas (compra e venda) só são aceitas nos primeiros **2:30**. Em vez de gráfico de preço, o frontend exibe a transmissão ao vivo da câmera com overlay de contagem, via WebSocket do serviço `oraculo-cars-count`.

---

## Arquitetura de Integração

```
oraculo-cars-count (FastAPI + YOLO + ByteTrack)
    ├── REST: POST /cameras/{id}/rounds/start → inicia rodada, retorna round_id
    ├── REST: GET /rounds/{round_id} → status, final_count
    ├── HLS: GET /rounds/{round_id}/hls/index.m3u8 → stream de vídeo
    └── WS: ws://host/rounds/{round_id}/ws → updates em tempo real

Oraculo API (Hono)
    ├── Cron: CarCountMarketCronService → cria mercado a cada 5min
    │         ├── chama oraculo-cars-count POST /cameras/{id}/rounds/start
    │         └── salva round_id no CarCountMarketConfig
    ├── Route: GET /api/v1/markets/car-count/active
    ├── Route: GET /api/v1/markets/car-count/past
    ├── Route: GET /api/v1/markets/car-count/stream-info → URL HLS + WS para o frontend
    ├── WS Proxy: wss://oraculo-api/ws/car-count/{roundId} → proxy do WS externo
    ├── Resolver: CarCountMarketResolver → resolve na expiração (final_count vs threshold)
    └── Webhook: POST /api/v1/webhooks/cars-count → recebe evento round_finished

Oraculo Web (Next.js)
    ├── MarketCard: detecta marketType === "car_count_5min"
    ├── HLS Player: <VideoPlayer hlsUrl=... /> (usando hls.js)
    ├── CarCountLive: contador em tempo real via WS proxy
    ├── Dois timers: betting deadline (2:30) e market end (5:00)
    └── Bloqueia UI de bet após 2:30 (e API rejeita ordens também)
```

---

## Fase 1 — Database (Prisma)

### 1.1 Novo MarketType

```prisma
// schema.prisma
enum MarketType {
  standard
  price_5min
  tweet_count
  match_result
  car_count_5min   // NOVO
}
```

### 1.2 Novo modelo CarCountMarketConfig

```prisma
model CarCountMarketConfig {
  id              Int       @id @default(autoincrement())
  marketId        Int       @unique
  cameraId        String    // ex: "SP055-KM092"
  cameraName      String    // ex: "SP-055 KM 092" (nome legível)
  roundId         String?   // UUID da rodada no oraculo-cars-count
  threshold       Int       // limiar para YES/NO (ex: 50 carros)
  finalCount      Int?      // contagem final registrada ao término
  roundStatus     String    @default("pending") // "pending" | "active" | "finished" | "cancelled"
  bettingEndsAt   DateTime  // market.startDate + 150s (2:30)
  market          Market    @relation(fields: [marketId], references: [id], onDelete: Cascade)

  @@map("car_count_market_configs")
}
```

### 1.3 Campo bettingEndsAt no Model Market

O campo `bettingEndsAt` é específico por config. Não precisamos alterar o modelo `Market` global — a restrição de apostas é verificada consultando `carCountMarketConfig.bettingEndsAt` ao processar ordens desse tipo de mercado.

### 1.4 Migration SQL

```sql
-- prisma/scripts/feat-car-count-market.sql

CREATE TYPE "MarketType_new" AS ENUM (
  'standard', 'price_5min', 'tweet_count', 'match_result', 'car_count_5min'
);

-- Atualizar coluna marketType
ALTER TABLE "markets"
  ALTER COLUMN "market_type" TYPE "MarketType_new"
  USING ("market_type"::text::"MarketType_new");

DROP TYPE "MarketType";
ALTER TYPE "MarketType_new" RENAME TO "MarketType";

-- Nova tabela
CREATE TABLE "car_count_market_configs" (
  "id"            SERIAL PRIMARY KEY,
  "market_id"     INT NOT NULL UNIQUE,
  "camera_id"     VARCHAR NOT NULL,
  "camera_name"   VARCHAR NOT NULL,
  "round_id"      VARCHAR,
  "threshold"     INT NOT NULL,
  "final_count"   INT,
  "round_status"  VARCHAR NOT NULL DEFAULT 'pending',
  "betting_ends_at" TIMESTAMPTZ NOT NULL,
  CONSTRAINT "car_count_market_configs_market_id_fkey"
    FOREIGN KEY ("market_id") REFERENCES "markets"("id") ON DELETE CASCADE
);
```

---

## Fase 2 — oraculo-cars-count Service Client

### 2.1 `shared/services/cars-count.service.ts` (NOVO)

```typescript
// Client HTTP para o serviço oraculo-cars-count
interface CarsCountCamera {
  id: string;
  name: string;
  stream_url: string;
  active: boolean;
  active_round?: { id: string; status: string; vehicle_count: number; time_remaining: number };
}

interface CarsCountRound {
  id: string;
  camera_id: string;
  round_number: number;
  status: "active" | "finished" | "cancelled";
  vehicle_count: number;
  started_at: string;
  ended_at: string | null;
  duration: number;
  cancel_reason: string | null;
}

class CarsCountService {
  private baseUrl: string; // CARS_COUNT_API_URL env var

  async getCameras(): Promise<CarsCountCamera[]>
  async startRound(cameraId: string, durationSeconds = 300): Promise<CarsCountRound>
  async getRound(roundId: string): Promise<CarsCountRound>
  async getRoundEvents(roundId: string): Promise<RoundEvent[]>

  // URL helpers (não fazem HTTP, só constroem URLs)
  getHlsUrl(roundId: string): string       // {baseUrl}/rounds/{roundId}/hls/index.m3u8
  getMjpegUrl(roundId: string): string     // {baseUrl}/rounds/{roundId}/video-feed
  getWsUrl(roundId: string): string        // ws://host/rounds/{roundId}/ws
}
```

### 2.2 Env Var

```env
CARS_COUNT_API_URL=http://localhost:8000   # URL do serviço oraculo-cars-count
CARS_COUNT_CAMERA_ID=SP055-KM092          # Câmera padrão para mercados automáticos
CARS_COUNT_THRESHOLD=50                   # Limiar padrão (pode ser dinâmico no futuro)
```

---

## Fase 3 — Resolver

### 3.1 `modules/markets/resolvers/car-count-market.resolver.ts` (NOVO)

```typescript
class CarCountMarketResolver implements IMarketResolver {
  marketType = MarketType.car_count_5min;

  async resolve(market: Market): Promise<void> {
    const config = market.carCountMarketConfig;

    // 1. Busca o round final no oraculo-cars-count
    const round = await carsCountService.getRound(config.roundId);

    if (round.status === "cancelled") {
      // Cancelar o mercado, estornar apostas
      await this.cancelMarket(market, `round_cancelled: ${round.cancel_reason}`);
      return;
    }

    if (round.status !== "finished") {
      throw new Error(`Round ${round.id} not finished yet (status: ${round.status})`);
    }

    const finalCount = round.vehicle_count;

    // YES = "Mais" → final_count > threshold
    const winner = finalCount > config.threshold;

    // 2. Persiste resultado
    await prisma.carCountMarketConfig.update({
      where: { marketId: market.id },
      data: { finalCount, roundStatus: "finished" },
    });

    // 3. Resolve o mercado
    await marketResolutionService.resolveMarket({
      market,
      winner,
      evidence: {
        notes: `Contagem final: ${finalCount} veículos. Limiar: ${config.threshold}. Resultado: ${winner ? "MAIS (YES)" : "MENOS (NO)"}`,
        links: [`${carsCountApiUrl}/rounds/${config.roundId}`],
      },
    });
  }
}
```

---

## Fase 4 — Cron Service de Criação

### 4.1 `modules/markets/services/car-count-market-cron.service.ts` (NOVO)

**Lógica de criação (similar ao PriceMarketCronService):**

```typescript
class CarCountMarketCronService {
  // Roda a cada 30s para detectar janela de criação
  // cronExpression: "*/30 * * * * *"

  async createCarCountMarket(): Promise<void> {
    // 1. Verifica se já existe mercado ativo
    const activeMarket = await marketRepository.findActiveByType("car_count_5min");
    if (activeMarket) {
      const timeToExpiry = activeMarket.endDate - Date.now() / 1000;
      if (timeToExpiry > 30) return; // Ainda tem tempo, aguarda
    }

    // 2. Adquire distributed lock (120s)
    const lock = await acquireLock("car-count-market-creation", 120);
    if (!lock) return;

    try {
      // 3. Chama oraculo-cars-count para iniciar rodada
      const round = await carsCountService.startRound(CARS_COUNT_CAMERA_ID, 300);

      // 4. Calcula timestamps
      const now = Date.now();
      const startTimestamp = Math.floor(now / 1000);
      const endTimestamp = startTimestamp + 300;       // +5min
      const bettingEndsAt = new Date(now + 150_000);  // +2:30

      // 5. Determina threshold (dinâmico no futuro, fixo agora)
      const threshold = await this.getThreshold(CARS_COUNT_CAMERA_ID);

      // 6. Cria mercado no Oraculo
      const question = `${cameraName} - Mais de ${threshold} carros em 5min? (${formatTimestamp()})`;

      await marketCreationService.createMarket({
        question,
        marketType: "car_count_5min",
        endDate: endTimestamp,
        carCountMarketConfig: {
          cameraId: CARS_COUNT_CAMERA_ID,
          cameraName: camera.name,
          roundId: round.id,
          threshold,
          roundStatus: "active",
          bettingEndsAt,
        },
        seedConfig: { enabled: true, totalUsdc: 2, levels: 3, spreadCents: 0 },
      });
    } finally {
      lock.release();
    }
  }

  private async getThreshold(cameraId: string): Promise<number> {
    // Futuramente: média histórica dos últimos 10 rounds dessa câmera
    // Por agora: variável de ambiente
    return parseInt(env.CARS_COUNT_THRESHOLD) || 50;
  }
}
```

---

## Fase 5 — Resolver Cron (Resolution)

### 5.1 `modules/markets/services/car-count-market-resolution-cron.service.ts` (NOVO)

```typescript
class CarCountMarketResolutionCronService {
  // Roda a cada 30s
  // cronExpression: "*/30 * * * * *"

  async resolveExpiredMarkets(): Promise<void> {
    const expiredMarkets = await prisma.market.findMany({
      where: {
        marketType: "car_count_5min",
        resolved: false,
        closed: false,
        endDate: { lte: Math.floor(Date.now() / 1000) },
      },
      include: { carCountMarketConfig: true },
    });

    for (const market of expiredMarkets) {
      await carCountMarketResolver.resolve(market);
    }
  }
}
```

---

## Fase 6 — Betting Window Enforcement

### 6.1 Lógica no Order Service / Route

A janela de apostas (compra e venda) fecha aos 2:30. Após isso, o mercado ainda existe e pode ser acompanhado, mas não aceita novas ordens.

**Onde aplicar a restrição:**

No handler de criação de ordem (`POST /api/v1/markets/:marketPda/orders`), antes de processar:

```typescript
// Em order.service.ts ou na rota
if (market.marketType === "car_count_5min") {
  const config = market.carCountMarketConfig;
  if (config && new Date() > config.bettingEndsAt) {
    throw new AppError(403, "Janela de apostas encerrada. Apostas permitidas apenas nos primeiros 2:30.");
  }
}
```

**Mesmo para cancelamento de ordens:**
```typescript
// Cancelamento (sell/exit) também não permitido após 2:30
if (market.marketType === "car_count_5min" && new Date() > config.bettingEndsAt) {
  throw new AppError(403, "Período de apostas encerrado.");
}
```

---

## Fase 7 — WebSocket Proxy

O frontend precisa receber updates em tempo real da contagem. Para não expor a URL interna do `oraculo-cars-count`, o Oraculo API faz um proxy WebSocket.

### 7.1 `modules/markets/routes/car-count-ws.route.ts` (NOVO)

**Endpoint:** `wss://oraculo-api/ws/markets/car-count/{roundId}`

```typescript
// Usando Hono com upgrade para WebSocket nativo
// ou biblioteca ws

app.get("/ws/markets/car-count/:roundId", (c) => {
  // Upgrade para WebSocket
  const { roundId } = c.req.param();

  // Valida que esse roundId pertence a um mercado ativo do tipo car_count_5min
  const market = await marketRepo.findByRoundId(roundId);
  if (!market) return c.json({ error: "Not found" }, 404);

  return upgradeWebSocket(c, (clientWs) => {
    // Conecta ao oraculo-cars-count WS
    const upstream = new WebSocket(carsCountService.getWsUrl(roundId));

    upstream.on("message", (data) => {
      if (clientWs.readyState === WebSocket.OPEN) {
        clientWs.send(data);
      }
    });

    upstream.on("close", () => clientWs.close());
    upstream.on("error", () => clientWs.close());

    clientWs.on("close", () => upstream.close());
  });
});
```

### 7.2 Mensagens que o frontend recebe (pass-through do oraculo-cars-count)

```typescript
// Update em tempo real (1x/segundo)
{
  "event": "update",
  "round_id": "uuid",
  "camera_id": "SP055-KM092",
  "count": 42,
  "time_remaining": 182.3,
  "duration": 300,
  "round_active": true,
  "connected": true,
  "fps": 14.2
}

// Rodada finalizada
{
  "event": "round_finished",
  "round_id": "uuid",
  "final_count": 87,
  "ended_at": "2026-03-24T15:05:00Z"
}

// Rodada cancelada (stream caiu > 60s)
{
  "event": "round_cancelled",
  "round_id": "uuid",
  "reason": "stream_timeout",
  "final_count": 34
}
```

---

## Fase 8 — Routes REST

### 8.1 Novas rotas em `modules/markets/routes.ts`

```typescript
// GET /api/v1/markets/car-count/active
// Retorna mercado ativo com config completa
{
  marketPda: string;
  question: string;
  endDate: number;
  bettingEndsAt: string; // ISO
  carCountMarketConfig: {
    cameraId: string;
    cameraName: string;
    roundId: string;
    threshold: number;
    roundStatus: string;
    hlsUrl: string;    // URL do stream HLS (via proxy ou direto)
    wsUrl: string;     // wss://oraculo-api/ws/markets/car-count/{roundId}
  };
}

// GET /api/v1/markets/car-count/past?limit=20
// Mercados resolvidos + cancelados

// GET /api/v1/markets/car-count/stream-info/:marketPda
// Retorna hlsUrl e wsUrl para o frontend conectar
```

---

## Fase 9 — Webhook do oraculo-cars-count

Para evitar polling e garantir resolução rápida ao fim da rodada:

### 9.1 Configurar oraculo-cars-count para notificar

Adicionar ao `app.py` do oraculo-cars-count um POST para o Oraculo quando uma rodada terminar:

```python
# No counter.py, após marcar round como finished:
import httpx
oraculo_webhook_url = os.getenv("ORACULO_WEBHOOK_URL")
if oraculo_webhook_url:
    httpx.post(f"{oraculo_webhook_url}/api/v1/webhooks/cars-count", json={
        "event": "round_finished",
        "round_id": str(round_id),
        "camera_id": camera_id,
        "final_count": final_count,
        "ended_at": ended_at.isoformat()
    }, headers={"X-Cars-Count-Secret": os.getenv("ORACULO_WEBHOOK_SECRET")})
```

### 9.2 Handler no Oraculo API

```typescript
// POST /api/v1/webhooks/cars-count
app.post("/api/v1/webhooks/cars-count", async (c) => {
  // Valida X-Cars-Count-Secret
  const body = await c.req.json();

  if (body.event === "round_finished") {
    const market = await marketRepo.findByRoundId(body.round_id);
    if (market && !market.resolved) {
      await carCountMarketResolver.resolve(market);
    }
  }

  if (body.event === "round_cancelled") {
    const market = await marketRepo.findByRoundId(body.round_id);
    if (market && !market.resolved) {
      await cancelMarketWithRefund(market, body.reason);
    }
  }

  return c.json({ ok: true });
});
```

---

## Fase 10 — Frontend Web

### 10.1 Types (`web/types/market.ts`)

```typescript
interface CarCountMarketConfig {
  cameraId: string;
  cameraName: string;
  roundId: string;
  threshold: number;
  finalCount?: number | null;
  roundStatus: string;
  bettingEndsAt: string; // ISO string
  hlsUrl?: string;
  wsUrl?: string;
}

interface Market {
  // ... campos existentes
  marketType?: "standard" | "price_5min" | "tweet_count" | "match_result" | "car_count_5min";
  carCountMarketConfig?: CarCountMarketConfig | null;
}
```

### 10.2 Labels (`web/lib/utils/price-market-labels.ts`)

Adicionar ao `getYesLabel` e `getNoLabel`:

```typescript
// "Mais" = mais carros do que o threshold (YES)
// "Menos" = igual ou menos carros (NO)
if (market.marketType === "car_count_5min") {
  return "Mais";  // getYesLabel
}
if (market.marketType === "car_count_5min") {
  return "Menos"; // getNoLabel
}
```

### 10.3 Market Card (`web/components/markets/market-card.tsx`)

Adicionar lógica para `car_count_5min`:
- Badge "Ao vivo" vermelho (igual ao price_5min)
- Mostrar câmera e threshold: "SP-055 KM 092 · Limiar: 50 carros"
- Countdown duplo: tempo para fechar apostas (2:30) e tempo para fim do mercado (5:00)
- Labels "Mais" / "Menos"

### 10.4 Hook `use-active-car-count-market.ts` (NOVO)

```typescript
// web/lib/hooks/use-active-car-count-market.ts
export function useActiveCarCountMarket(enabled = true) {
  return useQuery({
    queryKey: ["car-count-active"],
    queryFn: () => fetch("/api/v1/markets/car-count/active").then(r => r.json()),
    enabled,
    staleTime: 30_000,
    refetchInterval: 30_000,
  });
}
```

### 10.5 Hook `use-car-count-ws.ts` (NOVO)

```typescript
// web/lib/hooks/use-car-count-ws.ts
interface CarCountUpdate {
  event: "update" | "round_finished" | "round_cancelled";
  count: number;
  time_remaining: number;
  final_count?: number;
  fps: number;
  connected: boolean;
}

export function useCarCountWs(roundId: string | null, wsUrl: string | null) {
  const [data, setData] = useState<CarCountUpdate | null>(null);
  const [connected, setConnected] = useState(false);

  useEffect(() => {
    if (!roundId || !wsUrl) return;

    const ws = new WebSocket(wsUrl);
    ws.onopen = () => setConnected(true);
    ws.onclose = () => setConnected(false);
    ws.onmessage = (e) => setData(JSON.parse(e.data));

    return () => ws.close();
  }, [roundId, wsUrl]);

  return { data, connected };
}
```

### 10.6 Hook `use-betting-deadline.ts` (NOVO)

```typescript
// web/lib/hooks/use-betting-deadline.ts
// Retorna tempo restante para fechar apostas e se ainda é possível apostar
export function useBettingDeadline(bettingEndsAt: string | null) {
  const [canBet, setCanBet] = useState(true);
  const [timeLeft, setTimeLeft] = useState<string | null>(null);

  useEffect(() => {
    if (!bettingEndsAt) return;

    const interval = setInterval(() => {
      const remaining = new Date(bettingEndsAt).getTime() - Date.now();
      if (remaining <= 0) {
        setCanBet(false);
        setTimeLeft("0:00");
      } else {
        const m = Math.floor(remaining / 60000);
        const s = Math.floor((remaining % 60000) / 1000);
        setTimeLeft(`${m}:${s.toString().padStart(2, "0")}`);
      }
    }, 1000);

    return () => clearInterval(interval);
  }, [bettingEndsAt]);

  return { canBet, timeLeft };
}
```

### 10.7 Componente `CarCountLive` (NOVO)

```typescript
// web/components/market/car-count-live.tsx

interface Props {
  market: Market; // marketType === "car_count_5min"
}

export function CarCountLive({ market }: Props) {
  const config = market.carCountMarketConfig!;
  const { data: wsData, connected } = useCarCountWs(config.roundId, config.wsUrl);
  const { canBet, timeLeft: bettingTimeLeft } = useBettingDeadline(config.bettingEndsAt);
  const marketTimeLeft = usePriceMarketCountdown(market.endDate); // reuse existing hook

  const currentCount = wsData?.count ?? 0;
  const progress = Math.min((currentCount / config.threshold) * 100, 100);

  return (
    <div className="flex flex-col gap-4">
      {/* Video stream */}
      <div className="relative aspect-video rounded-lg overflow-hidden bg-black">
        <HlsPlayer url={config.hlsUrl} />

        {/* Overlay: contagem atual */}
        <div className="absolute top-3 left-3 bg-black/70 text-white px-3 py-1 rounded-full text-sm font-mono">
          {currentCount} carros
        </div>

        {/* Overlay: conexão */}
        {!connected && (
          <div className="absolute top-3 right-3 bg-red-600 text-white px-2 py-1 rounded-full text-xs">
            Reconectando...
          </div>
        )}
      </div>

      {/* Barra de progresso: contagem vs threshold */}
      <div>
        <div className="flex justify-between text-xs text-muted-foreground mb-1">
          <span>0</span>
          <span className="font-medium">Limiar: {config.threshold}</span>
          <span>{config.threshold * 2}</span>
        </div>
        <div className="h-2 bg-muted rounded-full overflow-hidden">
          <div
            className={cn(
              "h-full transition-all duration-1000",
              currentCount > config.threshold ? "bg-green-500" : "bg-blue-500"
            )}
            style={{ width: `${progress}%` }}
          />
        </div>
        <div className="flex justify-between text-xs mt-1">
          <span className="text-green-600">Mais ↑</span>
          <span className="text-red-600">Menos ↓</span>
        </div>
      </div>

      {/* Timers */}
      <div className="flex gap-4 text-sm">
        <div className="flex flex-col items-center">
          <span className="text-muted-foreground text-xs">Apostas fecham em</span>
          <span className={cn("font-mono font-bold text-lg", !canBet && "text-muted-foreground line-through")}>
            {canBet ? bettingTimeLeft : "Encerrado"}
          </span>
        </div>
        <div className="flex flex-col items-center">
          <span className="text-muted-foreground text-xs">Mercado fecha em</span>
          <span className="font-mono font-bold text-lg">{marketTimeLeft}</span>
        </div>
      </div>

      {/* Warning se apostas fechadas */}
      {!canBet && !market.resolved && (
        <div className="rounded-md bg-amber-50 border border-amber-200 p-3 text-sm text-amber-800">
          Janela de apostas encerrada. Aguarde o resultado em {marketTimeLeft}.
        </div>
      )}
    </div>
  );
}
```

### 10.8 Componente `HlsPlayer` (NOVO)

```typescript
// web/components/market/hls-player.tsx
// Usa hls.js para reproduzir stream HLS no browser

"use client";
import { useEffect, useRef } from "react";
// npm install hls.js
import Hls from "hls.js";

interface Props {
  url: string | null | undefined;
  autoPlay?: boolean;
  muted?: boolean;
}

export function HlsPlayer({ url, autoPlay = true, muted = true }: Props) {
  const videoRef = useRef<HTMLVideoElement>(null);

  useEffect(() => {
    if (!url || !videoRef.current) return;
    const video = videoRef.current;

    if (Hls.isSupported()) {
      const hls = new Hls({ lowLatencyMode: true });
      hls.loadSource(url);
      hls.attachMedia(video);
      if (autoPlay) hls.on(Hls.Events.MANIFEST_PARSED, () => video.play());
      return () => hls.destroy();
    } else if (video.canPlayType("application/vnd.apple.mpegurl")) {
      // Safari nativo
      video.src = url;
      if (autoPlay) video.play();
    }
  }, [url, autoPlay]);

  return (
    <video
      ref={videoRef}
      className="w-full h-full object-cover"
      autoPlay={autoPlay}
      muted={muted}
      playsInline
    />
  );
}
```

### 10.9 Página do Mercado

Na página de detalhe de um mercado (`/markets/[marketPda]`), detectar `marketType === "car_count_5min"` e renderizar `<CarCountLive />` em vez de `<MarketChart />`.

```typescript
// Em market-page.tsx (ou similar):
{market.marketType === "car_count_5min" ? (
  <CarCountLive market={market} />
) : market.marketType === "price_5min" ? (
  <PriceMarketChart ... />
) : (
  <MarketChart ... />
)}

// Desabilitar botões de bet se canBet === false
const { canBet } = useBettingDeadline(market.carCountMarketConfig?.bettingEndsAt ?? null);
// Passar canBet para o componente de ordem
```

---

## Fase 11 — Admin

### 11.1 Criação manual de mercado car_count_5min

Na página `/admin/markets/create`, adicionar tipo "Contagem de Carros 5min":

```typescript
// Campos específicos:
// - Camera: select com câmeras disponíveis (busca GET /api/v1/cars-count/cameras)
// - Threshold: número inteiro (ex: 50)
// - Duration: fixo em 300s (não editável)
// - A API: POST /api/v1/markets com marketType: "car_count_5min" + carCountMarketConfig
```

### 11.2 Endpoint de câmeras disponíveis no Oraculo API

```typescript
// GET /api/v1/cars-count/cameras
// Proxy para GET oraculo-cars-count/cameras
// Retorna lista de câmeras ativas com status de rodada atual
```

---

## Fase 12 — Env Vars e Configuração

### api/.env

```env
# oraculo-cars-count integration
CARS_COUNT_API_URL=http://localhost:8000
CARS_COUNT_CAMERA_ID=SP055-KM092
CARS_COUNT_THRESHOLD=50
CARS_COUNT_WEBHOOK_SECRET=<secret_aleatorio>

# No oraculo-cars-count/.env (adicionar):
ORACULO_WEBHOOK_URL=https://api.oraculo.com
ORACULO_WEBHOOK_SECRET=<mesmo_secret>
```

---

## Fase 13 — Registro nos Crons/App

### api/src/app.ts (ou onde os crons são registrados)

```typescript
// Registrar os novos crons
carCountMarketCronService.start(); // cria mercados a cada 30s de verificação
carCountMarketResolutionCronService.start(); // resolve mercados expirados

// Registrar nova rota WS
app.route("/ws/markets/car-count", carCountWsRoute);

// Registrar webhook
app.route("/api/v1/webhooks", carsCountWebhookRoute);

// Registrar rotas REST
app.route("/api/v1/markets/car-count", carCountMarketRoute);
app.route("/api/v1/cars-count", carsCountProxyRoute);
```

### Registrar resolver

```typescript
// Em market-resolver.registry.ts (ou similar)
resolverRegistry.register(new CarCountMarketResolver());
```

---

## Ordem de Implementação

```
[X] Fase 1  — Prisma: novo enum + model CarCountMarketConfig + migration SQL
[X] Fase 2  — CarsCountService (HTTP client)
[X] Fase 3  — CarCountMarketResolver
[X] Fase 4  — CarCountMarketCronService (criação automática)
[X] Fase 5  — CarCountMarketResolutionCronService
[X] Fase 6  — Betting window enforcement no order service
[X] Fase 7  — WebSocket proxy route
[X] Fase 8  — REST routes (/active, /past, /stream-info)
[X] Fase 9  — Webhook do oraculo-cars-count + handler no Oraculo
[X] Fase 10 — Frontend: types, labels, hooks, HlsPlayer, CarCountLive
[X] Fase 11 — Admin: novo tipo de mercado + seleção de câmera
[X] Fase 12 — Env vars em ambos os serviços
[X] Fase 13 — Registro de crons, rotas, resolvers no app.ts
```

---

## Considerações de Segurança

- **Betting window**: Enforced tanto no frontend (UI desabilita) quanto no backend (API rejeita com 403)
- **Webhook secret**: Validar `X-Cars-Count-Secret` antes de processar eventos
- **WS proxy autenticado**: Verificar que o `roundId` pertence a um mercado ativo antes de fazer proxy
- **Cancelamento automático**: Se rodada for `round_cancelled` (stream caiu >60s), cancelar mercado e estornar todas as apostas — mesma lógica existente de cancelamento de mercado
- **Lock de resolução**: Evitar dupla resolução (idempotência via `market.resolved` check)

---

## Diferenças vs Bitcoin 5min (Resumo)

| Aspecto | Bitcoin 5min | Carros 5min |
|---------|--------------|-------------|
| Duração | 15min | 5min |
| Janela de apostas | 15min (todo período) | 2:30 (metade) |
| Dado principal | Preço BTC (Chainlink+Kraken) | Contagem de veículos (YOLO+ByteTrack) |
| Frontend "chart" | Gráfico de preço ao vivo | Stream de vídeo HLS ao vivo |
| Resolução | Preço final > preço inicial | Contagem final > threshold |
| YES label | "Sobe" | "Mais" |
| NO label | "Desce" | "Menos" |
| Fonte de dados | btc-price-websocket.service.ts | ws://oraculo-cars-count/rounds/{id}/ws |
| Cancelamento | Raramente (dados sempre disponíveis) | Possível (se stream cair >60s) |
| Config DB | PriceMarketConfig | CarCountMarketConfig |
| Threshold | N/A (preço é contínuo) | Fixo por câmera/horário |

---

## Dependências a Instalar

### web/package.json
```bash
bun add hls.js
bun add -D @types/hls.js
```

---

## Próximos Passos (após implementação base)

1. **Threshold dinâmico**: Calcular média histórica dos últimos N rounds da câmera no mesmo horário do dia
2. **Multi-câmera**: Suporte a múltiplas câmeras simultâneas (mercados paralelos)
3. **Notificação de resultado**: Notify via WebSocket quando round_finished (socket já existe para outros eventos)
4. **Histórico de rounds**: Exibir últimas 5 rodadas com contagem final no frontend
5. **GPU inference**: Configurar YOLO_DEVICE=cuda no oraculo-cars-count para melhor performance
