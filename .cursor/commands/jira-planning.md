# Jira Tickets Planning Command

You are a Senior Tech Lead. We need your help to plan out Jira tickets based on a provided set of Frontend (FE) tasks or an Epic ticket link. The output is consumed by **junior developers**, so every ticket must be self-contained, unambiguous, and actionable without tribal knowledge.

## CRITICAL: Accuracy Requirement

**Accuracy is non-negotiable.** Before producing any output, you MUST thoroughly explore and gather full context from the codebase. Speculation, assumptions, or fabricated details (hallucinated endpoints, services, models, flags, or roles) are **not acceptable**.

Mandatory codebase exploration checklist (use Grep/Glob/Read/SemanticSearch tools as needed):

- **Frontend (`fm`):** Inspect existing modules, components, stores, API clients/SDK calls, routing, and feature flag usage related to the tickets. Confirm where new UI/logic fits and what already exists.
- **Backend (`bm`):** Inspect existing services (`@services/*`), controllers/routes, DTOs, modules, guards/RBAC decorators, repositories/queries, and database models related to the tickets. Confirm naming, structure, and existing patterns before proposing new endpoints.
- **Feature Flags:** Read `bm/packages/@libs/purepm-lovs/src/global/launchDarklyFlags.ts` to determine the existing naming convention. New flags MUST match this convention exactly. Do not invent flag names without verifying the pattern.
- **RBAC / Roles / Permissions:** Search the codebase for the actual existing roles, personas, guards, and permission strings. Reference these by their real names. Do NOT guess.
- **Data Models:** Reference real database/schema/table/field names where they exist in the codebase. If the schema is not yet available (e.g. coming from the Data team), explicitly mark it as `TBD - pending Data team` rather than fabricating a shape.
- **DB Entities (`@purepm/db-entities`):** When a BE ticket introduces or modifies a data model, check the `@purepm/db-entities` package to see if a matching entity already exists. If not, plan to add a new entity; if it exists but needs new/changed fields, plan to update it. This must be reflected as a concrete sub-task in the BE ticket.
- **Endpoints:** Match HTTP verbs, base paths, and route prefixes to the conventions used by the relevant service. Do NOT make up paths.
- **Search / ElasticSearch:** If any read path is (or should be) served from ElasticSearch, inspect `bm/packages/@services/search-service` (GraphQL subgraph), the query builder `buildElasticSearchQuery` in `@purepm/nest-common`, the `Elastic.*` interfaces and the `ElasticSearchIndex` enum in `bm/packages/@libs/purepm-lovs/src/search/`, and the reindex/sync processors under `bm/packages/@cloud-functions/*-elasticsearch-processor`. Use the REAL index name (e.g. `pure_work_order`, `pure_lease`) — do NOT invent an index.

If something cannot be determined from the codebase, **explicitly call it out as an open question or assumption** in the relevant ticket — never silently fill it in with a guess.

## Honor the Repo's Own Coding Conventions

Every ticket's technical detail MUST conform to the conventions already enforced in the repos. Discover and apply the convention files that govern the directories the work touches — do NOT restate the full rule, reference it by path and bake its requirement into the ticket:

- **Cursor Rules:** Read the relevant `.cursor/rules/**/*.mdc` files (both always-applied and path-scoped) in `fm` and `bm`. Examples to honor: vendor-before-local import order, imported `data-test` constants (no inline strings), `PureError`/`PEC` error handling with the `catcher` abstraction, endpoint input validation, db-entity definition patterns, Sequelize transactions for multi-entity ops, PrimeVue/Vue conventions.
- **Bugbot Rules:** Include the repo-root `.cursor/BUGBOT.md` (uppercase, inside `.cursor/`) plus any nested `.cursor/BUGBOT.md` found while traversing upward from the directories the tickets touch. Treat each as review criteria the planned implementation must satisfy. As of now, `bm/.cursor/BUGBOT.md` exists (backend review guidelines); `fm` may not have one yet. The correct location is `.cursor/BUGBOT.md` — NOT a lowercase `bugbot.md`, NOT a repo-root `BUGBOT.md`, and NOT `.cursor/rules/`. If a `BUGBOT.md` does not exist for a touched area, skip it silently — do not fabricate one.
- **User/Global Standards:** Strictly no `any` type; add/update JSDoc on functions; no inline comments; remove unused imports/variables; deduplicate logic and extract helpers to reduce complexity; tests must reach **≥ 80% coverage**. Fold these into each ticket's Acceptance Criteria and Definition of Done.

