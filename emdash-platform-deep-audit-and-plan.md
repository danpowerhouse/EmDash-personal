# Emdash Platform — Deep Code Audit & Implementation Plan

**Reviewed artifact:** `emdash-platform-cloudflare-v5-enhanced.zip`  
**Review date:** 1 November 2026  
**Purpose:** Establish what is genuinely implemented, identify technical and operational gaps, and define a safe path to an intelligent operating system without replacing EmDash or adding a competing database.

---

## 1. Executive assessment

The project is a credible EmDash-on-Cloudflare foundation with a working narrow intake/triage implementation. It is **not yet an organisational intelligence system**. There is no service that reads the operational collections, assembles cross-record context, detects risks, prepares a daily brief, or proposes coordinated plans.

The next leap should be delivered as a sequence of testable platform capabilities:

1. Make the existing deployment reproducible and prove it against a staging environment.
2. Harden the data contract and query paths for operational intelligence.
3. Build a read-only Context and Relationship Engine over EmDash's existing records.
4. Add deterministic operational signals and risk detection.
5. Turn those signals into a Chief-of-Staff briefing and weekly review.
6. Improve intake review and add a properly gated draft-conversion workflow.
7. Only then consider bounded actions, external integrations and specialist agents.

**Do not start with autonomous agents, n8n, a vector database or a second source of truth.** Those would add moving parts before the application has dependable context, permissions, observability and tests.

## 2. What was inspected and validated

The enhanced ZIP was extracted and inspected file-by-file, including `package.json`, Astro and Worker configuration, all application source, the intake plugin, all intelligence modules, the seed schema, scripts, tests, CI workflows, automation policy and project documentation.

The following checks were run in the available Node 22.16.0 environment:

| Check | Result |
|---|---|
| `scripts/check-scaffold.mjs` | Pass |
| `scripts/check-automation-policy.mjs` | Pass |
| `scripts/check-intelligence.mjs` | Pass |
| `scripts/check-bootstrap.mjs` | Pass |
| JavaScript syntax checks for all `.mjs` scripts | Pass |
| TypeScript parse checks using Node type stripping | Pass |
| JSON parsing for package, TypeScript config, seed and policy files | Pass |
| ZIP integrity | Pass |
| `scripts/check-build-config.mjs` | Not passable yet: no `dist/server/wrangler.json` build output exists |

The archive has **no `package-lock.json` and no `node_modules`**. Therefore I could not verify dependency installation, the EmDash seed CLI, Astro typecheck, Astro production build, generated Worker configuration, or a live deployment. The CI workflows intentionally fail immediately when the lockfile is absent. The live MCP script was not run because a real endpoint and token were not supplied.

This is a source-level review plus the dependency-free checks above, not a production certification.

## 3. Current architecture — verified from source

### Platform

- Astro 7.3.2 and EmDash 1.4.0.
- Cloudflare Worker runtime using `@emdash-cms/cloudflare` 1.4.0.
- D1 database bound as `DB` and R2 bucket bound as `MEDIA`; the bootstrap script requires EU jurisdiction for both.
- EmDash live content collection registered in Astro.
- EmDash admin and MCP server enabled.
- Worker scheduled entry point delegates to EmDash's `createScheduledHandler()`.
- First-party Audit Log and Forms plugins are installed; Forms defaults to honeypot spam protection.
- Native `platform-intake` plugin is registered in `astro.config.mjs`.

The native plugin registration is important: the intake implementation is wired into the application. It is not just a file sitting unused in the repository.

### Data model

The seed defines ten non-routable collections:

- Structure: `workspaces`, `organisations`.
- Planning: `goals`.
- Operations: `projects`, `tasks`, `contacts`, `knowledge`, `decisions`, `resources`, `commitments`.

There are **21 reference fields** connecting these collections. The current schema includes useful operational concepts such as goal/project/task state, priority, due dates, next-action flag, decisions, commitments, contacts and knowledge verification dates.

The model is a useful relational foundation. It is not yet a knowledge-graph service: there is no code traversing relationships, resolving related records, building reverse links or answering cross-record questions.

### Intake and triage

`src/plugins/platform-intake/index.ts` provides MCP tools/routes for:

- `capture`: validate and persist raw text to plugin storage, then run triage;
- `list`: retrieve a bounded list of summaries and triage proposals;
- `get`: retrieve one item, including its original content.

The queue is stored in the plugin's own `items` collection, not in the ten operational content collections.

