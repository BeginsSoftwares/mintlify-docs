# Migração de cloud: AWS → Railway (design)

**Data:** 2026-06-02
**Status:** Design aprovado (4 seções validadas em brainstorm) — aguardando revisão da spec escrita
**Objetivo:** Sair 100% da AWS no backend do Oráculo, cortando o custo de ~$592/mês para ~$40-80/mês (~85-90%), mantendo o produto funcional.

---

## 1. Contexto e motivação

Custo AWS (maio/2026, ~$592/mês total) é insustentável. Breakdown:

| Serviço AWS | $/mês |
|---|---|
| EC2 Compute | 175.79 |
| RDS Aurora (Postgres) | 107.13 |
| ECS (Fargate) | 87.08 |
| EC2-Other (NAT Gateway, EBS, transfer) | 86.76 |
| ALB | 35.42 |
| VPC | 33.49 |
| ElastiCache (Redis) | 27.30 |
| Amazon MQ (RabbitMQ) | 22.08 |
| CloudWatch / ECR / Secrets / R53 | ~16 |

~$418 é compute + rede (EC2/ECS/NAT/ALB/VPC) — evapora no Railway. A infra é Pulumi (VPC completa: compute/data/dns/loadbalancing/networking/secrets/storage).

**Provedor escolhido: Railway** (managed, migração rápida, baixo ops). Cloudflare na frente (DNS/proxy/WAF). Frontend (`web/`, `admin/`) **fica no Vercel** (já não é AWS).

## 2. Topologia alvo

```
  Cloudflare (DNS + proxy)  →  api.oraculo.fun
        ▼
  Railway project "oraculo"
   ├─ Service: oraculo-api      (Dockerfile, domínio público)
   ├─ Service: oraculo-worker   (Dockerfile, sem domínio público) — roda os crons
   ├─ Plugin:  Postgres         (managed, backups automáticos)
   ├─ Plugin:  Redis            (managed)
   └─ Service: rabbitmq         (container + volume persistente)
  Frontend (Vercel, inalterado): oraculo.fun / backoffice
  Externos (inalterados): Solana RPC (Helius), 3xChange/Bitso, Clerk
```

**Mapeamento AWS → Railway:**

| AWS | Railway |
|---|---|
| ECS Fargate + EC2 (api+worker) | 2 serviços (oraculo-api, oraculo-worker) do Dockerfile (`api/Dockerfile`) |
| Aurora Postgres | Plugin Postgres (managed) |
| ElastiCache Redis | Plugin Redis (managed) |
| Amazon MQ (RabbitMQ) | Serviço rabbitmq (container + volume) |
| ALB + VPC + NAT | some — Railway dá domínio + TLS + rede privada |
| ECR | some — build do source |
| Secrets Manager (~45 secrets) | Railway variables (encrypted) |
| CloudWatch | logs nativos do Railway |

**Decisões fixadas:**
- Frontend fica no Vercel (não migra).
- RabbitMQ vira container no Railway (auto-gerenciado por nós, com volume) — replicar como está; avaliar simplificar depois.
- Worker roda os crons (settlement/reconciliação/expire-orders) como serviço long-running (não Railway Cron).
- CI/CD via integração nativa do Railway (auto-deploy no push pra `main`); remove o deploy ECS do GitHub Actions.

## 3. Migração do banco (Aurora → Railway Postgres)

Dados pequenos (clientes de teste, dezenas de MB). Aurora já está no schema chain-as-truth (pós-cutover). **Estratégia: `pg_dump` completo (schema+dados, formato custom) → `pg_restore` no Railway PG** numa janela curta (não precisa de replicação lógica nessa escala).

**Ensaio primeiro (de-risk, sem downtime):** provisiona o Railway PG → dump Aurora → restore Railway → sobe a app no domínio Railway (sem cutover de DNS) → smoke (drift-audit, health, trade de teste). Valida tudo antes de cortar.

**Pontos de atenção:**
- **Integridade:** contagem de linhas por tabela (Aurora vs Railway) + `trading-v2-drift-audit.ts` no Railway (0 negativos; saldos batendo com a chain).
- **Connection pool:** Railway PG tem limite de conexões por plano. Setar `DATABASE_POOL_SIZE` compatível (o fix `PrismaPg max=poolSize` já está no código); avaliar PgBouncer se necessário.
- **Extensões/sequences:** conferir extensões usadas (ex.: `pgcrypto`) existem no Railway PG; `pg_restore` cuida das sequences.
- **Rollback:** Aurora fica viva e intocada (só lemos no dump) → reverter é imediato (re-apontar DATABASE_URL).

