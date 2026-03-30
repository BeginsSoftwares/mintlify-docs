# Economia do Mercado de Predição — Análise de "Always Win"

## Contexto

Quando um usuário aposta $1 em um mercado 50/50, ele recebe **1.98 tokens YES**.
A questão: isso permite que o usuário "sempre ganhe"?

---

## As duas fórmulas centrais

### Compra (modelo Kalshi)
```
tokens = USDC_líquido / preço
       = ($1.00 × 0.99) / 0.50
       = 1.98 tokens YES
```

### Venda via AMM (antes da resolução)
```
usdcReceived = tokens × preçoAtual × 0.99
             = 1.98 × 0.50 × 0.99
             = $0.98
```

Comprar e imediatamente vender = **perda de $0.02 (~2%)** — dois fees de 1% cada.

---

## Caso 1 — Venda via AMM (antes da resolução)

O usuário pode vender seus tokens a qualquer momento ao preço de mercado atual.

### Venda imediata (preço não mudou)
| Ação | Valor |
|------|-------|
| Investido | $1.00 |
| Tokens recebidos | 1.98 YES |
| Venda ao preço 50% | 1.98 × 0.50 × 0.99 = **$0.98** |
| **Resultado** | **−$0.02 (perda)** ✅ |

### Venda após preço subir (usuário acertou a direção)
| Ação | Valor |
|------|-------|
| Investido | $1.00 |
| Tokens recebidos | 1.98 YES (comprados a 50%) |
| Venda ao preço 70% | 1.98 × 0.70 × 0.99 = **$1.37** |
| **Resultado** | **+$0.37 (lucro)** ✅ |

### Venda após preço cair (usuário errou a direção)
| Ação | Valor |
|------|-------|
| Investido | $1.00 |
| Tokens recebidos | 1.98 YES (comprados a 50%) |
| Venda ao preço 30% | 1.98 × 0.30 × 0.99 = **$0.59** |
| **Resultado** | **−$0.41 (perda)** ✅ |

> **Conclusão**: O usuário lucra na venda AMM somente se acertou a direção do mercado. Isso é **correto e intencional** — é o funcionamento de qualquer mercado de predição.

---

## Caso 2 — Resolução do mercado

Ao fim do mercado, o lado vencedor paga **$1.00 por token**.

| Cenário | Tokens | Payout | Investido | Resultado |
|---------|--------|--------|-----------|-----------|
| YES ganha | 1.98 YES | 1.98 × $1 = **$1.98** | $1.00 | **+$0.98** |
| NO ganha  | 1.98 YES | 1.98 × $0 = **$0.00** | $1.00 | **−$1.00** |

### Valor esperado (mercado 50/50)
```
EV = 50% × $1.98 + 50% × $0.00 = $0.99
```

O usuário investe $1.00 e o valor esperado de retorno é **$0.99** → **casa tem edge de 1%** ✅

---

## Por que o vault não quebra

Para cada $1 apostado em YES, o sistema faz um matching complementar e compra NO:

```
Usuário:  $1 → vault, recebe 1.98 YES tokens
Platform: $1 → vault, recebe tokens NO (quantidade varia conforme preço)

Vault total: $2.00
```

Na resolução:
- YES ganha → paga 1.98 × $1 = **$1.98** (vault tinha $2.00) ✅
- NO ganha  → paga tokens_NO × $1 (vault tinha $2.00) ✅

O vault é sempre solvente porque o matching complementar deposita $1 para cada $1 do usuário.

---

## Deriva de preço após o matching complementar

O matching usa **o mesmo USDC** para YES e NO, mas em preços diferentes (pois a primeira compra move o preço):

```
1. Usuário compra $1 YES a 50% → 1.98 YES tokens
   → yesSupply sobe, yesPrice vai para ~60%

2. Platform compra $1 NO a ~40% → 2.47 NO tokens
   → noSupply sobe, yesPrice cai para ~45%
```

Após o matching: `yesPrice ≈ 45%`

Venda dos 1.98 tokens a 45%: `1.98 × 0.45 × 0.99 = $0.88` → **perda maior ainda**

> O usuário que tenta arbitrar vendendo imediatamente perde mais do que os 2% de fee.

---

## Vulnerabilidade teórica: janela entre transações

O matching complementar envolve **duas transações separadas** (~0.7s de intervalo):

1. `buyYes` → yesPrice sobe para ~60%
2. `buyNo`  → yesPrice volta para ~45%

Se um usuário conseguisse vender seus YES tokens **entre as duas transações** (yesPrice = 60%):
```
1.98 × 0.60 × 0.99 = $1.175 → lucro de $0.175
```

**Na prática**: não explorável por usuários normais (não têm como observar e reagir em 0.7s via API).

---

## Resumo

| Cenário | Resultado para o usuário | É um problema? |
|---------|--------------------------|----------------|
| Compra + venda imediata (mesmo preço) | −$0.02 (perda de 2%) | ✅ Não |
| Compra + venda após preço cair | Perda > 2% | ✅ Não |
| Compra + venda após preço subir | Lucro (acertou a direção) | ✅ Não — é o ponto do mercado |
| Resolução com lado certo | Lucro de até ~98% no valor apostado | ✅ Não — é o ponto do mercado |
| Resolução com lado errado | Perda de 100% | ✅ Não |
| Front-run entre transações do matching | Lucro de ~17% | ⚠️ Teórico, não prático |

**O usuário NÃO pode "sempre ganhar".** O valor esperado de qualquer aposta é **$0.99 por $1.00 investido** — a casa tem edge de 1% (o fee de taker).
