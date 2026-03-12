---
name: pr-generator
description: >
  Generate a pull request with an auto-filled description based on code changes and optional JIRA tickets.
  Use this skill whenever the user wants to create a PR, generate a pull request, open a PR, write a PR description,
  or submit their branch for review. Also triggers when the user says "pr", "pull request", "open pr", "submit pr",
  "create pr", "/pr", or mentions generating a PR description. Accepts optional JIRA ticket IDs as arguments
  (e.g. "/pr DOM-123, DOM-456"). If no ticket IDs are provided, the skill will ask the user whether to add them or continue without.
---
 
# PR Generator
 
Generate a pull request with a fully populated description by analyzing the branch's code changes and optional JIRA ticket context.
 
## Invocation
 
The user invokes this skill with optional JIRA ticket IDs:
 
- `/pr` — no tickets, will prompt
- `/pr DOM-123` — single ticket
- `/pr DOM-123, DOM-456` — multiple tickets
 
Parse the arguments to extract ticket IDs. Ticket IDs look like `[A-Z]+-[0-9]+` (e.g. DOM-123, PROJ-4567). If the user provides no ticket IDs, ask:
 
> I don't see any JIRA ticket IDs. Would you like to add ticket IDs (e.g. DOM-123) or continue without them?
 
If the user provides IDs, proceed with JIRA integration. If they decline, skip the JIRA steps entirely.
 
---
 
## Step 1: Validate git state
 
Run these checks before anything else:
 
```bash
# Confirm we're in a git repo
git rev-parse --is-inside-work-tree
 
# Get current branch
CURRENT_BRANCH=$(git branch --show-current)
```
 
If on `main`, `master`, or `develop`, stop and tell the user:
 
> You're on the base branch (`main`). Switch to a feature branch before creating a PR.
 
---
 
## Step 2: Check for existing PR
 
```bash
gh pr view 2>/dev/null
```
 
- If a PR already exists, show the user the PR URL and title. Ask if they want to **update the existing PR description** or **skip PR creation**.
- If updating, fetch the existing PR number with `gh pr view --json number --jq '.number'` and later use `gh pr edit <number> --body "..."`.
- If no PR exists, continue to create one.
 
---
 
## Step 3: Determine base branch
 
```bash
BASE_BRANCH=$(git remote show origin 2>/dev/null | grep 'HEAD branch' | awk '{print $NF}')
```
 
Fallback: try `main`, then `master`, then `develop`. Verify with `git rev-parse --verify origin/$BASE_BRANCH`.
 
Make sure the local repo has up-to-date remote info:
 
```bash
git fetch origin $BASE_BRANCH --quiet
```
 
---
 
## Step 4: Analyze the code changes
 
This is the core of the skill. Gather all the information you need to understand what changed and why.
 
### 4a. File-level summary
 
```bash
git diff --stat origin/$BASE_BRANCH...HEAD
git diff --name-status origin/$BASE_BRANCH...HEAD
```
 
This tells you what files were added, modified, deleted, or renamed — and how much changed in each.
 
### 4b. Commit history
 
```bash
git log --oneline origin/$BASE_BRANCH..HEAD
git log --format="%h %s%n%n%b" origin/$BASE_BRANCH..HEAD
```
 
Read all commit messages and bodies. These are the primary signal for understanding the developer's intent.
 
### 4c. Full diff
 
```bash
git diff origin/$BASE_BRANCH...HEAD
```
 
Read the actual code changes. This is important — commit messages don't always tell the full story. When reviewing the diff:
 
- Focus on **new files** first — they usually contain the core feature/fix.
- Look at **modified files** for the pattern of changes (new parameters, new fields, new conditions).
- Check **test files** to understand the expected behavior.
- Check **configuration or migration files** for infrastructure changes.
- Note **deleted code** to understand what was removed and why.
 
If the diff is very large (>2000 lines), prioritize reading the most-changed files and new files. Use `--stat` output to identify the heaviest files and read those diffs individually:
 
```bash
git diff origin/$BASE_BRANCH...HEAD -- path/to/important/file.ts
```
 
### 4d. Synthesize understanding
 
After reading the commits and diff, form a clear mental model of:
 
