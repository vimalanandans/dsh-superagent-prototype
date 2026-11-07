# 01 Module map and request trace

Source: Explore-agent analysis of `src/` (import graph computed by script, tests excluded; request path traced by reading code). Paths are repo-relative. Unverified items are marked.

**Key finding:** GitHub input is not a webhook. `src/sources/github.ts` polls the `gh` CLI every `pollSeconds` (60 to 3600, default 60). `src/app/http.ts` serves only the loopback `/orchestrator/api/*` panel routes.

## 1. Layers (all under `src/`)

| Layer | Modules | Role |
|---|---|---|
| 0 leaves | `limits`, `setup`, `navigation`, `admission`, `infra/text`, `infra/paths` | Pure values, error codes, admission table |
| 1 core | `core/{model,ports,reasons,errors,ids,stage,part,intake,evals,tickets,assignments,replies,engine,memory,index}` | Domain. Imports nothing outside `core/` (verified). Engine is the single entry point; five parts: Intake, Evals, Tickets, Assignments, Replies |
| 2 config/policy/env | `policy`, `deployment`, `kv`, `infra/{commands,environment,health}` | Settings schemas, gate, deployment resolution, CLI transport, worker environment |
| 3 sources | `sources/{kind,host,registry,poll,admission-context}`, `github*`, `feishu*` | Plugins. Nothing in `sources/` imports `src/app` (verified) |
| 4 executor | `executor/dsh.ts` (673 lines), `executor/background.ts` | `DshExecutor`, worker slots, launch |
| 5 app | `app/*` | Store, effects queue, outbox, passes (2 s pump), sessions, prompts, assessment, notices, tools, http/api/views. `app/runtime.ts` is the composition root (33 imports) |
| 6 client | `client/*.tsx` | Panel. Imports server code for types plus a few plain values |

## 2. Dependency diagram

```
index.ts ──► app/runtime.ts (composition root, ~33 imports)
                │
   ┌────────────┼──────────────────────────────────────────────┐
   ▼            ▼                                              ▼
app/http ─► app/api ─► app/views/*             app/tools/{main,executor,tool}
   │
   ▼
app/{settings,prompts,sessions,assessment,resources,notices,links,cards,context,outbox,passes,setup,source-context}
   │          │
   │          ▼
   │     executor/dsh ◄──value──► app/notices   (value-level 2-cycle)
   ▼
app/store ◄─► app/{effects,migration,versions,store-read}   (type-only back-edges)
   ├──► policy.ts ──► admission.ts ──► core/model
   ├──► kv.ts ──(type-only)──► app/prompts, app/store, policy, sources/github
   ▼
sources/{registry,kind,host,poll,github*,feishu*}   (no import of src/app)
   ▼
infra/{commands,environment,health,text,paths} ──► deployment.ts ──► setup.ts, limits.ts
   (infra/health ──► sources/host: infra reaches up into sources)

core/index.ts ◄── imported by 41 modules; core imports nothing outside core/
client/*.tsx ──► types (plus a few values) from app/api, sources/{kind,host}, core/*, admission, limits, setup, navigation
```

## 3. Hubs and cycles

Fan-in: `core/index` 41, `app/store` 23, `deployment` 18, `infra/commands` 18, `policy` 16, `sources/host` 16, `sources/registry` 14, `infra/text` 13, `kv` 13, `sources/kind` 13.

Fan-out: `app/runtime` 33, `app/http` 22, `app/resources` 16, `app/sessions` 13, `executor/dsh` 12, `app/assessment` 12, `sources/github` 12.

Cycles:
- **Value-level 2-cycle:** `app/notices` imports `triggerMessage` from `executor/dsh`; `executor/dsh` imports `resultRelay` from `app/notices`. Safe in ESM, but a layering tangle.
- **10-module value-level SCC:** `app/{assessment,cards,context,links,notices,prompts,resources}`, `app/tools/tool`, `app/views/records`, `executor/dsh`. Whether any edge is used at module-evaluation time is **not verified**.
- **35-module SCC** when type-only imports count: most of `app/`, `executor/dsh`, all `sources/*`, `kv`, `infra/health`.
- Layering oddities: `executor/dsh` sits above `app/` yet is imported back by `notices`, `links`, `cards`, `views/*`, `resources`. `kv.ts` points type-only at four higher layers. `sources/host` imports `kv`, `deployment`, `setup`, `registry`.

## 4. One request: `@agent please do X` on a GitHub Issue

**A. Ingress (polling)**
1. `sources/github.ts:112-121` `apply()` registers the kind and starts `watch()`.
2. `github.ts:280-294` `pollRound` runs `pollNotifications` (L298), `pollRepository`, `pollTicketPulls` (L404), `pollThreads` (L430). Notifications come from `gh api notifications` and honour `x-poll-interval`; `intakeComments` (L384) calls `ctx.receive` (L400).
3. `sources/github-trigger.ts:65-76` `commentTrigger`: id hashed from comment URL, id and version; `addressed: 'mention'`; thread `github:<number>`.

