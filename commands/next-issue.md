Pick up the next open GitHub/GitLab issue from the repo in the current directory.

**Input:** $ARGUMENTS (optional issue number, e.g., "42" or "#42")

1. Run `git remote get-url origin` to determine the platform:
   - Contains `github.com` → use GitHub MCP
   - Contains `gitlab.com` or self-hosted GitLab → use GitLab MCP

2. Fetch open issue:
  - If `$ARGUMENTS` is provided: fetch that specific issue
  - If `$ARGUMENTS` is empty: fetch open issues sorted by label priority (high → medium → low), pick the first unassigned issue or one labeled "next"

3. Create a new branch: `feature/issue-{number}-{short-description}`
  - Use Git worktree for this

4. Implement the fix or feature

5. Write tests
  - Add the tests in a file name reflecting the class/module it was found in
  - Make sure tests pass

6. Update CHANGELOG.md:
   - If CHANGELOG.md doesn't exist, create it with [Keep a Changelog](https://keepachangelog.com) format
   - Add entry under `## [Unreleased]` section
   - Use the appropriate category: `Added` / `Changed` / `Fixed` / `Removed`
   - Format: `- {short description} (#{issue number})`
   - Keep existing entries intact, only prepend the new one

7. Commit with message: `fix #{number}: {issue title}`
   - Include CHANGELOG.md in the commit
   - Push branch
   - Open PR using bot token: run `GH_TOKEN=$(generate-gh-token.sh) gh pr create` with title "fix #{number}: {title}"
   - Write the plan for the feature or fix as description for the PR

8. Summarize what was done and which files were changed

Determine the repo using `git remote get-url origin`.

9. Cleanup local worktree branches
