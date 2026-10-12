---
name: remove-claude-attribution
description: |
  Remove Claude Code attribution from git commits and GitHub pull requests: the
  "Co-Authored-By: Claude" trailer, the "Claude-Session" link, and the "Generated
  with Claude Code" PR footer. Use when the user asks to remove, hide, or strip the
  Claude watermark, co-author line, or attribution, or to stop it being added.
---

# Remove Claude attribution

Claude Code marks its work in three places. Each has its own fix.

| Where | What it looks like | Fix |
| --- | --- | --- |
| Commit message | `Co-Authored-By: Claude ... <noreply@anthropic.com>` | `attribution.commit` = `""` |
| Commit message (cloud / Remote Control) | `Claude-Session: https://claude.ai/code/session_...` | `attribution.sessionUrl` = `false` |
| PR description | `🤖 Generated with [Claude Code](https://claude.com/claude-code)` and a session link | `attribution.pr` = `""` |

## 1. Stop new attribution

Make sure `.claude/settings.json` (shared) or `~/.claude/settings.json` (every project) contains:

```json
{
  "attribution": {
    "commit": "",
    "pr": "",
    "sessionUrl": false
  }
}
```

On Claude Code v2.1.281 or later, `"attribution": false` does the same in one line. Older versions reject that value and skip the whole settings file, so keep the object form in a file that older versions also read. Managed (organization) settings override all of these.

Also enable the backstop hook, which strips any trailer that still slips into a commit message:

```sh
git config core.hooksPath .githooks
```

## 2. Clean commits that already have it

First check what is affected:

```sh
git log --format='%h %s' --grep='noreply@anthropic.com' --grep='Claude-Session:' --grep='Generated with \[Claude Code\]'
```

**Unpushed commits** (safe to rewrite):

- Last commit only: `git commit --amend`, delete the trailer lines, save.
- Several commits: rewrite messages from the base with a message filter.

  ```sh
  git -c core.hooksPath=/dev/null filter-branch -f --msg-filter \
    "grep -v -i -E '^(Co-Authored-By: Claude.*<noreply@anthropic\.com>|Claude-Session: https://claude\.ai/.*|.*Generated with \[?Claude Code\]?.*)$'" \
    <base>..HEAD
  ```

  `<base>` is the last commit that should stay untouched, for example `origin/main`.

**Already pushed commits**: rewriting them changes their hashes and needs a force push. Before doing it:

- Confirm with the user. Never do this on a shared branch such as `main` without explicit approval.
- Only rewrite branches the user owns. Others who pulled the branch must reset to the new history.
- Push with `git push --force-with-lease origin <branch>`, never a bare `--force`.

## 3. Clean pull requests

GitHub stores the PR description separately from the commits. Edit the description and delete the `Generated with Claude Code` line and the claude.ai session link. With the GitHub MCP tools, read the body with `pull_request_read`, remove those lines, and write it back with `update_pull_request`. With the `gh` CLI: `gh pr edit <number> --body-file <cleaned-body.md>`.

The PR's commits keep their trailers until you rewrite them as in step 2.

## Limits

- Squash merges build the final message from the PR's commits, so clean the commits before merging.
- A commit's history on GitHub (reflog, forks, cached views) may still show the old message after a rewrite.
- This removes labels, not authorship. Follow any policy at your school, employer, or project that asks you to disclose AI help.
