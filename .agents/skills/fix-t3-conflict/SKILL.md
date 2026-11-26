---
name: fix-t3-conflict
description: Resolve a failed nightly upstream merge in the abishop177-tech/t3code fork and open a PR into the `fork` branch. Use when a "Fork sync conflict" issue is open, Fork Sync failed on a merge, or the user asks to pull a nightly into the fork by hand.
---

# Resolve a fork sync conflict

Fork Sync (`.github/workflows/fork-sync.yml`) merges the newest upstream `v*-nightly.*` tag into
`fork` nightly. When the merge conflicts, it opens (or comments on) a "Fork sync conflict" issue
and leaves `fork` untouched. Your job: do that merge, resolve it, verify it, and open a PR into
`fork`. A human merges the PR.

Hard limits:

- Every `gh` command uses `-R abishop177-tech/t3code`. Never open anything against `pingdotgg/t3code`.
- Never push to `fork` or `main` directly, and never force-push them. Push only `resolve/*` branches.
- Never merge the PR yourself.

## 1. Find the tag and the issue

```sh
R=abishop177-tech/t3code
gh issue list -R $R --state open --search 'in:title "Fork sync conflict"'
git fetch origin fork
git fetch --no-tags upstream main
tag=$(git ls-remote --tags --refs upstream 'v*-nightly.*' | awk -F/ '{print $3}' | sort -V | tail -1)
git fetch --no-tags upstream tag "$tag"
```

Always resolve against the newest tag, even if the issue names an older one. The newer tag includes it.
If `git merge-base --is-ancestor "$tag" origin/fork` succeeds, `fork` already has it: comment on the
issue, close it, and stop.

## 2. Merge in a separate worktree

Don't disturb the user's checkout. Put the worktree next to this one, not in `/tmp`:

```sh
git worktree add "../t3code-resolve-$tag" -b "resolve/$tag" origin/fork
cd "../t3code-resolve-$tag"
git config rerere.enabled true
git merge --no-ff --no-edit -m "chore(fork): merge upstream $tag" "$tag"
git diff --name-only --diff-filter=U
```

## 3. Understand both sides before editing a file

For each conflicted file:

```sh
base=$(git merge-base HEAD "$tag")
git log --oneline "$base..origin/fork" -- <file>   # our fork-only changes
git log --oneline "$base..$tag" -- <file>          # upstream changes since we last merged
git diff "$base" origin/fork -- <file>
git diff "$base" "$tag" -- <file>
```

Read the commit messages and any PRs they name. Then resolve:

- **Default:** take upstream's version and reapply the fork's change on top of it, adapted to upstream's
  new shape (renames, moved code, new signatures).
- **Upstream already fixed it:** if upstream now contains the fork change, or a better fix for the same
  problem, take upstream's version and drop ours. Say so in the PR.
- **Can't tell what the fork change was for, or the two sides really disagree:** keep the version that
  builds, then list the file under "Needs a human" in the PR. Don't guess at product behavior.

Never silently drop a fork-only feature. Grep for other call sites of anything you rename or move.
Check that no conflict markers remain: `git diff --check` and `rg -n '^(<<<<<<<|>>>>>>>|=======$)'`.

## 4. Verify

Follow AGENTS.md: wait for or run `vp i`, then:

```sh
vp run typecheck
vp run check:changed
vp test run <test files next to every file you resolved>
```

Don't run repo-wide suites. Fork CI runs everything on the PR. If a failure also happens on the
untouched tag (`git stash; git checkout "$tag"` in a scratch worktree), it's upstream's, not yours:
note it in the PR and don't fix it here.

## 5. Commit, push, open the PR

```sh
git add -A && git commit --no-edit
git push -u origin "resolve/$tag"
gh pr create -R $R --base fork --head "resolve/$tag" \
  --title "chore(fork): merge upstream $tag" --body-file <body>
```

The PR body, in this order:

1. `Closes #<issue>` (works because `fork` is the default branch).
2. A table of the conflicted files, with one line each on what upstream changed, what the fork had,
   and how you resolved it.
3. **Needs a human:** each judgment call, or "None".
4. The checks you ran and their results. Upstream failures go here too.
5. The model and harness that did the work.

Then wait for Fork CI on the PR. If it fails because of your resolution, fix it and push again.
Report the PR URL, the CI state, and the "Needs a human" list to the user.

## 6. Clean up

Once the PR is open and pushed, go back to the main checkout and run
`git worktree remove "../t3code-resolve-$tag"`. Keep the branch.
