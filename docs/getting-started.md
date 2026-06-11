# Getting Started — GapRadar API

Guia rápido para criar sua primeira análise de mercado com a API do GapRadar.

**Base URL**: `https://gapradar.com.br`

---

## Pré-requisitos

- Conta no GapRadar (ou criar abaixo)
- `curl` ou qualquer cliente HTTP

---

## Passo 1 — Criar conta

```bash
curl -X POST https://gapradar.com.br/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "password": "minhasenha123",
    "name": "Maria Silva"
  }'
```

**Resposta:**
```json
{
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "user": {
      "id": "uuid-example-user-123",
      "email": "user@example.com",
      "name": "Maria Silva",
      "plan": "free",
      "analysesUsed": 0,
      "analysesLimit": 3
    }
  }
}
```

> O token retornado no registro já pode ser usado. Salve-o.

---

## Passo 2 — (Opcional) Login em sessões futuras

Se já tem conta, faça login para obter um novo token:

```bash
curl -X POST https://gapradar.com.br/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "password": "minhasenha123"
  }'
```

Guarde o `token` da resposta — você vai usá-lo em todos os requests autenticados.

---

## Passo 3 — Criar uma análise

```bash
curl -X POST https://gapradar.com.br/api/analyze \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -H "Content-Type: application/json" \
  -d '{
    "idea": "Plataforma de agendamento para pet shops",
    "category": "software",
    "targetMarket": "Brasil"
  }'
```

**Resposta:**
```json
{
  "data": {
    "analysisId": "uuid-example-123",
    "status": "queued"
  }
}
```

A análise é **assíncrona** — o sistema enfileira o trabalho e processa em background.

---

## Passo 4 — Acompanhar o resultado

### Opção A: Polling (simples)

Faça GET no resultado a cada 3-5 segundos até `status === "completed"`:

```bash
curl https://gapradar.com.br/api/analyze/uuid-example-123 \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

**Resposta enquanto processa:**
```json
{
  "data": {
    "id": "uuid-example-123",
    "status": "running",
    "idea": "Plataforma de agendamento para pet shops"
  }
}
```

**Resposta quando concluído:**
```json
{
  "data": {
    "id": "uuid-example-123",
    "status": "completed",
    "idea": "Plataforma de agendamento para pet shops",
    "result": {
      "gapScore": 8,
      "demandScore": 7,
      "competitors": [
        {
          "name": "PetScheduler",
          "url": "https://petscheduler.example.com",
          "description": "Plataforma de agendamento para clínicas veterinárias",
          "source": "producthunt",
          "sentiment": "neutral"
        }
      ],
      "marketSignals": [
        {
          "source": "hackernews",
          "signal": "Discussão sobre dificuldade de agendar serviços para pets",
          "sentiment": "positive"
        }
      ],
      "summary": "O mercado de pet shops no Brasil apresenta demanda crescente...",
      "opportunities": ["Mercado crescendo 15% ao ano", "Poucos players com UX mobile-first"],
      "risks": ["Grande player pode entrar no segmento"]
    }
  }
}
```

### Opção B: SSE Streaming (recomendado)

Receba eventos em tempo real sem polling. Veja [stream-analysis.md](endpoints/analyze/stream-analysis.md) para o guia completo.

```javascript
const token = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...';
const analysisId = 'uuid-example-123';

const response = await fetch(
  `https://gapradar.com.br/api/analyze/stream?analysisId=${analysisId}`,
  { headers: { Authorization: `Bearer ${token}` } }
);

const reader = response.body.getReader();
const decoder = new TextDecoder();

while (true) {
  const { done, value } = await reader.read();
  if (done) break;

  const lines = decoder.decode(value).split('\n');
  for (const line of lines) {
    if (!line.startsWith('data: ')) continue;
    const msg = JSON.parse(line.slice(6));
    console.log(msg.type, msg.data);
    if (msg.type === 'done') return;
  }
}
```

---

## Passo 5 — Ler o relatório completo

Após a análise concluída, obtenha o relatório formatado:

```bash
curl https://gapradar.com.br/api/reports/uuid-example-123 \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

---

## Próximos passos

- [Autenticação em detalhe](authentication.md)
- [Guia completo de streaming SSE](guides/streaming-integration.md)
- [Planos e billing](guides/billing-integration.md)
- [Referência GraphQL](endpoints/graphql/graphql.md)
- [Schema OpenAPI completo](../schema/openapi.yaml)
