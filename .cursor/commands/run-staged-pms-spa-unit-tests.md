# Run staged pms-spa unit tests (one-line command)

Builds a single shell command that runs `npm run test:unit` for **only** staged test files under `packages/@apps/pms-spa`, so you can copy it into a terminal and run tests locally without scanning the whole package.

## Usage

Invoke from Cursor chat when you want a copy-paste command for staged pms-spa tests.

**Command:**

```
/run-staged-pms-spa-unit-tests
```

## Instructions for AI

When this task is executed, follow these steps.

### Step 1: Repository root

1. Resolve the **fm** monorepo root (the git repository that contains `packages/@apps/pms-spa`).
2. Run all git commands with `working_directory` set to that root (or `cd` there first in a shell).

### Step 2: Collect staged test paths

1. Run: `git diff --cached --name-only --diff-filter=ACM`
2. From the output, keep only paths that satisfy **all** of:
   - Path starts with `packages/@apps/pms-spa/`
   - Path matches test file pattern: ends with `.spec.ts`, `.spec.js`, `.test.ts`, or `.test.js`
3. Strip the prefix `packages/@apps/pms-spa/` from each kept path so paths are **relative to the pms-spa package** (e.g. `src/helpers/foo.spec.ts`).
4. Deduplicate; sort alphabetically for stable output.

### Step 3: Handle empty list

If no paths remain after filtering:

- Respond with a short message that there are no staged pms-spa unit test files, and that the user should stage `*.spec.ts` / `*.spec.js` (or `*.test.ts` / `*.test.js`) under `packages/@apps/pms-spa`, or run tests another way.
- Stop.

### Step 4: Emit one-line command

1. Join the relative paths with a single space between each path.
2. Output **exactly one** runnable line for manual execution, using **only** this shape. Do **not** prefix with `cd packages/@apps/pms-spa &&` or any other shell prefix:

```bash
npm run test:unit -- <file1> <file2> ...
```

3. Present that line in a single fenced `bash` code block so the user can copy it in one gesture.
4. Do not split the command across multiple lines except inside the one code block.

### Notes

- Only **staged** files are considered (same idea as pre-commit focus).
- Non-test files under pms-spa are ignored; only paths ending with the test suffixes above are included.
- The person running the command must use **working directory** `packages/@apps/pms-spa` (or equivalent). File paths in the command are relative to that directory; nothing in the emitted line adds `cd` for them.
