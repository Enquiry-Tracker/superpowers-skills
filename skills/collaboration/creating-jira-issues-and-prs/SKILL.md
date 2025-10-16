---
name: Creating Jira Issues and Bitbucket PRs
description: Set up jira CLI and create issues with Bitbucket PRs for completed work
when_to_use: when work is complete and ready for review, needs tracking in Jira and code review via Bitbucket pull requests
version: 1.0.0
languages: all
dependencies: jira CLI (go-jira), git, Bitbucket
---

# Creating Jira Issues and Bitbucket PRs

## Overview

Use the jira CLI tool to create Jira issues and link them with Bitbucket pull requests for code review. This workflow ensures traceability between issue tracking and code changes.

## When to Use

- Work is complete and committed to git
- Need to create Jira issue to track the work
- Need to open Bitbucket PR for code review
- First time using jira CLI (includes setup instructions)

**Do NOT use when:**
- Using GitHub (this is for Bitbucket/Jira workflows)
- Issue already exists (just link to existing issue)

## Quick Reference

| Task | Command |
|------|---------|
| Install jira CLI | `brew install go-jira` |
| Check auth | `jira session` |
| Create issue | `jira create --noedit -t template-name` |
| List issues | `jira ls --query "project = XX"` |
| View issue | `jira view ISSUE-123` |

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

**Get API token:** https://id.atlassian.com/manage-profile/security/api-tokens

The jira CLI stores credentials in your system keyring after first authentication.

## Creating Issues with Templates

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

```
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

## Complete Workflow: Code → Issue → PR

### 1. Commit Your Changes

```bash
git add .
git commit -m "Your commit message"
git push origin your-branch
```

### 2. Create Jira Issue

Prepare issue template with:
- Summary of changes
- Security impact / priority
- Files modified
- Testing results
- Branch name and commit hash

```bash
export JIRA_API_TOKEN='your-token'
jira create --noedit -t your-template
```

**Capture the issue key** (e.g., ET-1234)

### 3. Create Bitbucket PR

1. Push branch to Bitbucket (if not already pushed)
2. Create PR via Bitbucket web UI or URL pattern:
   - `https://bitbucket.org/org/repo/pull-requests/new?source=branch-name&t=1`
3. Link Jira issue in PR description:
   - Format: `Closes ET-1234` or `Fixes ET-1234`
   - Bitbucket auto-links when it sees issue keys

### 4. Cross-Link

Add PR URL to Jira issue (optional but helpful):
- Comment with PR link
- Or update description with link

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

## Real-World Example

From our security validation work:

1. **Committed changes:** 9 files with input validation fixes
2. **Created template:** `~/.jira.d/templates/create-security-issue`
3. **Created issue:** `jira create --noedit -t create-security-issue`
4. **Result:** ET-8765 created with full context
5. **Created PR:** Used Bitbucket URL pattern with branch name
6. **Linked:** Added PR link to Jira issue description

**Time saved:** 5-10 minutes vs manual issue creation in web UI
