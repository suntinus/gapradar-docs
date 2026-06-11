# Guia: Primeira Análise — Do Zero ao Relatório

Tutorial completo do fluxo de validação de mercado com o GapRadar: criar conta, analisar uma ideia e ler o relatório final.

---

## O que você vai fazer

1. Criar uma conta
2. Fazer login e guardar o token
3. Submeter uma ideia para análise
4. Acompanhar o progresso
5. Ler o relatório completo

Tempo estimado: ~5 minutos (a análise leva ~1-3 minutos para completar).

---

## Passo 1 — Criar conta

```bash
curl -s -X POST https://gapradar.com.br/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "password": "minhasenha123",
    "name": "Maria Silva"
  }' | jq '.data.token'
```

> Substitua `user@example.com` e `minhasenha123` pelos seus dados.

Você verá o token JWT impresso. Guarde-o:

```bash
TOKEN="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

Se já tem conta, pule para o Passo 2 (login).

---

## Passo 2 — Login (sessões futuras)

```bash
TOKEN=$(curl -s -X POST https://gapradar.com.br/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"user@example.com","password":"minhasenha123"}' \
  | jq -r '.data.token')

echo "Token: $TOKEN"
```

---

## Passo 3 — Verificar seu plano e limite

```bash
curl -s https://gapradar.com.br/api/users/me/usage \
  -H "Authorization: Bearer $TOKEN" | jq '.data'
```

Resposta esperada (plano Free):
```json
{
  "analysesUsed": 0,
  "analysesLimit": 3,
  "resetDate": "2026-07-01T00:00:00.000Z"
}
```

---

## Passo 4 — Submeter ideia para análise

```bash
ANALYSIS_ID=$(curl -s -X POST https://gapradar.com.br/api/analyze \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "idea": "Plataforma de agendamento para pet shops com integração WhatsApp",
    "category": "software",
    "targetMarket": "Brasil"
  }' | jq -r '.data.analysisId')

echo "Analysis ID: $ANALYSIS_ID"
```

**Categorias disponíveis:**
- `software` — apps, SaaS, plataformas
- `physical` — produtos físicos
- `service` — prestação de serviços

---

## Passo 5 — Acompanhar o progresso

### Opção A: Polling (simples)

```bash
while true; do
  STATUS=$(curl -s "https://gapradar.com.br/api/analyze/$ANALYSIS_ID" \
    -H "Authorization: Bearer $TOKEN" | jq -r '.data.status')

  echo "Status: $STATUS"

  if [ "$STATUS" = "completed" ] || [ "$STATUS" = "failed" ]; then
    break
  fi

  sleep 3
done
```

### Opção B: SSE streaming (tempo real)

```bash
curl -N "https://gapradar.com.br/api/analyze/stream?analysisId=$ANALYSIS_ID" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Accept: text/event-stream"
```

Você verá eventos chegando:
```
data: {"type":"status","data":"Searching ProductHunt..."}
data: {"type":"status","data":"Searching HackerNews..."}
data: {"type":"status","data":"Analyzing with AI..."}
data: {"type":"result","data":{"gapScore":8,"demandScore":7,...}}
data: {"type":"done","data":""}
```

---

## Passo 6 — Ver resultado da análise

```bash
curl -s "https://gapradar.com.br/api/analyze/$ANALYSIS_ID" \
  -H "Authorization: Bearer $TOKEN" | jq '.data.result'
```

Resposta:
```json
{
  "gapScore": 8,
  "demandScore": 7,
  "competitors": [
    {
      "name": "PetScheduler",
      "url": "https://petscheduler.example.com",
      "description": "Agendamento para clínicas veterinárias",
      "source": "producthunt",
      "sentiment": "neutral"
    }
  ],
  "marketSignals": [
    {
      "source": "hackernews",
      "signal": "Usuários reclamam de agendamento manual via WhatsApp",
      "sentiment": "positive"
    }
  ],
  "summary": "O mercado de pet shops apresenta oportunidade real...",
  "opportunities": ["Crescimento 15% ao ano", "Integração WhatsApp ainda inexplorada"],
  "risks": ["Grandes plataformas podem entrar no segmento"]
}
```

**Interpretando os scores:**

| Score | Significado |
|-------|-------------|
| `gapScore` 8-10 | Oportunidade clara no mercado |
| `gapScore` 5-7 | Mercado existe, mas competição moderada |
| `gapScore` 0-4 | Mercado muito competitivo ou saturado |
| `demandScore` 8-10 | Alta demanda observada nas fontes |
| `demandScore` 5-7 | Demanda moderada |
| `demandScore` 0-4 | Demanda baixa ou pouco documentada |

---

## Passo 7 — Ler relatório formatado

```bash
curl -s "https://gapradar.com.br/api/reports/$ANALYSIS_ID" \
  -H "Authorization: Bearer $TOKEN" | jq '.data'
```

O relatório inclui:
- `executiveSummary` — resumo executivo narrativo
- `sections` — seções organizadas (Mercado, Concorrentes, Demanda, Oportunidades, Riscos, Recomendação)

---

## Passo 8 — Exportar PDF (planos Pro/Avulso)

```bash
curl -s "https://gapradar.com.br/api/reports/$ANALYSIS_ID/export" \
  -H "Authorization: Bearer $TOKEN" | jq '.data.url'
```

Retorna a URL do PDF para download.

---

## Fluxo completo em JavaScript

```javascript
const BASE_URL = 'https://gapradar.com.br';

async function runFullAnalysis() {
  // 1. Login
  const loginRes = await fetch(`${BASE_URL}/api/auth/login`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ email: 'user@example.com', password: 'minhasenha123' }),
  });
  const { data: { token } } = await loginRes.json();

  // 2. Criar análise
  const createRes = await fetch(`${BASE_URL}/api/analyze`, {
    method: 'POST',
    headers: { Authorization: `Bearer ${token}`, 'Content-Type': 'application/json' },
    body: JSON.stringify({
      idea: 'Plataforma de agendamento para pet shops com integração WhatsApp',
      category: 'software',
      targetMarket: 'Brasil',
    }),
  });
  const { data: { analysisId } } = await createRes.json();
  console.log('Análise criada:', analysisId);

  // 3. Aguardar resultado (polling)
  let result = null;
  while (!result) {
    await new Promise(r => setTimeout(r, 3000));

    const pollRes = await fetch(`${BASE_URL}/api/analyze/${analysisId}`, {
      headers: { Authorization: `Bearer ${token}` },
    });
    const { data } = await pollRes.json();

    if (data.status === 'completed') result = data.result;
    if (data.status === 'failed') throw new Error('Análise falhou');

    console.log('Status:', data.status);
  }

  console.log('Gap Score:', result.gapScore);
  console.log('Demand Score:', result.demandScore);
  console.log('Concorrentes encontrados:', result.competitors.length);

  // 4. Buscar relatório
  const reportRes = await fetch(`${BASE_URL}/api/reports/${analysisId}`, {
    headers: { Authorization: `Bearer ${token}` },
  });
  const { data: report } = await reportRes.json();
  console.log('Relatório:', report.title);
  console.log('Resumo:', report.executiveSummary);

  return { analysisId, result, report };
}

runFullAnalysis().then(console.log).catch(console.error);
```

---

## Próximos passos

- [Streaming SSE em vez de polling](streaming-integration.md)
- [Integração de billing para upgrade](billing-integration.md)
- [Referência GraphQL](../endpoints/graphql/graphql.md)
- [Schema OpenAPI completo](../../schema/openapi.yaml)
