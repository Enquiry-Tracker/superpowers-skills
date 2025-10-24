---
name: Creating Hotfixes
description: Emergency production fixes with direct-to-main deployment and develop backport workflow
when_to_use: when production is broken and requires immediate fix bypassing normal development workflow
version: 1.0.0
---

# Creating Hotfixes

## Overview

Hotfixes are emergency fixes deployed directly to production, bypassing the normal development branch workflow.

**Core principle:** Fix production fast → Deploy → Backport to develop → Verify alignment.

## When to Use

**Hotfix criteria (ALL must be true):**
- Production is broken or severely degraded
- Issue is affecting users RIGHT NOW
- Revenue/security/data loss is occurring
- Cannot wait for normal release cycle

**Examples:**
- Payment processing down
- Authentication broken
- Data corruption in progress
- Security vulnerability being exploited
- Critical API endpoint returning 500s

## When NOT to Use

**Use normal workflow instead:**
- Dashboard shows wrong numbers (not breaking core functionality)
- UI styling issue
- Feature request labeled "urgent" by stakeholder
- Bug that affects edge case only
- "Important for board meeting tomorrow"

**Red flag phrases that DON'T justify hotfix:**
- "Would be great to have..."
- "Important meeting tomorrow"
- "Customer is asking about..."
- "We should prioritize this"

**If production isn't currently broken, it's not a hotfix.**

## Hotfix Workflow

### Step 1: Verify It's Actually a Hotfix

Ask yourself:
- Is production broken RIGHT NOW?
- Are users unable to complete critical workflows?
- Is money being lost or data at risk?

If any answer is "no" → Use normal workflow (branch from develop, PR to develop)

### Step 2: Create Hotfix Branch

**Branch FROM main/master (not develop):**

```bash
# Ensure you're on latest production code
git checkout main
git pull origin main

# Create hotfix branch
git checkout -b hotfix/ET-XXXX-brief-description

# Example:
# git checkout -b hotfix/ET-1234-fix-payment-webhook-null
```

**Branch naming:** `hotfix/ET-[JIRA-ID]-description`

### Step 3: Implement Minimal Fix

**Keep changes minimal:**
- Fix ONLY the breaking issue
- Avoid refactoring or "while I'm here" changes
- Defer cleanup to post-hotfix follow-up

**Testing:**
- Run existing tests: `npm test`
- Add test for the specific bug if quick (<15 min)
- Manual verification of the fix
- DO NOT skip verification (use verification-before-completion skill)

### Step 4: Bump Version Numbers

**Update all version identifiers for the hotfix:**

```bash
# 1. Update package.json version
cd enrollment_node  # or enrollment_angular
npm version patch  # Increments patch version (e.g., 1.2.3 -> 1.2.4)

# 2. Update build version in constants.ts (backend only)
# Edit src/utils/constants.ts
# Increment the build number:
#   public static build = '0183';  →  public static build = '0184';

# 3. Commit all version changes together
git add package.json package-lock.json src/utils/constants.ts
git commit -m "Bump version for hotfix ET-XXXX"
```

**For frontend hotfixes (enrollment_angular):**
- Only need to bump package.json version
- No constants.ts build version

**For backend hotfixes (enrollment_node):**
- Bump package.json version (npm version patch)
- Increment build in src/utils/constants.ts
- Commit both changes together

**Version bump guidelines:**
- **Patch version** (x.x.X) - Bug fixes, hotfixes (most common)
- **Minor version** (x.X.x) - New features (rare for hotfix)
- **Major version** (X.x.x) - Breaking changes (never for hotfix)
- **Build version** - Increment by 1 (e.g., 0183 → 0184)

**Note:** Hotfixes are almost always patch bumps.

### Step 5: Create PR to Main

**Target branch: main/master (not develop)**

```bash
# Push hotfix branch
git push -u origin hotfix/ET-XXXX-description

# Create PR to main
gh pr create --base main --title "ET-XXXX: [HOTFIX] Brief description" --body "$(cat <<'EOF'
## HOTFIX - Production Issue

**Issue:** [One sentence: what's broken]
**Impact:** [Who's affected, revenue/security/data impact]
**Root cause:** [Brief technical cause]

## Fix
[2-3 bullets on what changed]

## Testing
- [ ] Existing tests pass
- [ ] Manual verification of fix
- [ ] [Specific verification steps]

## Post-Deploy
- [ ] Backport to develop
- [ ] Monitor for 24 hours
EOF
)"
```

