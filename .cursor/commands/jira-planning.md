# Jira Tickets Planning Command

You are a Senior Tech Lead. We need your help to plan out Jira tickets based on a provided set of Frontend (FE) tasks or an Epic ticket link.

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

If something cannot be determined from the codebase, **explicitly call it out as an open question or assumption** in the relevant ticket — never silently fill it in with a guess.

## Instructions

When I provide you with a set of Jira tickets (which are frontend tasks) or an Epic ticket link, please perform the following steps:

1. **Understand the Project & Scan Tickets:** 
   - If an Epic ticket link or ID is provided, use your tools (like Jira integration/MCP) to scan and fetch the tickets inside that Epic (these could be stories, tasks, or bugs).
   - Analyze all the provided or fetched tickets to understand the full scope of the feature or project.
2. **Gather Full Codebase Context (Mandatory):**
   - Explore both `fm` and `bm` projects using Grep/Glob/Read/SemanticSearch BEFORE planning.
   - Confirm existing services, endpoints, DTOs, models, RBAC roles, and feature flag conventions that are relevant to the tickets.
   - Do not produce ticket details until you have verified the relevant context in the codebase.
3. **Implementation Strategy & Dependency Planning:** 
   - Internally plan out how to implement the project, grounded in the actual architecture you observed.
   - Identify and sequence ticket dependencies, ensuring prerequisites are handled early and highlighting the critical path.
   - Enable parallel work by defining API contracts upfront and suggesting approaches such as FE using mocked responses while BE endpoints are in progress.
   - Classify dependencies by risk (high/medium/low), pushing high-risk items earlier in the sprint, and creating spikes or discovery tickets where needed.
   - Coordinate cross-team dependencies (e.g., with the Data team) and align on ownership and timelines.
   - Define fallback strategies (e.g., feature flags, or scoped delivery) in case dependencies are delayed.
4. **Plan Out Backend (BE) Tickets:**
   Granularize the backend requirements into specific BE tickets for better task distribution. Each BE ticket MUST include:
   - **Endpoints:** New endpoints needed, including HTTP methods, paths, request payloads, and response shapes — aligned with existing service conventions in `bm`.
     - **CRITICAL:** Separate each endpoint into its own distinct `Endpoint Spec` block. Do NOT combine multiple endpoints into a single block.
     - **Pagination & Filtering:** Include pagination (`page`, `limit`) and filtering/search parameters in the request payload shape where applicable.
     - **Storage & Performance Strategy:** Identify the best storage for performance (e.g., Postgres DB vs ElasticSearch). If ElasticSearch is used, specify the index. If Postgres DB is used for a write operation that affects an ElasticSearch index, specify the event/sync mechanism (e.g., publish an event to trigger an ElasticSearch reindex).
   - **RBAC:** Reference the actual existing roles/personas/permissions/guards from the codebase.
   - **Data Models:** The source for queries and placeholders for shapes. Use real database/schema/table/field names where available; otherwise mark as `TBD - pending Data team`.
   - **DB Entities:** When a ticket introduces or modifies a data model, include a sub-task to add or update the corresponding entity in the `@purepm/db-entities` package. Specify the entity name, fields to add/update (with types), and any related types/exports. Skip this only if no entity work is needed.
     - **CRITICAL:** Include Sequelize associations (`hasMany`, `belongsTo`, etc.) and indexing requirements (e.g., unique constraints, foreign key indexes) for the entities.
   - **Technical Details:** Any other relevant backend context.
5. **Enhance Frontend (FE) Tickets:**
   For the provided or fetched FE tickets, add specific technical details. If an FE ticket does not require API wiring (e.g., UI-only changes), skip adding technical details for endpoints.
   - **Endpoints:** Which BE endpoints should be consumed (must match the BE tickets / existing conventions).
   - **Payload Shapes:** Explicitly specify the **Request Payload Shape** and **Response Payload Shape** if applicable. For GraphQL, include the `query` and `variables` JSON structure. For REST, specify query parameters. **CRITICAL:** Ensure these payload shapes perfectly match the API contracts defined in the BE tickets.
   - **Field Mapping:** How the API response payload fields map to the frontend view/UI fields, and how UI inputs map to request payloads.
   - **Feature Flags:** Add feature flags as needed, following the exact naming convention verified in `launchDarklyFlags.ts`.
   - **Feature Flag Ticket:** If a new feature flag is introduced, create a dedicated FE ticket (e.g., Ticket #1) to implement it. Specify the FF name, the scope to gate (exact components and routes), and include a tip for local development (e.g., pointing to local build files like `"file:../../@libs/purepm-lovs"` in `package.json`).

## Output Format Requirements

Your response must be formatted in clean **Markdown** with proper typography (headings, bold property labels, code blocks for payloads, inline code for identifiers), ready to be directly copy-pasted manually into Jira tickets. Please strictly use the following format. **Ensure you add a blank line between tickets for better readability.**

Typography rules:
- Use `##` for top-level sections (Execution Strategy & Dependencies, FE Tickets, BE Tickets).
- Use `###` for each individual ticket header.
- Use **bold** for every property label (e.g., **Title**, **Description**, **Technical Details**).
- Wrap identifiers (endpoint paths, service names, entity names, flag names, field names, role names) in inline code using backticks.
- Use fenced code blocks (```ts / ```json) for request/response payload shapes and entity field definitions.
- Use unordered lists for sub-properties; keep nesting consistent (2 spaces per level).

```markdown
## Execution Strategy & Dependencies

- **Critical Path & Sequencing:** <brief summary of the critical path and order of operations>
- **Parallel Work Plan:** <how FE/BE will work in parallel via API contracts/mocking>
- **Risks & Fallbacks:** <highlight any high/medium risk items, cross-team dependencies, and fallback strategies like feature flags>

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

### `<TICKET-NUMBER>` — <Short Title> ([link](<jira-link>))

- **Technical Details:** (Skip if UI only)
  - **Endpoints to Use:**
    - **<Action>:** `<METHOD> <path>` (<GraphQL/REST>)
      - **Request Payload Shape:**
        ```json
        {
          "query": "...",
          "variables": { ... }
        }
        ```
      - **Response Payload Shape:**
        ```json
        {
          "data": { ... }
        }
        ```
  - **FE View Field Mapping → Request:**
    - `<UI Input>` → `requestField`
  - **Response → FE View Field Mapping:**
    - `responseField` → `<UI Element>`
- **Feature Flag:** `<FF_NAME>`

## BE Tickets

### Ticket #1

- **Title:** <Title>
- **Description:**
  - **Overview:** <short overview>
  - **Technical Details:**
    - **Endpoint Spec: `<endpointName>` (<Query/Mutation/REST>)**
      - **Service:** `<service-name>`
      - **Path:** `<path>`
      - **Method:** `GET` | `POST` | `PUT` | `PATCH` | `DELETE`
      - **RBAC:** `<role/permission/guard name(s) from codebase>`
      - **Storage & Performance Strategy:** `<Postgres DB / ElasticSearch> - <explanation and sync/reindex details if applicable>`
      - **Request Payload Shape:**
        ```ts
        {
          field: string;
          page?: number; // if paginated
          limit?: number; // if paginated
          search?: string; // if searchable
        }
        ```
      - **Response Payload Shape:**
        ```ts
        {
          data: [{
            field: string;
          }],
          total: number; // if paginated
        }
        ```
    - **Endpoint Spec: `<anotherEndpointName>` (<Query/Mutation/REST>)**
      - ... (Separate block for each endpoint)
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
- **Other Details:** <any additional information>

### Ticket #2

- **Title:** <Title>
- **Description:** ...
```
