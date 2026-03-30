# 3xChange — Webhook Events

Webhooks são enviados via `POST` para a URL configurada no seu merchant (`webhookUrl`) sempre que o status de um pedido muda. Cada entrega é assinada e possui um ID único para suporte a idempotência.

---

## Autenticação e segurança

Cada requisição inclui os seguintes headers:

| Header | Descrição |
|---|---|
| `X-3xchange-Event` | Nome do evento (ex: `order.completed`) |
| `X-3xchange-Delivery` | UUID único por entrega — use para idempotência |
| `X-3xchange-Signature` | Assinatura HMAC-SHA256 do body |
| `Content-Type` | `application/json` |

### Verificando a assinatura

A assinatura é gerada com HMAC-SHA256 usando o **webhook secret** do seu merchant sobre o body JSON bruto da requisição.

```js
const crypto = require('crypto');

function verifyWebhook(body, signature, secret) {
  const expected = 'sha256=' + crypto
    .createHmac('sha256', secret)
    .update(body)
    .digest('hex');
  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(expected)
  );
}

// No seu handler:
const rawBody = req.rawBody; // string JSON bruta, não parseada
const sig = req.headers['x-3xchange-signature'];
if (!verifyWebhook(rawBody, sig, process.env.WEBHOOK_SECRET)) {
  return res.status(401).end();
}
```

> **Importante:** sempre valide a assinatura antes de processar o evento.

### Retentativas

Se o seu endpoint retornar um status fora do range `2xx` ou não responder em 10 segundos, o sistema retentará automaticamente com backoff exponencial.

---

## Campos comuns

Todos os eventos incluem os seguintes campos no body:

| Campo | Tipo | Descrição |
|---|---|---|
| `event` | `string` | Nome do evento |
| `orderId` | `string` (UUID) | ID interno do pedido na 3xChange |
| `externalId` | `string \| null` | ID que você enviou ao criar o pedido |
| `type` | `"PAYIN" \| "PAYOUT"` | Tipo do pedido |
| `status` | `string` | Status atual do pedido |
| `timestamp` | `string` (ISO 8601) | Data/hora do evento |
| `deliveryId` | `string` (UUID) | ID único da entrega para idempotência |

---

## Eventos — PAYIN

Fluxo: cliente paga via PIX → 3xChange recebe e converte para crypto → transfere para a wallet de destino.

### `order.payment_received`

PIX recebido e confirmado. O pedido está sendo processado (conversão + transferência on-chain).

```json
{
  "event": "order.payment_received",
  "orderId": "b1c2d3e4-...",
  "externalId": "pedido-123",
  "type": "PAYIN",
  "status": "PAYMENT_RECEIVED",
  "amountBrl": 52.60,
  "pixEndToEndId": "E54811417202603231820RJUuDwUJw9l",
  "timestamp": "2026-03-23T18:20:45.713Z",
  "deliveryId": "61b66508-..."
}
```

| Campo | Tipo | Descrição |
|---|---|---|
| `amountBrl` | `number` | Valor recebido em BRL |
| `pixEndToEndId` | `string \| null` | ID end-to-end do PIX (rastreabilidade) |

---

### `order.completed` (PAYIN)

Crypto transferida para a wallet de destino. Fluxo finalizado com sucesso.

```json
{
  "event": "order.completed",
  "orderId": "b1c2d3e4-...",
  "externalId": "pedido-123",
  "type": "PAYIN",
  "status": "COMPLETED",
  "amountBrl": 52.60,
  "amountToken": 10.0,
  "targetToken": "USDC",
  "destinationWallet": "8xKp...solana",
  "txHash": "5FJZcffw7...",
  "timestamp": "2026-03-23T18:21:10.000Z",
  "deliveryId": "72c77609-..."
}
```

| Campo | Tipo | Descrição |
|---|---|---|
| `amountBrl` | `number \| null` | Valor em BRL (bruto) |
| `amountToken` | `number \| null` | Quantidade de crypto transferida |
| `targetToken` | `string \| null` | Token enviado (ex: `"USDC"`) |
| `destinationWallet` | `string \| null` | Endereço Solana de destino |
| `txHash` | `string \| null` | Hash da transação on-chain |

---

### `order.refunding` (PAYIN)

Pedido não pôde ser processado após o PIX recebido. Estorno em andamento.

```json
{
  "event": "order.refunding",
  "orderId": "b1c2d3e4-...",
  "externalId": "pedido-123",
  "type": "PAYIN",
  "status": "REFUNDING",
  "failureReason": "Swap failed: insufficient liquidity",
  "timestamp": "2026-03-23T18:22:00.000Z",
  "deliveryId": "83d88710-..."
}
```

---

### `order.refunded` (PAYIN)

Estorno via PIX concluído com sucesso.

```json
{
  "event": "order.refunded",
  "orderId": "b1c2d3e4-...",
  "externalId": "pedido-123",
  "type": "PAYIN",
  "status": "REFUNDED",
  "amountBrl": 52.60,
  "timestamp": "2026-03-23T18:23:00.000Z",
  "deliveryId": "94e99821-..."
}
```

