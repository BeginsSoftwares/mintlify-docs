# Migração AWS → Railway — Plano de Implementação

> **For agentic workers:** este é um RUNBOOK de infra. A maioria dos passos é executada pelo operador humano (conta Railway, credenciais AWS de prod, DNS Cloudflare). O agente coordena e verifica via read-only (profile AWS `oraculo`). Steps em checkbox (`- [ ]`).

**Goal:** Mover o backend do Oráculo (api + worker + Postgres + Redis + RabbitMQ) da AWS pro Railway, com Cloudflare na frente, cortando ~$592→~$40-80/mês. Frontend fica no Vercel.

**Arquitetura:** Railway project com 2 serviços (api, worker) do `api/Dockerfile` + plugins Postgres/Redis managed + serviço rabbitmq containerizado. Migração de DB via `pg_dump`/`pg_restore` em janela curta, com ensaio prévio. Cutover via flip de DNS no Cloudflare. AWS desmontada via `pulumi destroy` após estabilidade.

**Spec:** `docs/superpowers/specs/2026-06-02-railway-migration-design.md`

**Pré-requisitos do operador:**
- Conta Railway + `railway` CLI (`npm i -g @railway/cli`; `railway login`).
- Profile AWS `oraculo` (já configurado — conta 429441944959).
- Acesso ao DNS do Cloudflare (zona oraculo.fun).
- `pg_dump`/`pg_restore` (Postgres client) local.
- `DATABASE_URL` da Aurora de prod (do Secrets Manager / task def).

---

## FASE 0 — Preparação e decisões

### Task 0.1 — Determinar o tier do Railway + comando de start do worker ✅ RESOLVIDO
- [x] **Tier/conexões:** **Pro ($20/mo)**. `DATABASE_POOL_SIZE=20` (sem PgBouncer; reavaliar se houver esgotamento de conexões).
- [x] **Comando de start do worker:** confirmado via `describe-task-definition oraculo-prod-worker` → CMD override = `["worker.ts"]`. `api/Dockerfile` tem `ENTRYPOINT ["bun","run"]` + `CMD ["dist/index.js"]`.
  - **oraculo-api** → start command **default** (`bun run dist/index.js`). Nada a setar no Railway.
  - **oraculo-worker** → **Custom Start Command = `worker.ts`** (ENTRYPOINT `bun run` preservado → `bun run worker.ts`, idêntico à ECS).
- [x] **worker.ts na imagem:** `api/Dockerfile:41` copia `worker.ts` pro estágio final. OK.

---

## FASE 1 — Provisionar Railway (sem downtime)

### Task 1.1 — Criar projeto + serviços + plugins
> **Ambiente:** `railway init` (criar projeto) TRAVA nessa máquina (mutação GraphQL dá timeout — embora crie o projeto server-side; cuidado com duplicados). `railway add` e `railway delete` funcionam (retornam rápido). Projeto criado e linkado: **oraculo** `5dcab818-ec13-4eb2-8458-61b86083513d`, env `production`.
- [x] Projeto `oraculo` criado e linkado (`railway link --project 5dcab818-…`). Duplicados criados pelos retries do init foram deletados.
- [x] Postgres: `railway add --database postgres` → service `9683a873-7979-4f22-ab93-bb69cb294152`.
- [x] Redis: `railway add --database redis` → service `1c2c62be-7942-4398-96af-3b3346fc5213`.
- [x] rabbitmq: `railway add --service rabbitmq --image rabbitmq:3-management` → service `ac80454a-0657-4097-9ea0-84892c27903e`.
- [ ] **`oraculo-api`** — BLOQUEADO: `railway add --repo BeginsSoftwares/Oraculo` retorna "Unauthorized" (GitHub App do Railway não conectado à org). **Dashboard:** conectar GitHub (autorizar Railway App na org `BeginsSoftwares`, dar acesso ao repo Oraculo) → New → GitHub Repo → `BeginsSoftwares/Oraculo`. Settings: Service Name `oraculo-api`, **Root Directory `api`**, Builder Dockerfile, Start Command vazio (default `bun run dist/index.js`).
- [ ] **`oraculo-worker`** — mesmo repo. Settings: Service Name `oraculo-worker`, **Root Directory `api`**, Builder Dockerfile, **Start Command `worker.ts`**.
- [ ] **Volume do rabbitmq** (dashboard): rabbitmq → Volumes → New Volume → mount path `/var/lib/rabbitmq`.
- [ ] **Verificação:** os 5 componentes aparecem no projeto. `railway status --json` lista Redis/Postgres/rabbitmq (✅) + oraculo-api/oraculo-worker (pendente). Deixar deploys falharem por falta de env é esperado (resolve na Fase 2).