### Step 6: Expedited Review & Deploy

**Review:**
- Request immediate review from available team member
- Focus on: Does this fix the issue? Does it break anything else?
- Accept "good enough" over "perfect" for emergency

**Deploy:**
- Follow team's production deployment process
- Monitor deployment closely
- Verify fix in production

### Step 7: Backport to Develop

**Critical: Must happen same day as hotfix deploy**

```bash
# Switch to develop
git checkout develop
git pull origin develop

# Create backport branch
git checkout -b backport/ET-XXXX-from-hotfix

# Cherry-pick hotfix commits
git cherry-pick <hotfix-commit-sha>

# Or merge if multiple commits
git merge hotfix/ET-XXXX-description

# Resolve conflicts if needed
# Run tests
npm test

# Push and create PR to develop
git push -u origin backport/ET-XXXX-from-hotfix

gh pr create --base develop --title "ET-XXXX: Backport hotfix to develop" --body "$(cat <<'EOF'
## Backport from Hotfix

**Original hotfix:** [Link to main PR]
**Production deploy:** [Date/time]

Backporting hotfix changes to develop to keep branches aligned.

## Changes
[Brief description]

## Verification
- [ ] Tests passing
- [ ] No conflicts with develop
- [ ] Same logic as production hotfix
EOF
)"
```

### Step 8: Post-Hotfix Follow-up

**Within 48 hours:**

1. **Monitor production** for 24-48 hours after deploy
2. **Root cause analysis** - Why did this reach production?
3. **Prevention** - What tests/checks would have caught this?
4. **Technical debt** - File ticket for proper fix if hotfix was hacky

## Quick Reference

| Step | Branch | Target | Urgency |
|------|--------|--------|---------|
| 1. Hotfix | Branch from `main` | PR to `main` | Immediate |
| 2. Bump version | `npm version patch` | Include in hotfix | Required |
| 3. Backport | Branch from `develop` | PR to `develop` | Same day |
| 4. Follow-up | Branch from `develop` | PR to `develop` | Within 48h |

## Common Mistakes

**Treating non-emergencies as hotfixes**
- **Problem:** Bypasses code review, testing, normal quality gates
- **Fix:** Use checklist - if production isn't broken now, use normal workflow

**Branching from develop**
- **Problem:** Includes unreleased develop changes in production hotfix
- **Fix:** Always branch from main/master for hotfixes

**Forgetting to bump version**
- **Problem:** Deployment tracking unclear, can't correlate logs/errors to specific release
- **Fix:** Backend: bump package.json AND constants.ts build. Frontend: bump package.json only.

**Forgetting to backport**
- **Problem:** Next release from develop doesn't include hotfix, reintroduces bug
- **Fix:** Backport same day as hotfix. Add to hotfix checklist.

**Scope creep during hotfix**
- **Problem:** "While I'm here" refactoring delays critical fix
- **Fix:** Minimal change only. File follow-up ticket for improvements.

**Skipping tests**
- **Problem:** Hotfix introduces new bug
- **Fix:** Run test suite, add regression test if quick, manual verification

## Real-World Impact

**Cost of skipped backport:**
- Hotfix deployed Friday
- Team forgot to backport
- Normal release Monday included develop code without hotfix
- Bug reappeared in production
- Second emergency deploy required

**Cost of false hotfix:**
- Non-critical bug treated as hotfix
- Bypassed code review
- Introduced new bug
- Real emergency distracted from fixing previous "emergency"

## Red Flags

**Never:**
- Skip version bump
- Skip backport to develop
- Treat feature requests as hotfixes
- Refactor during hotfix
- Deploy without testing
- Branch hotfix from develop

**Always:**
- Verify it's truly production-breaking
- Branch from main/master
- Bump version with `npm version patch`
- PR to main/master first
- Backport same day
- Monitor post-deploy
- Follow up within 48h

## Integration

**Pairs with:**
- skills/collaboration/finishing-a-development-branch (for backport completion)
- skills/debugging/verification-before-completion (pre-deploy verification)
- skills/debugging/systematic-debugging (root cause analysis)
