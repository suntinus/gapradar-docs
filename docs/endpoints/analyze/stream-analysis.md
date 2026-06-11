# GET /api/analyze/stream — SSE Streaming

Acompanhe o progresso de uma análise em tempo real via Server-Sent Events (SSE).

**Auth:** Bearer token requerido

---

## Por que usar SSE em vez de polling?

| | Polling | SSE |
|---|---------|-----|
| Complexidade | Simples | Moderada |
| Latência | Alta (depende do intervalo) | Baixa (push imediato) |
| Conexões de rede | Múltiplas requests | Uma conexão persistente |
| Recomendado para | Scripts simples | Interfaces de usuário |

---

## Request

```
GET https://gapradar.com.br/api/analyze/stream?analysisId=uuid-example-123
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
Accept: text/event-stream
```

### Query params

| Parâmetro | Tipo | Obrigatório | Descrição |
|-----------|------|-------------|-----------|
| `analysisId` | string | ✅ | ID da análise retornado por `POST /api/analyze` |

---

## Tipos de evento

O stream envia eventos no formato SSE padrão (`data: <json>\n\n`). Cada mensagem tem `type` e `data`:

### `status` — Atualização de progresso

Enviado durante o processamento para informar o que está acontecendo.

```
data: {"type":"status","data":"Searching ProductHunt..."}

data: {"type":"status","data":"Searching HackerNews..."}

data: {"type":"status","data":"Analyzing with AI..."}

data: {"type":"status","data":"Generating report..."}
```

### `result` — Análise concluída

Enviado uma vez quando a análise termina com sucesso. `data` contém o `AnalysisResult` completo.

```
data: {"type":"result","data":{"gapScore":8,"demandScore":7,"competitors":[{"name":"PetScheduler","url":"https://petscheduler.example.com","description":"Agendamento para clínicas veterinárias","source":"producthunt","sentiment":"neutral"}],"marketSignals":[{"source":"hackernews","signal":"Discussão sobre dificuldade de agendamento pet","sentiment":"positive"}],"summary":"O mercado apresenta...","opportunities":["Crescimento 15% ao ano"],"risks":["Possível entrada de grande player"]}}
```

### `done` — Stream encerrado

Enviado após `result` ou após `error`, sinalizando que o stream pode ser fechado.

```
data: {"type":"done","data":""}
```

### `error` — Falha na análise

Enviado quando a análise falha. Sempre seguido de `done`.

```
data: {"type":"error","data":"Analysis failed: LLM timeout"}

data: {"type":"done","data":""}
```

---

## Implementações

### JavaScript — EventSource (mais simples)

`EventSource` é nativo no browser, mas **não suporta headers customizados**. Use somente se o token puder ser passado via query param (verifique disponibilidade na API):

```javascript
// Nota: EventSource não suporta Authorization header no browser
// Use fetch + ReadableStream abaixo para maior compatibilidade
const eventSource = new EventSource(
  `https://gapradar.com.br/api/analyze/stream?analysisId=uuid-example-123`
);

eventSource.onmessage = (event) => {
  const msg = JSON.parse(event.data);

  switch (msg.type) {
    case 'status':
      console.log('Progresso:', msg.data);
      break;
    case 'result':
      console.log('Resultado:', msg.data);
      break;
    case 'error':
      console.error('Erro:', msg.data);
      eventSource.close();
      break;
    case 'done':
      eventSource.close();
      break;
  }
};

eventSource.onerror = () => {
  console.error('Conexão SSE perdida');
  eventSource.close();
};
```

### JavaScript — fetch + ReadableStream (recomendado, suporta headers)

```javascript
async function streamAnalysis(analysisId, token, callbacks) {
  const { onStatus, onResult, onError, onDone } = callbacks;

  const response = await fetch(
    `https://gapradar.com.br/api/analyze/stream?analysisId=${analysisId}`,
    {
      headers: {
        Authorization: `Bearer ${token}`,
        Accept: 'text/event-stream',
      },
    }
  );

  if (!response.ok) {
    throw new Error(`HTTP ${response.status}`);
  }

  const reader = response.body.getReader();
  const decoder = new TextDecoder();
  let buffer = '';

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;

    buffer += decoder.decode(value, { stream: true });
    const lines = buffer.split('\n');
    buffer = lines.pop(); // manter linha incompleta no buffer

    for (const line of lines) {
      if (!line.startsWith('data: ')) continue;

      const msg = JSON.parse(line.slice(6));

      switch (msg.type) {
        case 'status':
          onStatus?.(msg.data);
          break;
        case 'result':
          onResult?.(msg.data);
          break;
        case 'error':
          onError?.(msg.data);
          break;
        case 'done':
          onDone?.();
          return;
      }
    }
  }
}

