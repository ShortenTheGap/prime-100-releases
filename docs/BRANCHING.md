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
| `staging` | Integration. Every `feature/*` is cut from here and PR'd back into here. Merge-commit. |
| `main` | **Production.** Deploy target. Only ever receives a PR from `staging`. Merge-commit, never squash. |

The name stays `main` — it is already the release source and already checked out in every
clone. One branch holding LIVE code beats two that drift.

Merge method matters. **Both branches allow merge commits only** — squash and rebase are
blocked by ruleset, for two different reasons:

- feature → staging: **merge commit**, to retain history. A squash flattens a branch's
  commits into one, and with automatic head-branch deletion on, the originals stop being
  reachable from any branch — still viewable in the PR on GitHub, but absent from a fresh
  clone, so no local `git blame` or `git bisect` on them. A merge commit's second parent
  keeps them all in the clone permanently.
- staging → main: **merge commit**, to keep ancestry intact. Squashing a promotion rewrites
  the promoted commits into one commit `staging` has never seen, so `main` stops being an
  ancestor of `staging` and the *next* promotion PR shows the entire history again as
  "changed".

Consequence of the first rule: `staging`'s graph is busy, and because status checks are
strict there, "Update branch" on an out-of-date feature branch merges `staging` into it and
that merge lands in the graph too. That is the cost of full history, accepted deliberately.

## What is enforced

Rulesets are **active on both branches** with an empty bypass list, so they apply to the
repo owner too. This repo is public, and rulesets are free on public repos — they cost money
only on private ones.

| Mechanism | Blocks what |
|---|---|
| Ruleset: require a pull request | **Direct pushes to `staging` and `main`.** Server-side. Cannot be bypassed. |
| Ruleset: allowed merge methods | **Squash and rebase merges.** Merge commits only, both branches. |
| Ruleset: block force pushes | **Force-pushes** to either branch. |
| Ruleset: restrict deletions | **Deleting** either branch. |
| Ruleset: required status checks | **A merge while `guard-promotion` is failing.** |
| `guard-promotion` (in `ci.yml`) | **A PR into `main` from any branch other than `staging`** — because it is a required check. |
| `ci.yml` test jobs | Nothing — there are none. This repo has no packages and no test commands. |
| `.githooks/pre-push` | Local direct pushes, on opted-in clones only. `--no-verify` bypasses it, but the ruleset then rejects the push anyway. Now a fast local failure rather than the last line of defence. |

Two things deliberately **not** required, because this is a single-account repo:

- **Required approvals: 0.** GitHub does not let a PR author approve their own PR, so with
  the bypass list empty, requiring 1 approval would make every merge impossible. Everything
  else above still holds; there is simply no second reviewer.
- **"Require approval of the most recent reviewable push": off.** It demands approval from
  someone other than the pusher and would lock the repo even at 0 required approvals.

Strict status checks ("require branches to be up to date") differ per branch on purpose:

- **`staging`: on.** Feature branches must be current before merging; cheap, and "Update
  branch" handles it.
- **`main`: off.** Every promotion leaves `main` holding a merge commit that `staging` does
  not have, so strict mode would flag every future promotion as stale and demand a back-merge
  of `main` into `staging` first — friction with no benefit, since `staging` already contains
  all of `main`'s content.

There is still **no gate on what clients see**. Releases are consumed straight from GitHub
(Release assets and `latest.json`), so nothing sits between a merge into `main` and clients
fetching it, and `ci.yml` has no test jobs to gate on. The rulesets control how code *reaches*
`main`; they do not verify that what reaches it is correct.

Note that `.githooks/pre-push` carries an inherited comment claiming the repo is "private
under a personal Free GitHub account" and that protection "cannot be enabled". That was true
of the repo the setup came from, not of this one — protection is enabled here.

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
# merge once CI is green — MERGE COMMIT (the only method allowed); head branch auto-deletes
```

`gh pr merge <n> --merge` is the command; `--squash` and `--rebase` are rejected by the
ruleset.

Promotion:

```sh
gh pr create --base main --head staging --title "Promote staging to production"
# 1. CI green on staging, guard-promotion passing (it is a required check)
# 2. merge with a MERGE COMMIT — gh pr merge <n> --merge (the only method allowed)
# 3. confirm the release assets / latest.json that clients fetch are the expected ones
```

Do **not** delete `staging` after a promotion (the ruleset blocks it anyway) — it is a
long-lived branch.

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

Also applied 2026-08-18, by hand in the GitHub UI (all four need repo admin):

1. **Default branch → `staging`.** Makes `staging` the base for every new PR and every fresh
   clone, which removes the single most likely mistake.
2. **"Automatically delete head branches" → on.**
3. **"Allow rebase merging" → off.** Repo-wide, on top of the per-branch ruleset restriction.
4. **Verified nothing keys off the default branch** for releases: assets hang off the *tag*
   (`whisper-assets-v1` → `ggml-base.en.bin`, `whisper-macos-arm64.tar.gz`,
   `whisper-windows-x64.tar.gz`), so the flip does not change what clients download.

   **One gotcha it does introduce:** `gh release create` with no `--target` uses the default
   branch, which is now `staging`. Pass `--target main` explicitly when cutting a release, or
   the tag gets created off `staging`.

Rulesets `Protect staging` and `Protect main` were created the same day — see the next
section for the live configuration.

Verify the GitHub half at any time:

```sh
gh api repos/ShortenTheGap/prime-100-releases \
  --jq '{default_branch, allow_squash_merge, allow_merge_commit, allow_rebase_merge, delete_branch_on_merge}'
