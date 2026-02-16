# Nivara Portal

**Internal Developer Portal** for the [Nivara AI](https://nivara.io) platform — built on [Backstage](https://backstage.io) by Spotify.

[![Live](https://img.shields.io/badge/live-portal.nivara.io-8B7355?style=flat-square)](https://portal.nivara.io)
[![Backstage](https://img.shields.io/badge/backstage-%230052CC?style=flat-square&logo=backstage&logoColor=white)](https://backstage.io)
[![TypeScript](https://img.shields.io/badge/typescript-5.8-3178C6?style=flat-square&logo=typescript&logoColor=white)](https://typescriptlang.org)
[![Node](https://img.shields.io/badge/node-22-339933?style=flat-square&logo=nodedotjs&logoColor=white)](https://nodejs.org)
[![Deploy](https://img.shields.io/badge/deploy-Railway-0B0D0E?style=flat-square&logo=railway&logoColor=white)](https://railway.app)

---

## What it does

Nivara Portal is the single pane of glass for Nivara's AI proof-of-concept portfolio. It centralizes service discovery, documentation, project scaffolding, team ownership, quality scorecards, and AI-powered catalog search across all Nivara repos.

| Capability | Description |
|---|---|
| **Software Catalog** | Auto-discovers all `catalog-info.yaml` files across the `nivara-ai` GitHub org (10-min sync) |
| **TechDocs** | MkDocs-based documentation built and served locally for every cataloged component |
| **Scaffolder** | "Nuovo PoC Nivara" template to spin up new projects with TechDocs, `.gitignore`, `.env.example` from day zero |
| **GitHub OAuth** | Sign in with GitHub, matched to User entities via `usernameMatchingUserEntityName` |
| **Search** | Full-text search across the catalog and TechDocs, powered by PostgreSQL |
| **Design System** | Custom `nivaraDarkTheme` with centralized design tokens (colors, spacing, typography, radius, shadows) |
| **GitHub Integrations** | Pull Requests, Actions CI/CD, Security Insights, Dependabot right on entity pages |
| **Tech Radar** | Technology radar for tracking adoption of languages, frameworks, and tools |
| **Announcements** | Team-wide announcements with banner and sidebar widget |
| **Tech Insights** | Quality scorecards with automated compliance checks (owner, title, TechDocs, lifecycle) |
| **RAG AI Assistant** | Hybrid AI with scheduled indexer (30-min), anti-hallucination guardrails, and full-page chat UI |
| **AI Chat Page** | Full-page conversational UI with streaming, GFM markdown (tables, code blocks), entity mention linking |
| **Custom RBAC** | Group-based permissions — `team-nivara` gets full access, guests are read-only |

## Architecture

```
Browser ──► Backstage Frontend (React 18 · Material UI · Design Tokens)
              │
              ├── AiChatPage (react-markdown + remark-gfm, streaming SSE)
              ├── HomePage (live stats, count-up, quick actions, PRs widget)
              └── EntityPage (scorecards, GitHub, iFrame tabs)
              │
              ▼
           Backstage Backend (Node 22)
              ├── Catalog (GitHub auto-discovery, 10-min sync)
              ├── TechDocs (local builder + mkdocs-techdocs-core)
              ├── Scaffolder (GitHub publish + Roadie HTTP/Utils actions)
              ├── Auth (GitHub OAuth + Guest + service-to-service)
              ├── Search (PostgreSQL engine)
              ├── Permissions (custom Nivara RBAC policy)
              ├── Notifications + Signals (real-time)
              ├── Announcements (team-wide)
              ├── Tech Insights (quality scorecards + JSON rules engine)
              └── RAG AI (scheduled indexer + anti-hallucination)
              │
              ▼
           PostgreSQL + pgvector (Railway)
```

### RAG AI Pipeline

The AI assistant uses a three-stage pipeline with built-in quality guardrails:

```
                 ┌─── Scheduled Indexer (every 30 min) ───┐
                 │                                         │
                 │  1. Catalog entities → chunk + embed    │
                 │  2. TechDocs markdown → chunk + embed   │
                 │  3. Store in pgvector (25+ embeddings)  │
                 └─────────────────────────────────────────┘

User query ──► AiChatPage / SidebarRagModal (frontend)
                 │
                 ▼
              rag-ai-backend (API + service-to-service auth)
                 │
                 ▼
              NivaraRagAiModule (custom backend module)
                 ├── Embeddings:   OpenAI text-embedding-3-small
                 ├── Storage:      pgvector on PostgreSQL (chunkSize: 1000, top-10)
                 ├── Retrieval:    Vector similarity + Backstage Search
                 ├── Chat/LLM:    DeepSeek deepseek-chat (temp: 0, maxTokens: 1024)
                 └── Guardrails:   Strict prompt — "answer ONLY from context,
                                   never invent" + explicit refusal when
                                   context is insufficient
```

## Enterprise plugins

The portal ships with a curated set of enterprise plugins organized in three phases:

### Phase 1 — GitHub & Governance

| Plugin | Purpose |
|---|---|
| `@backstage-community/plugin-github-pull-requests-board` | PR dashboard per entity |
| `@backstage-community/plugin-github-actions` | CI/CD workflow status on entity pages |
| `@backstage-community/plugin-security-insights` | Dependabot alerts + security advisories |
| `@backstage/plugin-adr` | Architecture Decision Records |
| `@backstage-community/plugin-tech-radar` | Technology adoption radar |
| `@backstage-community/plugin-announcements` | Team-wide announcements system |
| Custom `NivaraPermissionPolicy` | Group-based RBAC (team-nivara = full, guests = read-only) |

### Phase 2 — Scaffolder & Home

| Plugin | Purpose |
|---|---|
| `@roadiehq/scaffolder-backend-module-http-request` | HTTP actions in scaffolder templates |
| `@roadiehq/scaffolder-backend-module-utils` | File/JSON/YAML utilities for scaffolder |
| `@roadiehq/backstage-plugin-home-markdown` | GitHub Markdown widget on homepage |
| `@roadiehq/backstage-plugin-home-rss` | RSS feed widget on homepage |
| `@roadiehq/backstage-plugin-iframe` | iFrame embedding for external dashboards |
| `@roadiehq/backstage-entity-validator` | `yarn validate:catalog` for CI/CD validation |

### Phase 3 — Quality & AI

| Plugin | Purpose |
|---|---|
| `@backstage-community/plugin-tech-insights` | Quality scorecards with 4 automated checks |
| `@backstage-community/plugin-tech-insights-backend-module-jsonfc` | JSON rules engine for fact evaluation |
| `@roadiehq/rag-ai` | AI assistant sidebar + modal (frontend) |
| `@roadiehq/rag-ai-backend` | RAG AI core backend plugin |
| Custom `NivaraRagAiModule` | Hybrid RAG with scheduled indexer, anti-hallucination, service-to-service auth |

#### Quality Checks (Tech Insights)

| Check | What it verifies |
|---|---|
| Group Owner Check | A Group is set as `spec.owner` |
| Title Check | Entity has a non-empty title or description |
| TechDocs Check | `backstage.io/techdocs-ref` annotation is present |
| Lifecycle Check | Entity has a defined lifecycle (production, experimental, etc.) |

### AI Chat Page

The portal includes a dedicated full-page AI chat experience at `/ai-chat`:

| Feature | Description |
|---|---|
| **Streaming responses** | Real-time SSE streaming with typing indicator |
| **GFM Markdown** | Tables, code blocks with copy button, ordered/unordered lists, task lists, strikethrough |
| **Entity mentions** | `component:default/nivara-lobby` auto-linked as clickable chips to catalog pages |
| **Suggested prompts** | Multilingual quick-start prompts (IT/EN) |
| **Anti-hallucination** | Strict system prompt — answers only from catalog context, explicit refusal when unsure |
| **Design tokens** | Consistent styling via centralized `tokens.ts` (colors, spacing, typography, radius, shadows) |

Rendering powered by `react-markdown` + `remark-gfm` with safe React component rendering.

## Tech stack

| Layer | Technology |
|---|---|
| Framework | [Backstage](https://backstage.io) v1.47 (new backend system) |
| Frontend | React 18 · Material UI 4 · React Router 6 |
| Markdown | `react-markdown` + `remark-gfm` (GFM tables, code blocks, task lists) |
| Design system | Custom design tokens (`tokens.ts`) · `nivaraDarkTheme` |
| Backend | Node.js 22 · `@backstage/backend-defaults` |
| Auth | GitHub OAuth · Guest · service-to-service (Bearer) |
| Database | PostgreSQL + pgvector (Railway managed) |
| Search | `plugin-search-backend-module-pg` |
| AI Embeddings | OpenAI `text-embedding-3-small` |
| AI Chat | DeepSeek `deepseek-chat` (V3.2) via LangChain · temp 0 |
| AI Indexer | Scheduled every 30 min (catalog + TechDocs sources) |
| Vector Store | pgvector v0.8 on PostgreSQL (chunkSize: 1000, top-10 retrieval) |
| TechDocs | MkDocs · `mkdocs-techdocs-core` · local builder |
| Infra | Docker multi-stage · Railway auto-deploy |
| Package manager | Yarn 4 (Corepack) |
| Monorepo | Backstage CLI workspaces (`packages/*`, `plugins/*`) |

## Catalog overview

The portal automatically catalogs the **Nivara AI Platform** system. Each entity has enriched descriptions (1200-2050 chars) optimized for RAG AI indexing:

| Entity | Kind | Stack | Description |
|---|---|---|---|
| `nivara-ai-platform` | System | — | Enterprise AI PoC portfolio (5 components) |
| `team-nivara` | Group | — | Core team: ndavol, Emad-Nivara, FioCola, vstoican |
| `nivara-lobby` | Component | React 19 · FastAPI · LiteLLM · Supabase | Italian legislative amendment analysis with Explainable AI |
| `nivara-navigator` | Component | React 18 · Supabase · Deno Edge Functions | AI needs assessment with DAG-based graph engine (58 nodes) |
| `nivara-playground` | Component | Next.js 16 · Vercel AI SDK 6 · Supabase | Multi-provider AI testing showroom (10 models, 4 providers) |
| `nivara-portal` | Component | Backstage · React · Node.js · PostgreSQL | This portal — RAG AI, RBAC, Tech Insights, TechDocs |
| `otb-intelligence` | Component | React 18 · Supabase · Recharts · OpenAI | Open-to-Buy planning for luxury retail (10 modules, 30 canvas views) |

## Getting started

### Prerequisites

- Node.js 22+
- Yarn (via Corepack: `corepack enable`)
- A [GitHub Personal Access Token](https://github.com/settings/tokens) with `repo` and `read:org` scope
- PostgreSQL (for production; in-memory SQLite is used for local dev)

### Install and run

```bash
git clone https://github.com/nivara-ai/nivara-portal.git
cd nivara-portal
yarn install
```

Create a `.env` file (or export the variables):

```bash
GITHUB_TOKEN=ghp_...
AUTH_GITHUB_CLIENT_ID=...        # optional — for GitHub OAuth
AUTH_GITHUB_CLIENT_SECRET=...    # optional — for GitHub OAuth
```

Start in development mode:

```bash
yarn start
```

This launches the frontend on `http://localhost:3000` and the backend on `http://localhost:7007`.

### Production build

```bash
yarn build:backend
node packages/backend --config app-config.yaml --config app-config.production.yaml
```

Or with Docker:

```bash
docker build -t nivara-portal .
docker run -p 7007:7007 \
  -e GITHUB_TOKEN=ghp_... \
  -e OPENAI_API_KEY=sk-... \
  -e DEEPSEEK_API_KEY=sk-... \
  -e PGHOST=... -e PGPORT=5432 -e PGUSER=... -e PGPASSWORD=... -e PGDATABASE=... \
  nivara-portal
```

### Enable pgvector (required for RAG AI)

Connect to your PostgreSQL instance and run:

```sql
CREATE EXTENSION IF NOT EXISTS vector;
```

## Project structure

```
nivara-portal/
├── packages/
│   ├── app/                    # Backstage frontend (React)
│   │   └── src/
│   │       ├── App.tsx         # Routes + theme + RagModal + AiChatPage route
│   │       ├── apis.ts         # API factories (RoadieRagAiClient)
│   │       ├── theme/
│   │       │   ├── index.ts            # Theme exports
│   │       │   ├── nivaraDarkTheme.ts  # Custom dark theme
│   │       │   └── tokens.ts           # Design tokens (colors, spacing, typo, radius)
│   │       └── components/
│   │           ├── Root/       # Sidebar, LogoFull, LogoIcon, SidebarRagModal
│   │           ├── ai/         # AiChatPage (react-markdown, streaming, entity mentions)
│   │           ├── catalog/    # EntityPage (scorecards, GitHub, iFrame tabs)
│   │           └── home/       # HomePage (live stats, count-up, quick actions, PRs)
│   └── backend/                # Backstage backend (Node.js)
│       └── src/
│           ├── index.ts        # Plugin registration (all phases)
│           └── plugins/
│               ├── NivaraPermissionPolicy.ts  # Custom RBAC
│               └── NivaraRagAiModule.ts       # Hybrid RAG AI (indexer + LLM + guardrails)
├── catalog-entities/
│   └── system.yaml             # System, Group, and User entities
├── scaffolder-templates/
│   └── new-poc/
│       ├── template.yaml       # Scaffolder template definition
│       └── skeleton/           # Project skeleton (Next.js starter)
├── docs/plans/                 # Design documents for each phase
├── app-config.yaml             # Base config (local development)
├── app-config.production.yaml  # Production overrides (Railway)
└── Dockerfile                  # Multi-stage build for Railway
```

## Scaffolder template

The **"Nuovo PoC Nivara"** template creates new projects pre-configured with:

- `catalog-info.yaml` registered in Backstage
- TechDocs (`mkdocs.yml` + `docs/index.md`)
- `.gitignore` (Next.js/Node.js patterns)
- `.env.example` with common variables
- Dynamic sector tagging
- Choice of deploy target (Vercel, Railway, Netlify)
- LLM provider selection (OpenAI, Anthropic, Google, Groq)

## Environment variables

| Variable | Required | Description |
|---|---|---|
| `GITHUB_TOKEN` | Yes | GitHub PAT for catalog discovery and scaffolder |
| `AUTH_GITHUB_CLIENT_ID` | No | GitHub OAuth App client ID |
| `AUTH_GITHUB_CLIENT_SECRET` | No | GitHub OAuth App client secret |
| `OPENAI_API_KEY` | Prod | OpenAI API key for RAG AI embeddings |
| `DEEPSEEK_API_KEY` | Prod | DeepSeek API key for RAG AI chat |
| `PGHOST` | Prod | PostgreSQL host |
| `PGPORT` | Prod | PostgreSQL port |
| `PGUSER` | Prod | PostgreSQL user |
| `PGPASSWORD` | Prod | PostgreSQL password |
| `PGDATABASE` | Prod | PostgreSQL database name |

## License

Proprietary. Copyright Nivara AI.
