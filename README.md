# ci-shared

Shared GitHub Actions machinery for the `tsarna` repos. Public because
reusable workflows in a private repo on a user account cannot be called from
other repos.

## `ci-gate.yml` — the shared CI Success gate

One definition of "did CI pass", plus the Renovate → Devin handoff that only
makes sense when it didn't. Add it as the last job of a repo's `ci.yml`:

```yaml
  ci-success:
    name: CI Success
    if: always()
    needs: [build]              # every job that must pass
    permissions:
      actions: read             # read this run's job conclusions
      contents: read
      pull-requests: write      # marker comment + labels
      id-token: write           # OIDC -> AWS -> Devin key in SSM
    uses: tsarna/ci-shared/.github/workflows/ci-gate.yml@v1
    with:
      needs: ${{ toJSON(needs) }}
```

A reusable workflow cannot see its caller's jobs, but it can be *handed* them:
`toJSON(needs)` is legal in `with:`. So the gate aggregates any repo's results
without knowing the job names — which is why `build`, `test` and `checks` all
work unchanged.

> **All four permissions are required, even with `handoff: false`.** GitHub
> checks statically that no job in a called workflow requests more than the
> caller grants, and fails the **entire run at startup** if one does — `if:`
> conditions are not considered, so the skipped handoff job still counts.
> Granting less does not disable the handoff; it stops the repo's CI from
> running at all.

Then require only **CI Success** in branch protection. Adding, renaming or
splitting a job no longer means editing the protection rule.

### Inputs

| Input | Default | Meaning |
|---|---|---|
| `needs` | *(required)* | `${{ toJSON(needs) }}` from the caller |
| `allow_skipped` | `true` | a skipped job counts as success |
| `handoff` | `true` | open a Devin session on a red Renovate PR |
| `renovate_bot` | `tsarna-renovate[bot]` | only this author is eligible |
| `max_acu` | `5` | hard ACU ceiling for one session |
| `max_sessions_per_day` | `4` | cap across all repos, counted from Devin |

An empty or missing `needs` fails the gate. A gate that passes because it was
handed nothing is worse than no gate at all.

## The Devin handoff

The `handoff` job is a **sibling** of the gate, never a dependency of it, so
nothing it does — including failing outright — can change the CI Success
conclusion.

It runs only when all of this holds:

- the gate saw a `failure` (not a cancellation);
- the event is a same-repo `pull_request` (so: not `push`, not `merge_group`,
  not a fork, and not reachable by a schedule);
- the PR is open *now* (re-read live, not from the event payload) and not a
  draft;
- the author is the Renovate app and **every commit on the branch is the
  bot's** — a branch a person has started editing is left alone;
- the branch matches `^renovate/[A-Za-z0-9._/-]{1,200}$` and the head SHA is 40
  hex characters;
- no `devin-hold` label;
- no existing handoff comment for **this head SHA** — Renovate rebases its
  branches, so dedupe is per tree, not per PR. A rebase deserves a fresh look;
  a re-run of the same tree does not;
- the daily session cap has not been reached.

Failing job names come from the Actions API (`checks / checks`, as the UI shows
them), not from a hand-maintained list, and are character-bounded before they
reach the prompt. The prompt itself carries only the repo, PR number, branch,
head SHA, failing job names, run URL, and a pointer to
[`skills/renovate-ci-fix/SKILL.md`](skills/renovate-ci-fix/SKILL.md) — which
lives here, in git, once, rather than being copied into every repo.

### Credentials

No GitHub secret anywhere. The job mints an OIDC token, exchanges it directly
with STS (`aws sts assume-role-with-web-identity` — no third-party action in
the credential path), and reads the Devin API key from SSM. The key's lifetime
is one shell step.

The IAM trust policy is conditioned on both the environment
(`repo:tsarna/*:environment:devin`) and `job_workflow_ref` — so **only this
workflow file**, called from any repo, can read that parameter.

| | |
|---|---|
| Role | `arn:aws:iam::635166557291:role/gh-devin-handoff` |
| Key | `/awusoh02/apps/devin/api-key` (SecureString, `us-east-2`) |
| Org id | `/awusoh02/apps/devin/org-id` (String) |

The session API is `POST /v3/organizations/{org_id}/sessions`. A `cog_`
service-user key authorises **only** the org-scoped v3 routes: every `/v1/*`
endpoint answers `403 {"detail":"Unauthorized"}` for one, whichever endpoint it
is, so a 403 there is not a permissions problem to chase. The org id is on the
**Settings → Devin API** page in app.devin.ai, which is also where service
users and their keys are created.

The v3 schema has **no `idempotent` field** and **no `branch` field**. So the
per-head-SHA marker comment is the only thing between a persistently red PR and
one session per CI run, and the branch is named in the prompt instead.

### Labels

Created in a repo on first use:

| Label | Meaning |
|---|---|
| `devin-in-progress` | a session is working on this PR |
| `devin-assisted` | fixed by a session |
| `devin-no-fix` | the failure is not caused by this update (pre-existing, flaky, or upstream) |
| `devin-failed` | caused by the update, but the session could not fix it |
| `devin-hold` | never hand this PR to Devin (set by hand) |

### Turning it off

Per PR: add `devin-hold`. Per repo: `with: {handoff: false}`. Everywhere at
once: revoke the role's policy, or delete the SSM parameter.
