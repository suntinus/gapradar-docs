# Guia: Integração de Streaming SSE

Como consumir o stream de análise em tempo real usando Server-Sent Events (SSE).

---

## Por que SSE?

O GapRadar leva 1-3 minutos para completar uma análise. Em vez de fazer polling a cada 3 segundos (gerando muitas requests), SSE mantém uma conexão aberta e o servidor envia os eventos conforme acontecem — progressivamente.

Resultado: **melhor UX** (usuário vê o progresso), **menor carga de rede**, **implementação elegante**.

---

## Fluxo de eventos

```
POST /api/analyze          → obtém analysisId
GET  /api/analyze/stream   → abre conexão SSE
                                │
                                ├─ data: {"type":"status","data":"Searching ProductHunt..."}
                                ├─ data: {"type":"status","data":"Searching HackerNews..."}
                                ├─ data: {"type":"status","data":"Analyzing with AI..."}
                                ├─ data: {"type":"result","data":{...AnalysisResult}}
                                └─ data: {"type":"done","data":""}
```

---

## Implementação — JavaScript (browser)

### Opção 1: fetch + ReadableStream (recomendado)

Suporta headers customizados (necessário para o `Authorization: Bearer`):

```javascript
/**
 * Conecta ao stream SSE do GapRadar e chama callbacks por tipo de evento.
 */
async function streamAnalysis(analysisId, token, callbacks = {}) {
  const { onStatus, onResult, onError, onDone } = callbacks;

  const response = await fetch(
    `https://gapradar.com.br/api/analyze/stream?analysisId=${analysisId}`,
    {
      headers: {
        Authorization: `Bearer ${token}`,
        Accept: 'text/event-stream',
        'Cache-Control': 'no-cache',
      },
    }
  );

  if (!response.ok) {
    throw new Error(`Conexão SSE falhou: HTTP ${response.status}`);
  }

  const reader = response.body.getReader();
  const decoder = new TextDecoder();
  let buffer = '';

  try {
    while (true) {
      const { done, value } = await reader.read();
      if (done) break;

      buffer += decoder.decode(value, { stream: true });

      // Processar linhas completas
      const lines = buffer.split('\n');
      buffer = lines.pop(); // última linha pode ser incompleta

      for (const line of lines) {
        if (!line.startsWith('data: ')) continue;

        const raw = line.slice(6).trim();
        if (!raw) continue;

        const msg = JSON.parse(raw);

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
            return; // encerrar loop
        }
      }
    }
  } finally {
    reader.releaseLock();
  }
}
```

**Uso:**

```javascript
await streamAnalysis('uuid-example-123', 'eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...', {
  onStatus: (msg) => {
    document.getElementById('progress').textContent = msg;
    console.log('Progresso:', msg);
  },
  onResult: (result) => {
    console.log('Gap Score:', result.gapScore);
    console.log('Concorrentes:', result.competitors);
    renderResultCard(result);
  },
  onError: (err) => {
    console.error('Análise falhou:', err);
    showErrorMessage(err);
  },
  onDone: () => {
    hideSpinner();
  },
});
```

---

### Opção 2: EventSource (nativo, sem headers customizados)

`EventSource` é mais simples, mas **não suporta Authorization header** no browser por limitação da spec. Use somente se o backend aceitar token via query param:

```javascript
// Verifica se a API aceita token via query param
const eventSource = new EventSource(
  `https://gapradar.com.br/api/analyze/stream?analysisId=uuid-example-123&token=eyJ...`
);

eventSource.onmessage = (e) => {
  const msg = JSON.parse(e.data);
  if (msg.type === 'result') renderResult(msg.data);
  if (msg.type === 'done') eventSource.close();
};

eventSource.onerror = () => eventSource.close();
```

---

## Implementação — TypeScript completa

```typescript
interface AnalysisResult {
  gapScore: number;
  demandScore: number;
  competitors: {
    name: string;
    url: string;
    description: string;
    source: string;
    sentiment: 'positive' | 'neutral' | 'negative';
  }[];
  marketSignals: {
    source: string;
    signal: string;
    sentiment: 'positive' | 'neutral' | 'negative';
    url?: string;
  }[];
  summary: string;
  opportunities: string[];
  risks: string[];
}

type SSEEventType = 'status' | 'result' | 'error' | 'done';

interface SSEMessage {
  type: SSEEventType;
  data: string | AnalysisResult | '';
}

interface StreamCallbacks {
  onStatus?: (message: string) => void;
  onResult?: (result: AnalysisResult) => void;
  onError?: (error: string) => void;
  onDone?: () => void;
}

