---
name: Creating Jira Issues and Bitbucket PRs
description: Create or reuse Jira issues and link them with Bitbucket PRs
when_to_use: when work is complete and needs tracking in Jira and code review via Bitbucket pull requests
version: 1.3.0
languages: all
dependencies: jira CLI (go-jira), git, Bitbucket
---

# Creating Jira Issues and Bitbucket PRs

## Overview

Use jira CLI to create or reuse Jira issues and link them with Bitbucket pull requests. Check for existing issues first to avoid duplication, especially for similar work across repositories.

## When to Use

- Work is complete and committed to git
- Need to track work in Jira
- Need to open Bitbucket PR for code review
- First time using jira CLI (includes setup instructions)

**Do NOT use when:**
- Using GitHub (this is for Bitbucket/Jira workflows)

## Quick Reference

| Task | Command |
|------|---------|
| Install jira CLI | `brew install go-jira` |
| Check auth | `jira session` |
| Search issues | `jira ls --query "text ~ 'keywords'"` |
| Search by summary | `jira ls --query "summary ~ 'improve CLAUDE'"` |
| View issue | `jira view ISSUE-123` |
| Create issue | `jira create --noedit -t template-name` |
| List recent issues | `jira ls --query "project = XX ORDER BY created DESC"` |

## Initial Setup (One-Time)

### 1. Install jira CLI

```bash
brew install go-jira
```

### 2. Create Configuration

Create `~/.jira.d/config.yml`:

```yaml
endpoint: https://yourcompany.atlassian.net
user: your-email@company.com
password-source: keyring
project: XX  # Default project key
```

### 3. Authenticate

```bash
export JIRA_API_TOKEN='your-api-token'
echo $JIRA_API_TOKEN | jira session
```

