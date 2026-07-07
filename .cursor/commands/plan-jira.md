# Plan Jira Ticket Implementation

This command takes a Jira ticket, thoroughly scans the ticket description, attachments, and comments using the Atlassian MCP, and creates a comprehensive implementation plan.

## Usage

```
/plan-jira <JIRA-TICKET-ID-or-URL>
```

## Execution Steps

When this command is executed, follow these steps meticulously:

### 0. Setup and Mode Switch
- IMMEDIATELY call the `SwitchMode` tool with `target_mode_id: 'plan'` to switch to Plan mode.
- Ensure that you are using the **Gemini 3.1 Pro** model for this task. (If you cannot switch models yourself, politely ask the user to switch to Gemini 3.1 Pro before proceeding).

### 1. Information Gathering (MCP)
- Use the `user-mcp-atlassian` MCP server (e.g., `jira_get_issue`, `jira_get_comments`, `jira_get_attachments`) to pull all available information for the provided Jira ticket.
- **Critical:** Read the main description, ALL comments (to catch changing requirements or constraints), and review any attachments/images mentioned for UI/UX context.
- Ensure no detail is missed. If attachments contain mockups, explicitly note the required UI elements.

### 2. Codebase Context (Search)
- Search the `fm` (frontend) and `bm` (backend) repositories to understand the current state of the relevant systems.
- Identify existing components, API endpoints, database models, or utilities that need to be modified or can be reused.
- **Critical:** Identify and strictly adhere to existing architectural and coding patterns. Do not introduce novel solutions or anti-patterns that deviate from the established repository standards.

### 2b. Ticket type: feature vs bug (drives feature flags)
- From Jira (`issuetype.name` and any relevant labels), classify the ticket:
  - **Bug-only work:** issue type is **Bug** (or an equivalent defect type your org uses), or the ticket is explicitly a defect/regression fix with no new user-facing capability. **Do not** add or plan a new LaunchDarkly flag for these.
  - **Feature work:** issue types such as **Story**, **Epic**, **Task** (when delivering new capability), **Improvement**, **New Feature**, or any type that introduces new behavior/UI the business may want to gate. **Do** plan a new flag in `@purepm/purepm-lovs` ONLY if it involves frontend changes (see below). **Do not** add or plan a new LaunchDarkly flag for backend-only changes.
- If classification is unclear, state the assumption in **Open Questions** and default to **no new flag** until confirmed (avoids polluting `LaunchDarklyFlags`).

### 2c. Feature flags (`purepm-lovs`) — **frontend features only**
When the ticket is classified as **feature** work AND involves frontend changes:
- Add the flag in **`bm/packages/@libs/purepm-lovs/src/global/launchDarklyFlags.ts`** (exported via `src/global/index.ts`). Do not introduce parallel flag registries elsewhere for the same capability.
- **Naming:** Follow existing patterns in that file. Prefer:
  - **Product/UI:** `PMS_<DOMAIN>_<AREA>_<...>_ENABLED` (e.g. `PMS_LEASING_…`, `PMS_CORE_…`, `PMS_COMMS_HUB_…`, `PMS_REPORTING_…`). Place the enum member under the matching section comment (Global, Navigation, Hub-specific, etc.).
  - Enum **key** and **string value** should align with neighbors (usually the key and the LaunchDarkly key string are the same style, often identical).
- The plan must include: register flag in LaunchDarkly (or your team’s process), wire the app/service to read it, and default/off behavior when the flag is false.
- **Do not** bump `@purepm/purepm-lovs` versions across other frontend apps or monorepo packages as part of this command; the human will publish/bump consumers manually. The plan should only call out the lovs change and the app(s) that **need** the new enum at runtime.

When the ticket is **bug-only** or **backend-only:** omit feature-flag steps entirely (no new enum entry, no LD rollout section for a new key).

### 2d. Local testing with `file:` dependency on `purepm-lovs`
When the implementation touches `@purepm/purepm-lovs` and a frontend app under `fm` needs to test the change locally:
1. Build the library: from `bm/packages/@libs/purepm-lovs`, run `npm run build` (ensures `dist/` matches `src/`).
2. In **only** the frontend package(s) you are changing that depend on `@purepm/purepm-lovs`, temporarily set:
   ```json
   "@purepm/purepm-lovs": "file:<relative-path-from-that-package.json-to-bm/packages/@libs/purepm-lovs>"
   ```
   Compute the path from that app’s `package.json` directory — do not assume one size fits all. Example for `fm/packages/@apps/pms-spa` when `fm` and `bm` are sibling repos under the same parent directory:
   ```json
   "@purepm/purepm-lovs": "file:../../../../bm/packages/@libs/purepm-lovs"
   ```
   After editing, run `npm install` (or the repo’s package-manager equivalent) in that app so the link resolves.