The triage pipeline is in `src/lib/intelligence/`:

- deterministic rules run first;
- four rule families classify explicit task, decision, idea and reference markers;
- ambiguous input can optionally use the Anthropic Messages API;
- LLM output has category, confidence and length validation;
- LLM results still require human review;
- provider failure returns a manual-review fallback;
- LLM is disabled by default in `wrangler.jsonc`.

This is a sound narrow classifier/capture pattern. It does not create tasks, update projects, link contacts, or execute suggested actions. Its `source` enum accepts `email`, `file` and `voice` labels, but there is no email ingestion, file upload/OCR or voice transcription implementation behind those labels.

## 4. Gaps and findings that materially affect the plan

### A. No read-only intelligence path into operational content

The intake plugin declares `network:request` but not EmDash's `content:read` capability. It is deliberately isolated from operational records at the application level. There is no `context`, `planning`, `signals`, `briefing` or `orchestration` module in the source tree.

This is the central missing capability. The current code can classify an incoming string, but cannot inspect the state of goals, projects, tasks, commitments, decisions or knowledge to determine what matters.

### B. The seed is not a production migration mechanism

The seed is configured through `package.json#emdash.seed`. EmDash documents the seed as initial setup/schema structure, not a file automatically reapplied to mutate an existing live database on every deploy. Future schema changes must use EmDash's supported schema-evolution/migration process; editing `seed/seed.json` alone is not a production migration.

### C. Query/index design needs work before operational analytics

The seed declares **zero `indexed: true` non-reference scalar fields**. It has 40 non-reference scalar fields of indexable types, but none are currently marked indexed. The 21 reference fields are relation-backed and cannot use ordinary scalar-field indexes. State, priority, due-date, next-action, target-date and verification-date queries are therefore not prepared for indexed filtering/sorting.

The 21 reference fields use `targetCollection` relations. EmDash's current field documentation says relation-backed reference links are stored separately and returned under `references`, not in the normal `data` field. A future context/graph adapter must read `references` and not assume relationship IDs are simple values in `data`. Relation-bound references cannot use the ordinary scalar-field index/filter path.

Recommended high-value scalar indexes to evaluate first:

- `tasks`: `state`, `priority`, `due_date`, `next_action`;
- `projects`: `state`, `priority`, `due_date`;
- `commitments`: `state`, `priority`, `due_date`;
- `decisions`: `state`, `decided_at`;
- `goals`: `state`, `target_date`;
- `organisations`: `relationship_state`;
- `knowledge`: `category`, `verified_at`.

Only add indexes justified by the actual query plan; verify the pinned EmDash 1.4.0 behaviour first. Do not mark every field indexed by default.

### D. Intake has no human-facing review workflow

There is no intake admin page or route for approving, correcting, dismissing, snoozing or converting an item into an operational record. `list` and `get` are the only queue-reading operations. There is no `reviewed_by`, `reviewed_at`, `converted_to`, correction feedback or explicit lifecycle state machine.

The stored record is persisted once with a pending-review fallback, then overwritten with the triage result. That preserves availability if the provider fails, but it is not an immutable audit history of each classification attempt. There is also no deduplication/idempotency contract or retention policy for the intake queue.

The `get` MCP output schema is `z.unknown()`, weaker than the explicit schemas used for capture and list. This should be tightened.

### E. Cron is wired, but business workflows are not

`src/worker.ts` exports EmDash's scheduled handler and `wrangler.jsonc` triggers it every minute. This is for EmDash scheduled publishing/maintenance/plugin scheduling. There are no application handlers for the policy's `morning-brief`, `weekly-review` or `exception-alert` workflows.

`automation/operating-policy.json` defines the operating rules and desired schedules, but `scripts/check-automation-policy.mjs` only checks selected policy invariants. It does not execute these workflows. Email provider is explicitly `not_selected`; no email delivery is configured.

The configured 08:15 Lisbon and Friday 16:00 Lisbon schedules will need timezone-aware scheduling and durable deduplication. Cloudflare cron expressions are UTC; a fixed UTC cron will drift by one hour when Lisbon changes between standard and daylight time.

### F. Health and CI do not prove platform health

`/health.json` always returns `{status:"ok"}` without checking D1, R2, EmDash migrations, auth, MCP or plugin storage. The six-hour uptime workflow checks only this static endpoint. It can be green while database or MCP operations are broken.