export async function streamAnalysis(
  analysisId: string,
  token: string,
  callbacks: StreamCallbacks
): Promise<void> {
  const url = `https://gapradar.com.br/api/analyze/stream?analysisId=${encodeURIComponent(analysisId)}`;

  const response = await fetch(url, {
    headers: {
      Authorization: `Bearer ${token}`,
      Accept: 'text/event-stream',
      'Cache-Control': 'no-cache',
    },
  });

  if (response.status === 401) throw new Error('Token inválido ou expirado');
  if (response.status === 404) throw new Error('Análise não encontrada');
  if (!response.ok) throw new Error(`Erro HTTP ${response.status}`);

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

        const raw = line.slice(6).trim();
        if (!raw) continue;

        const msg = JSON.parse(raw) as SSEMessage;

        if (msg.type === 'status') {
          callbacks.onStatus?.(msg.data as string);
        } else if (msg.type === 'result') {
          callbacks.onResult?.(msg.data as AnalysisResult);
        } else if (msg.type === 'error') {
          callbacks.onError?.(msg.data as string);
        } else if (msg.type === 'done') {
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

---

## React Hook — useGapRadarStream

```typescript
import { useState, useCallback, useRef } from 'react';
import { streamAnalysis, AnalysisResult } from './gapradar';

interface StreamState {
  statusMessage: string;
  result: AnalysisResult | null;
  error: string | null;
  isStreaming: boolean;
}

export function useGapRadarStream() {
  const [state, setState] = useState<StreamState>({
    statusMessage: '',
    result: null,
    error: null,
    isStreaming: false,
  });

  const start = useCallback(async (analysisId: string, token: string) => {
    setState({ statusMessage: 'Iniciando...', result: null, error: null, isStreaming: true });

    try {
      await streamAnalysis(analysisId, token, {
        onStatus: (msg) =>
          setState(s => ({ ...s, statusMessage: msg })),
        onResult: (result) =>
          setState(s => ({ ...s, result })),
        onError: (err) =>
          setState(s => ({ ...s, error: err, isStreaming: false })),
        onDone: () =>
          setState(s => ({ ...s, isStreaming: false })),
      });
    } catch (err) {
      setState(s => ({
        ...s,
        error: String(err),
        isStreaming: false,
      }));
    }
  }, []);

  return { ...state, start };
}
```

**Uso no componente:**

```tsx
function AnalysisPage() {
  const { statusMessage, result, error, isStreaming, start } = useGapRadarStream();

  const handleAnalyze = async () => {
    const { analysisId } = await createAnalysis({ idea: '...', category: 'software' });
    await start(analysisId, token);
  };

  return (
    <div>
      {isStreaming && <p>Analisando: {statusMessage}</p>}
      {error && <p>Erro: {error}</p>}
      {result && <ResultCard result={result} />}
    </div>
  );
}
```

---

## Node.js — server-side

```javascript
const fetch = require('node-fetch');

async function streamAnalysisNode(analysisId, token) {
  const response = await fetch(
    `https://gapradar.com.br/api/analyze/stream?analysisId=${analysisId}`,
    { headers: { Authorization: `Bearer ${token}` } }
  );

  return new Promise((resolve, reject) => {
    let buffer = '';

    response.body.on('data', (chunk) => {
      buffer += chunk.toString();
      const lines = buffer.split('\n');
      buffer = lines.pop();

      for (const line of lines) {
        if (!line.startsWith('data: ')) continue;
        const msg = JSON.parse(line.slice(6));
        if (msg.type === 'result') resolve(msg.data);
        if (msg.type === 'error') reject(new Error(msg.data));
      }
    });

    response.body.on('error', reject);
  });
}
```

---

## Reconexão automática

Se a conexão cair antes do evento `done`, reconecte:

```javascript
async function streamWithReconnect(analysisId, token, callbacks, maxRetries = 3) {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    try {
      await streamAnalysis(analysisId, token, callbacks);
      return; // sucesso
    } catch (err) {
      if (attempt === maxRetries - 1) throw err;

      const wait = Math.pow(2, attempt) * 1000;
      console.log(`Reconectando em ${wait}ms...`);
      await new Promise(r => setTimeout(r, wait));
    }
  }
}
```

---

## Ver também

- [GET /api/analyze/stream — Referência do endpoint](../endpoints/analyze/stream-analysis.md)
- [POST /api/analyze — Criar análise](../endpoints/analyze/create-analysis.md)
- [Tratamento de erros](../error-handling.md)
