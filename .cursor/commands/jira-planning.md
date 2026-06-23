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

## Instructions

When I provide you with a set of Jira tickets (which are frontend tasks) or an Epic ticket link, please perform the following steps:

1. **Understand the Project & Scan Tickets:**
   - If an Epic ticket link or ID is provided, use your tools (like Jira integration/MCP) to scan and fetch the tickets inside that Epic (these could be stories, tasks, or bugs).
   - Analyze all the provided or fetched tickets to understand the full scope of the feature or project.
2. **Gather Full Codebase Context (Mandatory):**
   - Explore both `fm` and `bm` projects using Grep/Glob/Read/SemanticSearch BEFORE planning.
   - Confirm existing services, endpoints, DTOs, models, RBAC roles, search/ES indexes, and feature flag conventions that are relevant to the tickets.
   - Discover and read the convention files that apply to the directories the work touches: relevant `.cursor/rules/**/*.mdc` and any `.cursor/BUGBOT.md` (root + nested) in `fm` and `bm`. Capture the conventions they enforce so the planned implementation conforms and will pass Bugbot review.
   - Do not produce ticket details until you have verified the relevant context in the codebase.
3. **Solidify API Contracts (Source of Truth):**
   - Define every API contract **once** in a dedicated `API Contracts` section, each with a stable **Contract ID** (e.g. `C1`, `C2`). FE and BE tickets MUST reference the contract by ID instead of re-declaring shapes, so the contract cannot drift between the two sides.
   - Each contract must be complete and unambiguous: operation type (GraphQL Query/Mutation or REST), exact request shape (including pagination/filter/sort params), exact response shape, error cases, and the owning BE ticket.
   - Lock contracts EARLY so FE can build against them with mocked responses while BE is in flight. Any later change to a contract must bump/annotate the Contract ID and call out the affected FE and BE tickets.
4. **Implementation Strategy & Dependency Planning:**
   - Internally plan out how to implement the project, grounded in the actual architecture you observed.
   - Identify and sequence ticket dependencies, ensuring prerequisites are handled early and highlighting the critical path.
   - Enable parallel work by locking the API contracts upfront (see step 3) and suggesting approaches such as FE using mocked responses while BE endpoints are in progress.
   - Classify dependencies by risk (high/medium/low), pushing high-risk items earlier in the sprint, and creating spikes or discovery tickets where needed.
   - Coordinate cross-team dependencies (e.g., with the Data team) and align on ownership and timelines.
   - Define fallback strategies (e.g., feature flags, or scoped delivery) in case dependencies are delayed.
5. **Plan Out Backend (BE) Tickets:**
   Granularize the backend requirements into specific BE tickets for better task distribution. Each BE ticket MUST include:
   - **Endpoints:** New endpoints needed, including HTTP methods, paths, request payloads, and response shapes — aligned with existing service conventions in `bm`. Reference the locked **Contract ID** rather than duplicating shapes.
     - **CRITICAL:** Separate each endpoint into its own distinct `Endpoint Spec` block. Do NOT combine multiple endpoints into a single block.
     - **Pagination & Filtering:** Include pagination (`page`, `limit`) and filtering/search/sort parameters in the request payload shape where applicable.
     - **Storage & Performance Strategy:** Identify the best storage for performance (Postgres DB vs ElasticSearch) and justify it.
   - **ElasticSearch (when reads are served from ES):**
     - **CRITICAL — Define the query:** Specify the real `ElasticSearchIndex` (e.g. `pure_work_order`), the `buildElasticSearchQuery` call from `@purepm/nest-common`, the `Elastic.*` query interface, filters/sort/pagination, and any script fields. Do NOT hand-wave the query.
     - **CRITICAL — Flag BE work + create a ticket:** If the index/mapping/query does not yet support what the FE needs (new field, new filter, new analyzer, new index), this is **net-new BE work** — call it out explicitly and create a dedicated BE ticket for the index/mapping/query change and its reindex.
     - **Write → ES sync:** If a Postgres write must surface in an ES-backed read, specify the sync mechanism (Pub/Sub event → `*-elasticsearch-processor` reindex) and note the **eventual-consistency sync delay**. Any FE ticket that reads this data after a CRUD write MUST plan an optimistic update (see step 6).
   - **RBAC:** Reference the actual existing roles/personas/permissions/guards from the codebase. Per `bm/.cursor/BUGBOT.md`, RBAC and multi-tenant (tenant/brand/property) scoping MUST be enforced on every new endpoint.
   - **Data Models:** The source for queries and placeholders for shapes. Use real database/schema/table/field names where available; otherwise mark as `TBD - pending Data team`. New columns must be nullable or have defaults (backward-compatible).
   - **DB Entities:** When a ticket introduces or modifies a data model, include a sub-task to add or update the corresponding entity in the `@purepm/db-entities` package. Specify the entity name, fields to add/update (with types), and any related types/exports. Skip this only if no entity work is needed.
     - **CRITICAL:** Include Sequelize associations (`hasMany`, `belongsTo`, etc.) and indexing requirements (e.g., unique constraints, foreign key indexes) for the entities.
   - **Technical Details:** Any other relevant backend context.
