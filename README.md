# Multi-Tenant AI Operations Platform — Engineering Case Study

> **Business goal:** Run a multi-department team of AI agents from one self-hosted control plane. See what each department is working on, what it has cost, what tools it is allowed to touch, and what is waiting for human sign-off. Multi-tenant, self-hosted, no SaaS lock-in.

**Status:** Client engagement (MVP). Phase 1 is delivered, deployed, and accepted. The client's product name, UI screenshots, and repository remain private; this is the sanitized engineering account.

**Scale (measured at `HEAD`):** `574` files · `~96,000` lines of application, engine, schema, and verification code · `39` Prisma models · `26` enums · `17` migrations · `97` executable verification & invariant scripts.

---

## Architecture: Four Runtimes, One Hard Database Boundary

```text
                         browser
                            │
                     [ proxy: Caddy ]          ◄── only container with a published port (80/443)
                            │
                     [ app: Next.js 16 ] ──────┬──► PostgreSQL 16 (Prisma 7)
                       │       │               ├──► Valkey / Redis (queue & cache)
                       │       │               └──► MinIO (S3-compatible uploads)
                 HTTP  │       │ queue claims
                       ▼       ▼
          [ engine: Python ]  [ worker: Node ] ──► PostgreSQL 16 (claims QUEUED tasks)
          FastAPI · MCP (67   DAG waves (3x) ·
          tools) · Mem0 +     cancellation ·
          embedded Qdrant     crash resume
          (NEVER touches DB)
                       ▲
                       │ HTTP turn (RBAC + budget gated)
            [ voice worker: LiveKit Agents ]
              Deepgram STT · ElevenLabs TTS
                       │
                 LiveKit Cloud ◄── browser joins via scoped token
```

**Why split TypeScript and Python instead of one monolith:**  
The product needed React 19 / Next.js App Router and strict relational tenancy; the agent, MCP, memory, and voice layers needed Python. Separating them over HTTP enforced one rule that paid for itself repeatedly: **the Python engine never gets a database connection.** When the engine needs tenant knowledge or task state, it calls bearer-authenticated machine endpoints on the app (`/api/brain/read`, `/api/tasks/status`). If the engine crashes, restarts, or is redeployed, the dashboard stays up, reports the degraded dependency plainly, and no agent bug can corrupt Postgres.

---

## Core Capabilities & Engineering Depth

### 1. Two-Tier MCP Tool Security (`67` Live-Discovered Tools)
The engine connects to **5 MCP servers**:
- `filesystem` (`14` tools, confined to `/workspace`)
- `github` (`26` tools)
- `terminal` (`1` tool, confined to `/workspace`)
- `playwright` (`25` tools, `--headless --isolated`)
- `brain` (`1` HTTP callback)
- Plus an optional HTTP automation connector (`activepieces`).

Tools are discovered dynamically at runtime—never hardcoded—and binaries are pinned at Docker image build time rather than downloaded at boot.

Tool access is enforced at two distinct trust boundaries:
- **Interactive chat boundary (`CHAT_TOOL_ALLOWLIST`):** The executive chat is hardcoded to `filesystem` + `brain` only. A prompt injection in chat cannot invoke Playwright scripts or shell commands.
- **Role-scoped task boundary (`AgentRole.allowedTools`):** Async tasks execute under their department role's explicit grants (e.g., Playwright is restricted to QA, content, and SEO audit roles).

### 2. Async DAG Task Orchestration, Fair-Share Claiming & Crash Recovery
- **Parallel wave execution:** A planner decomposes work into up to 8 dependent steps, executing up to 3 steps in parallel per wave with parent-failure blocking and a fan-in reconciliation stage.
- **Fair-share queue claiming:** The Node worker enforces per-company concurrency limits (1–5 active tasks) and orders claims across departments so one noisy department or tenant cannot starve another.
- **Mid-run cancellation & crash resume:** Cancelling an active task halts cleanly between waves, preserving completed step outputs and recorded token spend while leaving unfinished steps `QUEUED`. On worker restart, tasks stranded in `RUNNING` by a dead process are automatically re-queued using compound step identity `(taskId, index)` with the recovery reason written to the audit trail.