The deployment smoke test also tests only `/health.json`. It skips when no URL is configured, which is useful for the very first deployment but should not remain a silent steady-state condition.

`check-mcp.mjs` exists, but it is a manual read-only discovery script. It is not part of `npm run check` or CI, and it does not invoke the platform-intake tools or test their route-level permissions.

### G. Security is well considered in policy, but needs live evidence and runtime gates

Positive design choices:

- LLM fallback is off by default;
- content is persisted before model triage;
- model output is validated and never treated as an executed action;
- workspace/organisation fields are explicitly not represented as access-control boundaries;
- production secrets are injected separately from source control;
- Cloudflare provisioning refuses non-EU resource collisions;
- the audit plugin's limitations are documented;
- the policy forbids automatic permission escalation and high-impact autonomous action.

Remaining work:

- prove the Contributor + MCP scope combination against the deployed 1.4.0 runtime;
- prove list/get/capture route permissions separately;
- confirm the plugin tool surface is explicitly enabled in EmDash;
- add negative permission tests, not just positive tool discovery;
- define intake and model-provider retention/data classification rules;
- provide D1 and R2 backup/restore drills;
- introduce meaningful dependency health checks and operational alerts;
- protect production deploys with repository/environment controls.

The official EmDash MCP reference says plugin tools require both explicit plugin MCP scope and the route's permission, and that content-create requires the Contributor (level 20) role. That makes Contributor a sound candidate for the current narrow intake route, but the actual installed 1.4.0 deployment still needs an end-to-end test. A plugin token should not be given `content:write` merely to append to the plugin-owned intake queue.

Also note that `platform-intake` is a **native** plugin, which runs in the host Worker rather than in a sandbox boundary. Native plugins must be treated as trusted application code; a capability declaration is not equivalent to isolating arbitrary plugin code.

### H. Data model is useful, but operational semantics remain thin

The schema supports key records, but some future intelligence use cases need explicit semantics:

- task dependencies and blockers are not modelled;
- blocked projects have a state but no structured blocker reason or next review date;
- decisions link to projects but not directly to goals;
- commitments can link to a contact but also have a free-text counterparty, so duplicate identities are possible;
- there is no explicit stale/last-reviewed signal for projects and tasks beyond platform metadata;
- relation fields for workspace and organisation are context/grouping only, not security partitions.

Do not add every possible field now. Add only those required by the first live use cases, after inspecting real records and agreeing the meaning of each state.

## 5. Target architecture

Keep EmDash as the only source of truth. Add a small application-owned Intelligence Layer that consumes EmDash content and returns evidence-backed insights.

```text
ChatGPT / EmDash Admin
          |
          v
       MCP + Admin UI
          |
          v
  Intelligence Application Layer
   ├── Context Engine (read-only)
   ├── Relationship/Graph Projection
   ├── Operational Rules Engine
   ├── Briefing & Review Composer
   ├── Approval / Action Gate (later)
   └── Scheduler & Delivery Adapter (later)
          |
          v
     EmDash Content API
          |
          v
 D1 / R2 — EmDash remains system of record
```

Implementation boundaries:

- Prefer EmDash `ctx.content`/supported content interfaces over direct SQL or a duplicated data store.
- Use typed records generated from the actual EmDash schema where possible.
- Build a bounded, paginated context snapshot; do not load every record or every full text body for every question.
- Keep derived graph edges and risk findings explainable and reproducible. Start with in-memory projections; persist only operational state that needs durable deduplication or user feedback.
- Rules generate the facts/signals; an LLM may summarise those facts, but it must not invent dates, statuses, owners or relationships.
- Every recommendation must cite the source collection and record ID(s), show why it was raised, and expose uncertainty.
- Keep all writes behind an explicit approval service. No model call should hold a general write credential.

## 6. Detailed implementation roadmap

### Phase 0 — Reproducible build and production-readiness gate

**Why first:** Intelligence work on an unbuildable or unverified deployment creates avoidable risk.

Tasks:

1. Generate a genuine `package-lock.json` with registry access using the exact `package.json`; do not fabricate a lockfile.
2. Run `npm ci`, `npm run check`, `npm run typecheck`, `npm run build`, and `npm run check:build-config` in Node 24 as selected by `.nvmrc`.
3. Review `npm audit` output and decide on every high/critical finding rather than relying only on a non-blocking deployment audit.
4. Validate `seed/seed.json` with the installed EmDash CLI and record warnings. Confirm behavior against pinned EmDash 1.4.0.
5. Add a separate staging Worker, D1 database and R2 bucket. Never use production D1/R2 for preview/testing.
6. Run first-time setup on staging; test sign-in/passkey, admin, content creation, revisions, search, media upload/download, Forms and MCP.
7. Test `platform-intake` capture/list/get as a Contributor using only the intended scopes. Test denied calls using a Subscriber and a token missing the plugin MCP scope.
8. Add a release gate: successful dependency install, tests, typecheck, production build, seed validation and staging smoke tests before production deployment.
9. Test and document D1 restore and R2 recovery separately; preserve the encryption key outside the database.

**Acceptance criteria:** clean `npm ci`; all CI validations pass; a staging deployment is usable; a Contributor can use only the intended intake/read operations; negative permission tests fail as expected; backup restore steps have been exercised.

### Phase 1 — Operational data contract and query readiness

**Why:** The Context Engine needs consistent state values, efficient filters and a known interpretation of every relationship.

Tasks:

1. Generate EmDash collection types and document the actual runtime record shape, especially reference fields under `references`.
2. Add only the high-value indexes listed in Section 4.C, after confirming 1.4.0 support and performance impact.
3. Write a field dictionary for state, priority, due date, target date, next-action and commitment semantics. Define what `waiting`, `blocked`, `inbox`, `ready`, `done`, `fulfilled`, `achieved` and `superseded` mean operationally.
4. Decide a canonical rule for contact/organisation identity: linked contact versus free-text counterparty.
5. Identify missing semantics required by the first use cases (e.g. blocker reason, waiting-on, task dependency) but defer unrelated schema expansion.
6. Plan and test schema evolution on staging. Do not assume updating the seed changes already-deployed records/schema.

**Acceptance criteria:** the model has a reviewed field dictionary; the Context Engine can page through records and resolve references; due-date/state queries use supported indexes where needed; schema evolution succeeds on a staging copy without losing content.

### Phase 2 — Read-only Context Engine

**Purpose:** Give ChatGPT and future briefings a reliable view of the actual operating state.

Suggested modules:

- `src/lib/intelligence/context/types.ts`
- `src/lib/intelligence/context/repository.ts`
- `src/lib/intelligence/context/snapshot.ts`
- `src/lib/intelligence/context/relationships.ts`
- `src/lib/intelligence/context/scoping.ts`

Tasks:

1. Add a read-only EmDash-backed repository adapter using supported content access methods and pagination.
2. Build a `ContextSnapshot` with a consistent `asOf` timestamp and timezone; include active goals, active/blocked projects, actionable tasks, open commitments, proposed decisions and relevant knowledge/contacts.
3. Enforce explicit workspace/organisation filters in application logic when the operator asks for a particular context. Document that these filters are not security boundaries.
4. Build a relationship projection from EmDash's `references` data, with forward and reverse lookups and cycle-safe traversal.
5. Add bounded retrieval: maximum page sizes, maximum records per snapshot, date windows, text truncation and relevant-record selection.
6. Add MCP read tools such as `get_operating_context`, `get_project_context`, `get_due_items` and `get_related_records`. Keep these read-only.
7. Return evidence references for every material statement: collection, record ID, title, state/date and the relation used to reach it.
8. Add structured errors for missing, inaccessible, stale or incomplete data; never silently claim a complete overview when pagination or retrieval limits were hit.

**Acceptance criteria:** questions about a project return its project, linked tasks, goal, decisions and related commitments where present; all records are traceable to EmDash IDs; no write API is available from the Context Engine; pagination and scoping tests pass.

### Phase 3 — Deterministic Operational Signals

**Purpose:** Detect operational risks consistently before asking an LLM to summarise them.

Initial signal catalogue:

- overdue open commitments;
- critical commitments due within 24 hours;
- other commitments due within the configured lookahead;
- overdue or near-due high-priority tasks;
- blocked projects older than the configured threshold;
- active projects with no actionable next step;
- proposed decisions older than the review threshold;
- goals with no linked active projects;
- knowledge records whose verification date is stale (only if the user confirms this signal is useful).

Each signal should contain:

- stable rule ID and version;
- severity and trigger facts;
- source record references;
- observed time and threshold used;
- explanation and suggested next step;
- deduplication key and optional acknowledged/snoozed state.

Tasks:

1. Use the existing operating policy as configuration, but validate its complete structure with a schema rather than a few ad-hoc assertions.
2. Implement the first rules as pure functions over a ContextSnapshot.
3. Freeze time in tests; test due-date edges, empty/missing dates, completed/cancelled states, DST transitions and duplicated records.
4. Keep the no-composite-productivity-score rule.
5. Separate urgent interrupts from routine brief items and apply quiet hours to delivery, not to signal computation.
6. Add deduplication so the same unresolved issue does not generate a new alert every minute.

**Acceptance criteria:** the same snapshot and `asOf` produce the same signals; each signal names the source record and rule; false-positive edge cases are covered; no writes to business content occur.

### Phase 4 — Chief of Staff: daily brief and weekly review

**Purpose:** Turn verified context and deterministic signals into useful, low-noise guidance.

Tasks:

1. Compose a weekday brief: at most three priorities, up to three waiting items and up to three heads-up items, following the existing policy.
2. Compose a Friday review: flow changes, blocked/stale work, commitments, proposed decisions and no more than three recommendations.
3. Use an LLM only as a bounded summariser of structured facts. Require structured output with source IDs; reject claims that cannot be traced back to the snapshot.
4. First expose the brief through a protected MCP read tool and/or EmDash admin page. This allows validation before enabling notifications.
5. Add scheduled execution only after the generated brief is trusted. Handle Europe/Lisbon DST using timezone-aware evaluation and durable idempotency keys; do not assume Cloudflare Cron is Lisbon-local.
6. Keep the provider choice for email explicit. The current project has no email provider; do not send to a guessed/default provider. Add provider credentials through the platform's secure secret mechanism after selection.
7. Store a delivery record (brief ID, snapshot time, channel, status, retry count, dedupe key) without duplicating the full EmDash content database.

**Acceptance criteria:** an operator can inspect the brief and follow every claim to source records; repeated scheduler invocations cannot send duplicate briefs; the quiet-hours and urgent-interrupt policy is tested; failed delivery can retry safely; routine summaries do not interrupt during quiet hours.

### Phase 5 — Intake review and human-controlled conversion

**Purpose:** Make the existing intake queue a useful operational workflow.

Tasks:

1. Add an admin review page or equivalent first-party UI for the intake queue.
2. Add explicit lifecycle states such as `new`, `triaged`, `reviewed`, `promoted`, `dismissed` and `snoozed`, with actor and timestamps.
3. Tighten the `get` output schema; add input/output tests for every route.
4. Add idempotency/deduplication where the source provides a stable reference; otherwise surface possible duplicates without silently discarding user input.
5. Let the human correct classification, select a target collection, add workspace/organisation, resolve links and preview the exact record to be created.
6. Initially create only EmDash drafts after explicit user approval. Preserve the original intake record and link it to the created EmDash record.
7. Add retention and deletion/export rules for raw intake content. Define which sources may be sent to Anthropic and apply redaction/data policy before enabling LLM fallback.

**Acceptance criteria:** every intake item can be reviewed and corrected; conversion is idempotent; a human explicitly confirms the target record; the source item remains traceable; no model-generated classification alone creates operational content.

### Phase 6 — Approval and bounded action service

**Purpose:** Introduce controlled writes without giving agents unrestricted credentials.

Tasks:

1. Define an `ActionProposal` contract with target, operation, before/after preview, rationale, source evidence, impact, reversibility, expiry and idempotency key.
2. Keep proposal generation separate from execution.
3. Require authenticated operator approval bound to the exact proposal digest; edits invalidate the approval.
4. Re-read current records immediately before execution and use EmDash revision/optimistic-concurrency facilities to avoid overwriting intervening human changes.
5. Start with a tiny allowlist of reversible draft operations in specific collections. Keep publish, send, delete, permission/schema, financial and external-account changes human-gated every time.
6. Record actor, approver, proposal digest, affected IDs, prior/current revisions, result and failure in an audit trail.
7. Add explicit negative tests for denied action classes and attempts to alter the proposal after approval.

**Acceptance criteria:** no write is possible without the approval service; approval covers the exact current action; duplicate requests do not duplicate records; failures are recoverable; high-impact action classes remain impossible to execute autonomously.

### Phase 7 — Integrations and specialist agents (later)

Only after Phases 0–6 are stable:

- choose and implement one email provider;
- add file upload and extraction/OCR with file type/size controls and malware scanning;
- add voice transcription if there is a confirmed workflow;
- add inbound email only with sender verification and clear retention;
- add external project/calendar systems only where the user has a concrete need;
- introduce specialist agents as narrow, stateless planners over the same Context Engine, never as separate databases or privileged operators.

