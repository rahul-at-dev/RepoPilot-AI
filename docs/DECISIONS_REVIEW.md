# RepoPilot AI — Review of Open Decisions D1–D15

> Status: **Approved**. This reviews ARCHITECTURE.md §2; the approved choices are now reflected in ARCHITECTURE.md and IMPLEMENTATION_PLAN.md. Nothing is implemented yet.
> Prices and limits below were checked against vendor docs on **2026-10-02**. They change often, so check again before buying anything.

**Legend.** **Verdict**: KEEP (no change), ADJUST (same direction, different details) or CHANGE (different choice). **Devin complexity**: Low (< ½ session), Med (about ½–1 session), High (more than 1 session or many moving parts).

---

## Summary

| # | Decision | Original proposal | Verdict | Revised recommendation |
|---|---|---|---|---|
| D1 | GitHub access | Service token now, GitHub App for private repos (Phase 7) | **KEEP** | Same. Make Phase 7 (private repos) an optional stretch phase. |
| D2 | Backend hosting | Render: separate web service + background worker | **ADJUST** | Render, but **one service** runs both the API and the worker. Free plan during development, Starter ($7/mo) for the always-on demo. Split into two services only if needed. |
| D3 | Job queue | Job queue stored in MongoDB | **KEEP** | Same. Runs inside the API process (see D2). |
| D4 | Embedding model | OpenRouter if suitable, otherwise Voyage `voyage-code-3` | **ADJUST** | **`voyage-code-4` via OpenRouter** ($0.12/M tokens), stored as `binData` float32. Voyage's own API is the fallback (same model, so no re-index needed). |
| D5 | Chat / summary models | Strong `CHAT_MODEL` + cheap `FAST_MODEL` | **ADJUST** | Default `CHAT_MODEL` = `google/gemini-2.5-flash`. `FAST_MODEL` = `google/gemini-2.5-flash-lite`. Optional premium chat model (Claude Sonnet class) for demos. Pick the final models with the Phase 3 evaluation set. No `:free` models in production. |
| D6 | Tenancy | Personal accounts + `workspaceId` on every document | **ADJUST** | Personal only, keyed by `userId`. Drop `workspaceId` for v1. |
| D7 | Shared indexes | Shared index per (repo, commit) | **KEEP** | Same. Every read goes through the access check. |
| D8 | Repo size limits | 5,000 files / 50 MB / 1 MB per file / 3 repos | **ADJUST** | 2,000 files / 20 MB text / 512 KB per file / 3 repos per user, plus a global daily import cap. |
| D9 | Index freshness | Manual now, webhooks in Phase 7 | **KEEP** | Same. |
| D10 | Branch / ref | Default branch, any ref allowed | **KEEP** | Same. The UI shows the default branch; the API also accepts `ref`. |
| D11 | Atlas tier | M0/Flex for dev, M10+ for production | **CHANGE** | **M0 (free) for dev and the demo**. Flex ($8–30/mo) once storage passes 512 MB. M10 only with real users. Local development uses the `mongodb-atlas-local` Docker image. |
| D12 | Code-parsing languages | TS/JS, Python, Go, Java | **ADJUST** | TS/JS + Python in v1. Go/Java later. Other languages use line-based chunking. |
| D13 | Monetisation | None; usage metered from day one | **KEEP** | Same, plus a **hard spending cap on the OpenRouter API key**. |
| D14 | Data retention & privacy | Store file contents; OpenRouter data-collection opt-out | **ADJUST** | Don't store full file contents: fetch them from GitHub at the indexed commit, with a cache. Send `provider.data_collection: "deny"` on every LLM call. |
| D15 | Monorepo tooling | pnpm workspaces (+ Turborepo if needed) | **KEEP** | pnpm workspaces only. |

Net effect: fewer services, fewer collections, fewer vendors, $0–7/mo fixed cost, and less implementation work.

---

## Per-decision review

### D1 — GitHub access ★

