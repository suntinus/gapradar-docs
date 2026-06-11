# /explain-flow

**Descrição:** Explica o fluxo completo de uma operação na API GapRadar — da request ao resultado.

**Uso:** `/explain-flow <operação>`

**Exemplos:**
- `/explain-flow análise de mercado` → fluxo completo desde criar até receber resultado
- `/explain-flow autenticação` → registro/login → uso do token
- `/explain-flow upgrade de plano` → verificar limite → checkout → retorno
- `/explain-flow streaming sse` → criar análise → conectar SSE → eventos

**Fluxos disponíveis:**

### Fluxo: Análise de mercado completa

```
POST /api/auth/login → token
  ↓
POST /api/analyze → analysisId (status: queued)
  ↓
[Polling] GET /api/analyze/:id (a cada 3s)
    OU
[Streaming] GET /api/analyze/stream (SSE: status → result → done)
  ↓
status: completed → result.gapScore + result.demandScore
  ↓
GET /api/reports/:analysisId → relatório formatado
  ↓
GET /api/reports/:analysisId/export → PDF URL (Pro/Avulso)
```

### Fluxo: Autenticação

```
POST /api/auth/register → token (nova conta)
    OU
POST /api/auth/login → token (conta existente)
  ↓
Usar token em: Authorization: Bearer <token>
  ↓
GET /api/auth/me → dados do usuário atual
```

### Fluxo: Upgrade de plano

```
GET /api/users/me/usage → analysesUsed >= analysesLimit → 402
  ↓
GET /api/billing/plans → escolher plano
  ↓
POST /api/billing/checkout → checkoutUrl
  ↓
Redirecionar para Stripe → pagamento
  ↓
successUrl?upgraded=true → GET /api/auth/me → plano atualizado
```

**Docs relacionadas:**
- Fluxo completo: `docs/guides/first-analysis.md`
- SSE: `docs/guides/streaming-integration.md`
- Billing: `docs/guides/billing-integration.md`
- Autenticação: `docs/authentication.md`
