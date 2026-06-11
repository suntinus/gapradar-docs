# POST /api/auth/login

Autentica o usuário e retorna um JWT Bearer token.

**Auth:** Não requerida

---

## Request

```
POST https://gapradar.com.br/api/auth/login
Content-Type: application/json
```

### Body

| Campo | Tipo | Obrigatório | Descrição |
|-------|------|-------------|-----------|
| `email` | string (email) | ✅ | Email cadastrado |
| `password` | string | ✅ | Senha da conta |

```json
{
  "email": "user@example.com",
  "password": "minhasenha123"
}
```

---

## Resposta

### `200 OK` — Login realizado

```json
{
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "user": {
      "id": "uuid-example-user-123",
      "email": "user@example.com",
      "name": "Maria Silva",
      "plan": "free",
      "analysesUsed": 1,
      "analysesLimit": 3,
      "createdAt": "2026-06-01T10:00:00.000Z"
    }
  }
}
```

### `401 Unauthorized` — Credenciais inválidas

```json
{
  "error": "Invalid credentials",
  "statusCode": 401
}
```

> Por segurança, a resposta não informa se o email existe ou não.

---

## Usando o token

Inclua o token no header de todas as requests autenticadas:

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

---

## Exemplo curl

```bash
curl -X POST https://gapradar.com.br/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"email":"user@example.com","password":"minhasenha123"}'
```

## Exemplo JavaScript

```javascript
async function login(email, password) {
  const response = await fetch('https://gapradar.com.br/api/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ email, password }),
  });

  if (!response.ok) {
    throw new Error('Credenciais inválidas');
  }

  const { data } = await response.json();
  return data.token; // Salvar em localStorage ou estado seguro
}
```

## Exemplo TypeScript com armazenamento de token

```typescript
interface LoginResponse {
  data: {
    token: string;
    user: {
      id: string;
      email: string;
      name: string;
      plan: 'free' | 'starter' | 'pro' | 'avulso';
      analysesUsed: number;
      analysesLimit: number;
    };
  };
}

async function login(email: string, password: string): Promise<string> {
  const res = await fetch('https://gapradar.com.br/api/auth/login', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ email, password }),
  });

  if (res.status === 401) throw new Error('Credenciais inválidas');
  if (!res.ok) throw new Error('Erro no servidor');

  const body: LoginResponse = await res.json();
  const token = body.data.token;

  // Salvar token (ex: sessionStorage para segurança)
  sessionStorage.setItem('gapradar_token', token);

  return token;
}
```