- **Proposed:** a read-only GitHub token on the server for public repos (Phase 1). A GitHub App with `Contents: read` + `Metadata: read` for private repos (Phase 7).
- **Why it fits:**
  - Without a token, GitHub allows 60 requests/hour per IP. That's unusable from a shared server. A token gives 5,000/hour.
  - Each import downloads the repo as one tarball, so ~3 API calls per import. Rate limits stop being a concern.
  - A GitHub App only sees the repos the user picks and uses tokens that expire after an hour. Clerk's GitHub sign-in with the `repo` scope would give full read **and write** access to all of the user's repos. A recruiter or reviewer would flag that, so we avoid it.
- **Cost:** $0.
- **Devin complexity:**
  - Phase 1 token: Low.
  - GitHub App: **High**. It needs the install and callback flow, linking installations to users, webhook signature checks, and a public URL to receive webhooks (smee.io in development). It also needs you to register the App, which is a manual step.
- **Verdict: KEEP.** Make Phase 7 an **optional stretch**. Public repos are enough for a strong demo, and skipping Phase 7 saves 1–2 sessions.

### D2 — Backend hosting ★

- **Proposed:** Render web service + Render background worker, built from one Docker image.
- **What I verified (Render pricing and docs):**
  - Free web service: 512 MB RAM, 0.1 CPU. It **spins down after 15 minutes** without incoming requests and takes **~1 minute** to wake. 750 free hours/month.
  - Background workers have **no free plan**; they start at $7/mo (Starter: 512 MB, 0.5 CPU).
  - Starter web service: $7/mo, never spins down.
- **Why change the shape:** two services means two bills, two deploy configs, and two processes to keep healthy. At demo scale, one Node process can run the HTTP API and claim jobs from the queue in the background.
  - The code keeps two entry points, so splitting later is a deploy change, not a code change.
  - The worker is turned on by an env flag (`RUN_WORKER=true`).
- **Free-plan caveats** (fine for development, weak for a demo):
  - The 1-minute cold start is the first thing a visitor sees.
  - 0.1 CPU makes ingestion slow.
  - If no one is watching the progress page, the service can sleep mid-job. The job lease (see D3) resumes the job on the next wake-up, but it's slower.
- **Cost:** $0 (Free) → **$7/mo** (Starter, recommended for the public demo) → $14/mo if a separate worker is ever needed.
  - Alternative: Railway Hobby at $5/mo, which includes $5 of usage. It's comparable; Render's docs and free plan make it the simpler default.
- **Devin complexity:** Low. One Dockerfile, one `render.yaml`. Devin can't create the Render account, so you need to do that.
- **Verdict: ADJUST.** Render, **one service running API + worker**, Free in development, Starter for the demo.

### D3 — Job queue ★

- **Proposed:** a `jobs` collection in MongoDB.
  - A worker claims a job atomically with `findOneAndUpdate` and holds a time-limited lease, renewed by heartbeat.
  - Failed jobs retry with backoff; a `dedupeKey` prevents duplicate jobs.
- **Why it fits:**
  - Ingestion jobs are few and long (minutes each). We don't need high throughput.
  - Redis/BullMQ would add a vendor (e.g. Upstash), another secret, and another local service, for no benefit at this scale.
  - The lease also survives Render free-plan restarts and redeploys.
- **Cost:** $0 extra. Storage is negligible, and finished jobs are deleted automatically after 30 days.
- **Devin complexity:** Med. About 200 lines plus concurrency tests, behind a `JobQueue` interface so BullMQ can be swapped in later.
- **Verdict: KEEP.**

### D4 — Embedding model ★

- **Proposed:** OpenRouter if it has a suitable model; otherwise Voyage `voyage-code-3` directly.
- **What I verified:** OpenRouter now has an `/embeddings` endpoint. Its embedding catalogue includes:
  - `voyageai/voyage-code-4` — $0.12/M tokens
  - `voyageai/voyage-4-lite` — $0.02/M
  - `openai/text-embedding-3-small` — $0.02/M
  - `qwen/qwen3-embedding-8b` — $0.01/M
  - `mistralai/codestral-embed-2505` — $0.15/M
  - a few `:free` models