1. **What problem is being solved** — infer from the nature of the changes
2. **What approach was taken** — the architectural/technical decisions
3. **What files are most important** — where a reviewer should start
4. **Any risks or notable decisions** — breaking changes, tradeoffs, workarounds
 
---
 
## Step 5: JIRA ticket analysis (if tickets provided)
 
If the user provided JIRA ticket IDs, use the Atlassian/JIRA MCP tools to fetch each ticket:
 
```
getJiraIssue(cloudId, issueIdOrKey)
```
 
For each ticket, read:
- **Summary** and **Description** — the requirement
- **Acceptance criteria** — what "done" looks like (often in the description or a custom field)
- **Comments** — clarifications, design decisions, scope changes
 
Then do a **spec-vs-implementation comparison**:
 
1. List each requirement or acceptance criterion from the ticket(s).
2. For each one, check whether the code changes address it. Reference the specific files or code patterns.
3. Surface any gaps clearly:
 
> **Spec Check:**
> ✅ Add optional query param `id` — implemented in `CollaboratorController.scala`
> ✅ Add `isDirectCollaborator` response field — added to `CollaboratorResponse` model
> ❌ Update API documentation — no doc changes found in this branch
> ⚠️ Add unit tests for new filters — test file modified but only covers `role` filter, not `id` filter
 
Present this to the user and ask them to confirm before proceeding:
 
> Your changes cover most of the acceptance criteria. I found one gap: [describe]. Do you want to proceed with the PR as-is, or address this first?
 
If the user confirms, proceed. The spec check results will inform the PR description content.
 
---
 
## Step 5.5: Gather additional context from the user
 
Before generating the description, always pause and ask the user for anything you can't infer from code alone:
 