## Jira Write-Channel Constraints (CRITICAL)

- The MCP description/comment channel accepts **Markdown only** and performs a **lossy** Markdown↔wiki conversion. It will:
  - flatten `{panel}`, `!image!`, and `[text|url|smart-link]` smart cards
  - escape/mangle `snake_case` and `*` inside inline code (e.g. `pure_application_person`)
  - collapse tables and drop code-fence language hints in comments
- **RULES:**
  - **Never** rewrite existing FE ticket content **above** `Technical Details` via MCP — edit that block in the Jira editor by hand. MCP may only touch summary + `Technical Details` and below.
  - Treat the **plan file (in-repo) as the canonical source**; Jira is a lossy mirror. Keep full contracts/diagrams in the plan, not the ticket body. Do not paste full TS into ticket bodies (Jira strips the `ts` fence and mangles types). Tickets reference `C1` and link to the plan.
  - After any MCP write, **re-fetch and eyeball the rendered output** before declaring done.
  - Use `h3.`/`###` for ticket section headings (h2 renders oversized in Jira).

## Instructions

When I provide you with a set of Jira tickets (which are frontend tasks) or an Epic ticket link, please perform the following steps:

1. **Step 0: Reconcile with live Jira before editing (recurring)**
   - Before writing, fetch current `assignee`, `status`, issue links, and scope for every epic child. Treat Jira as authoritative for:
     - **Assignee/Owner** — never assume; the staffing table must match live assignees.
     - **Status** — reflect actual workflow state, not "To Do" by default.
     - **Issue links** — the dependency map in the plan MUST equal the `Blocks`/`Relates` links in Jira. Remove links to cancelled tickets.
   - Surface any mismatch to the user instead of silently overwriting.
2. **Understand the Project & Scan Tickets:**
   - If an Epic ticket link or ID is provided, use your tools (like Jira integration/MCP) to scan and fetch the tickets inside that Epic.
   - Analyze all the provided or fetched tickets to understand the full scope of the feature or project.
3. **Gather Full Codebase Context (Mandatory):**
   - Explore both `fm` and `bm` projects using Grep/Glob/Read/SemanticSearch BEFORE planning.
   - Confirm existing services, endpoints, DTOs, models, RBAC roles, search/ES indexes, and feature flag conventions that are relevant to the tickets.
   - Discover and read the convention files that apply to the directories the work touches.
   - Do not produce ticket details until you have verified the relevant context in the codebase.
4. **Solidify API Contracts (Source of Truth):**
   - Define every API contract **once** in a dedicated `API Contracts` section, each with a stable **Contract ID** (e.g. `C1`, `C2`). FE and BE tickets MUST reference the contract by ID instead of re-declaring shapes.
   - Each contract must be complete and unambiguous.
   - Lock contracts EARLY so FE can build against them with mocked responses while BE is in flight.
5. **Implementation Strategy & Dependency Planning:**
   - Internally plan out how to implement the project, grounded in the actual architecture you observed.
   - Identify and sequence ticket dependencies, ensuring prerequisites are handled early and highlighting the critical path.
   - Enable parallel work by locking the API contracts upfront.
   - Coordinate cross-team dependencies (e.g., with the Data team) and align on ownership and timelines.
   - Define fallback strategies (e.g., feature flags, or scoped delivery) in case dependencies are delayed.
   - **Scope Changes & Cancellations:** When a ticket is descoped/cancelled: transition it to Canceled in Jira, remove its issue links, add an explicit "Out of Scope for V1" callout in Execution Strategy, and sweep ALL plan artifacts for references (including removing fields from API contracts, dependency nodes, staffing rows, and all `Blocks` links).
