# compare-pr

Generates pull request descriptions in format from the current branch diff.

## Commands

| Command                        | Description                                                                                            |
|--------------------------------|--------------------------------------------------------------------------------------------------------|
| `/compare-pr:pr [base-branch]` | Generate a PR description for the current branch. Base branch defaults to value in `.compare-pr.json`. |

## Output format

```
What changed | Why | How to test | Screenshots (if UI)
```

The command also prints a pre-review self-check, categorised as **Critical** / **Suggestion** /
**Question**, so obvious issues are caught before a reviewer spends time on them.

## Magento 2 behaviour

Setup commands in "How to test" are derived from the diff rather than pasted from a fixed list.
A change under `etc/db_schema.xml` produces a `setup:upgrade` step; a change under `view/` produces
`static-content:deploy` and a Screenshots section; a pure PHP service change produces neither.

Security patterns are flagged as Critical: unbound SQL, unescaped `.phtml` output, adminhtml
controllers without ACL, and frontend POST controllers without `CsrfAwareActionInterface`.

## Adapting to a project

Create a `.compare-pr.json` file in your project root:

```json
{
  "base_branch": "master",
  "ticket_pattern": "ABC-[0-9]+",
  "platform": "Magento EE 2.4.x"
}
```

| Field | Description | Default |
| ----- | ----------- | ------- |
| `base_branch` | Branch to diff against | `master` |
| `ticket_pattern` | Regex to extract ticket key from branch name | `[A-Z]+-[0-9]+` |
| `platform` | Target platform, used in test steps | `Magento 2` |

If `.compare-pr.json` does not exist, the defaults above are used.

For non-Magento projects, fork the command into a sibling file (`pr-laravel.md`, `pr-shopify.md`)
and replace Step 2. Steps 1, 3, and 4 are stack-agnostic.

## Safety

`allowed-tools` is limited to `Bash(git:*)`, `Read`, `Grep`, and `Glob`. The command cannot edit
files, commit, push, or open a PR.
