# /acp - Add, Commit & Push

Streamlined git workflow: stage all changes, generate a conventional commit message, and push.

## Arguments

- $--edit: Manually edit the generated commit message before committing
- $--dry-run: Preview the commit message and actions without executing
- $--no-push: Commit only, skip pushing
- $--force: Allow force-push with --force-with-lease (will ask confirmation)

## Usage Examples

```bash
# Basic: stage, commit with auto-message, push
/acp

# Preview without executing
/acp --dry-run

# Edit the generated message before committing
/acp --edit

# Commit only, don't push
/acp --no-push

# Allow force-push if needed (will still ask confirmation)
/acp --force

# Combine flags
/acp --edit --no-push
```

## Instructions

Execute this workflow with minimal output (only final commit message and result).

### Step 1: Pre-flight Checks

1. Verify we're in a git repository. If not, exit with error.
2. Check for uncommitted changes using "git status --porcelain". If no changes, exit with message: "Nothing to commit, working tree clean."
3. Check for merge conflicts (files with conflict markers). If found:
   - Attempt to understand the conflicts
   - Try to resolve them logically based on the context
   - If resolution is ambiguous, show the conflicts and ask the user how to proceed
   - After resolution, stage the resolved files

### Step 2: Protected Branch Check

Check current branch with "git branch --show-current".

If on "master", "main", or "develop":
- Ask user for confirmation: "You're on [branch]. Are you sure you want to commit and push directly to this branch?"
- If user declines, exit gracefully
- If --dry-run, just note this would require confirmation

### Step 3: Stage Changes

Run "git add ." to stage all changes.

### Step 4: Analyze Changes & Generate Commit Message

Analyze the staged changes using "git diff --cached --stat" and "git diff --cached" to understand:
- Which files changed and their purpose
- The nature of changes (new feature, bug fix, refactor, etc.)
- Potential scope based on directory/module structure

**Generate a Conventional Commit message following these rules:**

#### Type (required)

Choose the most appropriate:
- feat: New feature or capability
- fix: Bug fix
- docs: Documentation only
- style: Code style (formatting, semicolons, etc.)
- refactor: Code restructuring without behavior change
- perf: Performance improvement
- test: Adding or updating tests
- chore: Maintenance tasks, dependencies
- ci: CI/CD configuration
- build: Build system or external dependencies

#### Scope (auto-detect)

Infer from changed files:
- Component/module name (e.g., auth, api, ui)
- Directory-based (e.g., config, utils, models)
- Omit if changes span multiple unrelated areas

#### Breaking Change Detection

If changes include:
- Removed or renamed public functions/methods/classes
- Changed function signatures or return types
- Removed configuration options
- Database schema changes requiring migration
- API endpoint changes

Then:
- Add "!" after type/scope (example: feat(api)!:)
- Include a BREAKING CHANGE: footer explaining the impact

#### Message Format

```
<type>(<scope>): <subject>

<body>

[BREAKING CHANGE: <explanation>]
```

**Subject line rules:**
- Imperative mood ("Add feature" not "Added feature")
- No period at the end
- Max 50 characters
- Clear, value-focused (what does this enable?)

**Important:**
- Do NOT add any Co-authored-by trailers
- Do NOT include any AI attribution or mention of Claude
- The commit should appear as if written entirely by the user

**Body rules:**
- Brief: 2-4 bullet points maximum
- Focus on the "why" and business/feature value
- Use simple, clear English (accessible to non-native speakers)
- Skip obvious details visible in the diff

### Step 5: Present Commit Message

Display the generated commit message to the user.

If --dry-run:
- Show the message and what would happen
- Show files that would be committed
- Exit without making changes

If --edit:
- Ask user for any modifications they want
- Apply the changes to the message

Otherwise:
- Show the message and proceed

### Step 6: Commit

Run git commit with the message (use -m for subject and -m for body if body exists).

### Step 7: Push

If --no-push: Skip this step, show success message and exit.

1. Check if remote branch exists using "git ls-remote --heads origin [branch]"

2. If branch doesn't exist remotely:
   - Run "git push --set-upstream origin [branch]"

3. If push fails due to diverged history and --force flag is present:
   - Always ask confirmation: "Remote has diverged. Force push with --force-with-lease? This will overwrite remote changes."
   - If confirmed: run "git push --force-with-lease"
   - If declined: exit with message suggesting "git pull" first

4. If push fails for other reasons, show the error clearly.

### Output Format

Keep output minimal. Final output should look like:

```
✓ feat(auth): Add OAuth2 login support

  - Enable Google and GitHub authentication
  - Add session persistence for returning users

→ Pushed to origin/feature/oauth (3 files changed)
```

Or for dry-run:

```
[DRY RUN] Would commit and push:

feat(auth): Add OAuth2 login support

  - Enable Google and GitHub authentication
  - Add session persistence for returning users

Files: src/auth/oauth.ts, src/auth/providers.ts, README.md
Branch: feature/oauth → origin/feature/oauth
```
