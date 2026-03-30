# Sistema de Afiliados e Indicação — Oraculo

## Estado Atual (já existe no schema)

O model `User` já possui os campos base:
```prisma
referralCount           Int    @default(0)   // total de indicados
qualifiedReferrals      Int    @default(0)   // indicados que realmente operaram
affiliateRewardsEarned  Float  @default(0)   // total ganho em comissões
affiliateRewardsPending Float  @default(0)   // aguardando pagamento
affiliateRewardsClaimed Float  @default(0)   // já pago
referredBy              String?              // walletAddress de quem indicou
```

**O que falta:** código de indicação, tabelas de rastreamento de recompensas, lógica de ativação, sistema de tiers, e saldo de boas-vindas.

---

## Visão Geral da Arquitetura

```
┌─────────────────────────────────────────────────────────┐
│                     FLUXO PRINCIPAL                      │
│                                                          │
│  Novo usuário visita oraculo.fun/?ref=ABC123             │
│         ↓                                               │
│  Cookie salva o código (30 dias)                         │
│         ↓                                               │
│  Usuário cria conta (Clerk webhook)                      │
│         ↓                                               │
│  Sistema cria Referral (PENDING) + BonusCredit           │
│         ↓                                               │
│  Usuário faz 1º trade real                               │
│         ↓                                               │
│  Referral → ACTIVE, bonuses → APPROVED → creditados      │
│         ↓                                               │
│  A cada trade do indicado → comissão → afiliado          │
└─────────────────────────────────────────────────────────┘
```

---

## Schema de Banco de Dados

### Novas tabelas necessárias

```prisma
model ReferralCode {
  id        String   @id @default(uuid())
  userId    Int      @unique
  code      String   @unique          // ex: "RAFAEL10" ou "xK7mP2"
  isCustom  Boolean  @default(false)  // vanity code para afiliados
  isActive  Boolean  @default(true)
  createdAt DateTime @default(now())

  user      User       @relation(fields: [userId], references: [id])
  referrals Referral[]

  @@map("referral_codes")
}

model Referral {
  id             String         @id @default(uuid())
  referralCodeId String
  referrerId     Int
  refereeId      Int            @unique  // um usuário só pode ser indicado 1x
  status         ReferralStatus @default(PENDING)
  createdAt      DateTime       @default(now())
  activatedAt    DateTime?      // quando fez o 1º trade

  referralCode   ReferralCode   @relation(fields: [referralCodeId], references: [id])
  referrer       User           @relation("referrer", fields: [referrerId], references: [id])
  referee        User           @relation("referee", fields: [refereeId], references: [id])
  bonusCredits   BonusCredit[]

  @@index([referrerId])
  @@index([refereeId])
  @@map("referrals")
}

enum ReferralStatus {
  PENDING    // cadastrou mas não operou
  ACTIVE     // fez o 1º trade (qualificado)
  REWARDED   // bonus distribuído
  EXPIRED    // passou 30 dias sem operar
  INVALID    // suspeita de fraude
}

model BonusCredit {
  id         String      @id @default(uuid())
  userId     Int
  amount     Decimal     @db.Decimal(18, 6)
  type       BonusType
  status     BonusStatus @default(PENDING)
  reason     String
  referralId String?
  expiresAt  DateTime?   // bônus expiram se não ativados
  createdAt  DateTime    @default(now())
  paidAt     DateTime?

  user       User        @relation(fields: [userId], references: [id])
  referral   Referral?   @relation(fields: [referralId], references: [id])

  @@index([userId])
  @@index([status])
  @@map("bonus_credits")
}

enum BonusType {
  WELCOME           // boas-vindas para o novo usuário
  REFERRAL_SIGNUP   // afiliado ganha quando indicado cadastra
  REFERRAL_TRADE    // afiliado ganha quando indicado opera pela 1ª vez
  AFFILIATE_FEE     // % das taxas geradas pelo indicado (ongoing)
  ADMIN_GRANT       // crédito manual pelo admin
}

enum BonusStatus {
  PENDING    // aguardando condição (ex: 1º trade)
  APPROVED   // condição cumprida, pronto para creditar
  PAID       // creditado no saldo
  EXPIRED    // condição nunca cumprida
  CANCELLED  // cancelado manualmente
}

model AffiliateEarning {
  id          String   @id @default(uuid())
  affiliateId Int
  refereeId   Int
  orderId     String   // ordem que gerou a taxa
  feeAmount   Decimal  @db.Decimal(18, 6)  // taxa total da plataforma
  commission  Decimal  @db.Decimal(18, 6)  // parte do afiliado
  tierLevel   String   // bronze / silver / gold
  createdAt   DateTime @default(now())

  affiliate   User     @relation("affiliate_earnings", fields: [affiliateId], references: [id])
  referee     User     @relation("referee_earnings", fields: [refereeId], references: [id])

  @@index([affiliateId])
  @@index([refereeId])
  @@map("affiliate_earnings")
}
```

