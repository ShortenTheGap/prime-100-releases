# Branching: staging → production

## The flow

```
feature/login         ─┐
feature/maintenance    ─┼──PR──> staging ──test everything──PR──> main ──> LIVE
feature/notifications  ─┘                                      (production)
```

Two long-lived branches:

| Branch | Role |
|---|---|
| `staging` | Integration. Every `feature/*` is cut from here and PR'd back into here. Squash-merge. |
| `main` | **Production.** Deploy target. Only ever receives a PR from `staging`. Merge-commit, never squash. |

The name stays `main` — it is already the release source and already checked out in every
clone. One branch holding LIVE code beats two that drift.

Merge method matters:

- feature → staging: **squash** (keeps staging history readable).
- staging → main: **merge commit**. Squashing a promotion rewrites the promoted commits into
  one commit `staging` has never seen, so `main` stops being an ancestor of `staging` and the
  *next* promotion PR shows the entire history again as "changed".

## The rules are not enforced yet

Read this before assuming anything here blocks a mistake.

| Mechanism | Blocks what |
|---|---|
| `ci.yml` test jobs | Nothing — and there are none. This repo has no packages and no test commands. |
| `guard-promotion` | Nothing. Visible, advisory only. |
| `.githooks/pre-push` | Local direct pushes, on opted-in clones only. `--no-verify` bypasses. |
| Deploy "Wait for CI" | Not applicable — see below. |

**There is currently no gate that actually blocks anything.** Two reasons specific to this
repo:

1. **No deploy platform to gate.** Releases are consumed straight from GitHub (Release
   assets and `latest.json`), so there is no build step between a merge into `main` and
   clients seeing it. In the setup this was ported from, a platform-side "Wait for CI" was
   the only real gate; here that slot is empty.
2. **No test jobs to wait on.** `ci.yml` runs only `guard-promotion`. Nothing is built,
   linted, or tested, because there is nothing in the tree to check.

The mitigation available today: **this repo is public, so GitHub rulesets and branch
protection are free** — they cost money only on private repos. The lockdown commands at the
bottom can be run now; they need repo *admin*, not a paid plan. Until someone with admin
runs them, the flow is convention plus a local hook.

Note also that `.githooks/pre-push` carries an inherited comment claiming the repo is
"private under a personal Free GitHub account" and that protection "cannot be enabled". That
was true of the repo the setup came from, not of this one.

## One-time setup, per clone

Existing clone:

```sh
git fetch origin
git switch staging
git config core.hooksPath .githooks   # enables the pre-push guard, this clone only
```

New clone: `git clone …`, then the same `git config core.hooksPath .githooks`.

Git does not share hooks. **Every dev must run that `git config` line or the guard does
nothing.**

## Day to day

Feature:

```sh
git switch staging && git pull
git switch -c feature/notifications
# ... commit ...
git push -u origin feature/notifications
gh pr create --base staging --fill
# squash-merge once CI is green and someone else approved; delete the branch
```

Promotion:

```sh
gh pr create --base main --head staging --title "Promote staging to production"
# 1. CI green on staging, guard-promotion passing
# 2. get an approval
# 3. merge with a MERGE COMMIT (not squash)
# 4. confirm the release assets / latest.json that clients fetch are the expected ones
```

Hotfix — still goes through `staging`:

```sh
git switch staging && git pull
git switch -c fix/<thing>
# PR into staging, merge, then immediately promote staging -> main
```

Branching a hotfix off `main` directly means the fix exists in production but not in
`staging`, and the next promotion silently reverts it. If it is ever unavoidable,
cherry-pick the same commit into `staging` in the same sitting.

Branch naming: `feature/<thing>`, `fix/<thing>`, `chore/<thing>`. Nothing enforces it; it
just keeps the branch list readable.

## Two gotchas that cost time

- **A branch cut before CI landed shows no checks at all.** GitHub runs a `pull_request`
  workflow from the file as it exists on the *head* branch, so a branch predating `ci.yml`
  produces an empty check list — which looks like "passing" at a glance. Rebase it on
  current `staging` and the checks appear.