n8n remains optional. It should orchestrate external systems only if there is a demonstrated gap that EmDash/Cloudflare cannot handle cleanly.

## 7. Test strategy required for every phase

### Unit tests

- context assembly with partial/missing records;
- relationship traversal, cycles, missing target records and reverse links;
- rule thresholds, timezone conversions and state exclusions;
- deterministic brief composition and evidence completeness;
- approval digest/idempotency/concurrency behaviour.

### Plugin and contract tests

- capture/list/get success and validation failure;
- storage write-before-provider behaviour;
- provider timeout, HTTP error, malformed JSON, unexpected categories and oversized output;
- Contributor and Subscriber permission matrices;
- missing MCP plugin scope and disabled plugin tool surface;
- route input/output schemas and stable error contracts.

### Integration tests

- EmDash 1.4.0 with real D1-compatible test database;
- schema seed validation and fresh setup;
- schema evolution from prior schema version;
- media upload/retrieval through R2;
- actual MCP `tools/list` and each enabled plugin tool;
- scheduled worker idempotency under repeated events;
- notification delivery sandbox and retry.

### Operational checks

Replace the single static health check with distinct checks:

- liveness: Worker can respond;
- readiness: D1 connectivity and required migrations ready;
- integration: MCP endpoint responds and expected tool surface is enabled;
- storage: R2 configuration/availability verified without exposing object data;
- workflow: last successful brief/monitoring run and delivery status.

Do not put secrets, raw user content or sensitive record bodies in health output or routine logs.

## 8. Delivery order and gates

| Gate | Deliverable | Must be true before proceeding |
|---|---|---|
| G0 | Reproducible release | Lockfile, install, checks, typecheck and build pass |
| G1 | Staging proof | Auth, D1, R2, forms, audit and MCP verified; backups tested |
| G2 | Query-ready model | Field dictionary, indexes and reference contract validated |
| G3 | Context Engine | Evidence-backed, read-only cross-collection context works |
| G4 | Operational Signals | Deterministic rules and timezone/idempotency tests pass |
| G5 | Chief of Staff | Brief/review grounded in source records; delivery controlled |
| G6 | Intake workflow | Human review/conversion is usable and auditable |
| G7 | Bounded actions | Exact approval and execution gates proven with negative tests |
| G8 | Integrations/agents | Added only for agreed use cases after the core is reliable |

## 9. Decisions to keep explicit

1. **System of record:** EmDash only. No separate tasks/CRM/knowledge database.
2. **First intelligence version:** read-only context plus deterministic risk signals; LLM summarisation is optional and evidence-constrained.
3. **User interface:** preserve ChatGPT as the conversational interface; add only the small admin review interface needed for intake and approvals.
4. **Email:** provider is not selected; do not activate scheduled email until selected and tested.
5. **Data boundaries:** workspaces/organisations are filters and relationships, not security boundaries. Use separate deployments/databases if true separation is required.
6. **Autonomy:** no automatic permission escalation. No autonomous high-impact actions.
7. **n8n:** deferred unless a concrete integration need justifies it.

## 10. Immediate next implementation slice

The best first development increment after the release gate is:

**G2 + a minimal G3 vertical slice:** add the indexes needed for `tasks`, `projects` and `commitments`; implement a paginated read-only context repository; assemble one `get_operating_context` response containing active projects, the next actionable tasks, overdue/upcoming commitments, proposed decisions and their related goals/contacts; expose it through MCP; add fixture-based tests proving every statement has source IDs and no write operation exists.

That single slice proves the central architectural premise. Once it works against real EmDash records, the deterministic signals and Chief-of-Staff brief become straightforward extensions rather than speculative AI features.

---

## External EmDash references consulted

- [Native plugins and their in-process execution model](https://docs.emdashcms.com/plugins/creating-native-plugins/your-first-native-plugin/)
- [Plugin API routes and MCP tool exposure](https://docs.emdashcms.com/plugins/creating-plugins/api-routes/)
- [MCP scopes and role requirements](https://docs.emdashcms.com/reference/mcp-server/)
- [Seed files, indexing and relation-backed reference fields](https://docs.emdashcms.com/themes/seed-files/)
- [Field types and reference field storage](https://docs.emdashcms.com/reference/field-types/)
- [Cloudflare Worker scheduler and deployment](https://docs.emdashcms.com/deployment/cloudflare/)