### Task 1.2 — Rede privada entre serviços ✅
Env names confirmados em `api/src/shared/utils/env.ts`: `REDIS_URI`, `RABBITMQ_URL` (z.string().url(), obrigatório). `DATABASE_URL` NÃO está no schema zod (Prisma lê direto do `process.env`). Pool=`DATABASE_POOL_SIZE`.
- [x] rabbitmq creds: `RABBITMQ_DEFAULT_USER=oraculo` + `RABBITMQ_DEFAULT_PASS` (rand 32 hex) setados no serviço rabbitmq.
- [x] `oraculo-api` e `oraculo-worker` (via `railway variable set ... --skip-deploys`, refs do Railway):
  - `DATABASE_URL=${{Postgres.DATABASE_URL}}` → resolve `postgres.railway.internal:5432/railway` ✓
  - `REDIS_URI=${{Redis.REDIS_URL}}` → `redis.railway.internal:6379` ✓
  - `RABBITMQ_URL=amqp://${{rabbitmq.RABBITMQ_DEFAULT_USER}}:${{rabbitmq.RABBITMQ_DEFAULT_PASS}}@${{rabbitmq.RAILWAY_PRIVATE_DOMAIN}}:5672` → `rabbitmq.railway.internal:5672` ✓
  - `DATABASE_POOL_SIZE=20`
- [x] **Verificação:** `railway variable list` confirma refs resolvidas nos dois serviços.
> Nota CLI: `railway variable set` às vezes dá timeout na resposta mas grava (igual ao `init`) — verificar com `variable list` em vez de re-tentar cegamente.

---

## FASE 2 — Migrar env + secrets

**Estrutura na AWS (task def `oraculo-prod-api`):** 33 envs planas (`environment[]`, texto puro) + 46 secrets (`secrets[]`). Os secrets vêm de DOIS Secrets Manager:
- `oraculo/prod/database-url` → standalone (NÃO migra; no Railway `DATABASE_URL` = ref do Postgres).
- `oraculo/prod/app` → **um JSON com as outras 45 chaves** (mesmos nomes das vars do Railway).

`MONGODB_URI`: legado, sem conexão no boot — não migra.

### Task 2.1 — Rodar `api/scripts/railway-migrate-env.sh` (operador) ✅
Script idempotente, sem valores hardcoded (lê tudo da AWS em runtime), seta nos 2 serviços via `railway variable set --stdin --skip-deploys`. Re-runs pulam o que já existe (snapshot inicial) e pulam valores vazios.
- [x] `bash api/scripts/railway-migrate-env.sh` — todas as vars não-vazias setadas em oraculo-api + oraculo-worker.
- Overrides embutidos: pula `RABBITMQ_URL`/`REDIS_URI` (refs do Railway), `LOKI_URL`/`OTEL_EXPORTER_OTLP_ENDPOINT` (Loki/Jaeger internos AWS), `NODE_TLS_REJECT_UNAUTHORIZED=0` (dropado → restaura validação TLS); força `ENABLE_TRACING=false`.
- `THREEXCHANGE_MERCHANT_WALLET_ADDRESS` está **vazio em prod** (string ""), é opcional no `env.ts` → ignorado (CLI não seta vazio). Não é pendência.
- Extras vindos do secret JSON: `GF_SECURITY_ADMIN_PASSWORD` (Grafana, inócuo), `SLACK_ALERTS_WEBHOOK_URL` (usado pelo env.ts).