3. **Do not** auto-update `@purepm/purepm-lovs` semver in other apps or shared packages; leave version bumps and broad consumer upgrades to the developer.

### 3. Implementation Planning
Create a highly detailed, step-by-step implementation plan. Break down the work into logical phases (e.g., Database, Backend, Frontend, Testing).
- Ensure the plan addresses **every** requirement and constraint found in the ticket description and comments.
- **Pattern Matching:** Ensure all planned code changes align perfectly with existing codebase patterns and explicitly avoid known anti-patterns.
- For **feature** tickets, include the `purepm-lovs` flag addition and wiring; for **bug** tickets, explicitly note that no new flag is introduced.
- Call out potential edge cases or risks.
- If something is ambiguous in the ticket, explicitly state assumptions or list questions that need to be asked.

## Output Format

Provide a comprehensive markdown response with the following sections:

### 🎫 Ticket Summary
- **Goal**: High-level summary of what needs to be achieved.
- **Ticket type (plan use):** **Feature** or **Bug-only** — one line stating how this was determined from Jira (`issuetype` / labels).
- **Key Requirements**: Bullet points of all explicit requirements from the description and comments.
- **Context/Constraints**: Any technical constraints, design considerations (from attachments), or business logic rules.
- **New LaunchDarkly flag (frontend features only):** If **Feature** AND involves frontend UI/behavior changes, the proposed enum key and string (e.g. `PMS_LEASING_FUNCTION_EXAMPLE_ENABLED = 'PMS_LEASING_FUNCTION_EXAMPLE_ENABLED'`) and which section of `launchDarklyFlags.ts` it belongs in. If **Bug-only** or **Backend-only**, state **None**.

### 🔍 Current State Analysis
- **Frontend (`fm`)**: What exists today and what needs to change.
- **Backend (`bm`)**: What exists today and what needs to change.

### 📋 Implementation Plan
Break down the implementation into clear, ordered steps. For each step, include:
- The specific files to be created or modified.
- The logic/changes required.
- Any new dependencies or configuration changes.

**Phase 0: Feature flag (`bm` — `@purepm/purepm-lovs`) — skip entirely for Bug-only or Backend-only tickets**
- [ ] **Frontend Feature tickets only:** Add enum entry to `bm/packages/@libs/purepm-lovs/src/global/launchDarklyFlags.ts` in the correct domain section; run `npm run build` in that package; use a `file:` dependency only in the `fm` app(s) actively being changed for local verification (correct relative path per app); do not bump lovs versions elsewhere.

**Phase 1: Database / Schema (if applicable)**
- [ ] Step 1...

**Phase 2: Backend API / Logic (`bm`)**
- [ ] Step 1...

**Phase 3: Frontend / UI (`fm`)**
- [ ] Step 1...

**Phase 4: Testing & QA**
- [ ] Step 1...

### ⚠️ Edge Cases & Considerations
- List any potential pitfalls, performance considerations, backward compatibility issues, or unhandled scenarios discovered during analysis.

### ❓ Open Questions
- If any requirements are contradictory or missing, list them here. Otherwise, state "None".

## Guidelines
- **Observe Code Patterns & Avoid Anti-Patterns:** Always mimic the existing codebase's architectural style, naming conventions, and design patterns. Do not introduce novel solutions or anti-patterns that deviate from established repository standards.
- **Thoroughness is paramount:** Do not hallucinate, but do not skip over minor details mentioned in comments.
- **Actionable steps:** The plan should be clear enough that an agent or developer can immediately start executing Phase 1 (or Phase 0 for feature-flag work when applicable).
- **Feature flags:** Prefer domain-accurate names and file placement consistent with `launchDarklyFlags.ts`; never add a flag for bug-only or backend-only tickets by default.
- **Code snippets:** Include brief code snippets or interface definitions in the plan if it helps clarify the required architecture or API contracts.