# expected: staging, true, true, false, true
```

## The live rulesets

Applied 2026-08-18 via Settings → Rules → Rulesets. Both are `enforcement: active` with an
empty bypass list. Current state:

| Setting | `Protect staging` | `Protect main` |
|---|---|---|
| Target | `refs/heads/staging` | `refs/heads/main` |
| Require a pull request | yes | yes |
| Required approvals | 0 | 0 |
| Require approval of most recent push | off | off |
| Allowed merge methods | `merge` | `merge` |
| Required status checks | `guard-promotion` (GitHub Actions) | `guard-promotion` (GitHub Actions) |
| Branches up to date before merging | **on** | **off** |
| Require conversation resolution | on | off |
| Block force pushes | yes | yes |
| Restrict deletions | yes | yes |
| Require linear history | **off** — incompatible with merge commits | **off** |

Verify at any time — the second command is authoritative, it reports the rules the server
will actually enforce for the calling account rather than the stored config:

```sh
gh api repos/ShortenTheGap/prime-100-releases/rulesets --jq '.[] | {name, enforcement}'
gh api repos/ShortenTheGap/prime-100-releases/rules/branches/main --jq '.[].type'
# expected: deletion, non_fast_forward, pull_request, required_status_checks
```

Bind the status check to the **GitHub Actions** app (integration id `15368`) rather than
entering `guard-promotion` as free text — a free-text context can be satisfied by any commit
status of that name.

### Recreating them from the API

One call per branch. The payload is nested, so pass JSON on stdin — `gh api -f/-F` only
builds flat keys and cannot express `rules[].parameters`. `guard-promotion` is the only
status check this repo has; add the test-job names alongside it if suites are ever added.
This reproduces the live configuration, including the two per-branch differences.

```sh
for BRANCH in staging main; do
  # strict up-to-date checks and thread resolution on staging only — see the table above
  if [ "$BRANCH" = staging ]; then STRICT=true; RESOLVE=true; else STRICT=false; RESOLVE=false; fi
  cat <<JSON | gh api -X POST repos/ShortenTheGap/prime-100-releases/rulesets --input -
  {
    "name": "Protect $BRANCH",
    "target": "branch",
    "enforcement": "active",
    "bypass_actors": [],
    "conditions": { "ref_name": { "include": ["refs/heads/$BRANCH"], "exclude": [] } },
    "rules": [
      { "type": "pull_request", "parameters": {
          "required_approving_review_count": 0,
          "dismiss_stale_reviews_on_push": false,
          "require_code_owner_review": false,
          "require_last_push_approval": false,
          "required_review_thread_resolution": $RESOLVE,
          "allowed_merge_methods": ["merge"]
      }},
      { "type": "required_status_checks", "parameters": {
          "strict_required_status_checks_policy": $STRICT,
          "do_not_enforce_on_create": false,
          "required_status_checks": [
            { "context": "guard-promotion", "integration_id": 15368 }
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

Two constraints to remember if these are ever rebuilt:

- **A status check cannot be marked required until it has reported at least once.** Let CI run
  on a PR into each branch first, then the contexts become selectable.
- **GitHub does not let a PR author approve their own PR.** That is why approvals are 0. With
  a single account and an empty bypass list, requiring 1 approval makes every merge
  impossible. Raise it when a second collaborator exists; adding the owner as a
  `bypass_actor` instead is weaker, because it re-opens direct pushes.

Proving enforcement end to end is destructive, so it has not been run:

```sh
git push --force origin staging   # must be rejected by the remote
```

The non-destructive equivalent is the `rules/branches/<branch>` call above.

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
`guard-promotion`, `protected=` in `.githooks/pre-push`, the `Protect main` ruleset's target
pattern, this doc, the README section, and anything that publishes releases from `main`.