**Get API token:** [https://id.atlassian.com/manage-profile/security/api-tokens](https://id.atlassian.com/manage-profile/security/api-tokens)

Jira CLI stores credentials in your system keyring after first authentication.

## Create Issues with Templates

### 1. Create Template Directory

```bash
mkdir -p ~/.jira.d/templates
```

### 2. Create YAML Template

Example template at `~/.jira.d/templates/bug-fix`:

```yaml
fields:
  project:
    key: ET
  issuetype:
    name: Bug
  summary: "Your issue summary here"
  priority:
    name: Highest
  description: |
    h2. Summary
    Brief overview of the issue

    h2. Description
    Detailed description using Jira wiki markup

    h3. What Changed
    * Bullet point 1
    * Bullet point 2

    h3. Files Modified
    * path/to/file1.ts
    * path/to/file2.ts

    h3. Testing
    All tests pass - provide evidence

    h3. Related Links
    * Branch: branch-name
    * Commit: commit-hash
```

### 3. Create Issue from Template

```bash
export JIRA_API_TOKEN='your-token'
jira create --noedit -t bug-fix
```

**Output:** `OK ET-1234 https://company.atlassian.net/browse/ET-1234`

## Jira Wiki Markup Reference

Common formatting for issue descriptions:

```text
h1. Heading 1
h2. Heading 2
h3. Heading 3

*bold text*
_italic text_
{{monospace}}
{code}code block{code}

* Bullet list item
# Numbered list item

[Link text|http://url.com]
```

## Complete Workflow: Issue → Branch → Code → PR

### 1. Check for Existing Issues First

**IMPORTANT:** Before creating a new issue, search for existing ones to avoid duplication.

```bash
# Search by keywords in summary and description
export JIRA_API_TOKEN='your-token'
jira ls --query "text ~ 'improve CLAUDE' AND project = ET"

# Search recent issues
jira ls --query "project = ET ORDER BY created DESC" | head -10

# View specific issue to check if reusable
jira view ET-8772
```

**When to Reuse an Existing Issue:**
- Similar work across different repositories (e.g., ET-8772 and ET-8773 both improve CLAUDE.md)
- Same logical task split into multiple PRs
- Follow-up work directly related to original issue
- Documentation updates applied to multiple repos

**When to Create a New Issue:**
- Completely different work, even if similar topic
- Different project goals or acceptance criteria
- Requires separate tracking/reporting
- Different priority or timeline

### 2. Create New Jira Issue (if needed)

If no existing issue fits, create one:

```bash
# Prepare issue template describing the work
export JIRA_API_TOKEN='your-token'
jira create --noedit -t your-template
```

**Capture the issue key** (e.g., ET-1234)

### 3. Create Feature Branch with Issue Prefix

**IMPORTANT:** Branch names must start with the issue key

```bash
# Create feature branch from develop/main
git checkout develop
git pull origin develop
git checkout -b ET-1234-short-description
```

**Branch naming pattern:** `ISSUE-KEY-brief-description`
- Example: `ET-8765-input-validation-security-fixes`
- Lowercase with hyphens
- Keep description concise but meaningful

**For reused issues across repos:**
- Use same issue key in different repositories
- Example: ET-8772 used for CLAUDE.md improvements in both `superpowers-skills` and other repos
- Branch name stays consistent: `ET-8772-improve-claude-md`

### 4. Make Changes and Commit

```bash
# Make your code changes
git add .
git commit -m "Your descriptive commit message"
git push -u origin ET-1234-short-description
```

### 5. Create Bitbucket PR

**PR Title Format:** `ET-1234: Description of changes`
- **Always prefix with issue key and colon**
- Example: `ET-8765: Add comprehensive input validation to all public endpoints`

**PR Description Format:**
```markdown
**Closes ET-1234**

## Summary
Brief overview of changes

## What Changed
- Bullet point 1
- Bullet point 2

## Testing
All tests pass

**Jira:** https://company.atlassian.net/browse/ET-1234
**Branch:** ET-1234-branch-name
**Commit:** commit-hash
```

**For reused issues:**
- Use `Relates to ET-1234` instead of `Closes ET-1234` for subsequent PRs
- Example: Second PR uses "Relates to ET-8772" since first PR already closed it
- All PRs still link to the issue, maintaining traceability

**Create PR:**
1. Via URL: `https://bitbucket.org/org/repo/pull-requests/new?source=ET-1234-branch-name&dest=develop&t=1`
2. Or via Bitbucket web UI

**Auto-linking keywords:**
- `Closes ET-1234` - Closes issue when PR merges (use for first/only PR)
- `Fixes ET-1234` - Same as Closes
- `Resolves ET-1234` - Same as Closes
- `Relates to ET-1234` - Links without closing (use for additional PRs)

### 6. Verify Auto-Linking

Check that:
- PR shows in Jira issue's "Development" section
- Issue key appears as link in PR
- Status transitions work (if configured)
- Multiple PRs from different repos all link to same issue (when reusing)

## Common Mistakes

### ❌ Using jira create Without --noedit

**Problem:** Opens editor (vim) which hangs in non-interactive contexts

**Fix:** Always use `--noedit` flag with templates

### ❌ Forgetting to Export JIRA_API_TOKEN

**Problem:** Authentication fails with "EOF" error

**Fix:** Export token in same command block:
```bash
export JIRA_API_TOKEN='token' && jira create ...
```

### ❌ Template Not Found Error

**Problem:** Template file not in `~/.jira.d/templates/`

**Fix:** Check template location and use filename without extension

### ❌ Using Markdown in Jira Descriptions

**Problem:** Jira uses wiki markup, not Markdown

**Fix:** Convert to Jira wiki markup:
- `##` → `h2.`
- `**bold**` → `*bold*`
- ` ```code``` ` → `{code}code{code}`

### ❌ Missing Project Key in Config

**Problem:** Must specify project for every create

**Fix:** Add `project: XX` to `~/.jira.d/config.yml`

### ❌ PR Title Without Issue Key Prefix

**Problem:** PR title doesn't start with issue key (e.g., "Add validation" instead of "ET-1234: Add validation")

**Fix:** Always prefix PR title with `ISSUE-KEY:` format
- ✅ Good: `ET-8765: Add comprehensive input validation`
- ❌ Bad: `Add comprehensive input validation`

### ❌ Branch Not Named After Issue

**Problem:** Branch name doesn't include issue key, breaking traceability

**Fix:** Always create branch starting with issue key:
```bash
git checkout -b ET-1234-descriptive-name
```

### ❌ Creating Duplicate Issues for Similar Work

**Problem:** Creating new issues for the same logical task across repos (e.g., ET-8772 and ET-8773 both improve CLAUDE.md)

**Fix:** Search first, reuse existing issues when appropriate:
```bash
jira ls --query "text ~ 'improve CLAUDE' AND project = ET"
```

**When to reuse:** Same work across repos, related PRs, follow-up work
**When to create new:** Different goals, different tracking needs

## Troubleshooting

### Session Authentication Issues

If `jira session` fails:

1. Check config file exists: `cat ~/.jira.d/config.yml`
2. Verify endpoint is correct (no trailing slash)
3. Generate fresh API token
4. Try interactive login: `jira login`

### Template Issues

If template doesn't work:

1. Validate YAML syntax (indentation matters)
2. Check field names match Jira fields
3. Test with minimal template first
4. Use `jira create --help` to see available fields

### Bitbucket Auto-Linking

For Jira issues to auto-link in Bitbucket:

1. Use exact issue key format (PROJECT-123)
2. Include in PR title or description
3. Use keywords: Closes, Fixes, Resolves
4. Ensure Jira-Bitbucket integration is enabled

## Real-World Examples

### Example 1: New Issue for Security Work

1. **Created Jira issue:** `jira create --noedit -t create-security-issue`
   - Result: ET-8765 created
2. **Created feature branch:** `git checkout -b ET-8765-input-validation-security-fixes`
3. **Made changes:** Modified 9 files with input validation fixes
4. **Committed and pushed:** `git commit` and `git push -u origin ET-8765-input-validation-security-fixes`
5. **Created PR with proper format:**
   - Title: `ET-8765: Add comprehensive input validation to all public endpoints`
   - Description started with: `**Closes ET-8765**`
   - Auto-linked to Jira issue
6. **Result:** Full traceability from issue → branch → commits → PR

### Example 2: Reusing Issue Across Repos

1. **Searched for existing issues:**
   ```bash
   jira ls --query "text ~ 'improve CLAUDE' AND project = ET"
   # Found: ET-8772 - Improve CLAUDE.md clarity following Elements of Style
   ```
2. **Viewed issue to confirm it fits:**
   ```bash
   jira view ET-8772
   # Confirmed: Same work (applying Elements of Style to CLAUDE.md)
   ```
3. **Reused ET-8772 in second repo:**
   - First PR in `superpowers-skills`: Used `Closes ET-8772`
   - Second PR in `enrollment_node`: Used `Relates to ET-8772`
4. **Result:**
   - Single issue tracks related work across repos
   - Both PRs visible in ET-8772's Development section
   - Avoided duplicate issue (ET-8773 would have been unnecessary)

**Time saved:** 5-10 minutes per issue vs manual creation
**Bonus:** Reduced issue clutter, better organization
