# /pr - Create Pull Request

Create a GitHub pull request for the current branch with auto-generated title and description.

## Arguments

- `--base <branch>` (optional): Target branch for the PR. Defaults to the repository's default branch (main/master).

## Instructions

Execute the following workflow step by step. Show progress for each step.

### Step 1: Preflight Checks

1. Verify `gh` CLI is installed and authenticated:
   ```bash
   gh auth status
   ```
   If not authenticated, exit with: "❌ GitHub CLI not authenticated. Run `gh auth login` first."

2. Ensure we're in a git repository:
   ```bash
   git rev-parse --is-inside-work-tree
   ```

3. Get the current branch name:
   ```bash
   git branch --show-current
   ```
   If on `main` or `master`, exit with: "❌ Cannot create PR from default branch. Switch to a feature branch first."

### Step 2: Determine Base Branch

1. If `--base` argument is provided, use that value.
2. Otherwise, detect the default branch:
   ```bash
   gh repo view --json defaultBranchRef --jq '.defaultBranchRef.name'
   ```

Display: "🎯 Target branch: `<base_branch>`"

### Step 3: Check for Existing PR

Check if a PR already exists for this branch:
```bash
gh pr view --json url,state 2>/dev/null
```

If a PR exists:
- Display: "ℹ️ PR already exists for this branch: <url>"
- Exit gracefully (not an error).

### Step 4: Check for Commits

Get commits ahead of base branch:
```bash
git log <base_branch>..HEAD --oneline
```

If no commits found, exit with: "❌ No commits ahead of `<base_branch>`. Nothing to create a PR for."

Display: "📝 Found <n> commit(s) to include"

### Step 5: Gather Context

1. **Get commit messages**:
   ```bash
   git log <base_branch>..HEAD --pretty=format:"%s%n%b"
   ```

2. **Parse branch name** using pattern `feature/<project>-<id>-<description>`:
   - Extract `<project>` (uppercase for issue reference)
   - Extract `<id>` (issue number)
   - Extract `<description>` (convert hyphens to spaces, title case)

3. **Get list of changed files**:
   ```bash
   git diff <base_branch>..HEAD --name-only
   ```

### Step 6: Generate PR Title

Create a concise, descriptive title by:
1. Starting with the humanized branch description (e.g., "add-favorites-article" → "Add favorites article")
2. Refining it based on commit message content if it provides more clarity
3. Keep it under 72 characters
4. Use sentence case (capitalize first word only)

**Do NOT include any reference to AI, Claude, or automated generation.**

### Step 7: Generate PR Description

Create the PR body with these sections:

```markdown
## Summary

<2-3 sentences describing what this PR does, synthesized from commits>

## Value Delivered

<1-2 sentences explaining the feature or value this brings to users/the product>

## Changes

<Bullet list of key changes, grouped logically — NOT just listing files>

## Testing Notes

<Suggestions for how to test this PR — infer from the type of changes>

---

Closes #<PROJECT>-<ID>
```

Rules:
- Write in clear, professional, human language
- **Do NOT mention AI, Claude, automation, or that this was generated**
- Be specific about what changed and why it matters
- Keep it concise but informative

### Step 8: Create the Pull Request

1. Get current GitHub username:
   ```bash
   gh api user --jq '.login'
   ```

2. Create the PR:
   ```bash
   gh pr create \
     --base <base_branch> \
     --title "<generated_title>" \
     --body "<generated_body>" \
     --assignee "@me"
   ```

   Note: `--assignee "@me"` assigns to the authenticated user.

3. Capture and display the PR URL.

### Step 9: Final Output

Display:
```
✅ Pull request created successfully!

   <PR_URL>

   Title: <title>
   Base:  <base_branch> ← <current_branch>
   Assigned to: <username>
```

## Error Handling

| Scenario | Message |
|----------|---------|
| `gh` not installed | "❌ GitHub CLI (`gh`) is not installed. Install from https://cli.github.com" |
| Not authenticated | "❌ GitHub CLI not authenticated. Run `gh auth login` first." |
| Not a git repo | "❌ Not inside a git repository." |
| On default branch | "❌ Cannot create PR from default branch. Switch to a feature branch first." |
| No commits ahead | "❌ No commits ahead of `<base>`. Nothing to create a PR for." |
| PR already exists | "ℹ️ PR already exists for this branch: <url>" (exit 0) |
| API/network error | "❌ Failed to create PR: <error_message>" |

## Examples

```bash
# Create PR to default branch (main/master)
/pr

# Create PR to a specific branch
/pr --base develop

# Create PR to a release branch
/pr --base release/v2.0
```
