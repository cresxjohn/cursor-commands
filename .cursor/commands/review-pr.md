# Review a GitHub PR or branch

Perform a rigorous code review of the changes described by a **GitHub pull request or branch link**—not whatever happens to be checked out locally. Act as a skeptical principal engineer reviewer, not a cheerleader.

**All context comes from GitHub via `gh` or the GitHub API.** Do not require, resolve, or read from a local clone. Never use workspace files, `git diff`, or `git checkout` for this review.

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

### Confirm before diffing

One line, then continue to Step 1:

```
Reviewing: owner/repo  <PR#n or base...head>  ref=<head-sha-or-ref>
```

Every `gh` invocation must include `--repo owner/repo` when the subcommand supports it.

If `gh` is missing or unauthenticated, stop and tell me to run `gh auth login` (or install `gh`). Do not fall back to a local clone.

## Step 1 — Establish scope via GitHub

Fetch metadata and the patch from GitHub only.

### Resolve refs and SHAs

**PR** (`/pull/n`):

```bash
gh pr view <n> --repo owner/repo --json number,title,body,baseRefName,headRefName,headRefOid,files,commits
gh pr diff <n> --repo owner/repo
```

Store `HEAD_SHA` = `headRefOid`, `HEAD_REF` = `headRefName`, `BASE_REF` = `baseRefName`.

**Compare or tree** (no PR number):

```bash
gh repo view owner/repo --json defaultBranchRef
gh api repos/owner/repo/compare/base...head
```

For a tree link, set `head` from the URL and `base` from `defaultBranchRef`. Store `HEAD_SHA` from the compare response (`commits[-1].sha` or `merge_base_commit` as appropriate).

### Diff triage

Use `gh pr diff` or the compare API patch. For file lists without full patches:

```bash
gh api repos/owner/repo/pulls/<n>/files --paginate
# or compare endpoint: .files[] with filename, patch, additions, deletions
```

If the diff is large (>1000 lines or >30 files), say so up front and propose a review strategy (review by subsystem, or ask which area to prioritize) before generating findings. Do not silently truncate.

## Step 2 — Read context via GitHub

For every non-trivial changed file, pull **full file contents at `HEAD_SHA`** (and at `BASE_REF` when before/after comparison matters)—not just diff hunks.

### Fetch a file at a ref

```bash
gh api repos/owner/repo/contents/<path> --field ref=<HEAD_SHA-or-ref> -q .content | base64 -d
```

For directory listings (e.g. to discover related modules or rules):

```bash
gh api repos/owner/repo/contents/<dir>?ref=<ref>
```

Use `-q` / `--jq` to extract fields; decode base64 content in the shell when posting inline comments is not needed.

### When the diff alone is not enough

- **Signature changes:** search for callers in the same repo:

  ```bash
  gh api "/search/code?q=<symbol>+repo:owner/repo" --paginate
  ```

  Then fetch each hit file at `HEAD_SHA` for surrounding context.

- **New env vars / config / feature flags:** fetch likely config paths at `HEAD_SHA` (`.env.example`, deploy manifests, infra dirs) via contents API or code search.

- **New DB queries:** fetch the model/repository files referenced in imports at `HEAD_SHA`.

If the change has obvious cross-repo implications (e.g. a BE API contract change with a FE consumer elsewhere), call that out as a **Question for the author**—do not silently assume access to sibling repos; use code search scoped to those repos only if URLs are provided.

## Step 3 — Review dimensions

Evaluate the diff against each of these. Skip dimensions that genuinely don't apply, but say so briefly so I know you considered them.

### Repository Conventions

Before evaluating the diff, fetch and read `.cursor/rules/` from GitHub at `HEAD_SHA`:

```bash
gh api repos/owner/repo/contents/.cursor/rules?ref=<HEAD_SHA>
# then fetch each rule file via contents API
```

If that path does not exist, note “no `.cursor/rules` found at HEAD” and proceed with general standards only.

1. Summarize these rules on the fly.
2. Include this summary as a rubric in your review.
3. Actively evaluate the PR/branch against them and call out violations.

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

Lead the report with the confirmation line from Step 0 (`owner/repo`, PR or ref range, `ref=<head-sha-or-ref>`). Then:

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

If I choose an option that involves submitting the verdict and/or comments, use `gh pr review` with `--repo owner/repo`. Inline comments require a PR number; for compare/tree-only reviews, explain that posting review comments needs an open PR.

- For **LGTM**: `gh pr review <n> --repo owner/repo --approve -b "LGTM"`
- For **LG. Approved with comments.**: `gh pr review <n> --repo owner/repo --approve -b "LG. Approved with comments."`
- For **Changes requested. Check comments.**: `gh pr review <n> --repo owner/repo --request-changes -b "Changes requested. Check comments."`

When creating line-by-line PR comment(s), use this format for each item:

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
- Post inline comments via the pulls review API (see below).
- Use GitHub format when suggesting code changes (e.g., ` ```suggestion ` blocks).

**Posting inline comments via GitHub API:**

1. Resolve `HEAD_SHA` again if needed: `gh pr view <n> --repo owner/repo --json headRefOid -q .headRefOid`
2. **Verify line numbers:** fetch the file at `HEAD_SHA` via the contents API, decode, and confirm the cited line matches the code you reference. Do not infer line numbers from diff hunk headers alone.
3. Build `review.json`:

```json
{
  "commit_id": "<HEAD_SHA>",
  "body": "Top-level review summary or body message",
  "event": "REQUEST_CHANGES",
  "comments": [
    {
      "path": "path/to/file.ts",
      "line": 42,
      "side": "RIGHT",
      "body": "### Issue:\n...\n\n### Why it matters:\n...\n\n### Suggested fix:\n```suggestion\n...\n```"
    }
  ]
}
```

`event` is `APPROVE`, `REQUEST_CHANGES`, or `COMMENT`.

4. Submit:

```bash
gh api --method POST -H "Accept: application/vnd.github+json" \
  /repos/owner/repo/pulls/<n>/reviews --input review.json
```

Delete temporary `review.json` after a successful post unless I ask to keep it.

## Constraints

- **GitHub-only context.** Use `gh` / GitHub API for diffs, file contents, rules, and code search. Do not read the workspace filesystem, enumerate local git remotes, or checkout branches for this review.
- **CRITICAL: Line number accuracy.** Verify every cited line by fetching the file at `HEAD_SHA` from GitHub and locating the exact line in that content. Diff hunk headers (`@@ ... @@`) are not sufficient.
- Cite `path:line` for every finding (paths as in the repo root on GitHub). No vague "somewhere in the auth module" feedback.
- Severity matters. Do not list a missing JSDoc comment next to a SQL injection. If everything is "important," nothing is.
- If you're not sure something is a bug, say "possible issue" and explain what you'd need to verify it.
- Do not invent issues to fill out the report. A short, accurate review beats a long, padded one.
- Do not propose changes outside the diff scope unless they directly enable a fix.
- Do not run project test runners or test commands (`npm test`, `pytest`, `go test`, `cargo test`, etc.). CI runs the suite; this review is static analysis of the diff and GitHub-fetched context only.
