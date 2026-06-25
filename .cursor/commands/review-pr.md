# Review a GitHub PR or branch

Perform a rigorous code review of the changes described by a **GitHub pull request or branch link**—not whatever happens to be checked out locally. Act as a skeptical principal engineer reviewer, not a cheerleader.

## Step 0 — GitHub link (required) and resolve the local repo

**I must paste a GitHub URL in the same message** (or the command invocation must include it). Without a link, ask once for a PR or branch URL and stop—do not guess from the current branch or enumerate unrelated repos.

### What to accept

Parse one of these forms (query strings and `/files` suffixes are fine):

- **Pull request:** `https://github.com/<owner>/<repo>/pull/<n>` (or `.../pull/<n>/files`)
- **Compare:** `https://github.com/<owner>/<repo>/compare/<base>...<head>` (or `..` per GitHub’s UI)
- **Branch (tree):** `https://github.com/<owner>/<repo>/tree/<branch>`

Extract at least `owner`, `repo`, and either:

- **PR:** number `n`, or
- **Compare:** `base` and `head` ref names, or
- **Tree:** `head` branch; `base` will be the default branch (see Step 1).

Do not use my “current branch” or editor file to choose a different project. The URL is the source of truth for **which** `owner/repo` to review.

### Resolve `REPO` (local clone) without asking “which repo”

1. Enumerate git repository roots in the workspace (same approach as before: `git -C "<root>" rev-parse --show-toplevel` for each candidate root).

2. For each toplevel, read `git remote get-url origin` and normalize it to `owner/repo` (handle `https://github.com/o/r.git`, `git@github.com:o/r.git`, trailing `.git`, case).

3. If **exactly one** toplevel matches the URL’s `owner/repo`, set `REPO` to that absolute path.

4. If **none** match: do **not** list unrelated repositories or ask “which repo.” Respond that no local clone was found for `owner/repo`, and that I should open the folder that contains that clone (or add it to the workspace) and re-run. Stop.

5. If **multiple** toplevels match the same `owner/repo` (unusual): only then ask me to paste the absolute path to the intended clone. This is path disambiguation, not a multi-project menu.

### Confirm before diffing

One line, then continue to Step 1:

```
Reviewing: owner/repo  <PR#n or base...head>  local=$REPO
```

Use `git -C "$REPO" …` for **every** git invocation. Never `cd`. Never assume cwd.

## Step 1 — Establish scope

**Do not use the local codebase to get the diff.** Use the GitHub API or CLI (`gh`) to fetch the diff directly from GitHub.

**For a PR link** (`/pull/n`):

- Run `gh pr diff n --repo owner/repo` to list the patch for triage.
- **CRITICAL for line number accuracy:** Check out the PR locally (e.g., `gh pr checkout n` in `$REPO`) so that when you read files for context, the line numbers perfectly match the PR's `HEAD`.

**For compare or tree links** (no PR number):

- Use `gh api repos/<owner>/<repo>/compare/<base>...<head>` or equivalent commands to fetch the comparison diff from GitHub.
- Checkout the `head` branch locally (`git -C "$REPO" fetch origin <head> && git -C "$REPO" checkout <head>`).

Use the local codebase (`$REPO`) **only** to fetch related files and read surrounding context when the diff alone isn't enough. Do not use local `git diff` commands for the review itself.

If the diff is large (>1000 lines or >30 files), say so up front and propose a review strategy (review by subsystem, or ask which area to prioritize) before generating findings. Do not silently truncate.

## Step 2 — Read what you're reviewing

For every non-trivial changed file, open it in the local codebase (`$REPO`) and read enough surrounding context to understand the call sites, not just the diff hunks. **Only open files inside `$REPO`.** A change that looks fine in isolation can be wrong because of how its caller uses it. Specifically check for:

- Functions/types whose signatures changed — find every caller within this repo.
- New env vars, config, or feature flags — confirm they're wired in deploy configs and `.env.example` if applicable.
- New DB queries or indexes — read the surrounding model/repository layer.

If the change has obvious cross-repo implications (e.g. a BE API contract change with a FE consumer in a sibling repo), call that out as a **Question for the author** — do not silently cross the repo boundary.

## Step 3 — Review dimensions

Evaluate the diff against each of these. Skip dimensions that genuinely don't apply, but say so briefly so I know you considered them.

### Repository Conventions

Before evaluating the diff, dynamically find and read the `.cursor/rules` files in the resolved repository (e.g., by listing files in `$REPO/.cursor/rules/` and reading their contents).

1. Summarize these rules on the fly.
2. Include this summary as a rubric in your review.
3. Actively evaluate the PR/branch against these dynamically retrieved repository conventions and call out any violations.

### Correctness

- Off-by-one, null/undefined handling, unhandled promise rejections, swallowed errors.
- Concurrency: shared mutable state, race conditions, missing transactions, non-idempotent handlers.
- **Date & timezone handling**: business dates stored as noon UTC, displayed in the brand's local timezone. Flag any `new Date(string)` without explicit UTC parsing, any `.toISOString().split('T')[0]` patterns, and any date math that could cross a DST boundary.

### Security

- Input validation on all external boundaries (HTTP handlers, Pub/Sub message handlers, webhook receivers).
- SQL/NoSQL injection: parameterized queries only.
- Secrets: nothing hardcoded; reads from Secret Manager or env.
- AuthN/AuthZ: every new endpoint or RPC explicitly enforces RBAC. Multi-tenant boundary checks (tenant/brand/property scoping) on every query that touches tenant data.
- PII / financial data exposure in logs.

### Performance & data access

