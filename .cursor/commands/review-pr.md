# Review a GitHub PR or branch

Perform a rigorous code review of the changes described by a **GitHub pull request or branch link**—not whatever happens to be checked out locally. Act as a skeptical principal engineer reviewer, not a cheerleader.

**Hybrid context model & Execution Strategy:**

- **PR scope (diff, metadata, file contents, code search):** GitHub only via `gh` or the GitHub API. Do not use local `git diff`, `git checkout`, or the checked-out branch to determine what changed.
- **Repository conventions (Cursor rules, Bugbot):** Read from the matching **local workspace clone** without changing its branch or working tree. Do not fetch `.cursor/rules` from GitHub unless no local clone exists.
- **Maximize Parallelism:** Group independent tool calls. Once the PR metadata and file list are known (Step 1), fetch GitHub file contents (Step 2a), local Cursor rules (Step 2b), and Jira ticket details (Step 2c) **concurrently in a single parallel tool call batch** to speed up the review.
- **Jira requirements:** When a ticket ID is in the PR title/body, fetch description, **all comments**, and **attachments** via Atlassian MCP — requirements often change in comments or mockups.

## Step 0 — GitHub link (required)

**I must paste a GitHub URL in the same message** (or the command invocation must include it). Without a link, ask once for a PR or branch URL and stop—do not guess from the current branch or workspace.

### What to accept

Parse one of these forms (query strings and `/files` suffixes are fine):

- **Pull request:** `https://github.com/<owner>/<repo>/pull/<n>` (or `.../pull/<n>/files`)
- **Compare:** `https://github.com/<owner>/<repo>/compare/<base>...<head>` (or `..` per GitHub’s UI)
- **Branch (tree):** `https://github.com/<owner>/<repo>/tree/<branch>`

Extract at least `owner`, `repo`, and either:

- **PR:** number `n`, or
- **Compare:** `base` and `head` ref names, or
- **Tree:** `head` branch; resolve `base` as the repo default branch (Step 1).

The URL is the sole source of truth for **which** repository and revision range to review.

### Review artifact id (for concurrent reviews)

Every file written during a review MUST be namespaced so parallel reviews do not clobber each other:

