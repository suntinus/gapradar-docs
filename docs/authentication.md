# Autenticação

O GapRadar usa **JWT Bearer** para autenticação. O token é obtido via login com email e senha.

---

## Fluxo básico

```
1. POST /api/auth/register  →  cria conta, retorna token
        OU
   POST /api/auth/login     →  autentica, retorna token

2. Usar token em todos os requests protegidos:
   Authorization: Bearer <token>
```

---

## Criar conta

```bash
curl -X POST https://gapradar.com.br/api/auth/register \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "password": "minhasenha123",
    "name": "Maria Silva"
  }'
```

**Resposta `201`:**
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

**Erros:**
- `409 Conflict` — email já cadastrado
- `422 Unprocessable Entity` — dados inválidos (email malformado, senha muito curta)

---

## Login

```bash
curl -X POST https://gapradar.com.br/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{
    "email": "user@example.com",
    "password": "minhasenha123"
  }'
```

**Resposta `200`:**
```json
{
  "data": {
    "token": "eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...",
    "user": { ... }
  }
}
```

**Erros:**
- `401 Unauthorized` — credenciais inválidas

---

## Usando o token

Inclua o token em todos os requests autenticados via header `Authorization`:

```bash
curl https://gapradar.com.br/api/auth/me \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

### Em JavaScript/TypeScript

```typescript
const TOKEN = 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...';

const response = await fetch('https://gapradar.com.br/api/auth/me', {
  headers: {
    Authorization: `Bearer ${TOKEN}`,
    'Content-Type': 'application/json',
  },
});

const { data } = await response.json();
console.log(data.email); // user@example.com
```

---

## Obter usuário atual

```bash
curl https://gapradar.com.br/api/auth/me \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

**Resposta:**
```json
{
  "data": {
    "id": "uuid-example-user-123",
    "email": "user@example.com",
    "name": "Maria Silva",
    "plan": "free",
    "analysesUsed": 1,
    "analysesLimit": 3,
    "createdAt": "2026-06-01T10:00:00.000Z"
  }
}
```

---

## Verificar uso do mês

```bash
curl https://gapradar.com.br/api/users/me/usage \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..."
```

**Resposta:**
```json
{
  "data": {
    "analysesUsed": 2,
    "analysesLimit": 3,
    "resetDate": "2026-07-01T00:00:00.000Z"
  }
}
```

---

## Atualizar perfil

```bash
curl -X PUT https://gapradar.com.br/api/users/me \
  -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  -H "Content-Type: application/json" \
  -d '{"name": "Maria Souza"}'
```

---

## Reset de senha

### Passo 1 — Solicitar reset

```bash
curl -X POST https://gapradar.com.br/api/auth/forgot-password \
  -H "Content-Type: application/json" \
  -d '{"email": "user@example.com"}'
```

A resposta sempre retorna `200` por segurança — independente de o email existir ou não.

```json
{
  "data": {
    "message": "Se o email existir, você receberá as instruções em breve."
  }
}
```

### Passo 2 — Redefinir senha

Use o token recebido no email:

```bash
curl -X POST https://gapradar.com.br/api/auth/reset-password \
  -H "Content-Type: application/json" \
  -d '{
    "token": "reset-token-example-abc123",
    "password": "novasenha456"
  }'
```

**Erros:**
- `400 Bad Request` — token inválido ou expirado

---

## Detalhes do JWT

- **Algoritmo**: HS256
- **Expiração**: configurada pelo servidor (tipicamente 7 dias)
- **Claims incluídos**: `sub` (user ID), `email`, `plan`, `iat`, `exp`

> Tokens expirados retornam `401 Unauthorized`. Faça novo login para obter token válido.