6. **Plan Out Backend (BE) Tickets:**
   Granularize the backend requirements into specific BE tickets for better task distribution. Each BE ticket MUST include:
   - **Endpoints:** New endpoints needed, including HTTP methods, paths, request payloads, and response shapes. Reference the locked **Contract ID**.
     - **CRITICAL:** Separate each endpoint into its own distinct `Endpoint Spec` block.
   - **ElasticSearch (when reads are served from ES):**
     - **CRITICAL — Define the query:** Specify the real `ElasticSearchIndex`, the `buildElasticSearchQuery` call, etc.
     - **CRITICAL — Flag BE work + create a ticket:** If the index/mapping/query does not yet support what the FE needs, create a dedicated BE ticket for the index/mapping/query change and its reindex.
     - **Write → ES sync:** Specify the sync mechanism and note the **eventual-consistency sync delay**.
   - **RBAC:** Reference the actual existing roles/personas/permissions/guards from the codebase.
   - **Data Models & DB Entities:** Define the source for queries and placeholders for shapes. Include sub-tasks to add or update corresponding entities in `@purepm/db-entities`.
7. **Plan Out Data (DATA) Tickets:**
   - Explicitly plan DATA tickets with a `[DATA]` summary prefix and a separate staffing row.
   - **Never** fold index/SQL work into FE or BE assignees.
8. **Enhance Frontend (FE) Tickets:**
   For the provided or fetched FE tickets, add specific technical details.
   - **Endpoints:** Which BE endpoints/contracts should be consumed (reference the **Contract ID**).
   - **Field Mapping:** How the API response payload fields map to the frontend view/UI fields.
   - **Optimistic Update:** If reading CRUD'd data from an ES-backed endpoint, specify exactly which store/cache is updated optimistically and the reconciliation trigger. **Note:** For read-only list screens (no inline edits), do not optimistically update; instead, refetch on return/navigation.
   - **Feature Flags:** Add feature flags as needed.
9. **Epic Implementer Summary (optional but recommended):**
   - Maintain a synced subset of the plan as a single epic comment.
   - Keep a **staging file in-repo** (`<epic-key>-comment-<id>.md`) generated from the synced sections; edit there, then publish via `jira_edit_comment`.
   - **Omit mermaid** (paste dependency bullets instead) and **keep tables minimal** — both degrade in the comment renderer.
   - Exclude the per-ticket specs and registry (too long / lossy); link to them.
10. **Requirement Traceability:**
    - Trace each net-new field/behavior to Figma / epic PRD, then assign ownership: FE = display + interaction, BE = persistence / business rules, DATA = index derivation / SQL.
11. **New-Ticket Creation Playbook:**
    - When adding epic children: mirror an existing sibling ticket, set project/issue type/priority/parent/team, use `[FE]`/`[BE]`/`[DATA]` prefix, create `Blocks` links, and update registry + dependency map + per-ticket spec in one pass.
12. **Corruption Recovery Playbook:**
    - If MCP damages a ticket: restore from Jira change history (`jira_batch_get_changelogs`) or manual editor paste — **not** another MCP rewrite of the damaged block.

## Output Format Requirements

Your response must be formatted in clean **Markdown** with proper typography. Please strictly use the following format. **Ensure you add a blank line between tickets for better readability.**

Every ticket MUST be comprehensive enough for a junior developer to start without follow-up questions. That means each ticket includes: a clear description, **Acceptance Criteria**, ordered **Implementation Steps**, **Files / Modules to Touch** (real paths), **Testing** requirements (≥ 80% coverage), **Dependencies** (Blocked by / Blocks), **Conventions to Follow**, an **Estimate**, and a **Definition of Done**.

Typography rules:
- Use `##` for top-level sections.
- Use `###` for each individual ticket header, using real Jira keys if they exist (e.g. `### PMHUB-XXXXX — <Title>`).
- Use `####` for ticket sub-sections (Goal, Done when, Files / Modules, etc.) instead of bold labels, as bold labels collapse into walls of text in Jira.
- Wrap identifiers in inline code using backticks.
- Use fenced code blocks (```ts / ```json) for payloads, but keep them in the plan, not in the Jira ticket body.
- Use unordered lists for sub-properties; keep nesting consistent.

