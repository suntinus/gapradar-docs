# /find-endpoint

**Descrição:** Encontra o endpoint da API GapRadar para uma ação específica.

**Uso:** `/find-endpoint <descrição da ação>`

**Exemplos:**
- `/find-endpoint criar análise` → `POST /api/analyze`
- `/find-endpoint streaming resultado` → `GET /api/analyze/stream`
- `/find-endpoint exportar pdf` → `GET /api/reports/:id/export`
- `/find-endpoint upgrade plano` → `POST /api/billing/checkout`

**Como o assistente deve responder:**

1. Identificar a ação descrita pelo usuário
2. Mapear para o endpoint correspondente nesta tabela:

| Ação | Método | Path | Auth |
|------|--------|------|------|
| Criar conta | POST | `/api/auth/register` | — |
| Login | POST | `/api/auth/login` | — |
| Usuário atual | GET | `/api/auth/me` | ✅ |
| Reset senha | POST | `/api/auth/forgot-password` | — |
| Criar análise | POST | `/api/analyze` | ✅ |
| Resultado de análise | GET | `/api/analyze/:id` | ✅ |
| Stream SSE | GET | `/api/analyze/stream` | ✅ |
| Listar análises | GET | `/api/analyses` | ✅ |
| Deletar análise | DELETE | `/api/analyses/:id` | ✅ |
| Relatório completo | GET | `/api/reports/:id` | ✅ |
| Exportar PDF | GET | `/api/reports/:id/export` | ✅ |
| Perfil do usuário | GET | `/api/users/me` | ✅ |
| Uso do mês | GET | `/api/users/me/usage` | ✅ |
| Listar planos | GET | `/api/billing/plans` | — |
| Iniciar checkout | POST | `/api/billing/checkout` | ✅ |
| Ver assinatura | GET | `/api/billing/subscription` | ✅ |
| Portal billing | POST | `/api/billing/portal` | ✅ |
| GraphQL | POST | `/graphql` | ✅ |

3. Retornar o endpoint, método, se requer auth, e link para a documentação detalhada.
