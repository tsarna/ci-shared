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
    uses: tsarna/ci-shared/.github/workflows/ci-gate.yml@<sha> # v1.0.0
    with:
      needs: ${{ toJSON(needs) }}
```

Pinned by commit, not by tag. A moving tag would let anyone who can push a tag
here run code in every calling repo, each of which grants this workflow
`pull-requests: write` and an OIDC path to the Devin key — and zizmor reports
it as a HIGH finding. Renovate keeps the hash current from the trailing version
comment and automerges the bump once the calling repo's own CI is green.

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
- the author is the Renovate app, and **every commit on the branch is both
  authored by the bot and carries a valid GitHub signature** — so a branch a
  person has started editing is left alone. The signature is the part that
  makes this a control: GitHub resolves authorship by *email*, so
  `git config user.email <id>+<slug>[bot]@users.noreply.github.com` alone makes
  the login read as the bot. Renovate's commits through the App are signed by
  GitHub (`verified: true`, `reason: valid`); a laptop push cannot produce that
  under the App's identity. A PR with more than 250 commits is skipped, because
  `/pulls/{n}/commits` cannot enumerate past that;
- the branch matches `^renovate/[A-Za-z0-9._/-]{1,200}$` and the head SHA is 40
  hex characters;
- no `devin-hold` label;
- no existing handoff comment for **this head SHA** — Renovate rebases its
  branches, so dedupe is per tree, not per PR. A rebase deserves a fresh look;
  a re-run of the same tree does not;
- the daily session cap has not been reached. This is counted from Devin's own
  session list, so it holds across repos, and it **fails closed**: if the list
  cannot be read or does not parse, no session opens. It is still a
  time-of-check race — several PRs going red at once all read the same count —
  so the per-head-SHA marker, not the cap, is the tight bound.

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

The IAM trust policy is conditioned on `job_workflow_ref`, so only **this
workflow file** can read that parameter, from any repo. Be clear about what
that is and is not:

- The `environment: devin` half carries **no authorization**. The environment
  has no protection rules, and GitHub auto-creates an unprotected one in any
  repo that names it. It only shapes the OIDC `sub` claim.
- The ref is a wildcard, because a commit-pinned caller produces
  `…ci-gate.yml@<sha>` rather than `@refs/tags/v1`. So the boundary is **any
  ref of this file**, which means **push access to this repository is
  equivalent to holding the Devin key** — via the dispatch-only preflight job,
  which runs with `environment: devin` and `id-token: write`. Protect this
  repo's default branch accordingly, and treat a collaborator here as a
  key holder.
- Fork PRs are kept out by GitHub refusing `id-token: write` to `pull_request`
  runs from forks. That is load-bearing: without it, a fork PR against a public
  caller could point `uses:` at a commit from its own unmerged PR ref.

The skill is fetched from the **same revision of this repo that the caller
pinned**, read out of the OIDC token's own `job_workflow_ref` claim — so
pinning the workflow pins the agent's instructions with it. Fetching `main`
would have left the one file the agent is told to obey verbatim mutable.

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
once: revoke the role's policy, or delete the SSM parameter — that is the only
real kill switch, and it stops new sessions only. **There is no kill switch for
a session already running**; stop that one in the Devin app.

Note also that the marker comment is not authenticated: anyone who can comment
on a PR can pre-plant `<!-- devin-ci-fix sha=… -->` and suppress the handoff
for that tree. That direction is benign, but it is not a guarantee.
