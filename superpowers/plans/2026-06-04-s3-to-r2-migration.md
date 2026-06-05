# Migração S3 → Cloudflare R2 — Plano de Implementação

> **For agentic workers:** runbook ops+código. Passos de infra (R2, AWS) e dados rodados pelo operador; mudanças de código pelo agente. Steps em checkbox.

**Goal:** Sair 100% do AWS S3 movendo uploads (KYC, fotos de perfil, imagens de mercado, metadata JSON dos tokens) pro Cloudflare R2, pra permitir o `pulumi destroy` completo (a conta AWS está em débito e pode ser suspensa).

**Arquitetura:** R2 é S3-compatível → reusar o `@aws-sdk/client-s3` com `endpoint` do R2 + `region:"auto"`. Acesso público (imagens/metadata) via **custom domain** `cdn.oraculo.fun` ligado ao bucket R2 (R2 não suporta ACL por objeto). Dados copiados S3→R2; URLs já gravadas no banco e dentro dos JSONs reescritas pro novo domínio.

**Tech:** Bun, `@aws-sdk/client-s3`, Cloudflare R2, Railway, Postgres.

**Contexto:** parte final da migração AWS→Railway (`2026-06-02-railway-migration.md`). Bloqueia o teardown total da AWS.

---

## FASE 0 — Pré-requisitos (operador)

### Task 0.1 — Backup de segurança do S3 ✅ (script `api/scripts/s3-backup.sh`)
- [x] `bash api/scripts/s3-backup.sh` → `./s3-backup-oraculo-prod-uploads/` (usa creds S3_UPLOADS do Railway; profile admin não lista o bucket).
- Rede de segurança caso a AWS suspenda no meio da migração.

### Task 0.2 — Provisionar R2 (Cloudflare dashboard)
- [ ] Criar bucket R2 `oraculo-uploads`.
- [ ] Criar **R2 API Token** S3-compatível (Access Key ID + Secret, read/write no bucket). Anotar.
- [ ] Anotar o **endpoint da conta**: `https://<ACCOUNT_ID>.r2.cloudflarestorage.com`.
- [ ] **Custom domain público:** ligar `cdn.oraculo.fun` ao bucket (R2 → bucket → Settings → Custom Domains). Cloudflare cria o DNS + TLS. Esse vira o `R2_PUBLIC_URL` (`https://cdn.oraculo.fun`).

---

## FASE 1 — Copiar dados S3 → R2 (operador)

### Task 1.1 — Sync do backup local pro R2
Usar aws cli com endpoint do R2 (mais simples que instalar rclone). Preserva as mesmas keys.
- [ ] ```bash
  AWS_ACCESS_KEY_ID=<R2_AK> AWS_SECRET_ACCESS_KEY=<R2_SK> \
    aws s3 sync ./s3-backup-oraculo-prod-uploads s3://oraculo-uploads \
    --endpoint-url https://<ACCOUNT_ID>.r2.cloudflarestorage.com --region auto --no-progress
  ```
- [ ] **Verificação:** `aws s3 ls s3://oraculo-uploads --recursive --endpoint-url <r2> --region auto | wc -l` == contagem do backup local (`find ./s3-backup-... -type f | wc -l`).

---

## FASE 2 — Código: apontar o cliente pro R2

### Task 2.1 — Env schema
**Files:** Modify `api/src/shared/utils/env.ts`
- [ ] Adicionar ao `envSchema` (todos opcionais, retrocompat com S3):
```ts
  // Cloudflare R2 (S3-compatible) — substitui o S3 da AWS
  R2_ENDPOINT: z.string().optional(),       // https://<acct>.r2.cloudflarestorage.com
  R2_ACCESS_KEY: z.string().optional(),
  R2_SECRET_KEY: z.string().optional(),
  R2_BUCKET: z.string().optional(),
  R2_PUBLIC_URL: z.string().optional(),     // https://cdn.oraculo.fun
```

### Task 2.2 — S3Service usa R2 quando configurado
**Files:** Modify `api/src/shared/services/s3.service.ts`
- [ ] Construtor: priorizar R2; guardar base pública:
```ts
  private publicBaseUrl: string;
  constructor() {
    const usingR2 = !!env.R2_ENDPOINT;
    const region = usingR2 ? "auto" : (env.S3_UPLOADS_REGION ?? env.AWS_REGION);
    const accessKeyId = env.R2_ACCESS_KEY ?? env.S3_UPLOADS_ACCESS_KEY ?? env.AWS_ACCESS_KEY;
    const secretAccessKey = env.R2_SECRET_KEY ?? env.S3_UPLOADS_SECRET_KEY ?? env.AWS_SECRET_ACCESS_KEY;
    this.client = new S3Client({
      region,
      ...(usingR2 ? { endpoint: env.R2_ENDPOINT } : {}),
      credentials: { accessKeyId, secretAccessKey },
    });
    this.bucketName = env.R2_BUCKET ?? env.S3_UPLOADS_BUCKET ?? env.AWS_S3_BUCKET_KYC;
    this.publicBaseUrl = env.R2_PUBLIC_URL
      ?? `https://${this.bucketName}.s3.${env.AWS_REGION}.amazonaws.com`;
  }