> Before I write the PR description, is there anything you'd like me to include? For example:
> - How you tested this manually (endpoints hit, environments used, specific scenarios verified)
> - Why something deviates from the JIRA spec
> - Technical tradeoffs or decisions worth calling out
> - Screenshots (paste or attach them here and I'll include them in the PR)
>
> Or just say "looks good" and I'll generate it from the code and ticket context.
 
**Wait for the user to respond.** Do not proceed until they reply.
 
### Handling screenshots
 
If the user provides screenshots (images attached to their message):
 
1. Save each image to the repo working directory with a descriptive name:
   ```bash
   # Example: save to a temp location for upload
   cp /path/to/uploaded/image.png ./pr-screenshot-1.png
   ```
 
2. After creating the PR, upload the images as PR comments or embed them in the body. GitHub accepts images in PR markdown when uploaded via the web UI, but via CLI the best approach is:
   - Create the PR first with placeholder text like `<!-- screenshot-1 -->` in the relevant section (usually "How was the solution tested?" or "Any background context?")
   - Then use `gh` to add a comment with the screenshots, or instruct the user to paste them into the PR on GitHub since `gh pr create` doesn't natively support image uploads in the body.
 
3. Let the user know: if image embedding via CLI is limited, tell them you've noted where screenshots should go and they can paste them directly in the PR on GitHub.
 
### Incorporating user-provided context
 
Weave whatever the user provides into the relevant sections of the PR template:
- Manual testing details → "How was the solution tested?"
- Spec deviation explanations → "Any background context you want to provide?"
- Technical decisions → "Any background context you want to provide?" or "What problem does this PR solve?" depending on fit
- Screenshots → "How was the solution tested?" or "Any background context?" with image placeholders
 
---
 
## Step 6: Generate the PR description
 
Use the following template exactly. Fill each section based on your analysis from Steps 4, 5, and 5.5.
 
```markdown
### What problem does this PR solve?
 
[1-3 sentence summary of the problem. Then a concise bullet list of the specific changes. If JIRA tickets were provided, reference the business context from the ticket. Be direct — state what changed, not what "this PR does".]
 
### How was the solution tested?
 
[Describe how the changes were tested based on what you see in the diff. Reference test files if they were added/modified. If no tests are visible in the diff, write "Manual testing" and note that automated tests may need to be added. If you see specific test commands or configurations, mention them.]
 
### Where should the reviewer start?
 
[Point to the most important file or change. Use the format: "Start at `path/to/file.ext`" or if you can construct a GitHub URL: "Start at the changes in `FileName.ext`". Pick the file that best explains the core change.]
 
### Any background context you want to provide?
 
[Include context from JIRA tickets if available — why this change is needed from a product/business perspective. Also note any technical decisions, tradeoffs, or things the reviewer should know. If there's nothing notable beyond what's already clear, write "N/A" or a brief one-liner.]
 
### PR Author Checklist
- [ ] Have e2e smoke tests run successfully on your latest commit? [Troubleshoot pre-merge e2e-test runs](https://dominodatalab.atlassian.net/wiki/spaces/ENG/pages/2379579581/Troubleshoot+pre-merge+e2e+test+runs)
- [ ] Is there appropriate test coverage?
    - [ ] Have they run successfully with [GitHub Pull Request Triggers](https://dominodatalab.atlassian.net/wiki/spaces/ENG/pages/2283864109/GitHub+Pull+Request+Triggers)?
- [ ] Has relevant documentation been updated?
- [ ] Have appropriate metrics been added?
- [ ] Has the CODEOWNERS file been updated?
- [ ] Does the code follow our style guides:
    - [Domino Scala Style Guide](https://dominodatalab.atlassian.net/wiki/spaces/ENG/pages/300580964/Domino+Scala+Style+Guide)
    - [Domino Contributing Guide](https://github.com/cerebrotech/domino/blob/develop/CONTRIBUTING.md)
- [ ] Has the JIRA ticket(s) been linked below?
 
### Link to JIRA
 
[For each ticket ID provided, output the full URL: `https://dominodatalab.atlassian.net/browse/TICKET-ID`. If no tickets were provided, write "N/A".]
```
 
### Writing guidelines for the description
 
- **Be concise.** A reviewer will read this quickly. Short sentences, clear bullet points. Don't restate what's obvious from the diff.
- **Be specific.** Reference actual file names, field names, parameter names. Vague summaries like "improved the API" are useless.
- **Don't over-explain.** If a change is straightforward (rename, add a field), one bullet is enough. Save detail for non-obvious decisions.
- **Flow matters.** Someone should read the description top to bottom and come away with a clear picture. Don't dump unstructured notes.
- **Use the ticket context naturally.** If a JIRA ticket explains the "why", weave that into the problem statement — don't just copy-paste ticket text.
 
---
 
## Step 7: Present and create the PR
 
Show the user the complete PR description in a code block so they can review it. Ask:
 
> Here's the PR description I generated. Want me to create the PR with this, or would you like to make changes first?
 
### Creating the PR
 
Generate a concise PR title from the changes. The title should be short (<70 chars) and descriptive — it's what shows up in PR lists. Pattern: `[type]: [brief description]`. Examples:
- `feat: add collaborator filtering to project settings API`
- `fix: resolve null pointer in user sync job`
- `refactor: extract payment validation into shared module`
 
Then create the PR:
 
```bash
gh pr create --title "<title>" --body "<body>"
```
 
Use a heredoc for the body to handle multiline content and special characters:
 
```bash
gh pr create --title "feat: add collaborator filtering" --body "$(cat <<'PRBODY'
### What problem does this PR solve?
...full description here...
PRBODY
)"
```
 
If updating an existing PR instead:
 
```bash
gh pr edit <PR_NUMBER> --body "$(cat <<'PRBODY'
...full description here...
PRBODY
)"
```
 
After creation, display the PR URL to the user.
 
---
 
## Error handling
 
- **No commits ahead of base:** Tell the user their branch has no changes compared to the base branch.
- **Merge conflicts:** If `git merge-base` fails, suggest the user rebase or merge the base branch first.
- **`gh` not authenticated:** If `gh` commands fail with auth errors, tell the user to run `gh auth login`.
- **JIRA not connected:** If JIRA MCP tools are unavailable, inform the user that JIRA integration requires the Atlassian connector and skip ticket analysis gracefully.
- **Very large diffs:** If the diff exceeds what you can reasonably read, focus on new files, the most-changed files, and commit messages. Note in the PR description that the reviewer should pay special attention to specific areas.
 