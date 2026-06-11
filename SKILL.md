# SKILL — GapRadar API Context

Use este arquivo como contexto quando o assistente precisar responder perguntas ou gerar código relacionado à API do GapRadar.

---

## O que é o GapRadar

GapRadar é um SaaS de validação de gaps de mercado com IA. O usuário submete uma ideia de produto, e o sistema:

1. Scrapa fontes de mercado: ProductHunt, HackerNews, IndieHackers, G2/Capterra/Amazon/Etsy/Trustpilot
2. Processa os dados com LLM para identificar concorrentes, sinais de demanda e oportunidades
3. Retorna um relatório estruturado com `gapScore` (0-10) e `demandScore` (0-10)

**Base URL**: `https://gapradar.com.br`

---

## Como usar este repositório como contexto

Para usar como contexto no Claude Code ou similar:

```
/add-dir gapradar-docs/
```

Ou referenciar arquivos específicos:
- Fluxo de autenticação → `docs/authentication.md`
- Criar e acompanhar análise → `docs/guides/first-analysis.md`
- Streaming SSE → `docs/guides/streaming-integration.md`
- Schema completo → `schema/openapi.yaml`
- GraphQL → `docs/endpoints/graphql/graphql.md`

---

## Entidades Principais

### Analysis
Representa uma análise de mercado em andamento ou concluída.

```typescript
interface Analysis {
  id: string;            // UUID
  idea: string;          // Ideia submetida pelo usuário
  category: 'software' | 'physical' | 'service';
  targetMarket?: string;
  status: 'queued' | 'running' | 'completed' | 'failed';
  result?: AnalysisResult;
  createdAt: string;     // ISO 8601
  updatedAt: string;
}
```

### AnalysisResult
Resultado gerado pela IA após a análise.

```typescript
interface AnalysisResult {
  gapScore: number;        // 0-10, quanto de gap existe no mercado
  demandScore: number;     // 0-10, nível de demanda observada
  competitors: Competitor[];
  marketSignals: MarketSignal[];
  summary: string;         // Texto narrativo do resultado
  opportunities: string[];
  risks: string[];
}
```

### Competitor
Concorrente identificado nas fontes de dados.

```typescript
interface Competitor {
  name: string;
  url: string;
  description: string;
  source: string;          // 'producthunt' | 'g2' | 'capterra' | etc.
  sentiment?: 'positive' | 'neutral' | 'negative';
}
```

### MarketSignal
Sinal de mercado extraído de discussões e avaliações.

```typescript
interface MarketSignal {
  source: string;          // 'hackernews' | 'indiehackers' | 'g2' | etc.
  signal: string;          // Texto do sinal identificado
  sentiment: 'positive' | 'neutral' | 'negative';
  url?: string;
}
```

### User
Usuário autenticado na plataforma.

```typescript
interface User {
  id: string;
  email: string;
  name: string;
  plan: 'free' | 'starter' | 'pro' | 'avulso';
  analysesUsed: number;
  analysesLimit: number;
}
```

---

## Padrão de Autenticação

Todos os endpoints protegidos requerem Bearer token JWT no header:

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

Obter token via `POST /api/auth/login`. Token é JWT HS256.

---

## Padrão de Eventos SSE

O streaming de análise usa Server-Sent Events em `GET /api/analyze/stream`.

Tipos de evento:

| type | Quando | data |
|------|--------|------|
| `status` | Durante processamento | String descritiva ("Searching ProductHunt...") |
| `result` | Análise concluída | Objeto `AnalysisResult` completo |
| `done` | Stream encerrado | String vazia `""` |
| `error` | Falha na análise | Mensagem de erro string |

---

## GraphQL

Endpoint: `POST /graphql`
Auth: mesmo Bearer token no header `Authorization`.

Query principal:
```graphql
query GetAnalysis($id: ID!) {
  analysis(id: $id) {
    id status idea
    result {
      gapScore demandScore
      competitors { name url description }
      marketSignals { source signal sentiment }
    }
  }
}
```

---

## Perguntas e Respostas Rápidas

**Q: Como criar uma análise e obter o resultado?**
A: `POST /api/analyze` retorna `analysisId`. Depois poll `GET /api/analyze/:id` até `status === 'completed'`, ou use SSE em `GET /api/analyze/stream` para tempo real.

**Q: Análise é síncrona ou assíncrona?**
A: Assíncrona. A criação retorna imediatamente com `status: "queued"`. Use polling ou SSE para acompanhar.

**Q: Qual a diferença entre `/api/analyze/:id` e `/api/reports/:analysisId`?**
A: `/analyze/:id` retorna o objeto bruto da análise (status + result). `/reports/:analysisId` retorna o relatório formatado com narrativa completa, adequado para exibição ao usuário.

**Q: O plano Free tem limitações?**
A: Sim, 3 análises/mês. Ao atingir o limite, `POST /api/analyze` retorna `402 Payment Required`. Use `GET /api/billing/plans` para ver opções de upgrade.

**Q: Como exportar relatório como PDF?**
A: `GET /api/reports/:analysisId/export` — retorna URL do PDF. Disponível nos planos Pro e Avulso.

**Q: A API suporta GraphQL além de REST?**
A: Sim. `POST /graphql` com header Bearer. Ver `docs/endpoints/graphql/graphql.md`.
