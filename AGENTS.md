<!-- BEGIN:nextjs-agent-rules -->
# This is NOT the Next.js you know

This version has breaking changes — APIs, conventions, and file structure may all differ from your training data. Read the relevant guide in `node_modules/next/dist/docs/` before writing any code. Heed deprecation notices.
<!-- END:nextjs-agent-rules -->

<!-- BEGIN:luiproxyai-rules -->
# luiproxyai — OpenAI-compatible LLM gateway

Single source of truth for agent instructions in this repo. `CLAUDE.md` imports this file; put rules here, not there.

Stack: Next.js 16 (App Router) · TypeScript 5 · Postgres 17 + pgvector · Valkey · Caddy · Docker Compose.
Fork of `jaturapornchai/bcproxyai` (upstream package name: `sml-gateway`).
`main` tracks upstream untouched; all work lands on `lui/*` branches.

```bash
git fetch upstream && git merge upstream/main   # refresh main
```

## Docs
- [README.md](README.md) — feature overview, catalog, routing pipeline (Thai)
- [docs/API-GUIDE.md](docs/API-GUIDE.md) — gateway config guide
- [docs/maintainability-notes.md](docs/maintainability-notes.md) — agreed refactor plan for the hot path
- [.env.production.example](.env.production.example) — every secret a server deploy needs

## Documentation = current state only
When writing/rewriting README.md, AGENTS.md, or any docs file: describe ONLY what exists right now. Never include "previously / migrated from / what's new / changelog / version history". If something was removed, just don't mention it. Treat every doc rewrite as if the reader has never seen the project. Changelogs belong in `git log`, not docs.

**Why:** Mixing history with current state confuses readers.

## Port map (do not confuse)
| Port | Service |
|------|---------|
| 3334 | gateway via in-compose Caddy — local dev (`docker-compose.yml`) |
| 8335 | gateway via in-compose Caddy — lui-cloud (`docker-compose.lui.yml`); firewalld keeps it off the internet, nginx-proxy-manager forwards to 172.18.0.1:8335 |
| 5434 | Postgres, host-exposed in local dev only |
| 6382 | Valkey, host-exposed in local dev only |

The app container exposes only internal port 3000; host traffic routes through the in-compose `caddy` service. Scale with `docker compose up -d --scale sml-gateway=N`.

## Deploy + verify (never claim "done" without all 3 passing)

Local:
```bash
npx next build                                                             # (1) 0 errors
docker compose up -d --build
sleep 5 && curl -s -o /dev/null -w "%{http_code}\n" http://localhost:3334/  # (2) 200
docker compose ps                                                          # (3) Up, not Restarting/Exited
```

lui-cloud (aarch64, Oracle Linux 9):
```bash
docker compose -f docker-compose.yml -f docker-compose.lui.yml up -d --build
sleep 5 && curl -s -o /dev/null -w "%{http_code}\n" http://127.0.0.1:8335/api/health
```

Tests: `npx vitest run` — 127 tests across 10 files.

## Environment facts that bite
- `.env.production` must set `APP_ENCRYPTION_KEY` and `ADMIN_COOKIE_SECRET`; without them provider keys sit in Postgres as plaintext and the admin cookie is signed with the admin password itself.
- Provider API keys are NOT env vars — they are entered in `/setup` and stored in the `api_keys` table.
- The free-model catalog is hardcoded in `src/lib/free-model-catalog.ts`. `SML_FREE_*` env vars are ignored by design; adding or removing a model means editing that file and redeploying.
- No native modules in the dependency tree — keep it that way so `docker build` stays clean on arm64/musl.

## Known debt (do not "fix" casually)
- `src/app/v1/chat/completions/route.ts` is ~3,100 lines. `docs/maintainability-notes.md` has the agreed split; do it in a dedicated no-behavior-change PR and compare `npm run loadtest:smoke` p50/p95/p99 before and after.

## Session rules
- Specify file paths directly. Do not search broadly.
- Use sub-agents for codebase exploration.
- Batch multiple questions in one message.
- Run /compact after completing each feature.
- Run /clear when switching to an unrelated task.

## Compact instructions
Keep: uncommitted changes, decisions made, bugs + fixes, pending TODOs, current task context.
Drop: command output, failed attempts, exploration results, general discussion.
<!-- END:luiproxyai-rules -->
