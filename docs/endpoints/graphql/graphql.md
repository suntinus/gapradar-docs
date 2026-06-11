# GraphQL

O GapRadar expõe um endpoint GraphQL em `POST /graphql` para queries flexíveis.

**Auth:** Bearer token requerido (mesmo JWT do REST)

---

## Endpoint

```
POST https://gapradar.com.br/graphql
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Content-Type: application/json
```

---

## Autenticação

```javascript
const response = await fetch('https://gapradar.com.br/graphql', {
  method: 'POST',
  headers: {
    Authorization: `Bearer ${TOKEN}`,
    'Content-Type': 'application/json',
  },
  body: JSON.stringify({ query: '...', variables: {} }),
});
```

---

## Queries

### Buscar análise com resultado completo

```graphql
query GetAnalysis($id: ID!) {
  analysis(id: $id) {
    id
    status
    idea
    category
    targetMarket
    createdAt
    result {
      gapScore
      demandScore
      summary
      opportunities
      risks
      competitors {
        name
        url
        description
        source
        sentiment
      }
      marketSignals {
        source
        signal
        sentiment
        url
      }
    }
  }
}
```

**Variáveis:**
```json
{
  "id": "uuid-example-123"
}
```

**Resposta:**
```json
{
  "data": {
    "analysis": {
      "id": "uuid-example-123",
      "status": "completed",
      "idea": "Plataforma de agendamento para pet shops",
      "category": "software",
      "targetMarket": "Brasil",
      "createdAt": "2026-06-11T10:00:00.000Z",
      "result": {
        "gapScore": 8,
        "demandScore": 7,
        "summary": "O mercado apresenta demanda crescente...",
        "opportunities": ["Mercado crescendo 15% ao ano"],
        "risks": ["Grande player pode entrar"],
        "competitors": [
          {
            "name": "PetScheduler",
            "url": "https://petscheduler.example.com",
            "description": "Agendamento veterinário",
            "source": "producthunt",
            "sentiment": "neutral"
          }
        ],
        "marketSignals": [
          {
            "source": "hackernews",
            "signal": "Frustração com agendamento via WhatsApp",
            "sentiment": "positive",
            "url": "https://news.ycombinator.com/item?id=example123"
          }
        ]
      }
    }
  }
}
```

---

### Listar análises do usuário

```graphql
query ListAnalyses($page: Int, $limit: Int) {
  analyses(page: $page, limit: $limit) {
    data {
      id
      idea
      category
      status
      createdAt
      result {
        gapScore
        demandScore
      }
    }
    metadata {
      total
      page
      totalPages
    }
  }
}
```

---

### Buscar usuário atual

```graphql
query Me {
  me {
    id
    email
    name
    plan
    analysesUsed
    analysesLimit
  }
}
```

---

### Buscar relatório

```graphql
query GetReport($analysisId: ID!) {
  report(analysisId: $analysisId) {
    id
    title
    executiveSummary
    sections {
      title
      content
    }
    createdAt
  }
}
```

---

## Exemplo completo — fetch

```javascript
async function graphqlQuery(query, variables, token) {
  const response = await fetch('https://gapradar.com.br/graphql', {
    method: 'POST',
    headers: {
      Authorization: `Bearer ${token}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({ query, variables }),
  });

  const body = await response.json();

  if (body.errors) {
    throw new Error(body.errors[0].message);
  }

  return body.data;
}

// Uso
const data = await graphqlQuery(
  `query GetAnalysis($id: ID!) {
    analysis(id: $id) {
      id status
      result { gapScore demandScore }
    }
  }`,
  { id: 'uuid-example-123' },
  'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...'
);

console.log(data.analysis.result.gapScore); // 8
```

---

## Erros GraphQL

Erros retornam com HTTP 200, mas com campo `errors` no body:

```json
{
  "errors": [
    {
      "message": "Unauthorized",
      "extensions": { "code": "UNAUTHENTICATED" }
    }
  ],
  "data": null
}
```

Sempre verifique `body.errors` antes de usar `body.data`.

---

## GraphQL vs REST — quando usar cada um

| Use REST quando | Use GraphQL quando |
|----------------|-------------------|
| Criar análise ou autenticar | Buscar análise com resultado em um request |
| Operações de billing | Precisar de campos específicos sem sobrecarga |
| Streaming SSE | Listar análises com scores sem campos extras |

> SSE (`GET /api/analyze/stream`) é exclusivo do REST — não disponível via GraphQL.