```markdown
## Context Snapshot

- **What exists today:** <brief summary of current implementation>
- **Architecture Decision Rationale:** <e.g. why a new ES index vs extending an existing one>
- **Locked Business Rules:** <key rules that dictate behavior>
- **Code References:** <links to relevant files/enums>

## Execution Strategy & Dependencies

- **Critical Path & Sequencing:** <brief summary of the critical path and order of operations>
- **Parallel Work Plan:** <how FE/BE work in parallel via locked API contracts + FE mocking>
- **Risks & Fallbacks:** <high/medium risk items, cross-team dependencies, ES reindex/sync risks, and fallback strategies>
- **Out of Scope (V1):** <explicitly list any cancelled or deferred scope/tickets>
- **Ticket Dependency Map:** (Must round-trip to Jira `Blocks`/`Relates` links. `Blocks` = hard sequencing, `Relates` = paired work)
  - `PMHUB-XXXX1` → unblocks `PMHUB-XXXX2`, `PMHUB-XXXX3`
  - `PMHUB-XXXX4 (ES index/query)` → unblocks `PMHUB-XXXX5`

## Jira Ticket Registry

> Source of truth for key / alias / owner / status.

| Jira key | Plan alias | Owner | Status |
|----------|-----------|-------|--------|
| `PMHUB-XXXXX` | <alias> | <owner> | <status> |

## Staffing & Schedule

**Team:** <list team members and roles>
**Target:** <e.g., 2-week target>
**If we slip:** <fallback plan>

| Who | Week 1 | Week 2 |
|-----|--------|--------|
| **Name (Role)** | <tasks> | <tasks> |

## API Contracts (Source of Truth)

> Lock these first. FE and BE tickets reference these by **Contract ID**; do not redeclare shapes elsewhere. The canonical contract lives in the plan/repo only; do not paste full TS into Jira ticket bodies.

### `C1` — <Operation Name> (<GraphQL Query | GraphQL Mutation | REST>)

- **Owned by:** `PMHUB-XXXXX`  ·  **Consumed by:** `PMHUB-YYYYY`
- **Service / Path:** `<service-name>` · `<METHOD> <path>`
- **Request Shape:**
  ```ts
  {
    field: string;
    page?: number; // if paginated
    limit?: number; // if paginated
    search?: string; // if searchable
    sort?: { field: string; direction: 'asc' | 'desc' };
  }
  ```
- **Response Shape:**
  ```ts
  {
    data: Array<{ field: string }>;
    total: number; // if paginated
  }
  ```
- **Error Cases:** `<PEC code / status>` → `<meaning>`

## Open Items

- <Unresolved TBDs, e.g. source of a field, sign-off needed>

## FE Tickets

### `PMHUB-XXXXX` — Implement Feature Flag for <Feature> ([link](<jira-link>)) · <Owner>

#### Goal
<what the user sees / can do once this ships>

#### Technical Details
- **Feature Flag Name:** `<LaunchDarklyFlags.FLAG_NAME>`
- **Scope to Gate (Components/Routes):**
  - **UI Component:** `<path/to/component.vue>` - <how to gate>
  - **Route Guard:** `<path/to/routes.ts>` - <how to gate>
- **Tip:** For local development, point to local build files by pointing to `"file:../../@libs/purepm-lovs"` in `package.json`.

#### Done when
- [ ] <observable, testable outcome>
- [ ] Code merged, tests ≥ 80% coverage, lint/typecheck clean, flag defaults safe.

#### Files / Modules to Touch
- `<path/to/file>`

#### Testing
Unit/component tests for <cases>; ≥ 80% coverage.

#### Dependencies
Blocked by `<ticket/contract>`; Blocks `<ticket>`. Link existing tickets for behavioral AC (e.g. `PMHUB-XXXXX` Scenario 5) instead of re-writing full scenario prose.

#### Conventions
- `<.cursor/rules/...mdc>` - <one-line note on what to honor>

#### Estimate
`<S | M | L>` (`<points/days>`)

### `PMHUB-YYYYY` — <Short Title> ([link](<jira-link>)) · <Owner>

#### Goal
<what the user sees / can do once this ships>

#### Technical Details
- **Consumes Contract(s):** `C1` (`<METHOD> <path>`, <GraphQL/REST>)
- **FE View Field Mapping → Request:**
  - `<UI Input>` → `requestField`
- **Response → FE View Field Mapping:**
  - `responseField` → `<UI Element>`
- **Optimistic Update:** (only if reading CRUD'd data from an ES-backed endpoint)
  - **Why:** `<endpoint>` reads from ES index `<index>`, which lags the write by the reindex sync delay.
  - **Store/Cache to update:** `<pinia store / query cache>` - apply change immediately on mutation success.
  - **Reconcile:** refetch on `<trigger>`; **Rollback:** revert local change if the mutation errors.

#### Implementation Steps
1. <ordered, concrete step referencing real files/components>
2. ...

#### Done when
- [ ] AC met, tests ≥ 80%, lint/typecheck clean, no `any`, JSDoc added, behind flag if applicable.

#### Files / Modules to Touch
- `<fm/packages/@apps/.../Component.vue>`
- `<store / composable / dataTest.ts>`

#### Testing
Unit/component tests for <cases>; ≥ 80% coverage; add `data-test` constants for new interactive elements.

#### Dependencies
Blocked by `<ticket/contract>`; Blocks `<ticket>`. Link existing tickets for behavioral AC (e.g. `PMHUB-XXXXX` Scenario 5) instead of re-writing full scenario prose.

#### Conventions
- `<.cursor/rules/...mdc>` - <one-line note on what to honor>
- `<fm/.cursor/BUGBOT.md if present>` - <Bugbot criterion to satisfy>

#### Estimate
`<S | M | L>` (`<points/days>`)

## BE Tickets

### `PMHUB-ZZZZZ` — <Short Title> ([link](<jira-link>)) · <Owner>

#### Goal
<short overview>

#### Technical Details
- **Implements Contract(s):** `C1`
- **Endpoint Spec: `<endpointName>` (<Query/Mutation/REST>)**
  - **Service:** `<service-name>`
  - **Path:** `<path>`
  - **Method:** `GET` | `POST` | `PUT` | `PATCH` | `DELETE`
  - **RBAC:** `<role/permission/guard name(s) from codebase>` (+ tenant/brand/property scoping)
  - **Storage & Performance Strategy:** `<Postgres DB | ElasticSearch>` - <justification, N+1/index considerations>
  - **Request / Response:** See Contract `C1`.
- **ElasticSearch:** (only if this read is ES-backed)
  - **Index:** `<ElasticSearchIndex.X>` (e.g. `pure_work_order`)
  - **Query:** built via `buildElasticSearchQuery` (`@purepm/nest-common`) using `Elastic.<IQueryInterface>`
  - **New ES Work?:** `<None | New field/mapping/analyzer/index>` - if not None, **see dedicated BE ticket** `<id>` for the mapping/query change + reindex.
  - **Write → ES Sync:** `<Pub/Sub event → *-elasticsearch-processor reindex>`; note eventual-consistency delay (FE must optimistically update).
- **Data Models Mapping:** (request → db fields if mutation; response → db fields if query)
  - **Database:** `<database>` · **Schema:** `<schema>` · **Table:** `<table>`
  - `requestField` ↔ `column_name`
- **DB Entities (`@purepm/db-entities`):**
  - **Entity:** `<EntityName>` (Add | Update | N/A)
  - **Fields to Add/Update:**
    ```ts
    fieldName: string;
    ```
  - **Associations:** `<hasMany/belongsTo>` `<TargetEntity>` (foreignKey: `<fk>`)
  - **Indexes:** `<Index description>`

#### Implementation Steps
1. <ordered, concrete step referencing real services/files>
2. ...

#### Done when
- [ ] <observable, testable outcome>
- [ ] AC met, tests ≥ 80%, lint/typecheck clean, no `any`, JSDoc added, RBAC + validation enforced, backward-compatible.

#### Files / Modules to Touch
- `<bm/packages/@services/.../*.resolver.ts | *.service.ts>`
- `<bm/packages/@libs/db-entities/...>`

#### Testing
Unit tests for resolver/service + validation; ≥ 80% coverage.

#### Dependencies
Blocked by `<ticket/contract>`; Blocks `<ticket>`. Link existing tickets for behavioral AC (e.g. `PMHUB-XXXXX` Scenario 5) instead of re-writing full scenario prose.

#### Conventions
- `bm/.cursor/BUGBOT.md` - <e.g. PureError/PEC, RBAC + tenant scoping, no `any`, backward-compatible response>
- `<.cursor/rules/...mdc>` - <one-line note on what to honor>

#### Estimate
`<S | M | L>` (`<points/days>`)
