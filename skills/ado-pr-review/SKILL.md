---
name: ado-pr-review
description: Code review a pull request in Azure DevOps. Use when the user invokes /ado-pr-review with a PR ID.
---

Provide a code review for the given Azure DevOps pull request.

The user invokes this skill as `/ado-pr-review <PR-ID>`. The PR ID is a numeric Azure DevOps pull request ID.

## Prerequisites

1. **Azure DevOps MCP server** must be configured in your Claude Code MCP settings with a valid Personal Access Token (PAT). Required PAT scopes: *Code: Read*, *Pull Request Threads: Read & Write*.
2. **Local clone**: run this skill from inside a local clone of the Azure DevOps repo you want to review.
3. No `az login` is needed — ADO authentication is handled entirely by the MCP server PAT.

Follow these steps precisely:

## Step 0: Setup

Derive the Azure DevOps context once by running:

```bash
git remote get-url origin
```

Parse org, project, and repo from the URL. For a URL like `https://myorg@dev.azure.com/myorg/MyProject/_git/MyRepo`:
- org = `myorg`
- project = `MyProject`
- repo = `MyRepo`

Also capture the full SHA of the PR's source branch tip — you will need it for file links.

## Step 1: Eligibility check (Haiku agent)

Use a Haiku agent to check whether the pull request is eligible for review. The agent should:

1. Call `mcp__azure-devops__repo_get_pull_request_by_id` with the PR ID to fetch its details
2. Call `mcp__azure-devops__repo_list_pull_request_threads` to list all existing threads on the PR
3. Return `SKIP` (with a reason) if any of the following are true:
   - PR status is `abandoned` or `completed` — **Check the `status` field specifically, NOT `mergeStatus`. These are two different fields: `status` tracks the PR lifecycle (1=active, 2=abandoned, 3=completed), while `mergeStatus` tracks merge policy results (3=succeeded means policies passed, not that the PR is done). Only skip for `status` 2 or 3. `status` 1 / "active" means the PR is open and should be reviewed.**
   - PR is a draft (isDraft = true)
   - PR title or description suggests it is automated (e.g. "Bump", "chore", "dependabot", "auto-merge")
   - Any existing thread contains a comment that was posted by "Claude Code" or contains the text "Generated with [Claude Code]" — indicating a prior review already exists
4. Otherwise return `PROCEED` along with the PR title, source branch, and target branch

If the agent returns `SKIP`, stop here and do not proceed.

## Step 2: Locate guidance files (Haiku agent)

Use a Haiku agent to find relevant `CLAUDE.md` and `AGENTS.md` guidance. The agent should:

1. Get the list of files changed in the PR via `mcp__azure-devops__repo_get_pull_request_by_id` (or from `git diff --name-only <targetBranch>...<sourceBranch>`)
2. Identify the unique directory paths of changed files
3. Check for `CLAUDE.md` and `AGENTS.md` existence at:
   - The repo root
   - Each unique directory that contains a changed file, walking up to the root
4. Return the list of `CLAUDE.md` and `AGENTS.md` file paths (relative to repo root) that exist

## Step 3: Summarize the PR (Haiku agent)

Use a Haiku agent to produce a brief summary of the change. The agent should:

1. Read the PR title and description from `mcp__azure-devops__repo_get_pull_request_by_id`
2. Run `git diff <targetBranch>...<sourceBranch>` to get the full diff
3. Return a 2–4 sentence summary of what the PR changes and why

## Step 4: Parallel code review (5 Sonnet agents)

Launch 5 Sonnet agents in parallel. Each should independently review the PR and return a list of issues (each with a description and reason it was flagged). Provide all agents with:
- The PR diff (from `git diff <targetBranch>...<sourceBranch>`)
- The list of changed files
- The PR summary from step 3
- The list of `CLAUDE.md` and `AGENTS.md` paths from step 2

### Agent 1 — Guidance file compliance

Read each `CLAUDE.md` and `AGENTS.md` file from the list. Audit the PR diff for violations of those instructions. Note: these files are guidance for AI agents when writing code, so not all instructions apply during review. Only flag issues that clearly apply to the code changes.

### Agent 2 — Bug scan

Read the diff carefully. Do a shallow scan for obvious bugs in the changed lines. Avoid reading extra context beyond the changes. Focus on large bugs — avoid nitpicks and small issues. Do not flag pre-existing issues (lines not touched by the PR).

### Agent 3 — Git history context

For each changed file, run:
- `git log --oneline -20 -- <file>` to see recent history
- `git blame <targetBranch> -- <file>` for context on unchanged surrounding lines

Identify any bugs in the PR changes that only become apparent given the historical context.

### Agent 4 — Previous PR comments