---

## Tiers de Afiliado

| Tier   | Indicados ativos | Comissão | Duração fee share | Código custom |
|--------|-----------------|----------|-------------------|---------------|
| Bronze | 0–9             | 10%      | 30 dias           | Não           |
| Silver | 10–49           | 15%      | 60 dias            | Não           |
| Gold   | 50+             | 20%      | 90 dias            | Sim           |

> "Indicado ativo" = fez pelo menos $5 em volume de trades.

### Exemplo de comissão

Indicado aposta $100 → plataforma ganha $1 (1% fee) → afiliado Gold recebe $0.20 (20%)

---

## Estratégia de Bônus

### Saldo inicial para novos usuários

| Condição                    | Bônus    | Tipo         |
|-----------------------------|----------|--------------|
| Cadastro simples            | $1 USDC  | WELCOME      |
| Cadastro via indicação      | $2 USDC  | WELCOME      |
| 1º depósito de qualquer valor | +$1 USDC | ADMIN_GRANT |

**Importante:** o bônus de boas-vindas só é creditado no saldo após o usuário fazer o **1º trade real** (com dinheiro próprio ou do bônus com depósito). Isso evita que pessoas criem contas só para sacar o bônus.

### Recompensas para o afiliado

| Evento                         | Recompensa |
|--------------------------------|------------|
| Indicado se cadastra           | $0.50      |
| Indicado faz 1º trade          | $0.50      |
| Cada trade do indicado (ongoing) | % das fees |

### Exemplo completo (afiliado Bronze)

```
Rafael indica João:
  → João cadastra: Rafael ganha $0.50 (PENDING)
  → João faz 1º trade de $10:
      - João recebe bônus de $2 (WELCOME creditado)
      - Rafael recebe +$0.50 (REFERRAL_TRADE creditado)
      - Rafael recebe +$0.01 (10% do fee $0.10 da plataforma)
  → João faz mais trades nos próximos 30 dias:
      - Rafael continua ganhando 10% das fees
```

---

## Fluxo de Implementação

### 1. Geração de código de indicação

```
GET /api/v1/referral/my-code
→ Se usuário não tem código: gerar automaticamente (6 chars base62)
→ Retorna: { code: "xK7mP2", url: "https://oraculo.fun/?ref=xK7mP2", stats: {...} }
```

Algoritmo de geração:
```typescript
// 6 chars base62 = 56 bilhões de combinações
const chars = '0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz'
const code = Array.from({length: 6}, () => chars[Math.floor(Math.random() * 62)]).join('')
```

### 2. Integração no cadastro (Clerk webhook)

Em `clerk-webhook-processor.use-case.ts`, após criar o usuário:

