# Fork maintenance

- `fork` is the default branch: the latest upstream nightly tag plus our changes. Build and run this.
- `main` is an exact mirror of `pingdotgg/t3code` main. Never commit here; cut upstream PR branches from it.
- `sync/nightly` is the scratch branch the nightly sync tests before promoting to `fork`.

`Fork Sync` (`.github/workflows/fork-sync.yml`) runs daily: it mirrors `main`, merges the newest
`v*-nightly.*` tag into `sync/nightly`, runs `Fork CI`, and fast-forwards `fork` if it passes. A merge
conflict or a red run opens an issue instead. Run it on demand with `gh workflow run fork-sync.yml`.

## Fork-only changes

Branch from `fork`, open a PR into `fork`. Fork CI runs on the PR.

## Upstream PRs

Upstream only takes bug fixes from outside the team, 3 open PRs max. Read `CONTRIBUTING.md` first.

```sh
git fetch upstream && git switch -c fix/thing upstream/main
# fix, then test locally
vp run check:changed
vp test run <changed test files>
git push -u origin fix/thing
gh workflow run fork-ci.yml -R abishop177-tech/t3code -f ref=fix/thing   # full CI on GitHub runners
gh pr create -R pingdotgg/t3code --head abishop177-tech:fix/thing --base main
```

To try the fix in the app first, merge it into a scratch branch off `fork` and run `vp run dev`.

## Sync conflicts

Ask an agent to use the `fix-t3-conflict` skill. It merges the newest nightly in a separate
worktree, resolves and verifies it, and opens a PR into `fork` for you to review.
