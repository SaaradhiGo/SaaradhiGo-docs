## Worktrees
- Worktrees go under one folder outside the repos, `~/sg-worktrees/`, at most two at a time.
- When a PR merges, remove its worktree and local branch in the same session.
- At the end of every run, `git worktree list` shows only the main checkout plus any worktree with an open PR, and every main checkout is on its default branch and clean.
- At the end of every agent session, run `scripts/worktree-status.sh` from `saaradhigo-internal-docs` and fix anything it flags.