### Task 2.2 — Resíduo AWS (afeta Fase 7, NÃO bloqueia agora)
- **S3 continua em runtime:** `AWS_S3_BUCKET_KYC`/`S3_UPLOADS_*` (bucket `oraculo-prod-uploads`) — upload de KYC/fotos/imagens. `pulumi destroy` derruba compute/db/cache/mq mas os **buckets S3 + IAM user ficam** (custo residual baixo). "0% AWS" só migrando S3→Cloudflare R2 (sub-projeto à parte, mudança de código no S3 client). `S3_UPLOADS_ACCESS_KEY`/`SECRET_KEY` estão em texto puro na task def (não em Secrets Manager).
- **Wallet no Secrets Manager:** `PRICE_MARKET_AUTHORITY_WALLET` lê a chave de `crypto-wallets/{wallet}` em runtime. Enquanto `AWS_ACCESS_KEY` existir (necessário pro S3), esse read segue funcionando. Pra eliminar: setar `AUTHORITY_WALLET_PRIVATE_KEY` (fallback local no `env.ts`). Deferido pra Fase 7.

---

## FASE 3 — Ensaio da migração do DB (sem downtime)

### Task 3.1 + 3.2 — Dump Aurora → Restore Railway ✅ (via `api/scripts/railway-db-dryrun.sh`)
Script lê Aurora (Secrets Manager `oraculo/prod/database-url`) + Railway PG público (`DATABASE_PUBLIC_URL` da CLI), faz pg_dump -Fc → pg_restore --clean, e compara count(*) por tabela.
- [x] Dump 7.6M, restore limpo (sem erros relevantes).
- [x] **49/50 tabelas idênticas.** Única diferença: `chat_messages` (aurora=3309, railway=3233) — **skew temporal**: pg_dump é snapshot consistente; a verificação conta a Aurora depois, e a tabela recebeu ~76 inserts no intervalo. Não é perda de dado. No cutover real (Fase 6) o ECS→0 congela escritas antes do dump → skew zero.
- **Gotchas resolvidos no script:** senha da Aurora tem chars especiais (`[ ( > % +`) → percent-encode; `sslmode=no-verify` (node-pg/Prisma) → reescrito p/ `require` (libpq não aceita no-verify).

### Task 3.3 — Smoke da app no Railway 🟢 oraculo-api OK / worker pendente
**Deploy:** `railway up` (upload local) TRAVA nessa rede (mesmo gargalo do `init`) → usar **deploy pelo GitHub** (dashboard Deploy / push). Domínio do api: `https://oraculo-api-production-4f3a.up.railway.app`.
- [x] **oraculo-api:** build OK (Dockerfile), `/health` → `{"status":"ok"}`, DB ✓, Redis ✓, **RabbitMQ ✓** (4 consumers "ready with DLQ").
- [x] **oraculo-worker:** Custom Start Command = **`bun run worker.ts`** (deploy `29483c85` SUCCESS). Logs mostram crons MM replenish/phantom-check (worker), sem HTTP server (≠ API). Sem domínio público, sem healthcheck.
  - **Gotcha:** `worker.ts` sozinho FALHA — o Railway **não preserva o `ENTRYPOINT ["bun","run"]`** do Dockerfile no Custom Start Command (diferente da ECS, que mantém). Precisa do comando completo `bun run worker.ts`.
- [ ] (Opcional) drift-audit no **Railway Console** do oraculo-api (env interno já presente): `bun scripts/trading-v2-drift-audit.ts` → 0 negativos. (Integridade já validada na Task 3.1/3.2.)