**B. Intake and admission**
4. `app/source-context.ts:46` to `runtime.ts:161` to `app/passes.ts:34-39` `receive` then `engine.receive`.
5. `core/intake.ts:19-62`: dedup (`'duplicate'`), scope filter via `SourceRegistry.scope`, store Trigger and Thread, fan out to subscribed Tickets, queue for Eval.
6. `passes.ts:63-68` `pumpOnce` (also every 2 s) then `engine.evaluateDue` then `Evals.start`.
7. Gate: `core/evals.ts:96` calls `policy.gate` (`policy.ts:191-211`). Allowed human plus mention gives `strength: 'explicit'`. Refusals: `not_a_member`, `no_mention`, `admission_ignored`, `not_human`. Machine triggers use `machineGate`. Eval created (`evals.ts:107-118`), `assessor.request` called.
8. `app/assessment.ts:104` enqueues a durable `assessment` effect; handler builds a candidate card and delivers it to the main Session (`sessions.prompt(..., 'steer')`), then `engine.candidateDelivered` starts the deadline clock. **Explicit requests still pass through the main Session LLM.**

**C. Decision and Ticket**
9. Main Session calls `orchestrator_route` (`app/tools/main.ts:127`, `route()` L88-104) for explicit requests, or `orchestrator_eval*` (`main.ts:203-220`) for inferred ones.
10. `core/evals.ts:131-156` `decide`, `accept` (L306-316): re-checks policy version, requires `admitsInferred` for non-explicit, builds Mandate and Decision.
11. `core/tickets.ts:37-56` `Tickets.create`: prepare (`dsh.ts:239`, `ticketExecution` L97-115), first Assignment, subscribe Threads, queue acknowledgement (`replies.ts:43-45`), enqueue `launch` effect (`dsh.ts:264-266`).

**D. Execution and prompts**
12. `executor/dsh.ts:232` handles `launch`: reserve a worker slot, `prepareDirectory`, `sessions.ensure(workerSessionId, ...)` (L441), send first message JSON (L455-466), `engine.assignmentStarted` (L468-469).
13. Prompts: `app/sessions.ts:244-249` registers system prompt section `dsh-superagent-policy`; `app/prompts.ts:102-119` composes. Order: `shared.md`, `orchestrator.md`/`worker.md`, `team/{communication,scope,delivery}.md`, `subscriptions.md`, `replies.md`, `dispatch.md`/`work.md`, `admission.md` (orchestrator only), `handoff*.md`, generated environment section, generated policy section (orchestrator only). Where `cli.md` is injected is **not traced**.
14. Worker calls `orchestrator_ack` (`app/tools/executor.ts:29-36`, idempotent) and `orchestrator_report` (L75-100, `core/assignments.ts:99-134`). `ready` ends the Ticket unless it follows a PR Thread; a later poll (`pollTicketPulls`) detects merge or close.

**E. Reply**
15. `orchestrator_reply` (`main.ts:224-238`) or Ticket end: `core/replies.ts:64-91` `answer` enforces readiness for `result`, resolves channel, calls `outbox.send`. PR end notices: `runtime.ts:145-152`, `sources/github-pull.ts:90`.
16. `app/outbox.ts:27-32` writes receipt plus `reply` effect atomically; `deliver` (L39-51) waits if Source off or an earlier message pending, then `sources.send` (`registry.ts:95-98`). Retry backoff 5 s doubling to 300 s (L24).
17. `sources/github-channel.ts:18-45` `sendGithub`: posts to `issues/<n>/comments` (or review-thread replies), skips if a comment with marker `<!-- dsh-superagent:<id> -->` exists (idempotency), posts via `gh api`. 410, or 403 on a locked Issue, returns `'gone'`.

```
GitHub ─poll─► github.ts ─Trigger─► Passes/Engine(Intake) ─► Evals+policy.gate
  ─► MasterAssessor(effect) ─card─► Main Session ─route/decide─► Tickets.create
  ─ack─► Outbox     ─launch effect─► Worker Session ─report─► Assignments
  ─► Replies.answer ─► Outbox(receipt+effect) ─► sendGithub ─POST comment─► GitHub
```

## 5. Docs vs code

Matches: core and sources isolation; runtime getters for executor and assessor; effects table limits; 2 s pump; Engine parts.

Gaps worth knowing:
1. No webhook anywhere in `src/`; docs consistently describe polling. `docs/008-cookbook.md` mentions `src/sources/webhook.ts`, a hypothetical Source that does not exist.
2. `docs/007` dependency rules do not mention the `notices`/`dsh` cycle, `infra/health` to `sources/host`, or `kv.ts` type edges.
3. The panel also imports type-only from `sources/kind.ts`, which the doc's allowed list omits.
4. Doc wording on "direct Decision for an explicit request" can read as explicit requests skipping the LLM; they do not (`directEval`, `evals.ts:271`, is only for Triggers no Eval holds).
5. `main.ts:97` hard-codes reason `explicitRouted`.

## Next steps

- Read `docs/007-architecture.md` against sections 1 and 3.
- Verify the 10-module SCC for evaluation-time use.
- Trace where `prompts/cli.md` is injected.