6. **Enhance Frontend (FE) Tickets:**
   For the provided or fetched FE tickets, add specific technical details. If an FE ticket does not require API wiring (e.g., UI-only changes), skip the API-related properties.
   - **Endpoints:** Which BE endpoints/contracts should be consumed (reference the **Contract ID**; must match the BE tickets exactly).
   - **Payload Shapes:** Reference the locked Contract ID. Only restate a shape inline when it aids the implementer; if restated, it MUST match the contract verbatim.
   - **Field Mapping:** How the API response payload fields map to the frontend view/UI fields, and how UI inputs map to request payloads.
   - **Optimistic Update (when reading CRUD'd data from ES):** If the screen creates/updates/deletes data whose list/detail is read from an ES-backed endpoint, the ES read lags behind the write (reindex sync delay). The FE MUST apply an **optimistic update** to local state/store/cache immediately on success, then reconcile on the next server fetch, and **roll back** on mutation failure. Specify exactly which store/cache is updated and the reconciliation trigger.
   - **Feature Flags:** Add feature flags as needed, following the exact naming convention verified in `launchDarklyFlags.ts`.
   - **Feature Flag Ticket:** If a new feature flag is introduced, create a dedicated FE ticket (e.g., Ticket #1) to implement it. Specify the FF name, the scope to gate (exact components and routes), and include a tip for local development (e.g., pointing to local build files like `"file:../../@libs/purepm-lovs"` in `package.json`).

## Output Format Requirements

Your response must be formatted in clean **Markdown** with proper typography (headings, bold property labels, code blocks for payloads, inline code for identifiers), ready to be directly copy-pasted manually into Jira tickets. Please strictly use the following format. **Ensure you add a blank line between tickets for better readability.**

Every ticket MUST be comprehensive enough for a junior developer to start without follow-up questions. That means each ticket includes: a clear description, **Acceptance Criteria**, ordered **Implementation Steps**, **Files / Modules to Touch** (real paths), **Testing** requirements (≥ 80% coverage), **Dependencies** (Blocked by / Blocks), **Conventions to Follow**, an **Estimate**, and a **Definition of Done**.

Typography rules:
- Use `##` for top-level sections (Execution Strategy & Dependencies, API Contracts, FE Tickets, BE Tickets).
- Use `###` for each individual ticket header.
- Use **bold** for every property label (e.g., **Title**, **Description**, **Technical Details**).
- Wrap identifiers (endpoint paths, service names, entity names, flag names, field names, index names, role names) in inline code using backticks.
- Use fenced code blocks (```ts / ```json) for request/response payload shapes, ES queries, and entity field definitions.
- Use unordered lists for sub-properties; keep nesting consistent (2 spaces per level).

```markdown
## Execution Strategy & Dependencies

- **Critical Path & Sequencing:** <brief summary of the critical path and order of operations>
- **Parallel Work Plan:** <how FE/BE work in parallel via locked API contracts + FE mocking>
- **Risks & Fallbacks:** <high/medium risk items, cross-team (e.g. Data team) dependencies, ES reindex/sync risks, and fallback strategies like feature flags>
- **Ticket Dependency Map:**
  - `BE-1` → unblocks `FE-2`, `FE-3`
  - `BE-2 (ES index/query)` → unblocks `FE-4`

## API Contracts (Source of Truth)

> Lock these first. FE and BE tickets reference these by **Contract ID**; do not redeclare shapes elsewhere.

### `C1` — <Operation Name> (<GraphQL Query | GraphQL Mutation | REST>)

- **Owned by:** `<BE ticket id>`  ·  **Consumed by:** `<FE ticket id(s)>`
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

## FE Tickets

### Ticket #1 — Implement Feature Flag for <Feature>

- **Description:**
  - **Overview:** <short overview>
- **Technical Details:**
  - **Feature Flag Name:** `<LaunchDarklyFlags.FLAG_NAME>`
  - **Scope to Gate (Components/Routes):**
    - **UI Component:** `<path/to/component.vue>` - <how to gate>
    - **Route Guard:** `<path/to/routes.ts>` - <how to gate>
  - **Tip:** For local development, point to local build files by pointing to `"file:../../@libs/purepm-lovs"` in `package.json`.
- **Acceptance Criteria:**
  - [ ] <observable, testable outcome>
- **Definition of Done:** Code merged, tests ≥ 80% coverage, lint/typecheck clean, flag defaults safe.

### `<TICKET-NUMBER>` — <Short Title> ([link](<jira-link>))

- **Description:**
  - **Overview:** <what the user sees / can do once this ships>
- **Technical Details:** (Skip API properties if UI only)
  - **Consumes Contract(s):** `C1` (`<METHOD> <path>`, <GraphQL/REST>)
  - **FE View Field Mapping → Request:**
    - `<UI Input>` → `requestField`
  - **Response → FE View Field Mapping:**
    - `responseField` → `<UI Element>`
  - **Optimistic Update:** (only if reading CRUD'd data from an ES-backed endpoint)
    - **Why:** `<endpoint>` reads from ES index `<index>`, which lags the write by the reindex sync delay.
    - **Store/Cache to update:** `<pinia store / query cache>` - apply change immediately on mutation success.
    - **Reconcile:** refetch on `<trigger>`; **Rollback:** revert local change if the mutation errors.
- **Implementation Steps:**
  1. <ordered, concrete step referencing real files/components>
  2. ...
- **Files / Modules to Touch:**
  - `<fm/packages/@apps/.../Component.vue>`
  - `<store / composable / dataTest.ts>`
- **Testing:** Unit/component tests for <cases>; ≥ 80% coverage; add `data-test` constants for new interactive elements.
- **Dependencies:** Blocked by `<ticket/contract>`; Blocks `<ticket>`.
- **Feature Flag:** `<FF_NAME>`
- **Conventions to Follow:** (omit if none apply)
  - `<.cursor/rules/...mdc>` - <one-line note on what to honor>
  - `<fm/.cursor/BUGBOT.md if present>` - <Bugbot criterion to satisfy>
- **Estimate:** `<S | M | L>` (`<points/days>`)
- **Definition of Done:** AC met, tests ≥ 80%, lint/typecheck clean, no `any`, JSDoc added, behind flag if applicable.

## BE Tickets

### Ticket #1 — <Short Title>

- **Title:** <Title>
- **Description:**
  - **Overview:** <short overview>
- **Technical Details:**
  - **Implements Contract(s):** `C1`
  - **Endpoint Spec: `<endpointName>` (<Query/Mutation/REST>)**
    - **Service:** `<service-name>`
    - **Path:** `<path>`
    - **Method:** `GET` | `POST` | `PUT` | `PATCH` | `DELETE`
    - **RBAC:** `<role/permission/guard name(s) from codebase>` (+ tenant/brand/property scoping)
    - **Storage & Performance Strategy:** `<Postgres DB | ElasticSearch>` - <justification, N+1/index considerations>
    - **Request / Response:** See Contract `C1`.
  - **Endpoint Spec: `<anotherEndpointName>` (<Query/Mutation/REST>)**
    - ... (Separate block for each endpoint)
  - **ElasticSearch:** (only if this read is ES-backed)
    - **Index:** `<ElasticSearchIndex.X>` (e.g. `pure_work_order`)
    - **Query:** built via `buildElasticSearchQuery` (`@purepm/nest-common`) using `Elastic.<IQueryInterface>`
      ```ts
      buildElasticSearchQuery(paginatedQuery, {
        corporation_id: corporationId,
        brand_id_field: 'brand_id',
        default_sort: '<field>',
      });
      ```
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
    - **Associations:**
      - `<hasMany/belongsTo>` `<TargetEntity>` (foreignKey: `<fk>`)
    - **Indexes:**
      - `<Index description>`
- **Acceptance Criteria:**
  - [ ] <observable, testable outcome>
- **Implementation Steps:**
  1. <ordered, concrete step referencing real services/files>
  2. ...
- **Files / Modules to Touch:**
  - `<bm/packages/@services/.../*.resolver.ts | *.service.ts>`
  - `<bm/packages/@libs/db-entities/...>`
- **Testing:** Unit tests for resolver/service + validation; ≥ 80% coverage.
- **Dependencies:** Blocked by `<ticket/contract>`; Blocks `<ticket>`.
- **Conventions to Follow:** (omit if none apply)
  - `bm/.cursor/BUGBOT.md` - <e.g. PureError/PEC, RBAC + tenant scoping, no `any`, backward-compatible response>
  - `<.cursor/rules/...mdc>` - <one-line note on what to honor>
- **Estimate:** `<S | M | L>` (`<points/days>`)
- **Definition of Done:** AC met, tests ≥ 80%, lint/typecheck clean, no `any`, JSDoc added, RBAC + validation enforced, backward-compatible.

### Ticket #2 — <Short Title>

- **Title:** <Title>
- **Description:** ...
```
