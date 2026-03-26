# Mercados de Contagem em Tempo Real

> Tipo de mercado onde o resultado é determinado pela contagem de eventos físicos observáveis em tempo real — câmeras, sensores, APIs públicas — dentro de uma janela de tempo curta.

---

## Conceito

O usuário aposta sobre um evento contável e observável que acontece agora:

> "Mais de 20 carros vão passar nessa rodovia nos próximos 60 segundos?"

A resposta é objetiva, auditável e resolve automaticamente. O diferencial é que o usuário **acompanha o resultado acontecendo** — contador subindo, tempo esgotando — criando tensão contínua até a resolução.

---

## Estrutura do Mercado

```ts
interface CountingMarket {
  marketId: string
  question: string          // "Mais de 20 carros nos próximos 60s?"
  dataSource: DataSource    // câmera, sensor, API
  threshold: number         // 20
  operator: ">" | ">=" | "<" | "<=" | "==" // ">"
  windowSeconds: number     // 60
  startTime: number         // unix timestamp
  endTime: number           // startTime + windowSeconds
  resolution: "auto"        // sempre automático
}
```

**Durações sugeridas:** 30s · 60s · 90s · 120s

**Tipos de pergunta:**
- Binária: "Mais de N eventos?"
- Faixa: "Entre N e M eventos?"

---

## Fontes de Dados

### Nível 1 — Simulação (MVP)

Antes de qualquer câmera real. Valida UX e mecânica de apostas.

```ts
// Distribuição de Poisson — simula tráfego real
function simulateCarsPerTick(lambda = 1.2): number {
  return poissonSample(lambda) // média de 1.2 carros/segundo
}
```

Permite testar toda a experiência sem infraestrutura de visão computacional.

---

### Nível 2 — Câmera + IA

**Pipeline:**

```
Stream de câmera (RTSP / HLS / MJPEG)
        ↓
Captura de frames (1–5 fps)
        ↓
YOLOv8 — detecção de veículos
        ↓
DeepSORT / centroid tracking — identidade única por veículo
        ↓
Linha virtual de contagem
        ↓
Cruzou a linha → count++
        ↓
Redis pub/sub → WebSocket → Frontend
```

**Linha virtual de contagem:**

```
─────────────────────────────
          |  ← linha
          |
  🚗 → cruza → count++
```

A linha é configurada por coordenadas no frame. Só conta uma vez por veículo (tracking evita duplicatas).

**Stack do worker de visão:**

| Componente | Tecnologia |
|---|---|
| Linguagem | Python |
| Detecção | YOLOv8 (ultralytics) |
| Tracking | DeepSORT ou ByteTrack |
| Stream | OpenCV (`cv2.VideoCapture`) |
| Saída | Redis pub/sub |

---

### Nível 3 — Câmeras Públicas

Fontes no Brasil com streams acessíveis:

| Fonte | Formato |
|---|---|
| DER (Dept. de Estradas) | RTSP / HLS |
| Concessionárias de rodovias | MJPEG / HLS |
| CET / SMT (São Paulo) | HLS |
| Prefeituras (CCO) | RTSP |
| Waze / Google Maps API | dados agregados |

---

## Fluxo de Dados em Tempo Real

```
Camera Stream
      ↓
Vision Worker (Python)
      ↓  count por segundo
Redis  ──────────────────→  Histórico do mercado
      ↓
WebSocket Server (Bun)
      ↓
Frontend (Next.js)
      ↓
UI atualiza contador + animação
```

**Evento WebSocket:**

```json
{
  "event": "count_update",
  "marketId": "cars-highway-001",
  "count": 14,
  "timeRemaining": 34,
  "timestamp": 1710000026
}
```

---

## Resolução

A resolução é **automática e determinística**:

```ts
async function resolveMarket(market: CountingMarket) {
  const finalCount = await redis.get(`market:${market.marketId}:count`)

  const result = evaluate(finalCount, market.operator, market.threshold)
  // ex: 14 > 20 → false → NÃO vence

  await settleMarket(market.marketId, result)
}
```

Não há julgamento humano. O resultado é o que o contador registrou.

---

## Auditoria

Cada mercado salva:

```
markets/{marketId}/
  ├── frames/          # snapshots de 1 em 1 segundo (jpg comprimido)
  ├── detections.jsonl # timestamp + bbox + track_id de cada detecção
  ├── count_log.jsonl  # { timestamp, count, delta } a cada segundo
  └── result.json      # contagem final + resultado + hash dos logs
```

Qualquer usuário pode contestar e verificar frame a frame.

---

## Experiência do Usuário

### Tela do mercado (durante a janela)

```
┌────────────────────────────────────────────┐
│  🛣️  Rodovia Anhanguera — Câmera km 47     │
│                                            │
│  🚗 🚗 🚗 🚗 🚗 🚗 🚗 🚗 🚗 🚗 🚗 🚗      │
│                                            │
│  Contados: 12          ⏱ 34s restantes    │
│                                            │
│  ┤░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░├  │
│  0                   20 ← meta        40  │
│                                            │
│  Mais de 20 carros?                        │
│                                            │
│  [ ✅ SIM  — 1.85x ]   [ ❌ NÃO — 2.10x ] │
└────────────────────────────────────────────┘
```

### Elementos de tensão

- Contador incrementando em tempo real
- Barra de progresso em direção ao threshold
- Odds se ajustando conforme a contagem avança
- Animação de carro ao cruzar a linha

### Estados visuais

| Estado | Visual |
|---|---|
| Aberto para apostas | Contador rodando, botões ativos |
| Apostas encerradas (últimos 5s) | Botões bloqueados, só acompanha |
| Resolvido — SIM | Verde, confete, "20+ atingido!" |
| Resolvido — NÃO | Vermelho, "Ficou em 18" |

---

## Modelo de Odds

As odds mudam em tempo real conforme a contagem avança:

```
janela = 60s, threshold = 20, tempo decorrido = 30s

count atual = 12
projeção linear = 24 → favorece SIM → odds de SIM caem
```

Isso cria dinâmica de mercado durante a janela — quem aposta cedo assume mais risco, quem aposta tarde tem informação mas odds piores.

---

## Variações do Formato

| Variante | Exemplo |
|---|---|
| Rodovia | "Mais de 20 carros em 60s?" |
| Pedestres | "Mais de 50 pessoas cruzam a praça em 2min?" |
| Clima | "Vai chover nos próximos 30min?" (sensor) |
| Eventos ao vivo | "Mais de 3 passes completos nessa descida?" |
| Dados públicos | "BTC vai subir nos próximos 60s?" |

A mecânica é idêntica — muda apenas a fonte de dados e o que é contado.

---

## Roadmap de Implementação

```
Fase 1 — MVP com simulação
  ├── Criar tipo CountingMarket no schema
  ├── Worker de simulação (Poisson)
  ├── WebSocket de count_update
  └── UI básica com contador e apostas

Fase 2 — Câmera real
  ├── Vision worker (Python + YOLOv8)
  ├── Integração Redis
  ├── Pipeline de auditoria (frames + log)
  └── Painel de configuração de câmera

Fase 3 — Escala
  ├── Múltiplas câmeras simultâneas
  ├── Odds dinâmicas em tempo real
  ├── Replay do mercado pós-resolução
  └── Câmeras públicas brasileiras
```
