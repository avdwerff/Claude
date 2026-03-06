Create a new issue for the repo in the current directory.

Arguments: $ARGUMENTS

1. Run `git remote get-url origin` to determine the platform:
   - Contains `github.com` → use GitHub MCP
   - Contains `gitlab.com` or self-hosted GitLab → use GitLab MCP
2. Use the argument as the issue title
3. Ask me for:
   - A short description
   - Labels (bug / enhancement / question)
   - Priority (high / medium / low) → add as label
4. Create the issue via the appropriate MCP
5. Output the issue number and URL
