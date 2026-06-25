# Analyze Jira ticket (fm + bm)

Pull a Jira ticket, skim `fm` and `bm`, and answer in **a very short summary** using **simple, non-technical language** (no jargon unless the ticket text requires it).

## Usage

```
/analyze-jira [-f] <JIRA-TICKET-ID-or-URL>
```

- `-f`: If provided, include suggested fixes in code snippets under the "Still to do" section.

## Execution steps

1. **Jira** — `user-mcp-atlassian` (e.g. `jira_get_issue`) for title, description, comments, and attachments as needed. Note **`issuetype.name`** (and any defect/feature labels) for the feature-flag decision below.
2. **Feature vs bug (feature flags)** — Same rules as `/plan-jira` §2b–2c:
   - **Bug-only** (e.g. issue type **Bug**, or explicit defect/regression with no new capability): **no** new LaunchDarkly flag; say so in the summary.
   - **Feature** work (Story, Epic, Improvement, New Feature, Task that adds new behavior/UI, etc.): **yes** — call out that a new flag belongs in `@purepm/purepm-lovs` (`bm/packages/@libs/purepm-lovs/src/global/launchDarklyFlags.ts`), with a **proposed name** matching existing patterns (e.g. product/UI: `PMS_<DOMAIN>_<AREA>_…_ENABLED`; backend-only: `SERVICE_<SERVICE>_<AREA>_ENABLED`). No version bumps or `file:` link instructions in this short summary unless the ticket is explicitly about release mechanics.
   - If unclear: state **assumption: no new flag** until confirmed (avoid extra `LaunchDarklyFlags` noise).
3. **Codebase** — Search `fm` and `bm` for existing work vs. ticket scope (including any flag already wired for the same area).
4. **Reply** — Output format below. **Under ~15 lines** (excluding code snippets) unless expanded detail explicitly requested.

## Output format (each part: 1–3 short phrases or sentences)

Use **plain paragraphs**, not a bullet list. Separate parts with blank lines. Each part starts with a **bold label** on its own line or inline, then the text on the same line or the next.

**Goal.** Plain-language outcome from the ticket.

**Ticket type (flags).** **Feature** or **Bug-only** (from Jira `issuetype` / labels); one short phrase.

**New LaunchDarkly flag.** **Feature:** proposed enum key + string (same style as neighbors in `launchDarklyFlags.ts`) and domain section if obvious; **Bug-only:** `None`; **Unclear:** `None (assumed until confirmed)`.

**Root cause.** (If applicable, e.g., for bugs) One short sentence explaining why the issue occurs.

**Already there.** One line for `fm`, one for `bm`; or `nothing relevant`.

**Still to do.** One line each for `fm` and `bm` (gaps only); mention flag wiring only when a new flag is indicated. If the `-f` flag was provided, include suggested fixes in code snippets to illustrate the required changes. Do not show code snippets that are identical to the existing codebase; only provide snippets that actually demonstrate the suggested new or modified code.

**Rough effort.** e.g. small / medium / large plus a short reason.

The entire answer should still fit **~15 short lines** total (excluding code snippets); combine or shorten parts when needed.

## Guidelines

- **Impersonal wording only** — No *I, we, you, he, she, they*; no possessives *our, your, their*; no people-oriented phrasing (*the user*, *someone*, *one*). Prefer labels, noun phrases, short fragments, or passive voice (e.g. “API endpoint missing”, “UI shows X”).
- **Feature flags** — Align with `/plan-jira`: new flag only for **feature** tickets; **bug-only** tickets omit flag work entirely. Do not invent parallel flag registries outside `purepm-lovs`. Full LD registration, wiring, default-off behavior, and local `file:` testing steps belong in `/plan-jira`, not in this terse summary unless the ask is expanded.
- Code changes: only when explicitly requested.
- No line-by-line plans, file lists, or long architecture write-ups unless expanded detail requested.
- Everyday words preferred over internal codenames and acronyms when possible.
