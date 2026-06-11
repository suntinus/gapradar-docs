# Guia: Integração de Billing

Como integrar o sistema de planos e pagamentos do GapRadar na sua aplicação.

---

## Visão geral

O GapRadar usa **Stripe** para processar pagamentos. O fluxo padrão é:

1. Verificar se o usuário pode criar análises (`GET /api/users/me/usage`)
2. Se atingiu o limite → redirecionar para checkout (`POST /api/billing/checkout`)
3. Stripe processa o pagamento e redireciona de volta
4. Usuário volta com plano atualizado

---

## Planos e limites

```javascript
const PLANS = {
  free:     { analysesLimit: 3,    price: 0,  currency: 'BRL', interval: 'month' },
  starter:  { analysesLimit: null, price: 29, currency: 'BRL', interval: 'month' },
  pro:      { analysesLimit: null, price: 99, currency: 'BRL', interval: 'month' },
  avulso:   { analysesLimit: 1,    price: 49, currency: 'BRL', interval: 'one_time' },
};
// analysesLimit === null significa ilimitado
```

---

## Passo 1 — Verificar uso antes de criar análise

```javascript
async function checkCanAnalyze(token) {
  const res = await fetch('https://gapradar.com.br/api/users/me/usage', {
    headers: { Authorization: `Bearer ${token}` }
  });
  const { data } = await res.json();

  // analysesLimit null = ilimitado (planos Starter/Pro)
  if (data.analysesLimit === null) return { canAnalyze: true };

  const remaining = data.analysesLimit - data.analysesUsed;
  return {
    canAnalyze: remaining > 0,
    remaining,
    resetDate: data.resetDate,
    needsUpgrade: remaining <= 0,
  };
}

// Uso
const { canAnalyze, needsUpgrade } = await checkCanAnalyze(token);

if (needsUpgrade) {
  showUpgradeModal();
} else {
  createAnalysis();
}
```

---

## Passo 2 — Listar planos

```javascript
async function loadPlans() {
  const res = await fetch('https://gapradar.com.br/api/billing/plans');
  // Sem auth necessário
  const { data } = await res.json();
  return data;
}

// Exibir na UI
const plans = await loadPlans();
plans.forEach(plan => {
  console.log(`${plan.name}: R$${plan.price}/${plan.interval === 'month' ? 'mês' : 'uso'}`);
  console.log(`  Análises: ${plan.analysesLimit ?? 'Ilimitado'}`);
  console.log(`  Features: ${plan.features.join(', ')}`);
});
```

---

## Passo 3 — Iniciar checkout

```javascript
async function startCheckout(planId, token) {
  const res = await fetch('https://gapradar.com.br/api/billing/checkout', {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${token}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      planId,
      successUrl: `${window.location.origin}/dashboard?upgraded=true&plan=${planId}`,
      cancelUrl: `${window.location.origin}/pricing`,
    }),
  });

  const { data } = await res.json();

  // Redirecionar para Stripe Checkout
  window.location.href = data.checkoutUrl;
}

// Uso
document.getElementById('btn-upgrade-pro').addEventListener('click', () => {
  startCheckout('pro', token);
});
```

---

## Passo 4 — Detectar retorno do checkout

Quando o usuário voltar da página de sucesso (a `successUrl` que você configurou), verifique o parâmetro `upgraded`:

```javascript
// Na página de dashboard
const params = new URLSearchParams(window.location.search);

if (params.get('upgraded') === 'true') {
  // Recarregar dados do usuário para refletir novo plano
  const userRes = await fetch('https://gapradar.com.br/api/auth/me', {
    headers: { Authorization: `Bearer ${token}` }
  });
  const { data: user } = await userRes.json();

  showSuccessMessage(`Bem-vindo ao plano ${user.plan}!`);

  // Limpar query params da URL
  window.history.replaceState({}, '', '/dashboard');
}
```

---

## Passo 5 — Verificar assinatura atual

