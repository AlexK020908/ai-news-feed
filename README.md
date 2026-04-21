# AI News Ranker

AI-focused news aggregator that ranks, summarizes, and deduplicates articles about artificial intelligence in real time.

## Features
- Aggregates AI news from 32 sources (OpenAI, Anthropic, DeepMind, Google AI, arXiv, HN, GitHub trending, HuggingFace, and more)
- Semantic deduplication via Voyage embeddings + pgvector (collapses "GPT-5 released" from 5 sources into one item)
- Importance scoring with Claude Haiku 4.5 (calibrated 0–100 rubric)
- Topic clustering with automatic cluster labeling
- Real-time ranking blending Claude importance, HN/GitHub/HF engagement, source trust, cluster size, and citation signals
- Real-time UI via Supabase Realtime — new items stream to the browser as they drop
- Discord webhook push for high-importance items
- No accounts — public read-only feed, filter by category / min importance, sort by hot / new / trending

## Stack

| Layer         | Tool                                                   |
| ------------- | ------------------------------------------------------ |
| Framework     | Next.js 16 (App Router) + TypeScript                   |
| UI            | Tailwind v4 + custom primitives + lucide-react icons   |
| DB + realtime | Supabase Postgres + pgvector + Realtime                |
| LLM           | Anthropic `claude-haiku-4-5-20251001` (prompt-cached)  |
| Embeddings    | Voyage AI `voyage-3` (1024-dim) — optional             |
| Ingestion     | Vercel Cron → Next.js API routes per source adapter    |
| Deploy        | Vercel (frontend + cron), Supabase (DB)                |

## Run locally

### 1. Supabase

1. Create a project at [supabase.com](https://supabase.com).
2. In the SQL editor, run (in order):
   - `supabase/migrations/001_schema.sql` — full schema, RLS, realtime, triggers, and RPCs (`similar_items`, `similar_recent_items`, `bump_duplicate_count`, `recompute_topic_sizes`, `trending_items`, `top_topics`)
   - `supabase/seed/sources.sql` — 32 seed sources + reputation weights

### 2. Environment

```bash
cp .env.example .env.local
```

Fill in:
- `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY` — from Supabase project settings → API
- `ANTHROPIC_API_KEY` — from [console.anthropic.com](https://console.anthropic.com)
- `VOYAGE_API_KEY` — optional; enables semantic dedup via `voyage-3` embeddings (200M free tokens on signup)
- `CRON_SECRET` — any long random string; required to call `/api/cron/*` endpoints
- `ITEM_RETENTION_DAYS` — default `14`; items older than this get auto-pruned at ingest time
- `GITHUB_TOKEN` — optional; raises GitHub trending/search rate limits from 60/hr to 5000/hr
- `SEMANTIC_SCHOLAR_API_KEY` — optional; raises S2 rate limits for arXiv citation enrichment

### 3. Dev server + ingest loop

You need **two terminals**: one for the Next.js dev server, one for the ingest/enrich loop. The dev server alone shows an empty feed — new items only flow in when the cron endpoints get hit.

**Terminal 1 — dev server:**
```bash
npm install
npm run dev
```

Open [http://localhost:3000](http://localhost:3000). Empty feed is expected until the loop kicks in.

**Terminal 2 — ingest + enrich loop (every 15 min):**
```powershell
powershell -ExecutionPolicy Bypass -File .\scripts\loop.ps1
```

`scripts/loop.ps1` reads `CRON_SECRET` from `.env.local`, hits `/api/cron/ingest` then `/api/cron/enrich`, sleeps 15 min, repeats. Leave it running — items start appearing in the feed within a minute.

**Quick one-off test (no loop):**
```bash
curl -H "Authorization: Bearer <CRON_SECRET>" http://localhost:3000/api/cron/ingest
curl -H "Authorization: Bearer <CRON_SECRET>" http://localhost:3000/api/cron/enrich
```

**Survive reboots:** register `scripts/loop.ps1` in Windows Task Scheduler as a "run at logon" task.

### 4. Automate in production (optional)

- **Vercel cron** — `vercel.json` already declares schedules. Requires Vercel Pro ($20/mo) — Hobby's 60s function timeout and daily-only crons won't work here.
- **GitHub Actions** — free (2000 min/mo). Add a workflow that `curl`s the deployed `/api/cron/*` endpoints on a `*/15 * * * *` schedule.

## Project layout

```
app/                  Next.js routes (feed, item detail, topic detail, search, API crons)
components/           UI primitives (item card, filter bar, topics strip, …)
lib/
  anthropic/          Claude client, enrichment prompt + parser, embeddings
  supabase/           browser + server + service-role clients
  ingest/             source adapters + dedup + normalization + engagement scoring
  topics/             cluster.ts (union-find) + label.ts (Claude cluster labeling)
  types.ts            shared domain types (Item, Source, Category)
supabase/
  migrations/         schema
  seed/               source registry seed
```

## License

MIT — see [LICENSE](LICENSE).