- **About `voyage-code-4`:**
  - Released 2026-08. Voyage built it specifically for code retrieval.
  - Voyage reports it beats `voyage-code-3` on their code benchmarks, and it costs a third less ($0.12 vs $0.18 per M tokens).
  - It supports 256/512/1024/2048 dimensions and int8/binary output.
  - Voyage (owned by MongoDB) is also the embedding provider MongoDB recommends for Atlas.
- **Recommendation:**
  - Use `voyage-code-4` **through OpenRouter**: one API key and one bill for all AI.
  - Use **1024 dimensions**. Use 512 if the OpenRouter endpoint lets us choose the dimension (to confirm in the Phase 2 spike), which halves storage.
  - Store vectors as **BSON `binData` float32**, not plain arrays of numbers. MongoDB's docs say this cuts vector disk storage by ~66%. That matters a lot on the 512 MB free tier (see D11).
  - Fallback: Voyage's own API (same model, so the vectors are compatible and no re-index is needed). Voyage's pricing page lists a 200M-token free allowance per account, but the page is inconsistent about which code model it covers, so treat the free allowance as a bonus, not something to plan around.
  - Cheapest alternative: `qwen3-embedding-8b` at $0.01/M (strong on code benchmarks, larger vectors). Kept as a config option.
- **Cost** (rough estimate, ~2M embedded tokens per 1,000-file repo once file headers and overlap are included):
  - **≈ $0.25 per repo** with `voyage-code-4`
  - ≈ $0.02 per repo with Qwen
  - Re-indexing an unchanged commit costs $0, because unchanged chunks reuse their embeddings.
- **Devin complexity:** Low. It's the same OpenAI-compatible client as chat, behind an `EmbeddingProvider` interface. Every snapshot records the model and dimension, so a model change triggers a re-index instead of silently mixing vectors.
- **Verdict: ADJUST** to `voyage-code-4` via OpenRouter, stored as `binData` float32.

### D5 — Chat / LLM models ★

- **Proposed:** two tiers set by env var: a strong `CHAT_MODEL` and a cheap `FAST_MODEL`.
- **OpenRouter list prices** (per million input/output tokens, checked via their API):

| Model | Input $/M | Output $/M | Context | Suggested role |
|---|---|---|---|---|
| `google/gemini-2.5-flash-lite` | 0.10 | 0.40 | 1M | FAST: summaries, query rewriting, finding triage |
| `openai/gpt-5-nano` | 0.05 | 0.40 | 400k | FAST alternative |
| `google/gemini-2.5-flash` | 0.30 | 2.50 | 1M | **Default CHAT** |
| `deepseek/deepseek-v3.2` | 0.28 | 0.42 | 164k | Budget CHAT alternative |
| `openai/gpt-5-mini` | 0.25 | 2.00 | 400k | CHAT alternative |
| `anthropic/claude-sonnet-4.6` | 3.00 | 15.00 | 1M | Premium / demo toggle |

- **Cost per chat answer** (~12k context tokens + ~1k output):
  - Gemini 2.5 Flash: **≈ $0.006**
  - DeepSeek V3.2: ≈ $0.004
  - Claude Sonnet: ≈ $0.05
- **Cost of summaries per repo** (~300 important files with Flash-Lite): **≈ $0.08**.
- **Cost of a full docs generation** (~150k input + 20k output with Flash): ≈ $0.10.
- **Why these defaults:**
  - The Flash tiers are cheap and fast to the first token.
  - Their 1M-token context gives room for generous retrieved code.
  - The premium toggle shows off top quality in demos without paying Sonnet prices on every message.
  - The final choice is made with data: Phase 3's evaluation set compares 2–3 candidates on citation accuracy, groundedness, latency and cost per answer.
- **Avoid `:free` models in production:**
  - They're capped at 50 requests/day, or 1,000/day after buying $10 of credits.
  - Providers throttle them.
  - Many free endpoints may keep or train on prompts, which conflicts with D14.
  - They're fine for local experiments.
