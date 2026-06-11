# Tratamento de Erros

Referência de códigos de status HTTP, formato de erros e como lidar com situações comuns.

---

## Formato de erro

Todos os erros seguem o envelope:

```json
{
  "error": "Mensagem descritiva do erro",
  "statusCode": 400
}
```

Erros de validação podem incluir detalhes:

```json
{
  "error": "Validation failed",
  "statusCode": 422,
  "details": [
    { "field": "email", "message": "must be a valid email" },
    { "field": "password", "message": "must be at least 8 characters" }
  ]
}
```

---

## Códigos de status

| Status | Quando ocorre | Como lidar |
|--------|---------------|------------|
| `200 OK` | Sucesso | — |
| `201 Created` | Recurso criado (ex: register) | — |
| `202 Accepted` | Análise enfileirada | Usar polling ou SSE para aguardar resultado |
| `204 No Content` | Deletado com sucesso | — |
| `400 Bad Request` | Dados inválidos, token de reset expirado | Verificar campos obrigatórios |
| `401 Unauthorized` | Token ausente, inválido ou expirado | Fazer novo login |
| `402 Payment Required` | Limite de análises atingido ou plano insuficiente | Upgrade de plano |
| `404 Not Found` | Recurso não encontrado | Verificar o ID |
| `409 Conflict` | Email já cadastrado | Usar login em vez de register |
| `422 Unprocessable Entity` | Validação falhou | Corrigir os campos indicados |
| `429 Too Many Requests` | Rate limit atingido | Aguardar e tentar novamente |
| `500 Internal Server Error` | Erro inesperado no servidor | Reportar ao suporte |

---

## 401 — Token inválido ou expirado

```json
{
  "error": "Unauthorized",
  "statusCode": 401
}
```

**Causas:**
- Header `Authorization` ausente
- Token malformado
- Token expirado

**Solução:** Fazer novo `POST /api/auth/login` e usar o novo token.

```javascript
async function withRetry(url, options) {
  let response = await fetch(url, options);

  if (response.status === 401) {
    const newToken = await refreshLogin();
    options.headers.Authorization = `Bearer ${newToken}`;
    response = await fetch(url, options);
  }

  return response;
}
```

---

## 402 — Limite de análises ou plano insuficiente

```json
{
  "error": "Analysis limit reached. Upgrade your plan to continue.",
  "statusCode": 402
}
```

**Causas:**
- Plano Free: 3 análises/mês esgotadas
- Tentativa de exportar PDF no plano Free ou Starter
- Tentativa de acessar API no plano Free ou Starter

**Solução:**

```javascript
const response = await fetch('https://gapradar.com.br/api/analyze', { ... });

if (response.status === 402) {
  // Redirecionar para upgrade
  window.location.href = 'https://gapradar.com.br/pricing';
}
```

Ou verificar limite antes:

```bash
curl https://gapradar.com.br/api/users/me/usage \
  -H "Authorization: Bearer eyJ..."
# Confira analysesUsed vs analysesLimit
```

---

## 429 — Rate Limit

```json
{
  "error": "Too many requests",
  "statusCode": 429
}
```

Limites por plano (requests por minuto na API REST):

| Plano | Limite |
|-------|--------|
| Free | 30 req/min |
| Starter | 60 req/min |
| Pro | 120 req/min |
| Avulso | 60 req/min |

**Como lidar:** Use exponential backoff:

```javascript
async function fetchWithBackoff(url, options, maxRetries = 3) {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    const response = await fetch(url, options);

    if (response.status !== 429) return response;

    const waitMs = Math.pow(2, attempt) * 1000; // 1s, 2s, 4s
    await new Promise(resolve => setTimeout(resolve, waitMs));
  }
  throw new Error('Max retries exceeded');
}
```

---

## Erros no SSE

No stream de análise, erros são entregues como eventos antes do fechamento:

```
data: {"type":"error","data":"Analysis failed: LLM timeout"}

data: {"type":"done","data":""}
```

**Como lidar:**

```javascript
eventSource.onmessage = (e) => {
  const msg = JSON.parse(e.data);

  if (msg.type === 'error') {
    console.error('Análise falhou:', msg.data);
    eventSource.close();
    // Mostrar mensagem de erro ao usuário
    return;
  }

  if (msg.type === 'result') {
    // Processar resultado
  }

  if (msg.type === 'done') {
    eventSource.close();
  }
};

eventSource.onerror = () => {
  // Conexão SSE perdida — tentar reconectar ou usar polling
  eventSource.close();
};
```

---

## Análise com status "failed"

Ao usar polling (`GET /api/analyze/:id`), verifique o campo `status`:

```javascript
async function pollAnalysis(analysisId, token) {
  while (true) {
    const res = await fetch(`https://gapradar.com.br/api/analyze/${analysisId}`, {
      headers: { Authorization: `Bearer ${token}` }
    });
    const { data } = await res.json();

    if (data.status === 'completed') return data.result;
    if (data.status === 'failed') throw new Error('Análise falhou');

    // Aguardar 3 segundos antes do próximo poll
    await new Promise(r => setTimeout(r, 3000));
  }
}
```

---

## Checklist de integração

- [ ] Tratar `401` com refresh de token ou redirect para login
- [ ] Tratar `402` com redirect para upgrade de plano
- [ ] Tratar `429` com exponential backoff
- [ ] No SSE, tratar eventos `error` e o callback `onerror`
- [ ] No polling, verificar `status === "failed"` além de `"completed"`