### 3. Document Ingestion, Self-Aware Retrieval & Transactional Governance
- **Deterministic multi-format ingestion:** Ingests PDF, DOCX, TXT, Markdown, CSV, JSON, ZIP, and `.eml`/`.mbox` archives (a 60-email archive imports as 60 provenance-tracked entries). Refuses rather than fakes: PDFs without a text layer or oversized files are rejected with an explicit reason and character measurement; unreadable blocks are counted and reported.
- **Self-aware context + active re-query:** The company brief injected into agent context declares its own width (e.g., `40 of 72 entries`) and gives the agent the `brain.read` tool so it can search again with new terms when initial lexical retrieval misses.
- **Provenance & closed-loop writeback:** Finished tasks write conclusions back to the knowledge store as an `OBSERVATION` badged `[earlier task result]`, keeping agent output separate from `VERIFIED_FACT` or `OWNER_DECISION` until promoted by an owner.
- **Atomic structural mutations (`BrainMutation`):** Uploading an SOP can propose new departments, workstreams, agents, and starter tasks, but nothing writes to the org chart directly. Proposals land as `PENDING` mutations and apply in one Postgres transaction only after human approval.

### 4. Scoped LiveKit Voice-to-Action Pipeline
- `/api/livekit/token` mints a tenant-scoped room token; the microphone stays off until tapped.
- A dedicated Python LiveKit Agents worker handles **Deepgram STT** and **ElevenLabs TTS**, routing spoken instructions through `/api/coo/voice/turn` under the exact same RBAC permissions (`task:create`, `task:approve`) and budget ceilings as the web UI.

### 5. Pre-Flight FinOps Guardrails & Multi-Provider Failover
- **Pre-flight budget refusal:** Every finished task is priced from real token usage at the serving model's configured rate (unconfigured rates are labelled `unpriced`, never `$0.00`). Before launching an LLM task or remote GPU job, the system checks `modelSpend + estimatedComputeSpend` against the department's monthly ceiling and refuses execution pre-flight, naming both figures.
- **Provider failover chain:** Routes across `Gemini → OpenRouter → Ollama` (OpenAI-compatible local floor), degrading gracefully with per-provider failure reasons when upstream keys fail or quota is hit.
- **Persistent per-tenant memory:** Mem0 over embedded Qdrant on a named Docker volume, partitioned by `companyId`. Memory is fail-open (never blocks chat) and writes asynchronously after the reply streams.

---

## Security & Data-Integrity Invariants

- **Session-derived tenancy:** `companyId` comes exclusively from the encrypted `iron-session` cookie—never from the request body, query string, or headers. Forged tenant IDs return `403`, not rows.
- **Default-deny routes & server-side RBAC:** Every route is gated except an explicit public allowlist, and permissions (`Role`, `Permission`, `UserRole`, `Membership`) are re-verified inside every `"use server"` action, never trusted to UI visibility.
- **Boot-time secret enforcement:** The app refuses to start without a real session secret rather than booting insecurely.
- **Intentional `ON DELETE RESTRICT`:** 15 of 22 company-owned tables use `RESTRICT` rather than `CASCADE` so an accidental tenant delete cannot silently wipe tasks, financial spend, or append-only `AuditEvent` rows.
- **Postgres `NULL`-distinct uniqueness handled explicitly:** Because `UserRole` is unique on `(userId, roleId, companyId, departmentId)` with a nullable `departmentId`, Postgres treats `NULL` as distinct in unique indexes; company-wide grants use transactional find-then-create rather than `upsert`.

---

## Edge Cases Handled

| Failure Mode | System Behaviour |
| :--- | :--- |
| **Engine unreachable** | Dashboard loads normally; chat falls back to direct provider calls without tools and names the degraded dependency. |
| **Stale session after removal from a company** | Session degrades cleanly to a company the user still holds; stale cookies cannot retain access. |
| **Unapplied optional migration** | Affected panel renders empty with status rather than 500-ing the entire route. |
| **Two workers, one queue** | Atomic claim-policy ensures each task is claimed once; saturated companies are excluded before claim. |
| **Memory/chat provider mismatch** | Status endpoint reports both effective providers side-by-side after a config mismatch caused silent `429` failures on memory writes. |
| **Failing or runaway tool loop** | Bounded round cap + tool result truncation prevent one large payload or broken tool from stranding a run. |
| **Knowledge archived** | Leaves retrieval and briefs immediately, stays restorable, and writes an audit event with actor and reason. |

---

## What AI Coding Tools Got Wrong — and How I Caught It

AI coding tools accelerated implementation, but every generated slice was treated as untrusted until verified against Postgres, real providers, and a rendered browser:

