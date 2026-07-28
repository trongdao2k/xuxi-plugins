---
description: Generate a PR description in format from the current branch
argument-hint: "[base-branch] (default: master)"
allowed-tools: Bash(git:*), Read, Grep, Glob
model: claude-sonnet-4-6
---

# Generate pull request description

## Configuration

**Step 0 — Load project config.** Use the `Read` tool to read `.compare-pr.json` from the project
root (the working directory). The file contains:

```json
{
  "base_branch": "master",
  "ticket_pattern": "ABC-[0-9]+",
  "platform": "Magento EE 2.4.x"
}
```

- `base_branch`: default branch to diff against. Overridden by `$1` if provided.
- `ticket_pattern`: regex to extract ticket key from the branch name.
- `platform`: target platform, used in test steps.

If the file does not exist, use these defaults: base_branch=`master`, ticket_pattern=`[A-Z]+-[0-9]+`, platform=`Magento 2`.

Set `BASE` to `$1` if provided, otherwise to `base_branch` from the config.

## Context

- Current branch: !`git branch --show-current`
- Config file: read `.compare-pr.json` with the `Read` tool (do not use `!` backtick for this)
- Merge base with base branch: use `Bash` tool to run `git merge-base HEAD origin/<BASE> 2>/dev/null || git merge-base HEAD <BASE>`
- Changed files with stats: use `Bash` tool to run `git diff --stat <merge-base>...HEAD`
- Commits on this branch: use `Bash` tool to run `git log --oneline --no-merges <merge-base>...HEAD`
- Full diff: use `Bash` tool to run `git diff <merge-base>...HEAD`

Replace `<BASE>` with the resolved base branch and `<merge-base>` with the commit hash from
the merge-base command. Run each git command as a separate `Bash` tool call.

If any command above returned an error (for example the base branch does not exist locally),
tell the user which base branch you actually used and continue. Do not stop.

## Step 1 — Understand the change

Read the diff. If the intent is unclear from the diff and commit messages alone, read the
surrounding source files before writing anything.

For the "Why" section, always write your best analysis of the reason based on the diff,
commit messages, branch name, and code context. Use format:

```
<AI analysis of why the change was made, based on evidence from the diff and code>

> ⚠️ Author: please review and correct the above if needed.
```

The analysis should explain the business or technical problem being solved, referencing
concrete evidence (e.g. "the old indexer re-triggered full inventory reindex on every
status change — the new subselect filters at query level"). Never fabricate facts — if
part of the reasoning is uncertain, say so explicitly (e.g. "likely to improve performance,
though the ticket context is not visible in the diff").

Extract the ticket key from the branch name using the pattern above. If no key is found, omit
the ticket line rather than fabricating one.

## Step 2 — Derive Magento 2 test steps from the diff

Include a setup command in "How to test" **only** when the diff justifies it:

| Change detected in diff                                                            | Required step                                          |
| ---------------------------------------------------------------------------------- | ------------------------------------------------------ |
| New module, `etc/module.xml`, `registration.php`                                    | `bin/magento module:enable Vendor_Module`               |
| `etc/db_schema.xml`, `Setup/Patch/**`, `etc/module.xml` version bump                 | `bin/magento setup:upgrade`                            |
| `etc/di.xml`, new/changed constructor args, new interface preference, factory/proxy | `bin/magento setup:di:compile`                          |
| `view/**` (`.phtml`, `.less`, `.js`, `requirejs-config.js`, `layout/**`)             | `bin/magento setup:static-content:deploy -f <locales>` |
| `etc/indexer.xml`, `etc/mview.xml`, indexer classes                                 | `bin/magento indexer:reindex <index>`                  |
| `i18n/*.csv`                                                                        | `bin/magento cache:clean translate`                    |
| Any config/layout/block change                                                      | `bin/magento cache:flush`                              |
| `etc/crontab.xml`, cron classes                                                     | note how to trigger: `bin/magento cron:run` or console  |

Then write concrete verification steps, not generic ones. Each step must name the exact entry
point a reviewer touches: admin path (`Stores > Configuration > ...`), storefront URL, CLI
command, or API endpoint. Include the expected result and at least one negative/edge case
(disabled config, guest vs logged-in customer, multi-store scope, empty cart, etc.).

State the test scope explicitly: which store view, which customer group, and whether the change
affects frontend, adminhtml, webapi_rest, webapi_soap, GraphQL, or crontab area.

## Step 3 — Pre-review self-check

Scan the diff for the items below and list only the ones you actually found, categorised.
Nothing found in a category means that category is omitted entirely — do not pad the list.

**Critical (must fix before review):**

- Raw SQL built by concatenation instead of bound parameters (SQL injection)
- Output in `.phtml` without `$escaper->escapeHtml()` / `escapeHtmlAttr()` / `escapeUrl()` (XSS)
- Adminhtml controller without `ADMIN_RESOURCE` constant or `_isAllowed()` (missing ACL)
- Frontend POST controller not implementing `CsrfAwareActionInterface` (CSRF)
- Credentials, API keys, or client-specific hostnames committed in code or config
- Customer/order data written to log without masking (PII)
- `etc/db_schema.xml` changed without regenerating `db_schema_whitelist.json`

**Suggestion (optional improvement):**

- `ObjectManager::getInstance()` used outside a factory, proxy, or `registration.php`
- Business logic in a block or controller instead of a service/model class
- Collection loaded inside a loop (N+1 query)
- `preference` used where a `plugin` or event observer would be less invasive
- `around` plugin where `before` or `after` would do (`callable` chain cost)
- Missing `@api` annotation on classes intended for third-party extension
- Hardcoded value that belongs in `core_config_data` via `system.xml`
- Non-English comment (Convention: English for code comments and docs)

**Question (need clarification):**

- Behaviour that changes for existing data without a migration path
- Compatibility uncertainty (PHP version, Magento patch level, conflicting third-party module)
- Anything the diff does that has no visible link to the ticket scope

## Step 4 — Output

Print the PR description in a single fenced markdown block so the user can copy it directly into
GitLab/GitHub, then print the self-check findings **outside** the block (they are for the author,
not the PR body).

````markdown
## What changed

<!-- One line per logical change, referencing the module. Not a file list. -->

- ...

**Ticket:** ABC-123
**Affected modules:** Vendor_Module (area: frontend/adminhtml/...)

## Why

<!-- AI analysis based on diff and code context. Author: review and correct if needed. -->

...

> ⚠️ Author: please review and correct the above if needed.

## How to test

**Setup**

```bash
...
```

**Steps**

1. ...
   - Expected: ...

**Edge cases**

- ...

**Test scope:** store view / customer group / area

## Screenshots

<!-- Required for any adminhtml or storefront change. Before / After. -->

| Before | After |
| ------ | ----- |
|        |       |
````

Rules for the output:

- English only.
- "What changed" describes behaviour, not files. `git diff` already shows files.
- Omit the Screenshots section entirely if the diff touches no `view/` or `.phtml` file.
- Do not modify any file. This command only reads and reports.