- N+1 queries — look for loops issuing DB calls.
- Missing indexes for new query shapes (esp. AlloyDB).
- Elasticsearch: mapping changes that require reindex, queries that bypass aliases, missing analyzers, fielddata risks.
- BigQuery: full table scans, missing partition/cluster predicates, costly JOINs.
- Pub/Sub: ack deadlines vs handler latency, lack of idempotency for at-least-once delivery.
- Cloud Run: handlers that block the event loop, cold-start-sensitive imports.

### Database & migrations

- Schema changes are backward-compatible for the duration of a rolling deploy (additive first, drop later).
- New columns nullable or have defaults.
- Migrations don't lock large tables.
- Replication slot impact considered for changes that bulk-update rows.

### Frontend (Vue/React/TS)

- State that should be derived isn't being stored.
- Effects with missing or over-broad dependencies.
- Accessibility: keyboard nav, focus management, alt text on meaningful images.
- Error/loading/empty states present for every async UI.
- Date display uses brand timezone, not user's local.

### Feature flags (LaunchDarkly)

- New flags use attribute-based targeting where possible (avoid list-based segments — they bloat MAU).
- Flag cleanup TODOs on temporary flags.
- Flag default value is the safe one.

### Observability

- Structured logs at the right boundaries (request in/out, external call in/out).
- Datadog tracing preserved across Pub/Sub boundaries.
- New error paths emit the right log level and include enough context to debug from the log alone.

### API & contract

- Backward-compatible response shapes (no removed fields, no changed types).
- New optional fields documented.
- Breaking changes versioned or coordinated with consumers.

### Style & maintainability

- Naming. Function/file size. Dead code. Commented-out code. `TODO` without a ticket reference.
- Don't nitpick formatter-owned things (Prettier/Black/gofmt). Skip them.

## Step 4 — Output format

Lead the report with the confirmation line from Step 0 (owner/repo, PR or ref range, `local=$REPO`). Then:

### **Summary** — 2–3 sentences: what this change does, and the single most important thing the author should fix first.

Generate a specific **Verdict** at the end of your summary based on the review results:

- If there are no issues/comments at all: "Verdict: LGTM"
- If there are non-blocking suggestions or nits but no blocking issues: "Verdict: LG. Approved with comments."
- If there are blocking issues: "Verdict: Changes requested. Check comments."

### **Number the review result items** sequentially across all categories below for identification.

### **Blocking issues** — anything that must be fixed before merge. Each entry:

<number>. File and line range

- **Issue:** What's wrong
- **Why it matters:** concrete failure mode, not "best practice says"
- **Suggested fix:** code snippet if small

### **Non-blocking suggestions** — same format, but stuff that would improve the change without gating it.

### **Nits** — formatting, naming, minor cleanup. Keep terse, one line each. Group if many.

### **Questions for the author** — things you genuinely can't tell from the diff (intent, missing context, cross-repo implications). Don't pad with rhetorical questions.

### **Things you checked and they look good** — short list. This is calibration, not flattery; it tells me what you actually verified vs. what you skipped.

## Step 5 — Verdict and Commenting to the PR

**IMPORTANT: Do not ask this question until AFTER you have printed the entire review report.**

At the end of your review, **ask me how I want to proceed using exactly the following question format and options**. (Output this as regular text. Do NOT use the `AskQuestion` tool, so that the review is fully visible to the user):

**How would you like to proceed with the PR review via `gh`?**
A. Perform suggested verdict
B. Comment all the issues only
C. Comment all issues and perform suggested verdict

If I choose an option that involves submitting the verdict and/or comments, use the `gh pr review` command to post the selected review items and/or the final verdict message.

- For **LGTM**: `gh pr review --approve -b "LGTM"`
- For **LG. Approved with comments.**: `gh pr review --approve -b "LG. Approved with comments."`
- For **Changes requested. Check comments.**: `gh pr review --request-changes -b "Changes requested. Check comments."`

When creating the line-by-line PR comment(s) alongside the review, use the following format for each item:

```markdown
### Issue:

<Explain the issue>

### Why it matters:

<Explanation>

### Suggested fix:

<Fix and explanation>
```

**Notes on comments:**

- Add proper formatting to elements like code.
- Add the comment to the correct lines the issue is referred to (e.g., using `gh pr review --comment -b "..." -f <file> -l <line>` or similar, or just clearly specifying the file/line if adding a general comment).
- Use GitHub format when suggesting code changes (e.g., ` ```suggestion ` blocks).

## Constraints

- **CRITICAL: Line Number Accuracy.** You MUST verify the exact line numbers by reading the checked-out file in `$REPO` (e.g., using file reading tools to view the specific lines). Do not guess or calculate line numbers purely from diff hunks (`@@ ... @@`), as LLMs frequently get this wrong. Ensure the code you are targeting exists exactly at the cited line number in the `HEAD` version before suggesting it.
- Cite file:line for every finding, with paths relative to `$REPO`. No vague "somewhere in the auth module" feedback.
- Severity matters. Do not list a missing JSDoc comment next to a SQL injection. If everything is "important," nothing is.
- If you're not sure something is a bug, say "possible issue" and explain what you'd need to verify it.
- Do not invent issues to fill out the report. A short, accurate review beats a long, padded one.
- Do not propose changes outside the diff scope unless they directly enable a fix.
- Never run a git command without `-C "$REPO"`. Never read or edit files outside `$REPO`.
- Do not run project test runners or test commands (`npm test`, `pytest`, `go test`, `cargo test`, etc.). CI runs the suite; this review is static analysis of the diff and code context only.