- **Memory silently wiped on every container restart.** Persistence was configured, not proved. Restarting the container and re-querying revealed the store adapter wiped its directory on init unless a specific flag was set.
- **Chat hung for 108 seconds.** Replies finished in ~3s, but the request awaited synchronous memory extraction before returning. Moved memory writes post-response.
- **Arrowhead rendered 350px from its target.** The agent graph looked plausible and every SVG `d` path was valid, but a 444-unit layout stretched across a 1036px canvas (2.33× distortion). Caught by measuring rendered bounding boxes in Playwright; anchors are now emitted on every card and asserted programmatically.
- **Build-manifest 500 from Prisma client leak.** Importing shared constants pulled `prisma` into a `"use client"` bundle. Split shared constants into `*-shared.ts` and enforced zero `prisma` imports in client modules.
- **Flaky runtime MCP binary installs.** Fetching MCP servers with `npx -y` at container boot intermittently half-installed and surfaced as "disconnected." Pinned all server binaries at image build time so broken images fail CI/build instead of shipping quietly.
- **Two hours lost to `127.0.0.1` vs `localhost`.** Next.js hydration silently fails on `127.0.0.1` in Playwright, masquerading as broken React components. Standardized all verification harnesses on `localhost`.

---

## Verification & Self-Checking Architecture Maps

**97 executable scripts** (`89 verify-*.ts` gates + `8 sweep-*.ts` invariant harnesses across viewports, themes, roles, empty states, and degraded dependencies) run against a real database, real engine, and real browser DOM—no mock layer.

Two additional generators keep documentation honest:
- `npx tsx scripts/map-system.ts --check` — exits `1` if routes, permissions, or models drift from `SYSTEM-MAP.md`.
- `npm run map:db` — regenerates the 39-model schema reference from `prisma/schema.prisma`.

```bash
$ npx tsc --noEmit
$ npx tsx scripts/verify-library.ts
  ✓ analytics: 4 agents (3 reserve), 7 department skills, 8 reachable
  ✓ operations: 4 agents (3 reserve), 9 department skills, 9 reachable
  (one line per department)
Library: 61 skills · 20 workstreams · 34 roles
All library checks passed.
```

> **Note:** The verification suites run on demand today; wiring them into GitHub Actions CI is the immediate next step.

---

## What Isn't Built (Published Honestly Inside the App)

The dashboard renders its own live gap list from a single source file (`src/lib/build-gaps.ts`) so nothing is quietly stubbed:

- **Telephony / SIP escalation:** LiveKit voice rooms work in-browser, but PSTN phone calls and SMS escalation are not wired (the voice escalation rung explicitly reports itself unbuilt).
- **Self-running autonomous loop:** Backlog queuing, cancellation, and resume work, but automatic test-signal chaining between finished tasks is scoped for Phase 2.
- **UI connection manager & live connectors:** MCP servers are read-only in the UI (added via engine registry); third-party Activepieces connectors await a client-provisioned instance, and outbound invite emails fall back to displaying a temporary password until a mail provider key is supplied.
- **Permanent knowledge deletion & scheduled backups:** Entries archive and restore cleanly; hard deletion and automated backup alerting are pending client product decisions.

---

## Tech Stack

| Layer | Technology |
| :--- | :--- |
| **Web app** | Next.js 16 (App Router), React 19 (Server Components & Server Actions), TypeScript (strict), Tailwind CSS v4 (`@theme` tokens), Zod v4, `iron-session`, custom SVG graphs |
| **Data & storage** | PostgreSQL 16, Prisma 7 (`@prisma/adapter-pg`, 39 models, 26 enums, 17 migrations), Valkey / Redis, MinIO |
| **Agent runtime** | Python 3.11, FastAPI, bounded OpenAI-style function calling, MCP (5 live stdio/HTTP servers, 67 tools), Mem0 + embedded Qdrant |
| **Voice** | LiveKit Agents (dedicated Python worker), Deepgram STT, ElevenLabs TTS |
| **Infrastructure** | 8-service Docker Compose stack behind Caddy (TLS, single published port), `tsx` + Playwright verification harnesses |

---

## Why There Are No Screenshots or Repo Link Here

The interface, product name, and source code belong to the client, so they stay private. What I can share publicly is the architecture, the security and data invariants, the real bugs caught against live systems, and the verification methodology. Happy to walk through the architecture and run the verification checks live on a call.
