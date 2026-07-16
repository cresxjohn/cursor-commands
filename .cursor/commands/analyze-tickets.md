# Analyze Multiple Jira Tickets (Orchestrator)

Accepts a list of Jira tickets (from text or an image), analyzes each one in parallel using the `/analyze-jira` logic, comments the results on each Jira ticket, and provides a final summary.

## Usage

```
/analyze-tickets <list-of-JIRA-TICKET-IDs-or-URLs-or-image>
```

## Execution steps

1. **Extract Tickets:** Parse the user's prompt (or analyze the provided image) to extract all Jira ticket IDs.
2. **Parallel Analysis & Commenting:** For **each** ticket, execute the following steps in parallel by launching background subagents. **Strictly use the `cursor-grok-4.5-high` ("Cursor Grok 4.5 High") model** for these parallel executions:
   - **Analyze:** Perform the exact same analysis steps as defined in the `/analyze-jira` command (evaluate feature/bug, check LaunchDarkly flags, search `fm` and `bm` codebases, estimate SP).
   - **Format Output:** Generate the exact same short summary output format as `/analyze-jira`.
   - **Comment:** Use the `user-mcp-atlassian` MCP server (specifically the `jira_add_comment` tool) to post the generated `/analyze-jira` output as a comment directly to the Jira ticket.
3. **Final Summary:** Wait for all parallel ticket analyses and commenting actions to complete, then output a consolidated summary to the user.

## Output format

Do not output the full analysis for each ticket in the chat (since it is already posted to Jira). Instead, provide a consolidated summary table.

**Format:**
Provide a Markdown table with the following columns:
- **Ticket:** Jira Ticket ID.
- **Goal:** Plain-language outcome from the ticket (1 sentence max).
- **Type:** Feature or Bug-only.
- **SP:** AI Suggested SP.
- **Comment Status:** Success / Failed.

After the table, provide the **Total SP** by summing up the story points of all tickets.

Example:
| Ticket | Goal | Type | SP | Comment Status |
|---|---|---|---|---|
| PROJ-123 | Add new payment gateway | Feature | 5 | ✅ Success |
| PROJ-124 | Fix typo in settings page | Bug-only | 1 | ✅ Success |

**Total SP:** 6

## Guidelines

- **Impersonal wording only** — Follow all impersonal wording guidelines from `/analyze-jira` when generating the comment payload.
- **Efficiency** — Maximize parallel tool calls for codebase searches and MCP Jira API calls across the different tickets to reduce total execution time.
