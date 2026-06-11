# GET /api/analyze/:id — Resultado da Análise

Retorna o status e resultado de uma análise pelo ID.

**Auth:** Bearer token requerido

---

## Request

```
GET https://gapradar.com.br/api/analyze/:id
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### Path params

| Parâmetro | Tipo | Descrição |
|-----------|------|-----------|
| `id` | string | ID da análise retornado por `POST /api/analyze` |

---

## Resposta — status possíveis

### `queued` — Na fila

```json
{
  "data": {
    "id": "uuid-example-123",
    "idea": "Plataforma de agendamento para pet shops",
    "category": "software",
    "targetMarket": "Brasil",
    "status": "queued",
    "result": null,
    "createdAt": "2026-06-11T10:00:00.000Z",
    "updatedAt": "2026-06-11T10:00:00.000Z"
  }
}
```

### `running` — Em processamento

```json
{
  "data": {
    "id": "uuid-example-123",
    "status": "running",
    "result": null,
    ...
  }
}
```

### `completed` — Concluído com resultado

```json
{
  "data": {
    "id": "uuid-example-123",
    "idea": "Plataforma de agendamento para pet shops",
    "category": "software",
    "targetMarket": "Brasil",
    "status": "completed",
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
          "signal": "Usuários reclamam de falta de opções de agendamento online para pets",
          "sentiment": "positive",
          "url": "https://news.ycombinator.com/item?id=example123"
        }
      ],
      "summary": "O mercado de pet shops no Brasil apresenta demanda crescente com poucos players especializados em agendamento digital.",
      "opportunities": [
        "Mercado crescendo 15% ao ano no Brasil",
        "Poucos competidores com UX mobile-first"
      ],
      "risks": [
        "Grandes plataformas (iFood/99) podem entrar no segmento",
        "Ciclo de vendas B2B2C pode ser longo"
      ]
    },
    "createdAt": "2026-06-11T10:00:00.000Z",
    "updatedAt": "2026-06-11T10:02:30.000Z"
  }
}
```

### `failed` — Falhou

```json
{
  "data": {
    "id": "uuid-example-123",
    "status": "failed",
    "result": null,
    ...
  }
}
```

---

## Listar análises do usuário

```
GET https://gapradar.com.br/api/analyses
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### Query params opcionais

| Parâmetro | Tipo | Default | Descrição |
|-----------|------|---------|-----------|
| `page` | integer | 1 | Página |
| `limit` | integer | 10 | Itens por página (máx 50) |

**Resposta:**

```json
{
  "data": [
    {
      "id": "uuid-example-123",
      "idea": "Plataforma de agendamento para pet shops",
      "category": "software",
      "status": "completed",
      "createdAt": "2026-06-11T10:00:00.000Z"
    }
  ],
  "metadata": {
    "total": 3,
    "page": 1,
    "limit": 10,
    "totalPages": 1
  }
}
```

---

## Deletar análise

```
DELETE https://gapradar.com.br/api/analyses/:id
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Resposta:** `204 No Content`

---

## Polling com backoff

```javascript
async function pollAnalysis(analysisId, token, intervalMs = 3000) {
  const maxAttempts = 60; // máximo 3 minutos

  for (let i = 0; i < maxAttempts; i++) {
    const res = await fetch(
      `https://gapradar.com.br/api/analyze/${analysisId}`,
      { headers: { Authorization: `Bearer ${token}` } }
    );

    const { data } = await res.json();

    if (data.status === 'completed') return data.result;
    if (data.status === 'failed') throw new Error('Análise falhou');

    // Aumentar intervalo progressivamente após várias tentativas
    const wait = i > 10 ? intervalMs * 2 : intervalMs;
    await new Promise(r => setTimeout(r, wait));
  }

  throw new Error('Timeout: análise demorou mais do que o esperado');
}
```

---

## Ver também

- [POST /api/analyze — Criar análise](create-analysis.md)
- [GET /api/analyze/stream — SSE](stream-analysis.md)
- [GET /api/reports/:id — Relatório formatado](../reports/get-report.md)