```typescript
// 1. Ler referral code do metadata do Clerk (passado no frontend)
const refCode = event.data.unsafe_metadata?.referralCode

// 2. Encontrar o código e o dono
if (refCode) {
  const code = await prisma.referralCode.findUnique({ where: { code: refCode } })
  if (code && code.userId !== newUser.id) {  // previne auto-indicação
    await prisma.referral.create({
      data: {
        referralCodeId: code.id,
        referrerId: code.userId,
        refereeId: newUser.id,
        status: 'PENDING',
      }
    })
    // Salva no User.referredBy
    await prisma.user.update({
      where: { id: newUser.id },
      data: { referredBy: code.user.walletAddress }
    })
    // Cria bônus de boas-vindas MAIOR para indicado (PENDING)
    await prisma.bonusCredit.create({
      data: { userId: newUser.id, amount: 2.00, type: 'WELCOME',
              status: 'PENDING', reason: 'Indicado por afiliado', referralId: referral.id,
              expiresAt: addDays(new Date(), 30) }
    })
    // Cria bônus de SIGNUP para o afiliado (PENDING)
    await prisma.bonusCredit.create({
      data: { userId: code.userId, amount: 0.50, type: 'REFERRAL_SIGNUP',
              status: 'PENDING', reason: `Indicação de ${newUser.username}`, referralId: referral.id }
    })
  }
} else {
  // Bônus padrão sem indicação
  await prisma.bonusCredit.create({
    data: { userId: newUser.id, amount: 1.00, type: 'WELCOME',
            status: 'PENDING', reason: 'Bônus de boas-vindas',
            expiresAt: addDays(new Date(), 30) }
  })
}
```

### 3. Ativação no 1º trade (settlement.service.ts)

Após um trade ser liquidado com sucesso, verificar se é o 1º trade do usuário:

```typescript
async function checkAndActivateReferral(userWallet: string) {
  // Contar trades anteriores
  const tradeCount = await prisma.marketTrade.count({ where: { userWallet } })
  if (tradeCount !== 1) return  // só no exato 1º trade

  const user = await prisma.user.findUnique({ where: { walletAddress: userWallet } })

  // 1. Ativar todos os BonusCredits PENDING do usuário → APPROVED
  await prisma.bonusCredit.updateMany({
    where: { userId: user.id, status: 'PENDING', type: 'WELCOME' },
    data: { status: 'APPROVED', paidAt: new Date() }
  })

  // 2. Creditar no saldo
  const bonuses = await prisma.bonusCredit.findMany({
    where: { userId: user.id, status: 'APPROVED', type: 'WELCOME' }
  })
  const totalBonus = bonuses.reduce((sum, b) => sum + Number(b.amount), 0)
  await userAccountRepository.incrementBalance(userWallet, totalBonus)

  // 3. Se veio por indicação, ativar a referral
  if (user.referredBy) {
    const referral = await prisma.referral.findFirst({
      where: { refereeId: user.id, status: 'PENDING' }
    })
    if (referral) {
      await prisma.referral.update({
        where: { id: referral.id },
        data: { status: 'ACTIVE', activatedAt: new Date() }
      })

      // Pagar bônus REFERRAL_SIGNUP ao afiliado (já estava PENDING)
      await payPendingBonus(referral.referrerId, 'REFERRAL_SIGNUP', referral.id)

      // Criar e pagar bônus REFERRAL_TRADE ao afiliado
      await createAndPayBonus(referral.referrerId, 0.50, 'REFERRAL_TRADE', referral.id)

      // Atualizar contadores do afiliado
      await prisma.user.update({
        where: { id: referral.referrerId },
        data: {
          referralCount: { increment: 1 },
          qualifiedReferrals: { increment: 1 }
        }
      })
    }
  }
}
```

### 4. Comissão contínua (settlement.service.ts)

A cada trade liquidado com sucesso, verificar se o usuário foi indicado e ainda está dentro do período de comissão:

```typescript
async function distributeAffiliateCommission(userWallet: string, feeAmount: number, orderId: string) {
  const user = await prisma.user.findUnique({ where: { walletAddress: userWallet } })
  if (!user?.referredBy) return

  const referral = await prisma.referral.findFirst({
    where: { refereeId: user.id, status: 'ACTIVE' },
    include: { referralCode: { include: { user: true } } }
  })
  if (!referral) return

  // Verificar se ainda está dentro do período de comissão
  const tier = getAffiliateTier(referral.referralCode.user.qualifiedReferrals)
  const daysActive = differenceInDays(new Date(), referral.activatedAt!)
  if (daysActive > tier.commissionDays) return

  const commission = feeAmount * (tier.commissionRate / 100)
  if (commission < 0.001) return  // mínimo para valer a pena

  // Registrar e creditar
  await prisma.affiliateEarning.create({
    data: {
      affiliateId: referral.referrerId,
      refereeId: user.id,
      orderId,
      feeAmount,
      commission,
      tierLevel: tier.name,
    }
  })

  await userAccountRepository.incrementBalance(referral.referralCode.user.walletAddress, commission)

  await prisma.user.update({
    where: { id: referral.referrerId },
    data: {
      affiliateRewardsEarned: { increment: commission },
      affiliateRewardsClaimed: { increment: commission },
    }
  })
}

function getAffiliateTier(qualifiedReferrals: number) {
  if (qualifiedReferrals >= 50) return { name: 'gold',   commissionRate: 20, commissionDays: 90 }
  if (qualifiedReferrals >= 10) return { name: 'silver', commissionRate: 15, commissionDays: 60 }
  return                               { name: 'bronze', commissionRate: 10, commissionDays: 30 }
}
```

---

## Rotas da API

### Usuário
```
GET  /api/v1/referral/my-code          → código + URL de indicação
GET  /api/v1/referral/stats            → indicados, ganhos, tier atual
GET  /api/v1/referral/history          → lista de indicados com status
POST /api/v1/referral/validate/:code   → valida se um código existe (pré-cadastro)
```

### Admin
```
GET  /api/v1/admin/affiliate/overview              → stats globais
GET  /api/v1/admin/affiliate/leaderboard           → top afiliados
GET  /api/v1/admin/affiliate/users/:userId         → detalhes de um afiliado
POST /api/v1/admin/affiliate/grant-bonus           → conceder bônus manual
POST /api/v1/admin/affiliate/set-custom-code       → código vanity para Gold
POST /api/v1/admin/affiliate/invalidate/:referralId → marcar como fraude
```

---

## Anti-Abuso

### Regras de validação

| Regra | Implementação |
|-------|---------------|
| Sem auto-indicação | `code.userId !== newUser.id` |
| 1 código por usuário | `refereeId @unique` na tabela Referral |
| Bônus só após 1º trade | status PENDING → APPROVED só no 1º trade |
| Expiração de bônus | `expiresAt` 30 dias; job diário move PENDING → EXPIRED |
| Volume mínimo para qualificar | `$5 em trades` para contar como qualifiedReferral |
| Depósito mínimo para comissão | só gerar AffiliateEarning em trades com USDC real depositado |

### Proteções adicionais (fase 2)

- **IP fingerprint**: máximo 3 contas por IP recebem bônus de boas-vindas
- **Email domain blacklist**: bloquear emails temporários (mailinator, guerrilla, etc.)
- **Phone verification**: exigir celular para liberar bônus acima de $2
- **Velocity check**: afiliado que indica 10+ usuários em 24h entra em revisão manual
- **Circular referral detection**: A indica B, B indica A → ambos invalidados

---

## Estratégias de Crescimento

### Estratégia 1 — Lançamento (mês 1-2)
- Todos os usuários existentes recebem código automaticamente
- Campanha de e-mail anunciando o programa
- Primeiros 100 afiliados que atingirem Silver ganham status Gold por 30 dias

### Estratégia 2 — Criadores de conteúdo
- Códigos vanity personalizados (`oraculo.fun/?ref=NOMECRIADOR`)
- Dashboard com analytics: cliques, conversões, ganhos em tempo real
- Revenue share de 20% por 90 dias para influencers com >1000 seguidores

