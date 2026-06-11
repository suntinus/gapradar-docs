# Changelog

Todas as mudanças notáveis na API do GapRadar são documentadas neste arquivo.

Formato baseado em [Keep a Changelog](https://keepachangelog.com/pt-BR/1.0.0/).

---

## [1.0.0] — 2026-06-11

### Adicionado

**Autenticação**
- `POST /api/auth/register` — criação de conta com email, senha e nome
- `POST /api/auth/login` — login com JWT HS256
- `GET /api/auth/me` — usuário autenticado atual
- `POST /api/auth/forgot-password` — solicitação de reset de senha por email
- `POST /api/auth/reset-password` — redefinição de senha com token

**Análise de mercado**
- `POST /api/analyze` — criação de análise assíncrona (software/physical/service)
- `GET /api/analyze/:id` — polling de status e resultado
- `GET /api/analyze/stream` — streaming em tempo real via SSE (eventos: status/result/done/error)
- `GET /api/analyses` — listagem paginada de análises do usuário
- `DELETE /api/analyses/:id` — exclusão de análise

**Relatórios**
- `GET /api/reports/:analysisId` — relatório formatado com narrativa e seções
- `GET /api/reports/:analysisId/export` — exportação como PDF (planos Pro e Avulso)

**Usuários**
- `GET /api/users/me` — perfil e estatísticas de uso
- `PUT /api/users/me` — atualização de perfil
- `GET /api/users/me/usage` — quota de análises do mês corrente

**Billing**
- `GET /api/billing/plans` — listagem de planos (Free/Starter/Pro/Avulso)
- `POST /api/billing/checkout` — criação de sessão Stripe Checkout
- `GET /api/billing/subscription` — assinatura atual do usuário
- `POST /api/billing/portal` — acesso ao Stripe Customer Portal

**GraphQL**
- `POST /graphql` — endpoint GraphQL com queries para Analysis, Report, User
- Schemas: Analysis, AnalysisResult, Competitor, MarketSignal, User, Report

**Fontes de dados**
- ProductHunt — lançamentos de software
- HackerNews — discussões e Show HN
- IndieHackers — produtos indie
- G2/Capterra — avaliações B2B (via Firecrawl categórico)
- Amazon/Etsy — produtos físicos (via Firecrawl categórico)
- Trustpilot — avaliações de serviços (via Firecrawl categórico)

**Sistema de cache**
- L1: Cache semântico por similaridade de ideia (TTL 24h)
- L2: Cache de dados brutos por fonte (TTL 6h)
- L3: Memória de mercado por categoria (TTL 7 dias)

**Planos**
- Free: 3 análises/mês, relatório básico
- Starter: R$29/mês, ilimitado, relatório básico
- Pro: R$99/mês, ilimitado, relatório completo, PDF, API
- Avulso: R$49/análise, relatório completo, PDF, API