| Review type | `REVIEW_ARTIFACT_ID` |
|-------------|----------------------|
| Pull request | `pr-<PR_NUMBER>` (e.g. `pr-1234`) |
| Compare / tree (no PR #) | `pr-<owner>-<repo>-<head-sha-first-7>` (e.g. `pr-acme-web-a1b2c3d`) |

Use this prefix on **all** review-produced files: `review.json`, drafts, logs, exports, etc.

- **Filename pattern:** `<REVIEW_ARTIFACT_ID>-<purpose>.<ext>` — e.g. `pr-1234-review.json`, `pr-1234-report.md`
- **Location:** Write under `.cursor/reviews/` in the **cursor** workspace root (create the directory if needed). Never write review artifacts inside `LOCAL_REPO_ROOT` or other application repos — reviews may leave files in the clone and must not be disturbed by git operations.
- **Cleanup:** Delete `<REVIEW_ARTIFACT_ID>-review.json` after a successful GitHub post unless I ask to keep it. Other artifacts may be kept until I delete them.

### Confirm before diffing

One line, then continue to Step 1:

```
Reviewing: owner/repo  <PR#n or base...head>  ref=<head-sha-or-ref>  artifact=<REVIEW_ARTIFACT_ID>  jira=<TICKET-ID|none>  rules=<local|github|none>
```

Every `gh` invocation must include `--repo owner/repo` when the subcommand supports it.

If `gh` is missing or unauthenticated, stop and tell me to run `gh auth login` (or install `gh`). Do not fall back to a local clone for the diff.

## Step 1 — Establish scope via GitHub

Fetch metadata and the patch from GitHub only. **Run these initial `gh` commands concurrently where possible.**

### Resolve refs and SHAs

**PR** (`/pull/n`):

```bash
gh pr view <n> --repo owner/repo --json number,title,body,baseRefName,headRefName,headRefOid,files,commits
gh pr diff <n> --repo owner/repo
```

Store `HEAD_SHA` = `headRefOid`, `HEAD_REF` = `headRefName`, `BASE_REF` = `baseRefName`, `PR_NUMBER` = `number` (when present).
**Look for a Jira ticket ID (e.g., `XXX-1234`) in the PR `title` or `body`.** Store as `JIRA_KEY` when found.

**Compare or tree** (no PR number):

```bash
gh repo view owner/repo --json defaultBranchRef
gh api repos/owner/repo/compare/base...head
```

For a tree link, set `head` from the URL and `base` from `defaultBranchRef`. Store `HEAD_SHA` from the compare response (`commits[-1].sha` or `merge_base_commit` as appropriate). Set `PR_NUMBER` = empty; derive an artifact id in Step 0 (below).

### Diff triage

Use `gh pr diff` or the compare API patch. For file lists without full patches:

```bash
gh api repos/owner/repo/pulls/<n>/files --paginate
# or compare endpoint: .files[] with filename, patch, additions, deletions
```

If the diff is large (>1000 lines or >30 files), say so up front and propose a review strategy (review by subsystem, or ask which area to prioritize) before generating findings. Do not silently truncate.

## Step 2 — Gather deep context (Run 2a, 2b, 2c in parallel)

Once the diff and file list are known, gather all necessary context **concurrently in a single tool call batch**.

### 2a. Read context via GitHub

For every non-trivial changed file, pull **full file contents at `HEAD_SHA`** (and at `BASE_REF` when before/after comparison matters)—not just diff hunks. **Execute these fetches in parallel.**

#### Fetch a file at a ref

```bash
gh api repos/owner/repo/contents/<path> --field ref=<HEAD_SHA-or-ref> -q .content | base64 -d
```

For directory listings (e.g. to discover related modules):

```bash
gh api repos/owner/repo/contents/<dir>?ref=<ref>
```

Use `-q` / `--jq` to extract fields; decode base64 content in the shell when posting inline comments is not needed.

#### When the diff alone is not enough

- **Signature changes:** search for callers in the same repo:

  ```bash
  gh api "/search/code?q=<symbol>+repo:owner/repo" --paginate
  ```

  Then fetch each hit file at `HEAD_SHA` for surrounding context.

- **New env vars / config / feature flags:** fetch likely config paths at `HEAD_SHA` (`.env.example`, deploy manifests, infra dirs) via contents API or code search.

- **New DB queries:** fetch the model/repository files referenced in imports at `HEAD_SHA`.

If the change has obvious cross-repo implications (e.g. a BE API contract change with a FE consumer elsewhere), call that out as a **Question for the author**—do not silently assume access to sibling repos; use code search scoped to those repos only if URLs are provided.

### 2b. Resolve local clone and load conventions

Before evaluating the diff, load coding conventions from the **local workspace**, not GitHub.

#### Map `owner/repo` → local root

1. Enumerate workspace roots (multi-root workspaces may include `fm`, `bm`, etc.).
2. For each root, resolve the GitHub remote and match `owner/repo`:

   ```bash
   git -C <workspace-root> remote get-url origin
   ```

   Accept `git@github.com:owner/repo.git`, `https://github.com/owner/repo`, and `.git` suffix variants.

3. The first matching root is `LOCAL_REPO_ROOT`. If none match, note “no local clone for owner/repo” and fall back to GitHub (below).

#### Read Cursor rules locally (read-only)

**Do not** checkout, pull, stash, reset, or otherwise change branch or working tree in `LOCAL_REPO_ROOT`. Reviews may produce files in the clone; concurrent reviews and in-progress work must not be disturbed.

Load convention files read-only, scoped to the changed paths in the PR diff:

1. **Preferred:** Glob/Read from disk at the clone's **current** checked-out state — do not use `gh api` for rules when a local clone exists.

2. **Optional freshness (still no checkout):** After `git -C <LOCAL_REPO_ROOT> fetch origin` (fetch only), read specific rule files from the remote default branch:

   ```bash
   git -C <LOCAL_REPO_ROOT> show origin/<defaultBranch>:.cursor/rules/<file>.mdc
   ```

   Use `defaultBranch` from `gh repo view owner/repo --json defaultBranchRef -q .defaultBranchRef.name`. Prefer disk when sufficient; use `git show` only when local rules look stale vs `origin/<defaultBranch>`.

Note in the review rubric which branch/state rules were read from (e.g. `rules=local (current branch: feature/foo)` or `rules=local (origin/main via git show)`).

**Which rules to load:**

| Rule kind | How to select |
|-----------|---------------|
| Always-applied | Every `.mdc` under `.cursor/rules/` whose frontmatter has no `globs` (or empty `globs`) |
| Path-scoped | Every `.mdc` whose `globs` match at least one changed file path in the diff |
| Bugbot | `.cursor/BUGBOT.md` at repo root; also any nested `.cursor/BUGBOT.md` found by walking upward from each changed file's directory (stop at repo root) |

If the PR touches paths in multiple repos (e.g. coordinated FE/BE PRs), repeat for each matched local root.

**Fallback when no local clone:**

```bash
gh api repos/owner/repo/contents/.cursor/rules?ref=<HEAD_SHA>
# then fetch each rule file via contents API
```

If neither local rules nor GitHub rules exist, note “no `.cursor/rules` found” and proceed with general standards only.

#### Apply conventions in the review

1. Summarize the loaded rules on the fly (always-applied + path-scoped for this diff).
2. Include this summary as a rubric in the review under **Repository conventions**.
3. Actively evaluate the PR/branch against them and call out violations — cite the rule file path (e.g. `.cursor/rules/services/error-handling.mdc`).

### 2c. Fetch Business Requirements (Jira)

If `JIRA_KEY` was found in the PR title or body, fetch the **full ticket context** via `user-mcp-atlassian`. Run these **in parallel** with 2a/2b when possible:

1. **`jira_get_issue`** — issue summary, description, type, labels, and comments:
   - Set `comment_limit` to **100** (read **all** comments — requirements and constraints often appear there, not in the description).
   - Use `expand=renderedFields` when available for readable acceptance criteria.
   - Note `issuetype.name` (Bug vs Story/Feature) — affects how strictly to gate on acceptance criteria vs. implementation gaps.

2. **`jira_get_issue_images`** — inline image attachments (mockups, screenshots, Figma exports). Compare UI in the PR against what the ticket shows.

3. **`jira_download_attachments`** — non-image attachments (specs, CSVs, logs, PDFs). Skim for requirements, edge cases, or repro steps not in the description.

**What to extract:**

| Source | Focus |
|--------|--------|
| Description | Acceptance criteria, steps to reproduce (bugs), scope |
| Comments | Scope changes, clarifications, QA notes, "actually we need…" — treat newer comments as potentially overriding older text |
| Images | Expected UI layout, labels, empty/error states |
| Other attachments | Data samples, repro payloads, written specs |

If no Jira ticket is found, note `jira=none` and evaluate based on the PR description alone.

If Jira MCP is unavailable or the ticket cannot be fetched, note the failure and proceed from the PR body only — do not invent ticket requirements.

## Step 3 — Review dimensions

Evaluate the diff against each of these. Skip dimensions that genuinely don't apply, but say so briefly so I know you considered them. Be rigorous: brainstorm edge cases, missing error handling, and unhandled states.

### Requirements & Acceptance Criteria

- Cross-check the diff against the Jira ticket — **description, comments, and attachments** (Step 2c), not the description alone.
- Does the code fully implement the requested feature or fix the bug as described?
- Flag requirements stated only in **comments** or **attachments** that the PR missed.
- Brainstorm and verify edge cases: What happens if inputs are empty, zero, null, or unusually large? Are there edge cases implied by the ticket (including comment thread or mockups) that the PR missed?

### Repository Conventions

Use the rubric built in Step 2b. Do not re-fetch rules from GitHub when a local clone was resolved.

### Correctness & Edge Cases

- Off-by-one, null/undefined handling, unhandled promise rejections, swallowed errors.
- **Edge cases:** Unhappy paths (API failures, network timeouts), empty arrays/lists, unexpected falsy values, malformed data structures.
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
- **Edge cases:** Error states, loading spinners, empty states, and partial data states are present for every async UI.
- Accessibility: keyboard nav, focus management, alt text on meaningful images.
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

Lead the report with the confirmation line from Step 0 (`owner/repo`, PR or ref range, `ref=<head-sha-or-ref>`). Then:

### **Summary** — 2–3 sentences: what this change does, and the single most important thing the author should fix first.

Generate a specific **Verdict** at the end of your summary based on the review results (for the **chat report**; when posting to GitHub use the shorter phrase from Step 5 only):

- If there are no issues/comments at all: "Verdict: LGTM" → GitHub: `LGTM`
- If there are non-blocking suggestions or nits but no blocking issues: "Verdict: LG. Approved with comments." → GitHub: `LG. Approved with comments.`
- If there are blocking issues: "Verdict: Changes requested." → GitHub: `Changes requested.`

### **Number the review result items** sequentially across all categories below for identification.

### **Repository conventions** — rubric from Step 2b: which rule files were loaded (always-applied + path-scoped), source (`local` vs `github` fallback), and a one-line summary per rule. Violations belong in Blocking/Non-blocking below with the rule path cited.

### **Blocking issues** — anything that must be fixed before merge. Each entry:

<number>. File and line range

- **Issue:** What's wrong
- **Why it matters:** concrete failure mode, not "best practice says"
- **Suggestion:** code snippet if small — in the review report use a fenced block; when **posting to GitHub**, wrap code fixes in a ` ```suggestion ` block per Step 5. The chat report is full detail; the GitHub top-level review body is **verdict text only** (Step 5).

### **Non-blocking suggestions** — same format, but stuff that would improve the change without gating it.

### **Nits** — formatting, naming, minor cleanup. Keep terse, one line each. Group if many.

### **Questions for the author** — things you genuinely can't tell from the diff (intent, missing context, cross-repo implications). Don't pad with rhetorical questions.

### **Things you checked and they look good** — short list. Include which convention files were applied, whether Jira comments/attachments were reviewed, and whether they came from the local clone. This is calibration, not flattery; it tells me what you actually verified vs. what you skipped.

## Step 5 — Verdict and Commenting to the PR

**IMPORTANT: Do not ask this question until AFTER you have printed the entire review report.**

At the end of your review, **ask me how I want to proceed using exactly the following question format and options**. (Output this as regular text. Do NOT use the `AskQuestion` tool, so that the review is fully visible to the user):

**How would you like to proceed with the PR review via `gh`?**
A. Perform suggested verdict
B. Comment all the issues only
C. Comment all issues and perform suggested verdict

### Model for applying verdict (options A, B, C only)

When I choose **A**, **B**, or **C**, switch to **Composer 2.5** before running any `gh` commands or posting `<REVIEW_ARTIFACT_ID>-review.json`. The review report itself may be produced on any model; only the **apply/post** step requires Composer 2.5.

If you cannot switch models yourself, ask me to switch to **Composer 2.5** and wait for confirmation before submitting the verdict or inline comments.

If I choose an option that involves submitting the verdict and/or comments, use `gh pr review` with `--repo owner/repo`. Inline comments require a PR number; for compare/tree-only reviews, explain that posting review comments needs an open PR.

### Top-level review body — verdict text only

The **full review report** (summary, rubric, blocking issues, nits, etc.) is printed in chat only. When posting to GitHub, the **top-level review `body` must be the verdict phrase alone** — short, personal, no headings, no summary, no "See inline comments", no nits, no questions.

| Verdict | GitHub `-b` / `body` value (exact) |
|---------|-----------------------------------|
| LGTM | `LGTM` |
| LG with comments | `LG. Approved with comments.` |
| Changes requested | `Changes requested.` |

**Do not** post to GitHub:

- `**Summary**` / `**Verdict:**` headings or markdown section headers
- PR description-style summaries of what the change does
- Pointers like "See inline comments for details"
- Nits, questions, or convention rubrics in the top-level body

Put **all** substantive feedback in **inline comments** on the relevant lines. The top-level comment should read like a quick human sign-off.

**Option A** (verdict only, no inline comments): `gh pr review` with `-b` set to the verdict phrase above.

**Option B** (inline comments only, no approve/request-changes): submit via the pulls review API with `"event": "COMMENT"` and `"body": ""` (empty) or omit a meaningful body — never duplicate the chat report in `body`.

**Option C** (verdict + inline comments): one review via the pulls review API — `"body"` is **only** the verdict phrase; all issues live in `"comments"`.

```bash
# Option A examples
gh pr review <n> --repo owner/repo --approve -b "LGTM"
gh pr review <n> --repo owner/repo --approve -b "LG. Approved with comments."
gh pr review <n> --repo owner/repo --request-changes -b "Changes requested."
```

When creating line-by-line PR comment(s), use this format for each item:

```markdown
### Issue:

<Explain the issue>

### Why it matters:

<Explanation>

### Suggestion:

<Short prose explanation, then a GitHub suggestion block — see below>
```

### GitHub suggestion blocks (required for code fixes)

When the fix is a **code change**, use GitHub's native **suggestion** fence so the author gets an **Apply suggestion** button on the inline comment. Do not use plain ` ```ts ` / ` ```javascript ` fences for replaceable code.

**Format:**

````markdown
### Suggestion:

<Brief explanation of the change.>

```suggestion
<exact replacement lines — what the file should contain after the fix>
```
````

**Rules:**

| Rule | Detail |
|------|--------|
| Fence label | Exactly ` ```suggestion ` — no language tag (`ts`, `vue`, etc.) |
| Content | Replacement text only, not a unified diff (`-`/`+` lines) |
| Scope | Lines in the block replace the line(s) the inline comment is attached to |
| Single-line fix | Comment on that line; suggestion block contains the one replacement line |
| Multi-line fix | Use GitHub's multi-line comment API: set `start_line` (first line) and `line` (last line) on the same `side: "RIGHT"` hunk |
| Context | Include unchanged surrounding lines inside the suggestion **only** when replacing a contiguous range (GitHub replaces the whole commented range) |
| Non-code fixes | Prose-only under **Suggestion:** (no suggestion block) — e.g. config steps, rename across repo, add a test file |
| JSON body | In `<REVIEW_ARTIFACT_ID>-review.json`, escape newlines as `\n`; keep the suggestion fence intact inside the string |

**Single-line example** (inline comment on line 42):

````markdown
### Issue:

Missing null guard before accessing `lease.tenant`.

### Why it matters:

Throws at runtime when `tenant` is absent on draft leases.

### Suggestion:

Return early when tenant is missing.

```suggestion
if (!lease.tenant) return null;
```
````

**Multi-line example** (`<REVIEW_ARTIFACT_ID>-review.json` comment object):

```json
{
  "path": "src/handler.ts",
  "start_line": 10,
  "line": 14,
  "side": "RIGHT",
  "body": "### Issue:\n...\n\n### Why it matters:\n...\n\n### Suggestion:\nReplace the try/catch swallow with explicit error mapping.\n\n```suggestion\n  try {\n    return await service.fetch(id);\n  } catch (err) {\n    throw mapServiceError(err);\n  }\n```"
}
```

**Do not:**

- Use ` ```typescript ` or other language fences for apply-able fixes
- Put suggestion blocks in the top-level review body only (they must be on **inline** comments to enable Apply)
- Suggest changes spanning non-contiguous lines in one block (split into separate inline comments)

**Notes on comments:**

- Add proper formatting to elements like code (inline `` `backticks` `` for identifiers).
- Post inline comments via the pulls review API (see below).
- Every blocking/non-blocking item that includes a code fix should use a ` ```suggestion ` block when posted to GitHub.

**Posting inline comments via GitHub API:**

1. Resolve `HEAD_SHA` again if needed: `gh pr view <n> --repo owner/repo --json headRefOid -q .headRefOid`
2. **Verify line numbers:** fetch the file at `HEAD_SHA` via the contents API, decode, and confirm the cited line matches the code you reference. Do not infer line numbers from diff hunk headers alone.
3. Build `<REVIEW_ARTIFACT_ID>-review.json` under `.cursor/reviews/` (see Step 0):

```json
{
  "commit_id": "<HEAD_SHA>",
  "body": "LG. Approved with comments.",
  "event": "APPROVE",
  "comments": [
    {
      "path": "path/to/file.ts",
      "line": 42,
      "side": "RIGHT",
      "body": "### Issue:\n...\n\n### Why it matters:\n...\n\n### Suggestion:\n```suggestion\n...\n```"
    }
  ]
}
```

`event` is `APPROVE`, `REQUEST_CHANGES`, or `COMMENT`.

4. Submit:

```bash
gh api --method POST -H "Accept: application/vnd.github+json" \
  /repos/owner/repo/pulls/<n>/reviews --input .cursor/reviews/<REVIEW_ARTIFACT_ID>-review.json
```

Delete `.cursor/reviews/<REVIEW_ARTIFACT_ID>-review.json` after a successful post unless I ask to keep it.

## Constraints

- **GitHub for PR scope; local for conventions.** Use `gh` / GitHub API for diffs, changed-file contents at `HEAD_SHA`, and in-repo code search. Use the local workspace filesystem **only** for read-only access to `.cursor/rules/**/*.mdc`, `.cursor/BUGBOT.md`, and resolving `owner/repo` → workspace root. **Never** checkout, pull, or reset in application repos. Never use local `git diff` or checkout to determine what changed in the PR.
- **Review artifacts.** All files produced during a review use `<REVIEW_ARTIFACT_ID>-*` naming and live under `.cursor/reviews/` — not inside `LOCAL_REPO_ROOT`.
- **CRITICAL: Line number accuracy.** Verify every cited line by fetching the file at `HEAD_SHA` from GitHub and locating the exact line in that content. Diff hunk headers (`@@ ... @@`) are not sufficient.
- Cite `path:line` for every finding (paths as in the repo root on GitHub). No vague "somewhere in the auth module" feedback.
- Severity matters. Do not list a missing JSDoc comment next to a SQL injection. If everything is "important," nothing is.
- If you're not sure something is a bug, say "possible issue" and explain what you'd need to verify it.
- Do not invent issues to fill out the report. A short, accurate review beats a long, padded one.
- Do not propose changes outside the diff scope unless they directly enable a fix.
- Do not run project test runners or test commands (`npm test`, `pytest`, `go test`, `cargo test`, etc.). CI runs the suite; this review is static analysis of the diff and GitHub-fetched context only.
- **GitHub top-level body.** When applying a verdict, post only the short verdict phrase — never the summary, rubric, or report sections from Step 4.
