# claude-ops

Central Claude workflows for `jonhanke-ai-projects` and `jonhanke-dev`.
Repos carry thin stubs that call these; the logic lives here, so a change
to a prompt or a model tier is one edit rather than one per repo.

Public on purpose — a private repo's reusable workflows are not callable
from another organization.

## Two workflows

| | `claude-review.yml` | `claude-implement.yml` |
|---|---|---|
| Trigger | every PR push | `@claude ...` on an issue, or the `claude-implement` label |
| Does | reads the diff, comments | writes code and tests, opens a PR |
| Tools | four read-only `gh pr` commands | full toolset (edit files, run tests) |
| Model | `claude-sonnet-5`, or `claude-opus-5` with `deep-review` | `claude-opus-5` |
| Merges? | never — posts `COMMENTED` | never — opens a PR for review |

They are meant to be used together. The implementer opens a PR; the
reviewer reviews it. Different Claudes, different context, neither able to
approve its own work.

## Adding a repo

1. Copy the stub from `stubs/` into `.github/workflows/` in the target repo.
   Copy it **verbatim** — it carries a `permissions:` block and explicit
   secret passing, both of which are load-bearing (see Gotchas).
2. Set `CLAUDE_CODE_OAUTH_TOKEN` as a repo secret:
   `gh secret set CLAUDE_CODE_OAUTH_TOKEN --repo OWNER/REPO`
3. Confirm the **official** Claude GitHub App is installed on the org:
   `gh api orgs/ORG/installations --jq '.installations[].app_slug'`
   should include `claude`.

Review is safe to enable everywhere. Implement is opt-in — it grants an
agent write access to the repo, so enable it where you want that.

## Labels

| Label | Effect |
|---|---|
| `skip-review` | PR is not reviewed |
| `deep-review` | re-runs the review on the deeper model tier |
| `claude-implement` | applied to an **issue**, starts an implementation run |

## Gotchas

Every one of these was found the hard way, and most of them **report
success** while doing nothing.

1. **The stub must declare `permissions:`.** A called workflow cannot
   request more permission than its caller holds, and these repos default
   `GITHUB_TOKEN` to read-only. Omitting it gives `startup_failure` with no
   logs and no message.

2. **`secrets: inherit` does not cross organizations.** It does not error —
   the secret arrives empty, the action no-ops in about seven seconds, and
   the job reports **success**. Pass secrets explicitly by name.

3. **A PR that edits its own workflow file cannot run it.** The action
   requires the file to be byte-identical to the default branch copy,
   otherwise a PR could rewrite the workflow and exfiltrate the token.
   Expect a skipped check on any PR that changes a stub.

4. **`gh pr review` / `gh pr comment` need `--allowedTools`.** Without
   them Claude reads the diff, forms a review, then cannot publish it —
   a billed, green run with no output. Watch `permission_denials_count`.

5. **A custom GitHub App is not enough.** The OIDC token exchange rejects
   anything but the official `claude` app: `401 Unauthorized - Claude Code
   is not installed on this repository`.

**Diagnosing a quiet failure:** in the run's result payload,
`total_cost_usd` near zero means Claude never ran; a nonzero cost with
`permission_denials_count > 0` means it ran but could not publish.

## Cost

A review of a small diff is roughly $0.07. Implementation runs are
substantially more — they read more, write more, and run tests. Both draw
on the Claude Max subscription rather than metered API billing.

GitHub Actions minutes are billed separately by GitHub, and are the more
likely thing to run out.
