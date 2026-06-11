# GET /api/reports/:analysisId — Relatório Completo

Retorna o relatório formatado de uma análise concluída, com narrativa completa, recomendações e seções organizadas.

**Auth:** Bearer token requerido
**Disponibilidade:** Todos os planos (conteúdo varia por plano)

---

## Diferença entre análise e relatório

| | `GET /api/analyze/:id` | `GET /api/reports/:analysisId` |
|-|------------------------|-------------------------------|
| Conteúdo | Dados brutos da análise (scores, arrays) | Relatório formatado com narrativa |
| Uso | Processamento programático | Exibição ao usuário |
| Disponibilidade | Todos os planos | Todos os planos (conteúdo completo no Pro/Avulso) |

---

## Obter relatório

```
GET https://gapradar.com.br/api/reports/uuid-example-123
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

### Resposta `200`

```json
{
  "data": {
    "id": "uuid-report-example-456",
    "analysisId": "uuid-example-123",
    "title": "Relatório de Mercado — Plataforma de agendamento para pet shops",
    "executiveSummary": "O mercado de agendamento para pet shops no Brasil apresenta uma oportunidade significativa. Com gapScore de 8/10 e demandScore de 7/10, a ideia tem potencial para prosperar em um mercado em crescimento com competição fragmentada.",
    "sections": [
      {
        "title": "Visão Geral do Mercado",
        "content": "O setor pet brasileiro movimenta R$68 bilhões por ano e cresce 15% anualmente. Agendamento digital ainda é subatendido..."
      },
      {
        "title": "Análise de Concorrentes",
        "content": "Foram identificados 3 competidores diretos. PetScheduler (ProductHunt) e AgendaPet (G2) são os principais, porém ambos focam no segmento veterinário, deixando espaço para solução focada em banho e tosa..."
      },
      {
        "title": "Sinais de Demanda",
        "content": "8 discussões relevantes encontradas no HackerNews e IndieHackers indicam frustração com agendamento manual via WhatsApp..."
      },
      {
        "title": "Oportunidades",
        "content": "1. Mercado crescendo 15% ao ano no Brasil\n2. Poucos players com UX mobile-first\n3. Proprietários de pet shops ainda usam WhatsApp para agendamentos"
      },
      {
        "title": "Riscos e Desafios",
        "content": "1. Plataformas grandes (iFood/99) podem entrar no segmento\n2. Ciclo de vendas B2B2C pode ser longo\n3. Alta rotatividade de funcionários em pet shops dificulta adoção"
      },
      {
        "title": "Recomendação",
        "content": "Alta viabilidade. Recomendamos validar com 10 pet shops locais antes de escalar. Diferenciar com integração WhatsApp Business e lembretes automáticos."
      }
    ],
    "createdAt": "2026-06-11T10:02:30.000Z"
  }
}
```

### `404 Not Found`

```json
{
  "error": "Report not found",
  "statusCode": 404
}
```

> O relatório é gerado após a análise completar. Se a análise ainda está em andamento, aguarde e tente novamente.

---

## Exportar como PDF

```
GET https://gapradar.com.br/api/reports/uuid-example-123/export
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

**Disponível apenas nos planos Pro e Avulso.**

### Resposta `200`

```json
{
  "data": {
    "url": "https://gapradar.com.br/exports/report-uuid-example-123-20260611.pdf",
    "expiresAt": "2026-06-12T10:02:30.000Z"
  }
}
```

A URL do PDF expira em 24 horas.

### `402 Payment Required` — Plano não suporta PDF

```json
{
  "error": "PDF export is available on Pro and Avulso plans.",
  "statusCode": 402
}
```

---

## Exemplos

### curl — obter relatório

```bash
curl https://gapradar.com.br/api/reports/uuid-example-123 \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

### curl — exportar PDF

```bash
curl https://gapradar.com.br/api/reports/uuid-example-123/export \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

### JavaScript — baixar PDF automaticamente

```javascript
async function exportReportPDF(analysisId, token) {
  const res = await fetch(
    `https://gapradar.com.br/api/reports/${analysisId}/export`,
    { headers: { Authorization: `Bearer ${token}` } }
  );

  if (res.status === 402) {
    throw new Error('Exportação PDF requer plano Pro ou Avulso');
  }

  const { data } = await res.json();

  // Abrir PDF no browser
  window.open(data.url, '_blank');

  return data.url;
}
```
