# claude-ops

Central Claude workflows for `jonhanke-ai-projects` and `jonhanke-dev`.
Repos carry thin stubs that call these; the logic lives here, so a change
to a prompt or a model tier is one edit rather than one per repo.

Public on purpose — a private repo's reusable workflows are not callable
from another organization.

## Three workflows

| | `claude-plan.yml` | `claude-implement.yml` | `claude-review.yml` |
|---|---|---|---|
| Trigger | `@claude plan` on an issue | `@claude ...` on an issue | every PR push |
| Does | posts an approach, stops | writes code and tests, opens a PR | reads the diff, comments |
| Writes? | **nothing** | branch + PR | nothing |
| Tools | read + `gh issue comment` | full toolset | four read-only `gh pr` commands |
| Model | `claude-opus-5` | `claude-opus-5` | `claude-sonnet-5`, or `claude-opus-5` with `deep-review` |

Meant to be used as a chain, with your attention needed twice and briefly:

```
@claude plan  ->  you read the plan, say go  ->  @claude implement  ->  review  ->  you merge
```

The point of the plan stage is that your decision gets cheaper the earlier it
happens. Approving a paragraph takes a minute; rejecting a finished PR costs an
implementation run and your attention on a diff you did not want.

Different Claudes at each stage, with different context, and none of them able
to approve their own work.

**Planning is optional.** Well-specified issues can go straight to implement.
Use plan when the issue is yours-to-yourself rather than written as a task.

## Adding a repo

1. Copy the stub from `stubs/` into `.github/workflows/` in the target repo.
   Copy it **verbatim** — it carries a `permissions:` block and explicit
   secret passing, both of which are load-bearing (see Gotchas).
2. Set `CLAUDE_CODE_OAUTH_TOKEN` as a repo secret:
   `gh secret set CLAUDE_CODE_OAUTH_TOKEN --repo OWNER/REPO`
3. Confirm the **official** Claude GitHub App is installed on the org:
   `gh api orgs/ORG/installations --jq '.installations[].app_slug'`
   should include `claude`.

Review and plan are safe to enable everywhere — neither writes anything.
Implement is opt-in: it grants an agent write access to the repo, so enable it
where you actually want that.

## Working through a queue

The pattern this is built for is not one issue at a time with you watching.
That is the worst case — all of the latency, none of the benefit.

Instead: comment `@claude plan this` on several issues at once. Nothing
serializes them; each gets its own run. Come back later and read the plans as a
batch, approving or redirecting in a line each. Then fire `@claude implement`
at the ones you approved, and read the resulting PRs as a batch too.

Your attention is needed twice, briefly, on your schedule, rather than
continuously while a run you are watching finishes.

## Labels

| Label | Effect |
|---|---|
| `skip-review` | PR is not reviewed |
| `deep-review` | re-runs the review on the deeper model tier |
| `claude-plan` | applied to an **issue**, posts an approach and stops |
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
