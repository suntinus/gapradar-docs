# POST /api/analyze

Cria uma análise de mercado assíncrona.

**Auth:** Bearer token requerido

---

## Como funciona

A análise é **assíncrona**. O endpoint retorna imediatamente com `status: "queued"` e um `analysisId`. O processamento acontece em background:

1. Scraping de ProductHunt, HackerNews, IndieHackers, G2/Capterra
2. Análise por LLM para identificar concorrentes e sinais de mercado
3. Geração de `gapScore` e `demandScore`

Para obter o resultado:
- **Polling**: `GET /api/analyze/:id` a cada 3-5 segundos
- **SSE**: `GET /api/analyze/stream` para receber eventos em tempo real (recomendado)

---

## Request

```
POST https://gapradar.com.br/api/analyze
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json
```

### Body

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `idea` | string | ✅ | Descrição da ideia de produto a analisar |
| `category` | `"software"` \| `"physical"` \| `"service"` | ✅ | Categoria do produto |
| `targetMarket` | string | — | Mercado-alvo (ex: "Brasil", "EUA") |

```json
{
  "idea": "Plataforma de agendamento para pet shops",
  "category": "software",
  "targetMarket": "Brasil"
}
```

**Boas práticas para o campo `idea`:**
- Seja específico: "App de agendamento para banho e tosa em pet shops" é melhor do que "app para pets"
- Inclua o problema resolvido quando possível
- 10–200 caracteres ideal

**Categorias:**
- `software` — apps, SaaS, plataformas digitais
- `physical` — produtos físicos, hardware, e-commerce
- `service` — consultorias, agências, prestação de serviços

---

## Resposta

### `202 Accepted` — Análise enfileirada

```json
{
  "data": {
    "analysisId": "uuid-example-123",
    "status": "queued"
  }
}
```

### `402 Payment Required` — Limite atingido

```json
{
  "error": "Analysis limit reached. Upgrade your plan to continue.",
  "statusCode": 402
}
```

Ocorre quando o usuário do plano Free atingiu 3 análises/mês.

### `401 Unauthorized` — Token inválido

```json
{
  "error": "Unauthorized",
  "statusCode": 401
}
```

---

## Exemplos

### curl

```bash
TOKEN="eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."

curl -X POST https://gapradar.com.br/api/analyze \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "idea": "Plataforma de agendamento para pet shops",
    "category": "software",
    "targetMarket": "Brasil"
  }'
```

### JavaScript (criar + aguardar com polling)

```javascript
const TOKEN = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...';

async function createAndWaitForAnalysis(idea, category, targetMarket) {
  // 1. Criar análise
  const createRes = await fetch('https://gapradar.com.br/api/analyze', {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${TOKEN}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({ idea, category, targetMarket }),
  });

  if (createRes.status === 402) {
    throw new Error('Limite de análises atingido. Faça upgrade do plano.');
  }

  const { data: { analysisId } } = await createRes.json();

  // 2. Polling até concluir
  while (true) {
    await new Promise(r => setTimeout(r, 3000)); // aguardar 3s

    const pollRes = await fetch(`https://gapradar.com.br/api/analyze/${analysisId}`, {
      headers: { Authorization: `Bearer ${TOKEN}` },
    });
    const { data } = await pollRes.json();

    if (data.status === 'completed') return data.result;
    if (data.status === 'failed') throw new Error('Análise falhou');
    // status === 'queued' ou 'running' → continuar polling
  }
}

// Uso
const result = await createAndWaitForAnalysis(
  'Plataforma de agendamento para pet shops',
  'software',
  'Brasil'
);
console.log('Gap score:', result.gapScore);
console.log('Concorrentes:', result.competitors);
```

### TypeScript com SSE (recomendado)

```typescript
import { createAnalysis, streamAnalysis } from './gapradar-client';

const { analysisId } = await createAnalysis({
  idea: 'Plataforma de agendamento para pet shops',
  category: 'software',
  targetMarket: 'Brasil',
});

// Usar SSE para resultado em tempo real
await streamAnalysis(analysisId, {
  onStatus: (msg) => console.log('Status:', msg),
  onResult: (result) => console.log('Resultado:', result),
  onError: (err) => console.error('Erro:', err),
});
```

Ver [stream-analysis.md](stream-analysis.md) para implementação completa do SSE.

---

## Ver também

- [GET /api/analyze/:id — Obter resultado](get-result.md)
- [GET /api/analyze/stream — SSE](stream-analysis.md)
- [GET /api/reports/:id — Relatório formatado](../reports/get-report.md)