### Estratégia 3 — Torneiro de afiliados (mensal)
- Ranking público dos top 10 afiliados do mês
- Prêmio extra para o 1º colocado: $50 USDC + badge exclusivo no perfil
- Gera gamificação e competição saudável

### Estratégia 4 — Saldo inicial escalonado
```
Sem código de indicação:  $1 USDC (expira em 30 dias se não usar)
Com código de indicação:  $2 USDC
Com depósito de $10+:    +$1 USDC extra
Com KYC completo:        +$2 USDC extra
```
Máximo de $5 em bônus por usuário.

### Estratégia 5 — "Boost" de indicações
- Durante eventos especiais (lançamento de novo mercado), dobrar as comissões por 48h
- Notificação push/email para afiliados: "Comissão 2x até domingo!"

---

## Implementação em Fases

### Fase 1 — Fundação (prioridade alta)
- [ ] Migration: tabelas `referral_codes`, `referrals`, `bonus_credits`, `affiliate_earnings`
- [ ] Gerar código automaticamente ao criar usuário (Clerk webhook)
- [ ] `GET /api/v1/referral/my-code`
- [ ] Saldo inicial ($1 USDC) creditado no 1º trade
- [ ] Bônus de indicação simples (sem tiers): $0.50 signup + $0.50 1º trade

### Fase 2 — Comissão contínua
- [ ] Integração no `settlement.service.ts` para distribuir commission por fee
- [ ] Cálculo de tier baseado em `qualifiedReferrals`
- [ ] `GET /api/v1/referral/stats` com ganhos acumulados
- [ ] Job diário para expirar BonusCredits PENDING

### Fase 3 — Admin e anti-abuso
- [ ] Dashboard admin com métricas de afiliados
- [ ] Sistema de invalidação de fraudes
- [ ] IP fingerprinting no cadastro
- [ ] Expiração de referrals PENDING após 30 dias

### Fase 4 — Growth features
- [ ] Códigos vanity para Gold
- [ ] Dashboard público de leaderboard
- [ ] Campanha boost temporária
- [ ] Integração com analytics (UTM tracking)

---

## Métricas para Acompanhar

| Métrica | Objetivo |
|---------|----------|
| Referral conversion rate | % de links clicados que viram cadastros |
| Activation rate | % de cadastros que fazem o 1º trade |
| Qualified referral rate | % dos ativados que atingem $5 de volume |
| Affiliate revenue share | % da receita da plataforma paga em comissões |
| CAC via afiliado vs. orgânico | custo de aquisição comparado |
| Churn de indicados | indicados ficam mais tempo que orgânicos? |

---

## Variáveis de Ambiente Necessárias

```env
WELCOME_BONUS_AMOUNT=1.00          # bônus padrão (sem indicação)
WELCOME_BONUS_WITH_REF_AMOUNT=2.00 # bônus com indicação
REFERRAL_SIGNUP_BONUS=0.50         # afiliado ganha no signup do indicado
REFERRAL_TRADE_BONUS=0.50          # afiliado ganha no 1º trade do indicado
AFFILIATE_MIN_VOLUME_USD=5.00      # volume mínimo para qualificar indicado
BONUS_EXPIRY_DAYS=30               # dias para expirar bônus não utilizado
```

---

## Notas de Integração com a Arquitetura Atual

1. **`clerk-webhook-processor.use-case.ts`** → ponto de entrada do referral no cadastro
2. **`settlement.service.ts`** → ponto de ativação e de comissão contínua
3. **`UserAccount.balance`** → onde os bônus são creditados (já existente)
4. **`ledger_events`** → registrar cada crédito de bônus como `bonus_credit` para auditoria
5. **Frontend** → passar `referralCode` no `unsafe_metadata` do Clerk no momento do signup
6. **Cookie** → salvar `?ref=CODE` por 30 dias no browser para usar no cadastro posterior