```
- [ ] `uploadFile`: remover ACL no R2 e usar a base pública na URL:
```ts
      const command = new PutObjectCommand({
        Bucket: this.bucketName, Key: key, Body: body, ContentType: contentType,
        Metadata: metadata, ServerSideEncryption: "AES256",
        ...(publicRead && !env.R2_ENDPOINT ? { ACL: "public-read" } : {}),
      });
      const result = await this.client.send(command);
      return { url: `${this.publicBaseUrl}/${key}`, key, etag: result.ETag || "" };
```
> `SSE: AES256` é aceito pelo R2 (ignora ou aplica). Presigned URLs (KYC) funcionam igual.

### Task 2.3 — metadata.service.ts usa o cliente compartilhado
**Files:** Modify `api/src/modules/markets/services/metadata.service.ts`
- [ ] `getMetadata` cria um `S3Client` cru (linha ~173) — trocar pra usar o `s3Service` (ou aplicar o mesmo `endpoint`/`region:"auto"` do R2). Garantir que lê do R2 quando configurado.

---

## FASE 3 — Env no Railway (operador)
### Task 3.1 — Setar as vars R2 nos 2 serviços
- [ ] `R2_ENDPOINT`, `R2_ACCESS_KEY`, `R2_SECRET_KEY`, `R2_BUCKET=oraculo-uploads`, `R2_PUBLIC_URL=https://cdn.oraculo.fun` em oraculo-api e oraculo-worker (via `railway variable set --skip-deploys` ou dashboard).

---

## FASE 4 — Reescrever URLs no banco

### Task 4.1 — Colunas com URL do S3 → custom domain R2
**Files:** script `api/scripts/r2-rewrite-db-urls.sh` (a criar)
- [ ] Substituir o host em todas as colunas que guardam URL de upload. Colunas candidatas (confirmar no schema): `users.profilePicture`, `users.userAvatar`, markets `image`/`tokenImage`/`logoURI`, e quaisquer `*Avatar`/`*image*`.
```sql
-- padrão por coluna (DATABASE_URL = Railway):
UPDATE "User" SET "profilePicture" =
  replace("profilePicture",'oraculo-prod-uploads.s3.us-east-1.amazonaws.com','cdn.oraculo.fun')
  WHERE "profilePicture" LIKE '%oraculo-prod-uploads.s3%';
-- repetir p/ cada coluna identificada.
```
- [ ] **Verificação:** `SELECT count(*) ... WHERE col LIKE '%oraculo-prod-uploads.s3%'` == 0 em todas.

### Task 4.2 — Reescrever referências DENTRO dos JSONs de metadata (no R2)
Os JSONs de metadata dos tokens podem ter URLs do S3 no conteúdo (ex.: campo `image`). Como `tokenUri` hoje fica só no DB (comentários em `market-creation.service.ts:598`), o app lê o JSON via URL do DB — mas o conteúdo interno ainda aponta pro S3.
- [ ] Script: pra cada objeto sob `prediction-markets/` e `*/metadata` no R2, baixar → `sed` o host S3→`cdn.oraculo.fun` → re-upload. (Ou aceitar quebra se forem mercados de teste descartáveis — decisão do operador.)

---

## FASE 5 — Deploy + verificação
### Task 5.1 — Deploy e smoke
- [ ] Merge/deploy do código (Railway auto-deploy). 
- [ ] **Verificações ao vivo:** upload de KYC (presigned PUT → R2 ok), GET presigned (view) ok, imagem pública carrega via `https://cdn.oraculo.fun/...`, criar mercado de teste (metadata sobe no R2), front mostra fotos/imagens (URLs reescritas).

---

## FASE 6 — Teardown total da AWS
### Task 6.1 — `pulumi destroy` COMPLETO (sem preservar S3)
> Agora que nada depende do S3, o destroy é total — não precisa orfanizar o S3.
- [ ] Confirmar: app no Railway usando R2 (Fase 5 verde), backup local + R2 com os dados.
- [ ] `cd infra && pulumi stack select prod && pulumi destroy` (revisar preview; `auroraDeleteProtection:false` e ECR `forceDelete` → não trava). Inclui S3, Aurora, ECS, Redis, MQ, ALB, NAT, VPC, ECR, secrets.
- [ ] (Opcional) deixar a conta AWS ser encerrada por falta de pagamento — o destroy evita cobranças residuais até lá.
- [ ] **Verificação:** custo AWS → ~$0; `cdn.oraculo.fun/<key>` serve imagens do R2; app saudável.

---

## Riscos
- **Suspensão da AWS no meio** → mitigado pelo backup local (Fase 0.1) feito primeiro.
- **R2 sem ACL por objeto** → público via custom domain (Task 0.2/2.2).
- **URLs internas nos JSONs de metadata** → Task 4.2 (ou aceitar quebra se test data).
- **`tokenUri` legado on-chain** (se existir em mercado antigo) → quebraria com a AWS caindo de qualquer forma; sem mitigação possível com a conta suspensa.
- **forcePathStyle:** se o R2 reclamar de bucket no host, adicionar `forcePathStyle:true` no S3Client.
