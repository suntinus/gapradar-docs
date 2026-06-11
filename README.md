# GapRadar API Docs

> **Valide gaps de mercado com IA** — Submeta uma ideia, receba análise de concorrentes, sinais de demanda e pontuação de oportunidade.

[![API Version](https://img.shields.io/badge/API-v1.0.0-blue)](https://gapradar.com.br)
[![License](https://img.shields.io/badge/license-MIT-green)](./LICENSE)

---

## Quick Start

### 1. Criar conta

```bash
curl -X POST https://gapradar.com.br/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{"email":"user@example.com","password":"minhasenha123","name":"Maria"}'
```

### 2. Fazer login e obter token

```bash
curl -X POST https://gapradar.com.br/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"user@example.com","password":"minhasenha123"}'

# Resposta:
# {"data":{"token":"eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...","user":{...}}}
```

### 3. Criar uma análise

```bash
curl -X POST https://gapradar.com.br/api/analyze \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -H "Content-Type: application/json" \
  -d '{"idea":"Plataforma de agendamento para pet shops","category":"software","targetMarket":"Brasil"}'

# Resposta:
# {"data":{"analysisId":"uuid-example-123","status":"queued"}}
```

### 4. Acompanhar resultado em tempo real (SSE)

```javascript
const eventSource = new EventSource(
  'https://gapradar.com.br/api/analyze/stream?analysisId=uuid-example-123',
  { headers: { Authorization: 'Bearer eyJ...' } }
);

eventSource.onmessage = (e) => {
  const msg = JSON.parse(e.data);
  if (msg.type === 'result') console.log('Resultado:', msg.data);
  if (msg.type === 'done') eventSource.close();
};
```

---

## Endpoints

| Método | Path | Descrição | Auth | Docs |
|--------|------|-----------|------|------|
| POST | `/api/auth/register` | Criar conta | — | [register.md](docs/endpoints/auth/register.md) |
| POST | `/api/auth/login` | Login → JWT | — | [login.md](docs/endpoints/auth/login.md) |
| GET | `/api/auth/me` | Usuário atual | ✅ | [authentication.md](docs/authentication.md) |
| POST | `/api/auth/forgot-password` | Enviar reset email | — | [authentication.md](docs/authentication.md) |
| POST | `/api/auth/reset-password` | Redefinir senha | — | [authentication.md](docs/authentication.md) |
| POST | `/api/analyze` | Criar análise (async) | ✅ | [create-analysis.md](docs/endpoints/analyze/create-analysis.md) |
| GET | `/api/analyze/:id` | Resultado da análise | ✅ | [get-result.md](docs/endpoints/analyze/get-result.md) |
| GET | `/api/analyze/stream` | SSE streaming | ✅ | [stream-analysis.md](docs/endpoints/analyze/stream-analysis.md) |
| GET | `/api/analyses` | Listar análises | ✅ | [get-result.md](docs/endpoints/analyze/get-result.md) |
| DELETE | `/api/analyses/:id` | Deletar análise | ✅ | [get-result.md](docs/endpoints/analyze/get-result.md) |
| GET | `/api/reports/:analysisId` | Relatório completo | ✅ | [get-report.md](docs/endpoints/reports/get-report.md) |
| GET | `/api/reports/:analysisId/export` | Exportar PDF | ✅ | [get-report.md](docs/endpoints/reports/get-report.md) |
| GET | `/api/users/me` | Perfil + usage | ✅ | [authentication.md](docs/authentication.md) |
| PUT | `/api/users/me` | Atualizar perfil | ✅ | [authentication.md](docs/authentication.md) |
| GET | `/api/users/me/usage` | Stats de uso | ✅ | [authentication.md](docs/authentication.md) |
| GET | `/api/billing/plans` | Planos disponíveis | — | [billing.md](docs/endpoints/billing/billing.md) |
| POST | `/api/billing/checkout` | Criar sessão Stripe | ✅ | [billing.md](docs/endpoints/billing/billing.md) |
| GET | `/api/billing/subscription` | Assinatura atual | ✅ | [billing.md](docs/endpoints/billing/billing.md) |
| POST | `/api/billing/portal` | Portal Stripe | ✅ | [billing.md](docs/endpoints/billing/billing.md) |
| POST | `/graphql` | GraphQL endpoint | ✅ | [graphql.md](docs/endpoints/graphql/graphql.md) |

---

## Arquitetura

GapRadar roda em NestJS + MySQL + Redis no backend, com frontend React/Vite servido pelo próprio servidor. A análise de mercado é assíncrona — o backend scrapa fontes (ProductHunt, HackerNews, IndieHackers, G2/Capterra) e processa com LLM antes de retornar o resultado. Veja [architecture.md](docs/architecture.md) para detalhes.

---

## Guias

- [Primeira análise — passo a passo completo](docs/guides/first-analysis.md)
- [Integração com SSE (streaming)](docs/guides/streaming-integration.md)
- [Integração de billing / planos](docs/guides/billing-integration.md)

---

## Planos

| Plano | Preço | Análises/mês | Relatório completo | PDF Export | API |
|-------|-------|--------------|-------------------|------------|-----|
| Free | Grátis | 3 | Básico | — | — |
| Starter | R$29/mês | Ilimitado | Básico | — | — |
| Pro | R$99/mês | Ilimitado | Completo | ✅ | ✅ |
| Avulso | R$49/análise | Pay-per-use | Completo | ✅ | ✅ |

---

## Links

- **Produto**: [https://gapradar.com.br](https://gapradar.com.br)
- **OpenAPI Schema**: [schema/openapi.yaml](schema/openapi.yaml)
- **Changelog**: [docs/changelog/CHANGELOG.md](docs/changelog/CHANGELOG.md)