```javascript
async function getSubscription(token) {
  const res = await fetch('https://gapradar.com.br/api/billing/subscription', {
    headers: { Authorization: `Bearer ${token}` }
  });
  const { data } = await res.json();
  return data;
}

const sub = await getSubscription(token);

// sub.status: 'active' | 'trialing' | 'past_due' | 'canceled'
if (sub.status === 'past_due') {
  showPaymentWarning('Pagamento pendente. Atualize seu cartão para continuar.');
}

if (sub.cancelAtPeriodEnd) {
  const endDate = new Date(sub.currentPeriodEnd).toLocaleDateString('pt-BR');
  showInfo(`Sua assinatura será cancelada em ${endDate}.`);
}
```

---

## Passo 6 — Portal de gerenciamento (cancelar, trocar cartão)

```javascript
async function openBillingPortal(token) {
  const res = await fetch('https://gapradar.com.br/api/billing/portal', {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${token}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({
      returnUrl: `${window.location.origin}/dashboard`,
    }),
  });

  const { data } = await res.json();
  window.location.href = data.portalUrl;
}
```

---

## Componente React — Pricing Page

```tsx
import { useState, useEffect } from 'react';

function PricingPage({ token, currentPlan }) {
  const [plans, setPlans] = useState([]);
  const [loading, setLoading] = useState(false);

  useEffect(() => {
    fetch('https://gapradar.com.br/api/billing/plans')
      .then(r => r.json())
      .then(({ data }) => setPlans(data));
  }, []);

  const handleUpgrade = async (planId) => {
    if (!token) {
      window.location.href = '/login';
      return;
    }

    setLoading(true);
    try {
      const res = await fetch('https://gapradar.com.br/api/billing/checkout', {
        method: 'POST',
        headers: {
          Authorization: `Bearer ${token}`,
          'Content-Type': 'application/json',
        },
        body: JSON.stringify({
          planId,
          successUrl: `${window.location.origin}/dashboard?upgraded=true`,
          cancelUrl: `${window.location.origin}/pricing`,
        }),
      });
      const { data } = await res.json();
      window.location.href = data.checkoutUrl;
    } finally {
      setLoading(false);
    }
  };

  return (
    <div>
      {plans.map(plan => (
        <div key={plan.id}>
          <h3>{plan.name}</h3>
          <p>
            {plan.price === 0
              ? 'Grátis'
              : `R$${plan.price}/${plan.interval === 'month' ? 'mês' : 'uso'}`}
          </p>
          <ul>
            {plan.features.map(f => <li key={f}>{f}</li>)}
          </ul>
          {plan.id !== 'free' && plan.id !== currentPlan && (
            <button
              onClick={() => handleUpgrade(plan.id)}
              disabled={loading}
            >
              {plan.id === 'avulso' ? 'Comprar análise' : 'Assinar'}
            </button>
          )}
          {plan.id === currentPlan && <span>Plano atual</span>}
        </div>
      ))}
    </div>
  );
}
```

---

## Tratamento do erro 402

Quando o limite é atingido, `POST /api/analyze` retorna `402`:

```javascript
const res = await fetch('https://gapradar.com.br/api/analyze', { ... });

if (res.status === 402) {
  const canBuyAvulso = confirm(
    'Você atingiu o limite do plano Free. ' +
    'Deseja comprar uma análise avulsa por R$49?'
  );

  if (canBuyAvulso) {
    await startCheckout('avulso', token);
  }
  return;
}
```

---

## Resumo do fluxo

```
Usuário clica "Analisar"
  │
  ├─ GET /api/users/me/usage
  │   ├─ limit === null ou used < limit → POST /api/analyze (continua normal)
  │   └─ used >= limit → mostrar modal de upgrade
  │
Modal de upgrade → usuário escolhe plano
  │
  └─ POST /api/billing/checkout → redireciona para Stripe
       │
       └─ Stripe processa → redireciona para successUrl
            │
            └─ GET /api/auth/me → plano atualizado → libera análise
```