- **Devin complexity:** Low. Models are env-configured, with OpenRouter fallback lists.
- **Verdict: ADJUST** to the defaults above. Final model choice comes from the evaluation.

### D6 — Tenancy

- **Proposed:** personal accounts, with a `workspaceId` on every document so Clerk Organizations can be added later.
- **Assessment:** `workspaceId` makes every query, index and test carry a concept the portfolio app never uses.
  - Adding teams later is a small, mechanical migration (backfill from `userId`).
  - Clerk's free plan includes 50,000 monthly retained users, so auth cost isn't a factor either way.
- **Cost:** $0.
- **Devin complexity:** dropping `workspaceId` removes a little work from every module.
- **Verdict: ADJUST.** Key everything by `userId`. Teams stays in the post-v1 backlog.

### D7 — Shared public repository indexes ★

- **Proposed:** each GitHub repo is stored once. Each (repo, commit) is indexed once and reused by every user with access. A `repoAccess` row links users to repos.
- **Why it fits:**
  - Demo visitors tend to import the same popular repos (React, Express, etc.).
  - Sharing turns the 2nd through Nth import into an instant, $0 operation, which also feels great in a demo.
  - It also saves storage on the free M0 tier.
- **Security angle:**
  - The shared data is the same code anyone with access can already read on GitHub.
  - Isolation depends entirely on the access check. All repo data is reached only through a lookup that confirms the user's `repoAccess` row; client-supplied IDs never go straight into a query. There are cross-user ID-guessing (IDOR) tests for every repo route.
  - Conversations, feedback and finding dismissals are **per user** and never shared.
  - For private repos (if Phase 7 ships), access is re-checked against the user's GitHub App installation.
- **Cost:** reduces embedding and LLM spend by roughly the reuse factor.
- **Devin complexity:** Med. It needs the `repoAccess` table, reference-counted cleanup of old snapshots, and the deduplicated import path. That's maybe +½ session versus per-user copies, paid back quickly.
- **Verdict: KEEP.**

### D8 — Repo size limits

- **Proposed:** ≤ 5,000 indexable files, 50 MB of text, 1 MB per file, 3 repos per user.
- **Assessment:** the new hosting and database plan (512 MB RAM and 0.1–0.5 CPU on Render, 512 MB storage on M0) can't sustain 50 MB repos.
  - A 2,000-file / 20 MB cap still covers most popular libraries and typical portfolio repos.
  - It keeps one ingestion under ~$0.50 and a few minutes.
  - A global daily import cap protects the budget if the demo gets traffic.
- **Cost:** caps both spend and storage.
- **Devin complexity:** Low. These are config values.
- **Verdict: ADJUST** to 2,000 files / 20 MB text / 512 KB per file / 3 repos per user, plus a global daily cap. All configurable.

### D9 — Index freshness

- **Proposed:** manual re-index in v1; GitHub `push` webhook triggers re-indexing in Phase 7.
- **Assessment:** manual is enough for a demo, and the re-index button shows off incremental re-indexing (cached embeddings make it fast and nearly free).
- **Cost:** $0.
- **Devin complexity:** Low (manual). Webhooks only come with the optional Phase 7.
- **Verdict: KEEP.**

### D10 — Branch / ref support

- **Proposed:** default branch by default, any branch/tag/SHA allowed; one active snapshot per repo.
- **Assessment:** almost no extra cost. Resolving a ref to a commit SHA is needed anyway to make citations reproducible.
- **Cost:** $0.
- **Devin complexity:** Low.
- **Verdict: KEEP.** The UI exposes only the default branch in v1; the API accepts `ref`.

### D11 — MongoDB Atlas tier ★

- **Proposed:** M0/Flex for development, M10+ for production.
- **What I verified (Atlas docs and pricing):**
  - **Free (M0):** 512 MB storage, ~100 operations/second, **max 3 search/vector indexes**, no backups, no dedicated search nodes.
  - **Flex:** $8–30/mo, up to 10 indexes.
  - **M10:** $0.08/hr (~$58/mo).
  - Vector Search works on all of these tiers.