**Gotchas de build/deploy resolvidos:**
- **Root Directory `api` é obrigatório** senão o Railpack roda na raiz do monorepo e falha. Com root dir `api` + `Dockerfile` presente, o Railway usa o Dockerfile automaticamente (não precisa trocar o Builder de Railpack p/ Dockerfile).
- **RabbitMQ first-boot:** `RABBITMQ_DEFAULT_USER/PASS` só criam o user no 1º boot com data dir vazio. O serviço bootou antes das envs → user `oraculo` não existia → `PLAIN login refused`. Fix: anexar o volume `/var/lib/rabbitmq` (boot com volume vazio + envs presentes recria o user). RabbitMQ escuta em `[::]:5672` (IPv6, ok pro private networking).

---

## FASE 4 — Custom domain + TLS (sem cutover)

### Task 4.1 — Adicionar api.oraculo.fun ao Railway
- [ ] No serviço `oraculo-api`: Settings → Domains → Custom Domain → `api.oraculo.fun`. Railway dá um alvo CNAME (ex.: `xxx.up.railway.app`) e provisiona TLS.
- [ ] **NÃO mudar o DNS ainda** — só anotar o alvo CNAME do Railway.
- [ ] **Verificação:** Railway mostra o domínio como "pending DNS" (esperado, DNS ainda → AWS).

---

## FASE 5 — CI/CD (Railway auto-deploy)

### Task 5.1 — Ligar auto-deploy do Railway + neutralizar deploy ECS
**Files:** Modify `.github/workflows/deploy-api.yml`
- [ ] No Railway: conectar o repo GitHub aos serviços api+worker com auto-deploy no branch `main` (dashboard → service → Settings → trigger on push to main, root `api/`).
- [ ] No `deploy-api.yml`: **remover o job `deploy`** (build+push ECR + `aws ecs update-service`) — pra não deployar em infra que será desmontada. Manter os jobs `unit-tests`/`e2e-tests`/`playwright` como gate.
  - Editar: deletar o bloco `deploy:` inteiro (linhas ~229+).
- [ ] Commit: `git commit -m "ci: remove deploy ECS (migrado pro Railway auto-deploy)"`.
- [ ] **Verificação:** push de teste → Railway builda/deploya; GitHub Actions roda só os testes (sem deploy ECS).

---

## FASE 6 — CUTOVER ✅ FEITO (2026-06-02, fora de ordem mas concluído)

> **Como aconteceu na prática:** o operador adicionou o custom domain `api.oraculo.fun` no Railway E **virou o DNS no Cloudflare antes** do freeze/sync final. Recuperação: (1) freeze AWS (ECS→0), (2) re-rodar `railway-db-dryrun.sh` com a Aurora já estática → Railway = cópia final da Aurora. Sem usuários ativos, perda real ≈ 0.
> - ✅ `api.oraculo.fun` → Railway (Cloudflare proxied, `x-railway-edge`, TLS HTTP/2, `/health` ok).
> - ✅ AWS ECS api+worker `desired=0/running=0`. Aurora VIVA (não apagada).
> - ✅ Dados: 49/50 tabelas idênticas à Aurora. `chat_messages` teve race de escrita ao vivo no restore (chatbots) → PK não criada; reparada com `api/scripts/railway-fix-chat-pk.sh` (dedup + ADD PRIMARY KEY + setval). 
> - **Lição:** num restore com o app ao vivo, tabelas que o app escreve (chat_messages) dão conflito de índice. Ideal seria pausar o oraculo-api durante o restore; como é tabela não-crítica, reparo in-place resolveu.

### Task 6.1 — Congelar AWS ✅
- [ ] `aws ecs update-service --profile oraculo --region us-east-1 --cluster oraculo-prod-cluster --service oraculo-prod-api --desired-count 0`
- [ ] idem `oraculo-prod-worker`.
- [ ] **Verificação:** `runningCount=0` nos dois (agente pode checar read-only). **Downtime começa.**