For each changed file:
1. Run `git log --oneline -10 -- <file>` to get recent commit SHAs
2. Call `mcp__azure-devops__repo_list_pull_requests_by_commits` for those SHAs to find related PRs
3. Call `mcp__azure-devops__repo_list_pull_request_threads` for each discovered PR
4. Call `mcp__azure-devops__repo_list_pull_request_thread_comments` for threads that look relevant
5. Flag any reviewer comments from previous PRs that also apply to the current PR

### Agent 5 — Code comment compliance

Read the full content of each changed file. Check whether the PR's changes comply with guidance in inline code comments (e.g. `// NOTE:`, `// IMPORTANT:`, `// TODO:`, `// WARNING:`). Flag discrepancies.

## Step 5: Confidence scoring (parallel Haiku agents)

For each issue identified in step 4, launch a parallel Haiku agent that scores the issue's confidence on a 0–100 scale:

Provide the agent with: the PR diff, the issue description, and the guidance file list (the agent should read those `CLAUDE.md` and `AGENTS.md` files).

Scoring rubric (provide this verbatim to the agent):

- **0**: Not confident at all. This is a false positive that doesn't stand up to light scrutiny, or is a pre-existing issue.
- **25**: Somewhat confident. This might be a real issue, but may also be a false positive. The agent wasn't able to verify it's real. If stylistic, it was not explicitly called out in the relevant `CLAUDE.md` or `AGENTS.md`.
- **50**: Moderately confident. The agent verified this is a real issue, but it may be a nitpick or rare in practice. Relative to the PR, it's not very important.
- **75**: Highly confident. The agent double-checked and confirmed it is very likely a real issue that will be hit in practice. The existing approach is insufficient. Either very important and directly impacts functionality, or directly mentioned in the relevant `CLAUDE.md` or `AGENTS.md`.
- **100**: Absolutely certain. The agent confirmed it is definitely a real issue that will happen frequently in practice. Evidence directly confirms this.

Filter out any issues scoring below 80.

Examples of false positives (for guidance to scoring agents):
- Pre-existing issues on lines not modified by the PR
- Something that looks like a bug but is not actually a bug
- Pedantic nitpicks a senior engineer wouldn't call out
- Issues a linter, typechecker, or compiler would catch — assume CI handles these
- General code quality issues (lack of test coverage, general security) unless required in CLAUDE.md
- Issues mentioned in `CLAUDE.md` or `AGENTS.md` but explicitly silenced in code (e.g. lint-ignore comment)
- Changes in functionality that are likely intentional or directly related to the broader change
- Real issues, but on lines the PR did not modify

## Step 6: Final eligibility re-check (Haiku agent)

Repeat step 1's eligibility check to confirm the PR is still open and not yet reviewed before posting. Remember: check the `status` field (not `mergeStatus`) — `status` 1=active (proceed), 2=abandoned (skip), 3=completed (skip). `mergeStatus` is unrelated and should be ignored for this check.

If `SKIP` is returned, stop here.

## Step 7: Post review comment

If there are no issues remaining after filtering (step 5), post a "no issues" thread.

If there are issues, post a review thread.

Use `mcp__azure-devops__repo_create_pull_request_thread` to post a top-level comment thread on the PR.

### Azure DevOps file link format

For each issue, construct a file permalink using the **source branch tip SHA**:

```
https://dev.azure.com/{org}/{project}/_git/{repo}?path=/{file}&version=GC{sha}&line={lineStart}&lineEnd={lineEnd}&lineStartColumn=1&lineEndColumn=1&lineStyle=plain&_a=contents
```

- `{file}`: repo-root-relative path with leading `/` (e.g. `/Api/source/src/Api/Controllers/FooController.cs`)
- `{sha}`: full 40-character git SHA of the source branch tip (get via `git rev-parse <sourceBranch>`)
- `{lineStart}` / `{lineEnd}`: provide at least 1 line of context before and after the relevant line(s)
- URL-encode special characters in the path (spaces → `%20`, etc.)

### Comment format (with issues found)

```markdown
### Code review

Found N issue(s):

1. <brief description> (CLAUDE.md / AGENTS.md says "<exact quote>")

<ADO file permalink with line range>

2. <brief description> (bug due to <file and code snippet>)

<ADO file permalink with line range>

🤖 Generated with [Claude Code](https://claude.ai/code)

---
*If this review was useful, please react with 👍. Otherwise, react with 👎.*
```

### Comment format (no issues)

```markdown
### Code review

No issues found. Checked for bugs and CLAUDE.md/AGENTS.md compliance.

🤖 Generated with [Claude Code](https://claude.ai/code)
```

## General notes

- Do not check build signal or attempt to build or typecheck the app
- Always make a todo list first
- You must cite and link each bug
- Keep the final comment brief — one line per issue plus the link
- Do not use `gh` — all PR interactions go through the `mcp__azure-devops__*` tools
- Use `git diff`, `git log`, `git blame`, `git rev-parse`, and `git remote` via Bash for local operations