## 4. Sequência de cutover + rollback

### PREP (antecipado, zero downtime)
1. Provisiona projeto Railway: Postgres, Redis, serviço rabbitmq, serviços oraculo-api + oraculo-worker (Dockerfile). Migra os ~45 secrets.
2. Liga via rede privada (DATABASE_URL→PG interno, REDIS→Redis, RABBITMQ→serviço).
3. Ensaio do DB (Seção 3) + smoke no domínio Railway.
4. Adiciona `api.oraculo.fun` como custom domain no Railway (provisiona TLS); DNS no Cloudflare ainda → AWS.

### CUTOVER (janela curta, ~10-20 min downtime)
1. Congela AWS: scale ECS api+worker → 0.
2. Sync final: `pg_dump` final da Aurora → `pg_restore` no Railway PG → **verifica contagem de linhas**.
3. Sobe app no Railway (api+worker) apontando pro Railway PG/Redis/RabbitMQ → confirma health.
4. Vira DNS: Cloudflare `api.oraculo.fun` CNAME → domínio Railway (era → ALB AWS).
5. Verifica: `/health` (→ Railway), drift-audit, fluxo real (depósito/trade/cancel + saldo/posições no front), webhook de teste (3xChange).
6. Downtime encerra.

### Rollback
- DNS: volta CNAME `api.oraculo.fun` → ALB AWS.
- Compute: scale ECS AWS → 1 (intacto).
- DB: Aurora viva e intocada → imediato.
- ⚠️ Tradeoff: escritas no Railway pós-cutover se perdem no rollback pra Aurora. Mitigação: verificar rápido; rollback imediato se quebrar. Aceitável p/ clientes de teste.

## 5. Decomissionamento da AWS (realizar a economia)

Após **janela de estabilidade de ~alguns dias** (AWS + Railway em paralelo, observando o Railway):
1. **Snapshot final da Aurora** (arquival) antes de deletar RDS — guardar semanas.
2. Confirmar todos os secrets migrados.
3. `pulumi destroy` do stack de prod (derruba ECS, Aurora, ElastiCache, MQ, ALB, NAT, VPC, ECR, CloudWatch de forma gerenciada).
4. Route53: se DNS for 100% Cloudflare, sai junto; senão manter.
- Resultado: bill AWS → ~$0 (resíduo de snapshots). Railway ~$40-80/mês.

## 6. CI/CD

Trocar `deploy-api.yml` (deploy ECS) pela integração nativa do Railway (auto-deploy no push pra `main`). Remover o job `deploy` (ECS) pra não deployar em infra morta. Manter os jobs de teste/e2e como gate (rodam contra serviços de teste do CI, não Railway).

## 7. Validação

- **Ensaio:** drift-audit + health + trade de teste no domínio Railway.
- **Pós-cutover:** `/health` em api.oraculo.fun → Railway; drift-audit; fluxo real (depósito→trade→cancel→saldo/posições); webhook de teste 3xChange/Bitso.
- **Janela de estabilidade** antes do `pulumi destroy`.

## 8. Riscos

- **Integridade do DB** no dump/restore → mitigado por contagem de linhas + drift-audit + ensaio prévio.
- **Connection pool** do Railway PG → setar `DATABASE_POOL_SIZE`; PgBouncer se preciso.
- **Webhooks** (3xChange/Bitso) apontados pra api.oraculo.fun → seguem funcionando após DNS flip (mesmo path); validar com evento de teste.
- **RabbitMQ** self-run no Railway → volume persistente; validar durabilidade.
- **Secrets** (~45, incl. SOLANA_PRIVATE_KEY) → migrar com cuidado, sem vazar.
- **Cloudflare proxied + Railway TLS** → conferir modo SSL (Full) no cutover de DNS.
- **Escritas perdidas no rollback** → verificar rápido pós-cutover.

## 9. Não-objetivos (YAGNI)

- Não migrar o frontend (fica no Vercel).
- Não fazer replicação lógica zero-downtime (overkill p/ o tamanho dos dados).
- Não reescrever/remover o RabbitMQ agora (replicar como está; simplificar é projeto à parte).
- Não mexer no contrato on-chain, Solana, ou na lógica de trading (só infra).
