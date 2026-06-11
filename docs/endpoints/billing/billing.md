# Billing — Planos e Pagamentos

Endpoints para gerenciar planos, assinaturas e pagamentos via Stripe.

---

## Planos disponíveis

```
GET https://gapradar.com.br/api/billing/plans
```

Endpoint público — não requer autenticação.

### Resposta

```json
{
  "data": [
    {
      "id": "free",
      "name": "Free",
      "price": 0,
      "currency": "BRL",
      "interval": "month",
      "analysesLimit": 3,
      "features": [
        "3 análises por mês",
        "Relatório básico",
        "Acesso ao histórico"
      ]
    },
    {
      "id": "starter",
      "name": "Starter",
      "price": 29,
      "currency": "BRL",
      "interval": "month",
      "analysesLimit": null,
      "features": [
        "Análises ilimitadas",
        "Relatório básico",
        "Acesso ao histórico"
      ]
    },
    {
      "id": "pro",
      "name": "Pro",
      "price": 99,
      "currency": "BRL",
      "interval": "month",
      "analysesLimit": null,
      "features": [
        "Análises ilimitadas",
        "Relatório completo",
        "Exportação PDF",
        "Acesso à API"
      ]
    },
    {
      "id": "avulso",
      "name": "Avulso",
      "price": 49,
      "currency": "BRL",
      "interval": "one_time",
      "analysesLimit": 1,
      "features": [
        "1 análise",
        "Relatório completo",
        "Exportação PDF",
        "Acesso à API"
      ]
    }
  ]
}
```

---

## Criar sessão de checkout

```
POST https://gapradar.com.br/api/billing/checkout
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json
```

### Body

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `planId` | `"starter"` \| `"pro"` \| `"avulso"` | ✅ | Plano desejado |
| `successUrl` | string | — | URL após pagamento bem-sucedido |
| `cancelUrl` | string | — | URL se o usuário cancelar |

```json
{
  "planId": "pro",
  "successUrl": "https://gapradar.com.br/dashboard?upgraded=true",
  "cancelUrl": "https://gapradar.com.br/pricing"
}
```

### Resposta `200`

```json
{
  "data": {
    "checkoutUrl": "https://checkout.stripe.com/pay/cs_example_123"
  }
}
```

Redirecione o usuário para `checkoutUrl` para completar o pagamento.

---

## Ver assinatura atual

```
GET https://gapradar.com.br/api/billing/subscription
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### Resposta

```json
{
  "data": {
    "plan": "pro",
    "status": "active",
    "currentPeriodEnd": "2026-07-11T10:00:00.000Z",
    "cancelAtPeriodEnd": false
  }
}
```

**Status possíveis:**
- `active` — assinatura ativa
- `trialing` — período de trial
- `past_due` — pagamento pendente
- `canceled` — cancelada

---

## Acessar portal Stripe (gerenciar/cancelar)

```
POST https://gapradar.com.br/api/billing/portal
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json
```

### Body (opcional)

```json
{
  "returnUrl": "https://gapradar.com.br/dashboard"
}
```

### Resposta `200`

```json
{
  "data": {
    "portalUrl": "https://billing.stripe.com/session/example_session"
  }
}
```

Redirecione o usuário para `portalUrl` para gerenciar cartão, ver histórico de pagamentos ou cancelar assinatura.

---

## Fluxo de upgrade — exemplo JavaScript

```javascript
async function upgradeToProPlan(token) {
  const res = await fetch('https://gapradar.com.br/api/billing/checkout', {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${token}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      planId: 'pro',
      successUrl: `${window.location.origin}/dashboard?upgraded=true`,
      cancelUrl: `${window.location.origin}/pricing`,
    }),
  });

  const { data } = await res.json();

  // Redirecionar para Stripe Checkout
  window.location.href = data.checkoutUrl;
}
```

## Verificar se usuário pode criar mais análises

```javascript
async function canCreateAnalysis(token) {
  const res = await fetch('https://gapradar.com.br/api/users/me/usage', {
    headers: { Authorization: `Bearer ${token}` }
  });
  const { data } = await res.json();

  if (data.analysesLimit === null) return true; // ilimitado
  return data.analysesUsed < data.analysesLimit;
}
```

---

## Comparativo de planos

| Feature | Free | Starter | Pro | Avulso |
|---------|------|---------|-----|--------|
| Análises/mês | 3 | Ilimitado | Ilimitado | 1 (por compra) |
| Relatório completo | — | — | ✅ | ✅ |
| Export PDF | — | — | ✅ | ✅ |
| Acesso API | — | — | ✅ | ✅ |
| Preço | Grátis | R$29/mês | R$99/mês | R$49/análise |
