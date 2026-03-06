Pick up the next open GitHub/GitLab issue from the repo in the current directory.

1. Run `git remote get-url origin` to determine the platform:
   - Contains `github.com` → use GitHub MCP
   - Contains `gitlab.com` or self-hosted GitLab → use GitLab MCP

2. Fetch open issues sorted by label priority (high → medium → low)
   Pick the first unassigned issue, or one labeled "next"

3. Create a new branch: `feature/issue-{number}-{short-description}`

4. Implement the fix or feature

5. Write tests

6. Update CHANGELOG.md:
   - If CHANGELOG.md doesn't exist, create it with [Keep a Changelog](https://keepachangelog.com) format
   - Add entry under `## [Unreleased]` section
   - Use the appropriate category: `Added` / `Changed` / `Fixed` / `Removed`
   - Format: `- {short description} (#{issue number})`
   - Keep existing entries intact, only prepend the new one

7. Commit with message: `fix #{number}: {issue title}`
   - Include CHANGELOG.md in the commit

8. Summarize what was done and which files were changed

Determine the repo using `git remote get-url origin`.