// Uso
await streamAnalysis('uuid-example-123', 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...', {
  onStatus: (msg) => updateProgressBar(msg),
  onResult: (result) => displayResult(result),
  onError: (err) => showError(err),
  onDone: () => hideSpinner(),
});
```

### TypeScript — Implementação tipada completa

```typescript
interface AnalysisResult {
  gapScore: number;
  demandScore: number;
  competitors: Competitor[];
  marketSignals: MarketSignal[];
  summary: string;
  opportunities: string[];
  risks: string[];
}

interface Competitor {
  name: string;
  url: string;
  description: string;
  source: string;
  sentiment: 'positive' | 'neutral' | 'negative';
}

interface MarketSignal {
  source: string;
  signal: string;
  sentiment: 'positive' | 'neutral' | 'negative';
  url?: string;
}

type SSEMessage =
  | { type: 'status'; data: string }
  | { type: 'result'; data: AnalysisResult }
  | { type: 'error'; data: string }
  | { type: 'done'; data: '' };

interface StreamCallbacks {
  onStatus?: (message: string) => void;
  onResult?: (result: AnalysisResult) => void;
  onError?: (error: string) => void;
  onDone?: () => void;
}

async function streamAnalysis(
  analysisId: string,
  token: string,
  callbacks: StreamCallbacks
): Promise<void> {
  const response = await fetch(
    `https://gapradar.com.br/api/analyze/stream?analysisId=${analysisId}`,
    {
      headers: {
        Authorization: `Bearer ${token}`,
        Accept: 'text/event-stream',
      },
    }
  );

  if (!response.ok) {
    throw new Error(`SSE connection failed: HTTP ${response.status}`);
  }

  const reader = response.body!.getReader();
  const decoder = new TextDecoder();
  let buffer = '';

  try {
    while (true) {
      const { done, value } = await reader.read();
      if (done) break;

      buffer += decoder.decode(value, { stream: true });
      const lines = buffer.split('\n');
      buffer = lines.pop() ?? '';

      for (const line of lines) {
        if (!line.startsWith('data: ')) continue;

        const msg: SSEMessage = JSON.parse(line.slice(6));

        if (msg.type === 'status') callbacks.onStatus?.(msg.data);
        else if (msg.type === 'result') callbacks.onResult?.(msg.data);
        else if (msg.type === 'error') callbacks.onError?.(msg.data);
        else if (msg.type === 'done') {
          callbacks.onDone?.();
          return;
        }
      }
    }
  } finally {
    reader.releaseLock();
  }
}
```

### React Hook

```typescript
import { useState, useCallback } from 'react';

function useGapRadarStream() {
  const [status, setStatus] = useState<string>('');
  const [result, setResult] = useState<AnalysisResult | null>(null);
  const [error, setError] = useState<string | null>(null);
  const [isStreaming, setIsStreaming] = useState(false);

  const startStream = useCallback(async (analysisId: string, token: string) => {
    setIsStreaming(true);
    setError(null);
    setResult(null);

    try {
      await streamAnalysis(analysisId, token, {
        onStatus: setStatus,
        onResult: setResult,
        onError: setError,
        onDone: () => setIsStreaming(false),
      });
    } catch (err) {
      setError(String(err));
      setIsStreaming(false);
    }
  }, []);

  return { status, result, error, isStreaming, startStream };
}
```

---

## Erros comuns

| Situação | Causa | Solução |
|----------|-------|---------|
| `HTTP 401` | Token inválido | Fazer novo login |
| `HTTP 404` | analysisId não encontrado | Verificar o ID |
| Evento `error` no stream | LLM timeout ou scraping falhou | Criar nova análise |
| Conexão cai sem evento `done` | Timeout de rede | Usar `onerror` para reconectar ou fazer polling |

---

## Ver também

- [POST /api/analyze — Criar análise](create-analysis.md)
- [GET /api/analyze/:id — Polling de resultado](get-result.md)
- [Guia completo de streaming](../../guides/streaming-integration.md)
