# Price5MinChart — Fluxo completo

Componente: `web/components/market/price-5min-chart.tsx`
Biblioteca: [lightweight-charts](https://tradingview.github.io/lightweight-charts/)

---

## Visão geral

Gráfico de área em tempo real que exibe o preço do BTC durante um mercado de 5 minutos. O preço é o foco central: ele fica sempre ao centro visual do eixo Y e a linha se move suavemente a 30 fps via interpolação linear. Uma linha tracejada (TARGET) marca o preço que o usuário precisa bater.

---

## Props

| Prop             | Tipo      | Descrição                                                  |
|------------------|-----------|------------------------------------------------------------|
| `priceToBeat`    | `number`  | Preço alvo do mercado (linha TARGET)                       |
| `startTimestamp` | `number`  | Unix timestamp em **segundos** do início do mercado        |
| `closed`         | `boolean` | Indica se o mercado já encerrou (reservado, não usado ainda) |

---

## Constantes de configuração

```ts
LOOKBACK_SECONDS   = 300    // histórico inicial carregado (5 min)
VISIBLE_WINDOW_SEC = 20     // janela visível no eixo X (segundos)
WS_RECONNECT_MS    = 3000   // delay de reconexão WebSocket
LERP_DURATION_MS   = 400    // duração da interpolação de preço entre updates
```

---

## Estrutura de refs

| Ref                   | Tipo              | Papel                                                              |
|-----------------------|-------------------|--------------------------------------------------------------------|
| `containerRef`        | `HTMLDivElement`  | Elemento DOM onde o chart é montado                                |
| `chartRef`            | `IChartApi`       | Instância do chart lightweight-charts                              |
| `seriesRef`           | `ISeriesApi`      | Série do tipo AreaSeries                                           |
| `lastTimeRef`         | `number`          | Último timestamp (segundos float) enviado ao `series.update()`     |
| `rafRef`              | `number \| null`  | ID do requestAnimationFrame atual; `null` = loop parado            |
| `fromPriceRef`        | `number \| null`  | Preço de partida da interpolação atual                             |
| `toPriceRef`          | `number \| null`  | Preço de destino da interpolação atual                             |
| `priceUpdateTimeRef`  | `number`          | `Date.now()` no momento em que o último `price.updated` chegou     |
| `currentVisualPrice`  | `number`          | Preço visual interpolado no frame atual (lido pelo autoscale)      |

---

## Fluxo de inicialização (useEffect #1)

```
mount
  │
  ├─ 1. GET /markets/btc-price?startTimestamp=…&lookbackSeconds=300
  │       └─ buildPoints(raw, startMs, priceToBeat)
  │           • insere ponto âncora em startSec com valor = priceToBeat
  │           • agrupa por segundo (Map), descarta duplicatas
  │           • retorna array ordenado por tempo
  │
  ├─ 2. createChart() — lightweight-charts montado no containerRef
  │       • autoSize: true
  │       • layout: fundo #001c1e, texto #3d5f62
  │       • grid: linhas horizontais sutis, sem verticais
  │       • rightPriceScale: scaleMargins top/bottom 10%
  │       • timeScale: barSpacing 8, labels a cada 4s (tickMarkFormatter)
  │       • handleScroll: false / handleScale: false
  │
  ├─ 3. addSeries(AreaSeries)
  │       • linha laranja #f97316, lineWidth 2, curva suave
  │       • fill: gradiente de rgba(249,115,22,0.12) → 0
  │       • lastPriceAnimation: Continuous
  │       • formato do label: $XX,XXX (arredondado, sem decimais)
  │
  ├─ 4. createPriceLine(TARGET)
  │       • preço fixo = priceToBeat
  │       • cor ciano #22d3ee, tracejado, lineWidth 1
  │       • axisLabelVisible: true (aparece no eixo Y mesmo fora do range)
  │
  ├─ 5. series.setData(initialData)
  │       • seeds: lastTimeRef, toPriceRef, fromPriceRef = último ponto
  │       • priceUpdateTimeRef = Date.now() - LERP_DURATION_MS (lerp já concluído)
  │       • setVisibleRange: janela de 20s terminando no último ponto
  │
  └─ 6. Inicia loop de animação (ver seção abaixo)
```

---

## Loop de animação (30 fps)

O RAF é limitado a 30 fps via comparação de timestamps:

```
requestAnimationFrame(ts)
  │
  ├─ ts - lastFrameMs < 33ms? → reagenda e retorna (frame pulado)
  │
  ├─ Calcula floatSec = Math.round(Date.now() / 100) / 10
  │   • resolução de 100ms → mesmo bucket por ~3 frames consecutivos
  │   • evita criar um ponto novo por frame; substitui o mesmo bucket
  │
  ├─ getLerpedPrice()
  │   t = clamp((now - priceUpdateTime) / 400ms, 0, 1)
  │   price = fromPrice + (toPrice - fromPrice) * t
  │
  ├─ currentVisualPrice.current = price   ← usado pelo auto-scale Y
  │
  ├─ series.update({ time: floatSec, value: price })
  │   • time = max(lastTimeRef, floatSec) — nunca vai para trás
  │
  └─ timeScale.setVisibleRange
       from: nowSec - 20
       to:   nowSec + 2   ← 2s de margem à direita
```

### Por que 30 fps e não 60?

O servidor envia updates a cada ~500ms–1s. A 60 fps, 5 de cada 6 frames fariam exatamente o mesmo cálculo (mesmo bucket de 100ms, mesmo `t` de lerp). 30 fps reduz chamadas ao DOM/canvas pela metade sem perda visual perceptível.

---

## Eixo Y — auto-scale dinâmico

Com `scaleMargins: { top: 0.1, bottom: 0.1 }` e sem `autoscaleInfoProvider`, o lightweight-charts calcula automaticamente o range Y baseado nos pontos **visíveis** na janela de 20 segundos. Isso significa:

- Se o BTC variar $5 em 20s → o gráfico mostra um range de ~$5 com 10% de padding
- Se variar $50 → abre proporcionalmente
- O preço atual fica sempre próximo ao centro visual (é o ponto mais recente)
- A linha TARGET aparece via label no eixo Y (`axisLabelVisible: true`) mesmo quando fora do range visível

---

## Eixo X — labels de 4 em 4 segundos

```ts
tickMarkFormatter: (time) => {
  if (Math.round(time) % 4 !== 0) return ""; // suprime label
  return "HH:MM:SS";
}
```

O `setVisibleRange` no RAF move o viewport continuamente com `nowSec` em float, produzindo scroll suave sem saltos.

---

## Fluxo WebSocket (useEffect #2)

```
connect()
  │
  ├─ ws.onopen
  │   └─ send: { type: "subscribe_btc_price", startTimestamp, lookbackSeconds: 300 }
  │
  ├─ ws.onmessage
  │   │
  │   ├─ event: "history.loaded"
  │   │   • buildPoints(payload.graphData) → series.setData()
  │   │   • Substitui dados REST por versão de alta resolução do WS
  │   │   • Seeds: lastTimeRef, toPriceRef, fromPriceRef, priceUpdateTimeRef
  │   │   • setVisibleRange recalibrado
  │   │
  │   └─ event: "price.updated"
  │       • fromPrice = getLerpedPrice() atual  ← snapshot do visual
  │       • toPrice   = point.price             ← novo alvo
  │       • priceUpdateTime = Date.now()        ← reinicia lerp t=0
  │
  ├─ ws.onclose → setTimeout(connect, 3000)
  └─ ws.onerror → ws.close()
```

### Dupla fonte de dados

| Fonte       | Quando         | Papel                                              |
|-------------|----------------|----------------------------------------------------|
| REST        | Antes do WS    | Histórico imediato para o chart não iniciar vazio  |
| WS `history.loaded` | Após conexão | Substitui REST por dados de maior resolução   |
| WS `price.updated`  | Contínuo     | Dispara novo lerp; RAF aplica suavemente      |

---

## Interpolação linear de preço (lerp)

Cada `price.updated` não atualiza o gráfico diretamente. Em vez disso:

```
price.updated recebido
  fromPrice = snapshot do visual atual   (evita salto)
  toPrice   = novo preço do servidor
  t₀        = Date.now()

A cada frame (30fps):
  t = clamp((now - t₀) / 400ms, 0, 1)
  visual = fromPrice + (toPrice - fromPrice) * t
```

Com `LERP_DURATION_MS = 400`, a linha leva 400ms para chegar ao novo preço. Se um novo update chegar antes dos 400ms, o lerp reinicia a partir da posição visual atual (não do `fromPrice` antigo), evitando acúmulo de "dívida" de animação.

---

## Cleanup

Ambos os `useEffect` limpam recursos ao desmontar:

- **useEffect #1**: `cancelAnimationFrame`, zera `rafRef`/`seriesRef`/`chartRef`, chama `chart.remove()`
- **useEffect #2**: envia `unsubscribe_btc_price`, fecha WebSocket, limpa `setTimeout` de reconexão

---

## Diagrama de sequência resumido

```
Component mount
    │
    ├──► REST fetch → initialData → chart criado → series.setData() → RAF inicia
    │
    └──► WS connect → subscribe
              │
              ├── history.loaded → series.setData() (sobrescreve REST)
              │
              └── price.updated ─────────────────────────┐
                                                          ▼
                                              fromPrice/toPrice atualizados
                                                          │
                                              RAF (30fps) ─► getLerpedPrice()
                                                          │
                                              series.update({ time, value })
                                                          │
                                              timeScale.setVisibleRange (scroll)
```
