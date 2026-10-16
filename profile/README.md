# 🌐 Agrega-Caragua

> Infraestrutura distribuída e edge-native para captação, triagem multimodal com IA, normalização e publicação em escala de vagas de emprego.

[![Production App](https://img.shields.io/badge/Production-agrega.pages.dev-blue?style=flat&logo=cloudflarepages)](https://agrega.pages.dev)
[![Organization](https://img.shields.io/badge/Org-Internal%20%2F%20Private-red?style=flat&logo=github)](https://github.com/Agrega-Caragua)
[![Runtime](https://img.shields.io/badge/Runtime-Node.js%2020%2B%20%7C%20Cloudflare%20Workers-orange?style=flat&logo=nodedotjs)](https://nodejs.org)
[![Database](https://img.shields.io/badge/Database-Neon%20Serverless%20Postgres-00E599?style=flat&logo=postgresql)](https://neon.tech)
[![Stack](https://img.shields.io/badge/Stack-TypeScript%20%7C%20Next.js%20%7C%20Hono-black?style=flat&logo=typescript)](https://typescriptlang.org)

---

## 🎯 Sobre a Organização

O **Agrega-Caragua** é um ecossistema modular construído para centralizar e qualificar o fluxo de vagas de trabalho. O pipeline unifica fontes fragmentadas — scrapers automatizados em portais externos, mensagens soltas de grupos de WhatsApp e fotos de anúncios físicos de rua —, processa os dados brutos através de OCR e Modelos de Linguagem (LLMs), e publica oportunidades verificadas com baixíssima latência no Edge.

---

## 🏗️ Topologia da Arquitetura

FONTES DE INGESTÃO
 [ Scrapers Externos ]      [ WhatsApp Mensagens ]     [ WhatsApp Fotos/Panfletos ]
 (Indeed, Gupy, Scripts)    (Grupos e Chats Privados)     (Fotos de Cartazes e Murais)
            │                          │                             │
            └──────────────────────────┼─────────────────────────────┘
                                       │
                                       ▼
                  ┌──────────────────────────────────────────┐
                  │        image-scrapper (Porta 4000)       │
                  │     Universal Gateway & OCR Multimodal   │
                  │ • Deduplicação SHA-256 antecipada        │
                  │ • Smart Relevance Guard (Anti-Spam/PIX)  │
                  │ • Groq Vision (Qwen) + Gemini Failover   │
                  │ • AI Key Balancer (Least-Inflight)       │
                  └────────────────────┬─────────────────────┘
                                       │ Least-Inflight Dispatch
                                       ▼
               ┌────────────────────────────────────────────────┐
               │          agent-agrega (Active-Active)          │
               │        2x Instâncias Cloudflare Workers        │
               │ • Formatação e enriquecimento via Hermes LLM   │
               │ • Extração de Salário, Turno e Requisitos      │
               │ • Validação tipada estrita com Zod             │
               │ • Despacho autenticado com segredo timing-safe │
               └───────────────────────┬────────────────────────┘
                                       │ POST /api/v1/webhooks/receive/:id
                                       │ Headers: x-agrega-secret + x-agrega-delivery
                                       ▼
                  ┌──────────────────────────────────────────┐
                  │             agrega (Monorepo)            │
                  │ • /server: Hono Edge API (Neon Postgres) │
                  │ • /web: Next.js App Router (SSR & SEO)   │
                  │ • Painel Administrativo (/admin)         │
                  │ • Rotação Dinâmica de Chaves de IA (M2M) │
                  └──────────────────────────────────────────┘

---

## 📦 Repositórios da Organização

Todos os repositórios listados abaixo são de uso **interno e privado**.

### 1. [`agrega`](https://github.com/Agrega-Caragua/agrega)
**Núcleo da Plataforma (Monorepo)**
* **`/web` (Frontend):** Next.js (App Router), TypeScript estrito, Tailwind CSS, Radix UI e Framer Motion. Renderização focada em SSR e SEO indexável de vagas.
* **`/server` (API Core):** Hono rodando no Cloudflare Workers, Prisma ORM com Edge Adapter e PostgreSQL Serverless no Neon (com Point-In-Time Recovery ativado).
* **Segurança e Hardening:** Autenticação via Cookies HTTP-Only (`SameSite=Lax`), barreiras anti-CSRF (cruzamento `Origin` / `Referer`), validação binária de uploads via Magic Bytes e rate limiting por IP.

### 2. [`image-scrapper`](https://github.com/Agrega-Caragua/image-scrapper)
**Gateway de Ingestão, OCR Multimodal e Load Balancer de Chaves**
* **Deduplicação Antecipada:** Calcula hash semântico SHA-256 (`empresa + cargo + local + id`). Duplicatas são descartadas como `DUPLICATE_SKIPPED` antes de gerar custos de LLM ou chamadas ao banco.
* **Pipeline de Visão:** OCR executado primariamente com **Groq Vision** (`qwen/qwen3.8-27b`) com transição automática (failover) para **Google Gemini** (`gemini-3.1-flash-lite` / `gemini-1.5-flash`).
* **Smart Relevance Guard:** Triagem semântica que descarta comprovantes Pix, cardápios, memes e conversas casuais.
* **Key Balancer & Circuit Breaker:** Algoritmo *Least-Inflight* com desempate por *Round-Robin*. Pausa automática (cooldown) em erros 429 respeitando o header `Retry-After`.

### 3. [`agent-agrega`](https://github.com/Agrega-Caragua/agent-agrega)
**Microsserviço de Normalização Inteligente (Serverless)**
* **Processamento Hermes:** Executa inferência contextual em Cloudflare Workers para inferir e estruturar cargos padronizados, faixas salariais, benefícios e modelos de trabalho.
* **Cofre M2M:** Não armazena tokens de IA localmente; requisita credenciais saudáveis do pool central sob demanda via `GET /api/v1/agent/keys/next`.
* **Despacho Confiável:** Entrega via webhook com garantia de idempotência através do cabeçalho `x-agrega-delivery` (UUID) e assinatura criptográfica com `x-agrega-secret`.

### 4. *[Pipeline Futuro]* `scrapper-indeed` *(Em importação)*
Módulo de extração pontual e agendada de vagas do Indeed, construído para enviar lotes formatados diretamente ao endpoint `POST /api/v1/gateway/ingest` do `image-scrapper`.

---

## 🔒 Padrões de Comunicação M2M e Segurança

| Origem | Destino | Protocolo / Transporte | Método de Autenticação |
| :--- | :--- | :--- | :--- |
| **Bots WhatsApp / Scrapers** | `image-scrapper` | HTTP JSON / Multipart | Rede interna / Chave de Gateway |
| **`image-scrapper`** | `agent-agrega` | HTTPS JSON | Balanceamento Least-Inflight ativo |
| **`agent-agrega`** | `agrega` (API Central) | HTTPS JSON (Webhook) | Headers `x-agrega-secret` + `x-agrega-delivery` |
| **`agent-agrega`** | `agrega` (Cofre M2M) | HTTPS JSON | Token Bearer M2M (`AGENT_API_KEY`) |

---

## 💻 Guia Unificado de Inicialização Local

Para rodar todo o ecossistema na sua máquina:

```bash
# 1. Monorepo Principal (Core + Web)
git clone [https://github.com/Agrega-Caragua/agrega.git](https://github.com/Agrega-Caragua/agrega.git)
cd agrega && npm install
cd server && cp .env.example .env && npx prisma db push
npm run dev
# -> Web: http://localhost:3000 | API: http://localhost:3333

# 2. Gateway de Ingestão & OCR (Porta 4000)
git clone [https://github.com/Agrega-Caragua/image-scrapper.git](https://github.com/Agrega-Caragua/image-scrapper.git)
cd image-scrapper && npm install
cp .env.example .env
npm run dev
# -> Gateway: http://localhost:4000

# 3. Worker de Formatação Serverless (Portas 8080 e 8081)
git clone [https://github.com/Agrega-Caragua/agent-agrega.git](https://github.com/Agrega-Caragua/agent-agrega.git)
cd agent-agrega && npm install
cp .dev.vars.example .dev.vars
npm run dev:1 # Inicia Instância 1 em http://localhost:8080
# npm run dev:2 para iniciar Instância 2 em http://localhost:8081

⚙️ Diretrizes de Engenharia
Test-Driven Development (TDD): Cobertura com Vitest e Node.js Test Runner para regras de negócio, parsers heurísticos e testes de segurança ofensiva (prevenção contra Mass Assignment e injeções).

Tipagem Estrita: 100% TypeScript com schemas de entrada e saída validados por Zod em todas as camadas.

Deploy Contínuo: Pipelines automáticos via GitHub Actions apontando diretamente para Cloudflare Pages e Cloudflare Workers, suportando rollback instantâneo via CLI do Wrangler.
