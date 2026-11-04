# Multi-tenant AI operations dashboard

**Business goal:** run a team of AI agents from one screen. See what each department is
working on, what it has cost, and what is waiting for a human decision. Multi-tenant,
self-hosted, no SaaS dependency.

Client MVP. Phase 1 is delivered, deployed and accepted. The client's product name, the
interface and the repository stay private; this is the engineering account.

## Trigger

A person signs up and the company is provisioned: departments, one agent per department, a
skill library, channels and model connections. No setup commands. From there everything
starts from a queued task, a chat message, or an uploaded document.

## How it's put together

```
browser
   |
   v
app  (Next.js 16 App Router, React 19, TypeScript)
   |                              \
   | HTTP                          \ claims queued tasks
   v                                v
engine (Python, FastAPI)          worker (Node)
agents, MCP tools, retrieval,
voice. Never touches Postgres.
   |
   v
PostgreSQL (Prisma 7)
```

Two runtimes instead of one, because the agent layer needed Python and the product needed
React. It cost me time and it paid for itself twice: when the engine is down the dashboard
still loads and says so, and changing how an agent reasons never touched the UI.

## Main flows

1. **Ask the COO** — the question goes to the engine with the company's own knowledge in
   scope. If the answer involves work, it becomes a queued task.
2. **Queue and run** — the worker claims the task, runs it in bounded steps, and writes the
   result back with a status the board can render.
3. **Read the cost** — each finished task is priced from the tokens it actually used, at the
   rate for the model that served it. Department caps can refuse work.
4. **Upload knowledge** — a document is split, classified and made retrievable. Archived
   entries leave retrieval and the company brief, and can be restored.
5. **Approve** — anything above an agent's spending authority waits as a pending change
   until a human approves it. One transaction applies it.

## Rules the system enforces

- The company ID comes from the session. Never from the request body, query or header. A
  forged ID gets a 403, not rows.
- Permissions are checked again inside the server action, not just hidden in the UI. A
  checkbox the browser enforces is a suggestion.
- Every finished task is priced, and a run is checked against a cost estimate before it
  starts.
- Each company has a concurrency limit the worker reads at claim time, so a company at its
  cap is simply not picked next.
- The agent graph is a projection of stored rows. A dependency that isn't in the database
  isn't drawn.

## Edge cases handled

- **Engine unreachable** → the dashboard loads and the panel says unreachable. It does not
  blank the page.
- **Session points at a company the user no longer belongs to** → degrades to a company they
  actually hold. A stale session can't grant access.
- **Optional table missing** (an unapplied migration) → that one panel renders empty. The
  page doesn't 500 because a feature isn't deployed yet.
- **Two workers, one queue** → a task is claimed exactly once, and a company at its cap is
  excluded before the claim.
- **A model with no configured rate** → counted as unpriced, not as free. Understating spend
  would be worse than not knowing it.
- **A failing tool call** → bounded round cap plus a fallback to a direct provider call. One
  bad round doesn't strand the run.
- **Knowledge archived** → leaves retrieval immediately, stays restorable, and the change is
  audited with who and why.
- **A task with no department** → answers to no department cap. That one is a known gap, not
  a feature.

## What the AI got wrong, and how I caught it

This is the part I'd actually point at, because it's why the 85 verification suites
exist.

- **An arrowhead 350px from its target.** The graph looked almost right and every path
  string was correct. The layout was 444 units wide and the canvas was 1036px, so
  everything was silently stretched 2.33 times. I found it by measuring a path's bounding
  box against its own `d`. The graph now emits an anchor on every card, and the check
  asserts every wire reaches one.
- **A page that 500'd on a build manifest.** The database client had leaked into a client
  component. Pure constants moved to a shared file. `prisma` never appears in a
  `"use client"` file now.
- **A chat that hung for 108 seconds.** A memory write happened before the response was
  sent. It moved after.
- **Memory silently wiped on every container restart.** Persistence was configured but
  never proved, until it was restarted and re-queried.
- **Two hours lost to `127.0.0.1` instead of `localhost`.** Browser automation against
  `127.0.0.1` renders with no hydration and looks exactly like a broken component. The bug
  was in my test setup, not the product.

## Verification

85 suites that run against a real database, the real engine, or a real rendered DOM. No
mock layer anywhere, so a pass means the real thing works.

```
$ npx tsc --noEmit
$ npx tsx scripts/verify-library.ts
  ✓ analytics: 4 agents (3 reserve), 7 department skills, 8 reachable
  ✓ operations: 4 agents (3 reserve), 9 department skills, 9 reachable
  (one line per department)

Library: 61 skills · 20 workstreams · 34 roles
All library checks passed.
```

83 more suites live in `scripts/`. There is no CI yet: the suites run on demand, and
putting them in GitHub Actions is the next thing I would do.

## What isn't built

The product lists its own gaps on a page, generated from one source file so the list can't
drift. Seven items today, including rented GPU compute (the call path exists, the endpoint
is the client's to create), live automation connectors, permanent deletion in the knowledge
base (deliberately not built, because archive-or-delete is a product decision, not a code
one), and scheduled backups with an alert destination.

## Stack

Next.js 16 App Router and React 19 server components, TypeScript strict, Prisma 7 with the
pg adapter over Postgres (39 models, 26 enums, 16 migrations), Tailwind v4 driven by
`@theme` tokens, Zod at every request boundary. The engine is Python and FastAPI. Deployed
as an 8-service Docker Compose stack behind Caddy, with Postgres, Redis and MinIO
self-hosted and only the proxy publishing a host port.


## What's not here, and why

There are no screenshots of the product and no link to the code. The interface is the
client's, so the visuals, the product name and the repository stay private. What I can show
is the engineering: the architecture above, the rules and edge cases below, and the
verification. Happy to walk through the running system and the code live on a call.