- **Check the base branch on every PR.** Even with `staging` as default, a hand-retargeted
  PR or one opened from a stale tab can land on `main`. Fix the base selector rather than
  relying on `guard-promotion` to catch it.

## Admin-task record

Applied 2026-08-18: `staging` created from `main`; `.githooks/pre-push`, `.gitattributes`,
`.github/workflows/ci.yml`, `.github/pull_request_template.md`, this doc, and the README
section added via `chore/branching-flow`.

**Not applied — these need repo admin, which the account used did not have.** Someone with
admin on `ShortenTheGap/prime-100-releases` must do them by hand:

1. **Default branch → `staging`.** Settings → General → Default branch, via the **⇄ swap
   icon**. (*Not* Settings → Branches, which has no such control.) Highest-value item in the
   whole setup: it makes `staging` the base for every new PR and every fresh clone, which
   removes the single most likely mistake.
2. **"Automatically delete head branches" → on.** Settings → General → Pull Requests.
3. **"Allow rebase merging" → off.** Leaves squash (feature→staging) and merge commit
   (staging→main) only, matching the merge rules above.
4. **Confirm nothing keys off the default branch** for publishing releases or serving
   `latest.json` after the flip — verify, do not assume.

Verify the GitHub half at any time:

```sh
gh api repos/ShortenTheGap/prime-100-releases \
  --jq '{default_branch, allow_squash_merge, allow_merge_commit, allow_rebase_merge, delete_branch_on_merge}'
# expected: staging, true, true, false, true
```

## Real protection (free here — public repo, needs admin)

One call per branch. The payload is nested, so pass JSON on stdin — `gh api -f/-F` only
builds flat keys and cannot express `rules[].parameters`. `guard-promotion` is the only
status check this repo has; add the test-job names alongside it if suites are ever added.

```sh
for BRANCH in staging main; do
  cat <<JSON | gh api -X POST repos/ShortenTheGap/prime-100-releases/rulesets --input -
  {
    "name": "Protect $BRANCH",
    "target": "branch",
    "enforcement": "active",
    "bypass_actors": [],
    "conditions": { "ref_name": { "include": ["refs/heads/$BRANCH"], "exclude": [] } },
    "rules": [
      { "type": "pull_request", "parameters": {
          "required_approving_review_count": 1,
          "dismiss_stale_reviews_on_push": true,
          "require_code_owner_review": false,
          "require_last_push_approval": true,
          "required_review_thread_resolution": true,
          "allowed_merge_methods": ["merge", "squash"]
      }},
      { "type": "required_status_checks", "parameters": {
          "strict_required_status_checks_policy": true,
          "do_not_enforce_on_create": false,
          "required_status_checks": [
            { "context": "guard-promotion" }
          ]
      }},
      { "type": "non_fast_forward" },
      { "type": "deletion" }
    ]
  }
JSON
done
```

`non_fast_forward` blocks force-pushes, `deletion` blocks branch deletion,
`strict_required_status_checks_policy` requires the branch to be up to date before merge,
`bypass_actors: []` applies the rules to the owner too.

Before turning it on:

- **A status check cannot be marked required until it has reported at least once.** Let CI
  run on a PR into each branch first, then the contexts become selectable.
- **GitHub does not let a PR author approve their own PR.** With
  `required_approving_review_count: 1` and no bypass actors, every merge needs a second
  account. A solo setup needs the owner added as a `bypass_actor` — weaker, but it still
  blocks force-pushes and accidental direct pushes.

Verify:

```sh
gh api repos/ShortenTheGap/prime-100-releases/rulesets --jq '.[].name'
git push --force origin staging   # must be rejected by the remote
```

## Optional: rename `main` to `production`

Admin-only, free on any plan; GitHub retargets open PRs automatically. It is a one-time chore
for every dev, so it is optional — the flow works identically with the name `main`.
Settings → Branches → rename, then each dev runs:

```sh
git branch -m main production
git fetch origin && git branch -u origin/production production
git remote set-head origin -a
```

Then update: `branches:` in `.github/workflows/ci.yml`, the `BASE` check in
`guard-promotion`, `protected=` in `.githooks/pre-push`, this doc, the README section, and
anything that publishes releases from `main`.