### Task 6.2 — Sync final do DB
- [ ] `pg_dump "$AURORA_URL" -Fc --no-owner --no-acl -f oraculo-final.dump`
- [ ] `pg_restore -d "$RAILWAY_PG_PUBLIC_URL" --no-owner --no-acl --clean --if-exists oraculo-final.dump`
- [ ] **Verificação:** re-rodar a contagem de linhas (Task 3.2) → todas `OK`.

### Task 6.3 — Subir app no Railway + verificar
- [ ] Garantir `oraculo-api`/`oraculo-worker` rodando no Railway (redeploy se necessário).
- [ ] **Verificação:** `curl https://<oraculo-api>.up.railway.app/health` → 200.

### Task 6.4 — Virar o DNS (Cloudflare)
- [ ] Cloudflare → zona oraculo.fun → registro `api` (CNAME): trocar o target de (ALB AWS) para o **alvo CNAME do Railway** (Task 4.1). Proxy: laranja (proxied). SSL/TLS mode: **Full**.
- [ ] **Verificação:**
  ```bash
  curl -s https://api.oraculo.fun/health   # → {"status":"ok"} servido pelo Railway
  ```
  (propaga rápido com Cloudflare proxied + TTL baixo).

### Task 6.5 — Validação pós-cutover
- [ ] `bun scripts/trading-v2-drift-audit.ts` (DATABASE_URL=Railway) → limpo.
- [ ] Fluxo real no front (oraculo.fun): saldo aparece, posições aparecem, depósito/trade/cancel funcionam.
- [ ] Disparar/aguardar 1 **webhook 3xChange** de teste → confirmar processamento (chega em api.oraculo.fun → Railway).
- [ ] **Downtime encerra.** Se algo crítico falhar → **ROLLBACK** (abaixo).

### ROLLBACK (se Task 6.5 falhar)
- [ ] Cloudflare: voltar o CNAME `api` → ALB AWS.
- [ ] `aws ecs update-service ... --service oraculo-prod-api --desired-count 1` (e worker).
- [ ] Aurora segue intocada → app volta a usá-la. (Escritas no Railway pós-cutover são perdidas — por isso validar rápido.)

---

## FASE 7 — Estabilidade + decomissionar AWS (realizar a economia)

### Task 7.1 — Janela de estabilidade
- [ ] Manter AWS de pé (já com ECS em 0) + Railway servindo por **~3 dias** (ou o prazo que o operador definir), observando logs/erros no Railway.
- [ ] **Verificação:** sem incidentes; métricas de negócio normais; webhooks ok.

### Task 7.2 — Snapshot final da Aurora (arquival)
- [ ] `aws rds create-db-cluster-snapshot --profile oraculo --region us-east-1 --db-cluster-identifier tf-20260227222129140600000005 --db-cluster-snapshot-identifier final-pre-aws-teardown-$(date +%Y%m%d)` (agente pode rodar — é criação de snapshot, não destrutivo).
- [ ] Aguardar `available`. Guardar por algumas semanas.

### Task 7.3 — `pulumi destroy` do stack de prod
**Files:** `infra/` (Pulumi)
- [ ] Confirmar **todos os secrets migrados** (Fase 2) e Railway estável.
- [ ] No `infra/`: `pulumi stack select prod && pulumi destroy` (revisar o preview com cuidado — derruba ECS, Aurora, ElastiCache, MQ, ALB, NAT, VPC, ECR, CloudWatch).
- [ ] **Verificação:** `aws ce get-cost-and-usage` nos dias seguintes → custo despencando pra ~$0.
- [ ] Route53: se DNS é 100% Cloudflare, remover a zona (ou deixar; ~$0.50).

---

## Verificação final
- [ ] `api.oraculo.fun/health` 200 servido pelo Railway.
- [ ] drift-audit limpo no Railway PG.
- [ ] Fluxo de trading + depósito/saque + webhooks funcionando em prod.
- [ ] Custo AWS → ~$0; Railway ~$40-80/mês.
- [ ] Snapshot final da Aurora guardado.