| Campo | Tipo | Descrição |
|---|---|---|
| `amountBrl` | `number` | Valor estornado em BRL |

---

## Eventos — PAYOUT

Fluxo: merchant envia USDC para a wallet da plataforma → 3xChange converte para BRL → envia PIX para o receiver.

### `order.deposit_received`

USDC detectado on-chain na wallet da plataforma. Processamento iniciado.

```json
{
  "event": "order.deposit_received",
  "orderId": "a828f06a-...",
  "externalId": "109",
  "type": "PAYOUT",
  "status": "DEPOSIT_RECEIVED",
  "amountCrypto": 10.0,
  "txHash": "5FJZcffw7fLRM...",
  "timestamp": "2026-03-23T18:18:00.000Z",
  "deliveryId": "11a00932-..."
}
```

| Campo | Tipo | Descrição |
|---|---|---|
| `amountCrypto` | `number` | Quantidade de USDC recebida on-chain |
| `txHash` | `string \| null` | Assinatura da transação Solana |

---

### `order.processing`

Conversão USDC → BRL e cálculo de taxas concluídos. PIX sendo enviado ao banco.

```json
{
  "event": "order.processing",
  "orderId": "a828f06a-...",
  "externalId": "109",
  "type": "PAYOUT",
  "status": "PIX_SENDING",
  "amountBrl": 51.98,
  "pixKey": "14516300699",
  "timestamp": "2026-03-23T18:18:10.000Z",
  "deliveryId": "22b11043-..."
}
```

| Campo | Tipo | Descrição |
|---|---|---|
| `amountBrl` | `number` | Valor líquido em BRL a ser enviado via PIX (após taxas) |
| `pixKey` | `string \| null` | Chave PIX de destino |

---

### `order.pix_sending`

PIX aceito pelo provedor de pagamentos. Aguardando liquidação bancária.

```json
{
  "event": "order.pix_sending",
  "orderId": "a828f06a-...",
  "externalId": "109",
  "type": "PAYOUT",
  "status": "PIX_SENDING",
  "amountBrl": 51.98,
  "pixKey": "14516300699",
  "threexpayTransactionId": "1fe6ae8b-bac2-4a1c-b609-f568123113c5",
  "timestamp": "2026-03-23T18:18:15.000Z",
  "deliveryId": "33c22154-..."
}
```

| Campo | Tipo | Descrição |
|---|---|---|
| `amountBrl` | `number` | Valor enviado via PIX em BRL |
| `pixKey` | `string` | Chave PIX de destino |
| `threexpayTransactionId` | `string` | ID da transação no provedor PIX |

---

### `order.completed` (PAYOUT)

PIX liquidado e confirmado pelo banco. Fluxo finalizado com sucesso.

```json
{
  "event": "order.completed",
  "orderId": "a828f06a-...",
  "externalId": "109",
  "type": "PAYOUT",
  "status": "COMPLETED",
  "amountBrl": 51.98,
  "pixKey": "14516300699",
  "pixEndToEndId": "E54811417202603231820RJUuDwUJw9l",
  "timestamp": "2026-03-23T18:20:45.713Z",
  "deliveryId": "61b66508-..."
}
```

| Campo | Tipo | Descrição |
|---|---|---|
| `amountBrl` | `number \| null` | Valor líquido enviado em BRL |
| `pixKey` | `string \| null` | Chave PIX de destino |
| `pixEndToEndId` | `string \| null` | ID end-to-end do PIX (rastreabilidade bancária) |

---

## Evento de erro — ambos os tipos

### `order.failed`

Ocorreu um erro em qualquer etapa do fluxo. O pedido foi encerrado.

```json
{
  "event": "order.failed",
  "orderId": "a828f06a-...",
  "externalId": "109",
  "type": "PAYOUT",
  "status": "FAILED",
  "failureReason": "PIX payout failed",
  "timestamp": "2026-03-23T18:20:45.713Z",
  "deliveryId": "44d33265-..."
}
```

| Campo | Tipo | Descrição |
|---|---|---|
| `failureReason` | `string` | Descrição do erro |

---

## Resumo dos fluxos

### PAYIN (PIX → Crypto)

```
order.payment_received
        ↓
  order.completed   ──── (erro) ──→  order.refunding → order.refunded
                                              ↓ (erro no estorno)
                                         order.failed
```

### PAYOUT (Crypto → PIX)

```
order.deposit_received
        ↓
order.processing
        ↓
order.pix_sending
        ↓
order.completed   ──── (erro em qualquer etapa) ──→  order.failed
```

---

## Configuração

Configure a URL e o secret do webhook no dashboard em **Configurações → Webhooks**, ou via API:

```http
PUT /api/v1/merchant/webhook
Authorization: Bearer <token>

{
  "webhookUrl": "https://sua-api.com/webhooks/3xchange",
  "webhookSecret": "seu_secret_aqui"
}
```

> O secret deve ter no mínimo 32 caracteres. Guarde-o com segurança — ele é usado para validar todas as entregas.