- **Fit for RepoPilot:**
  - We need only **2 search indexes** (`chunks_vector`, `chunks_text`), so M0's limit of 3 works.
  - Storage per chunk ≈ 1.5 KB of code + 4 KB vector (1024-dim `binData` float32), plus metadata and the text index. That's ~6–7 KB per chunk, so **~60k chunks ≈ 15–30 mid-size repos** on M0.
  - Fetching files from GitHub instead of storing them (D14) and dropping the `embeddingCache` collection (embeddings are reused from the previous snapshot's chunks by content hash) roughly doubles the usable headroom.
  - Cleanup keeps only the last 2 snapshots per repo.
- **Local development vs CI vs production:**
  - **Local:** the official `mongodb/mongodb-atlas-local` Docker image runs MongoDB with `$search` and `$vectorSearch` on a laptop. It's free and works offline (except embedding calls), via `docker compose up`.
  - **CI:** the same image runs as a GitHub Actions service, so search integration tests run without touching Atlas.
  - **Demo / production:** a separate Atlas project with an M0 cluster (free clusters are one per project, so dev and prod are separate projects).
- **Cost:** **$0**. Upgrade to Flex (~$8–30/mo) only when storage passes ~400 MB; M10 only with real users or when backups are needed.
- **Devin complexity:** Low–Med. A script creates the search indexes, and there's a Docker Compose file. Devin can't create your Atlas account or cluster.
- **Verdict: CHANGE** to M0 for development and demo, with the local Docker image for development and CI.

### D12 — Code-parsing (AST) languages

- **Proposed:** TypeScript/JavaScript, Python, Go, Java via `web-tree-sitter`.
- **Assessment:** each language needs its own work (rules for finding functions/classes and imports, plus import-path resolution) and its own tests. Most portfolio and demo repos are TS/JS or Python.
- **Cost:** $0 at runtime.
- **Devin complexity:** each extra language is ≈ +¼ session.
- **Verdict: ADJUST** to TS/JS + Python in v1. Other languages use line-based chunking and regex import detection; Go/Java are backlog.

### D13 — Monetisation / metering

- **Proposed:** no billing; record usage in `usageEvents` from day one.
- **Assessment:** metering is cheap and doubles as the evidence for cost claims in the portfolio write-up.
  - Add a hard safety net: OpenRouter API keys support a **credit limit per key**. Set, for example, $10/month on the production key so a bug or abuse can never overspend.
  - Use separate keys for development and production.
- **Cost:** $0. OpenRouter adds no markup on model prices, but charges a **5.5% fee (minimum $0.80) when you buy credits**, so buy credits in $10–20 chunks.
- **Devin complexity:** Low.
- **Verdict: KEEP**, plus the per-key spending cap.

### D14 — Data retention & privacy

- **Proposed:** store indexed file contents; use OpenRouter's data-collection opt-out.
- **Assessment:**
  - Storing every file duplicates what's in the chunks and on GitHub. Instead, the file viewer fetches the file at the **indexed commit**, which never changes, so it can be cached indefinitely. An in-memory cache avoids repeat fetches. This is one less collection to clean up and a big M0 storage saving.
  - Every OpenRouter request sends `provider: { data_collection: "deny" }`, so requests only go to providers that don't store or train on data (`zdr: true` is a stricter option).
    - Caveat: this can rule out some providers for a model. If no compliant provider exists, the request fails rather than falling back, so the Phase 3 evaluation must confirm the chosen models have compliant providers.
  - Secrets found in repos are redacted before embedding and before any LLM call (unchanged).
- **Cost:** saves storage; the data-collection restriction may occasionally route to a slightly pricier provider.
- **Devin complexity:** Low.
- **Verdict: ADJUST.**

### D15 — Monorepo tooling

- **Proposed:** pnpm workspaces, plus Turborepo if build times warrant it.
- **Assessment:** three packages don't need a build orchestrator.
- **Cost:** $0.
- **Devin complexity:** Low.
- **Verdict: KEEP** pnpm workspaces only.

---

## Cross-cutting topics

### Security and data isolation

- **Who you are:** Clerk login tokens are verified on the API with `@clerk/express`, restricted to our frontend domains.
- **What you can see:** one access-check middleware for repo routes. Search and storage functions accept only an authorized-snapshot object, never a raw ID, so an unscoped query is hard to write by accident. Cross-user ID-guessing tests cover every repo route.
- **Vector search scoping:**
  - Filter by `snapshotId` **inside** `$vectorSearch` (the field is declared as a filter in the index). Never filter with a `$match` afterwards; that can drop results and leak ranking information.
  - Every text search has the same `snapshotId` filter.
- **Repo content is never executed.** Tarball extraction is safe (path checks, size caps). Rendered Markdown allows no raw HTML. The LLM has no tools that take actions, so prompt injection can at worst distort an answer.
- **Secrets:** API keys are stored only in Render/Vercel env vars. GitHub tokens are never stored or sent to the browser. Secrets found in repos are redacted before embedding or LLM calls, and findings store only fingerprints.

### MongoDB Atlas Vector Search specifics

- **Two indexes on `chunks`:**
  - `chunks_vector`: the vector field (1024 or 512 dimensions, cosine), with `snapshotId`, `language`, `dirs` and `kind` as filter fields.
  - `chunks_text`: a code-aware analyzer (splits camelCase and snake_case).
- **Hybrid ranking:** combine the vector and text results in application code with Reciprocal Rank Fusion (~20 lines). I'm not relying on Atlas's native `$rankFusion` stage because I haven't confirmed which MongoDB version M0 runs. App-side code is portable and easy to test.
- **Tuning:** start with `numCandidates` = 10–20× `limit` and tune with the evaluation set. Enable automatic scalar quantization only past ~100k vectors (MongoDB's guidance).
- **Index management:** index definitions live in code and are applied by a script, never clicked together in the Atlas UI.

### OpenRouter usage and cost

- **One key covers chat, summaries, triage and embeddings.**
- **Pricing:**
  - Model prices are passed through with no markup.
  - Buying credits costs 5.5% (minimum $0.80).
  - Paid models have no OpenRouter rate limits (providers may still throttle).
  - `:free` models are capped at 50/1,000 requests per day.
- **Controls:**
  - Per-key credit limits (production cap).
  - Usage from each response is recorded in `usageEvents`.
  - Per-user monthly token caps and the global daily import cap.
  - Expensive work is cached by content hash: embeddings, file summaries, AI reviews.
  - Fallback models are listed per request.
- **Expected spend at demo scale:**
  - ~20 repos indexed: ~$5–7 (embeddings + summaries + analysis AI)
  - ~1,000 chat answers on Flash: ~$6
  - Total **≈ $10–15 per month**, capped by the key limit.

### Local development vs production

| Concern | Local dev | CI | Demo / production |
|---|---|---|---|
| Frontend | `vite` dev server | build only | Vercel Hobby (free; for personal, non-commercial use) |
| API + worker | `tsx watch` (one process, `RUN_WORKER=true`) | Vitest + Supertest | Render: one service, Free → Starter $7/mo |
| MongoDB | `mongodb-atlas-local` Docker image (search + vector) | same image as a service container | Atlas M0 (separate project) |
| Auth | Clerk development instance | Clerk test tokens (only in E2E tests) | Clerk production instance (free plan) |
| GitHub | Service token (public read) | Recorded HTTP responses | Service token |
| LLM / embeddings | OpenRouter dev key (small cap) or recorded responses | **Mocked** (no live calls) | OpenRouter prod key with credit limit |
| Errors / logs | pretty pino logs | — | pino JSON in Render logs; Sentry optional (free plan) |

---

## Using Devin credits efficiently

1. **Have every account and secret ready before Phase 0:** Clerk, Atlas, Render, Vercel, OpenRouter (with a credit limit), and the GitHub service token. Waiting on credentials is the most common way sessions stall.
2. **The MVP is Phases 0–3** (import → search → cited chat). That's the portfolio centrepiece; everything else adds to it.
3. **Merge Phases 4 and 5** into one "Insights" phase, since both reuse the same parsed files and the findings UI. Make Phase 6 (docs) one session and **Phase 7 (private repos) optional**.
4. **Test with small, pinned example repos and mocked LLM responses.** Live LLM calls only happen in the on-demand evaluation run.
5. **One PR per step from the plan**, CI must pass, and phases are reviewed at the end so they can't drift.
6. **Fewer vendors and services** (no Redis, no separate worker, no Turborepo, Sentry optional) means less setup and debugging per session.

**Revised effort estimate:** MVP (Phases 0–3) ≈ 4–6 sessions. Full v1 without private repos ≈ 7–10 sessions. Phase 7 adds 1–2.

---

## Final recommended configuration

| Area | Choice |
|---|---|
| Frontend | React + Vite + TypeScript + Tailwind v4 + shadcn/ui, TanStack Query, React Router. Deployed on **Vercel Hobby** |
| Backend | Node 22 + Express 5 + TypeScript. **One Render service** running API + worker in the same process (`RUN_WORKER=true`). Free in dev → **Starter $7/mo** for the demo |
| Jobs | **Job queue in MongoDB** (atomic claims, leases, retries, deduplication), behind a `JobQueue` interface |
| Database | **MongoDB Atlas M0** (separate dev/prod projects) + Mongoose. Upgrade to Flex at ~400 MB |
| Local DB / CI | `mongodb/mongodb-atlas-local` Docker image (supports `$search` + `$vectorSearch`) |
| Search | Atlas Vector Search (`chunks_vector`, pre-filtered by `snapshotId`) + Atlas Search (`chunks_text`), combined with **app-side RRF** |
| Embeddings | **`voyage-code-4` via OpenRouter**, 1024 dimensions (512 if supported), stored as **`binData` float32**, reused by content hash. Fallback: Voyage's own API (same vectors) |
| Chat / summaries | OpenRouter: `CHAT_MODEL=google/gemini-2.5-flash`, `FAST_MODEL=google/gemini-2.5-flash-lite`, optional premium Claude Sonnet toggle. `data_collection: "deny"`. Final pick from evaluation |
| Auth | **Clerk** free plan (`@clerk/clerk-react` + `@clerk/express`), personal accounts keyed by `userId` |
| GitHub | REST API with a **read-only service token**, tarball download per import. GitHub App for private repos = **optional Phase 7** |
| Code parsing | `web-tree-sitter`, **TS/JS + Python** in v1; line-based chunking for other languages |
| Limits | 2,000 files / 20 MB text / 512 KB per file / 3 repos per user / global daily import cap; per-user monthly token cap; **OpenRouter key credit limit** |
| Storage strategy | No full file copies stored (fetched from GitHub at the indexed commit + cached); no separate embedding cache; keep last 2 snapshots per repo |
| Monorepo | pnpm workspaces: `apps/web`, `apps/api`, `packages/shared` (shared Zod schemas) |
| Quality gates | ESLint + Prettier + Husky/lint-staged, Vitest + Supertest, GitHub Actions CI, Playwright E2E in Phase 6 (hardening) |
| Observability | pino structured logs + request IDs; Sentry free plan optional |

**Estimated monthly cost:**
- **Fixed:** $0 during development; **$7/mo** with an always-on demo (Render Starter). Vercel, Atlas M0, Clerk and GitHub are all $0.
- **Usage:** ≈ **$10–15/mo** on OpenRouter at demo scale, hard-capped by the key limit.

### Approval

Approved. ARCHITECTURE.md and IMPLEMENTATION_PLAN.md were updated accordingly (Insights merged into Phase 4, AI docs = Phase 5, hardening = Phase 6, private repos = optional Phase 7). Phase 0 starts only on an explicit go-ahead.
