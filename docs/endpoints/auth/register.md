# POST /api/auth/register

Cria uma nova conta de usuário.

**Auth:** Não requerida

---

## Request

```
POST https://gapradar.com.br/api/auth/register
Content-Type: application/json
```

### Body

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `email` | string (email) | ✅ | Email do usuário |
| `password` | string | ✅ | Senha (mínimo 8 caracteres) |
| `name` | string | ✅ | Nome do usuário |

```json
{
  "email": "user@example.com",
  "password": "minhasenha123",
  "name": "Maria Silva"
}
```

---

## Resposta

### `201 Created` — Conta criada

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
      "analysesLimit": 3,
      "createdAt": "2026-06-11T10:00:00.000Z"
    }
  }
}
```

O `token` retornado pode ser usado imediatamente em requests autenticados.

### `409 Conflict` — Email já cadastrado

```json
{
  "error": "Email already registered",
  "statusCode": 409
}
```

### `422 Unprocessable Entity` — Dados inválidos

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

## Exemplo curl

```bash
curl -X POST https://gapradar.com.br/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "password": "minhasenha123",
    "name": "Maria Silva"
  }'
```

## Exemplo JavaScript

```javascript
const response = await fetch('https://gapradar.com.br/api/auth/register', {
  method: 'POST',
  headers: { 'Content-Type': 'application/json' },
  body: JSON.stringify({
    email: 'user@example.com',
    password: 'minhasenha123',
    name: 'Maria Silva',
  }),
});

const { data } = await response.json();
const token = data.token; // Salvar para uso posterior
```
