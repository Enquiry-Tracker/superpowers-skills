---
name: Resizing Cloud SQL Databases
description: Safe procedure for changing Cloud SQL instance resources (CPU/RAM) without data loss
when_to_use: when needing to increase or decrease Cloud SQL database CPU or memory, during cost optimization, or responding to performance issues
version: 1.1.0
languages: bash, gcloud
dependencies: gcloud CLI, Cloud SQL Admin API access
---

# Resizing Cloud SQL Databases

## Overview

**Resizing Cloud SQL instances requires downtime and carries data risk if not done properly.**

Core principle: **Safety steps are MANDATORY regardless of urgency.** Production pressure, time constraints, and existing outages do NOT justify skipping backup, service shutdown, or verification steps.

**EMERGENCIES REQUIRE MORE PROCEDURE, NOT LESS.** When production is down, CEO is escalating, and users are locked out - that's when you follow the checklist most carefully. Pressure is not permission to skip steps.

## When to Use

Use this skill when:
- Increasing database resources due to memory/CPU pressure
- Decreasing resources for cost optimization
- Responding to OOM errors or performance degradation
- Finance requests database downsizing

## The Non-Negotiable Checklist

**ALL 9 steps required. NO exceptions. NO emergencies. NO shortcuts.**

**If you skip ANY step, you are violating this skill. Period.**

- ❌ "Services are down, skip maintenance page" - ASK FOR CONFIRMATION ANYWAY
- ❌ "Production is already down" - STOP SERVICES ANYWAY
- ❌ "We need to move fast" - CREATE BACKUP ANYWAY
- ❌ "Automatic backups exist" - CREATE MANUAL BACKUP ANYWAY
- ❌ "This is a safe operation" - FOLLOW ALL 9 STEPS ANYWAY
- ❌ "Taking backup adds X minutes of downtime" - WAIT FOR BACKUP ANYWAY
- ❌ "CEO is calling me" - FOLLOW CHECKLIST ANYWAY
- ❌ "All users are locked out" - THIS IS WHY YOU FOLLOW PROCEDURE

**There is NO "emergency exception" clause in this skill. If you think you found one, you're rationalizing.**

### 1. Enable Frontend Maintenance Mode FIRST

**BEFORE doing ANYTHING else, ask the user to confirm maintenance mode is enabled.**

**Action:** Verify https://app.enquirytracker.net/ displays "Maintenance Mode" page.

**Why MANDATORY:**
- Users should see maintenance page, not error messages
- Frontend must show status before backend goes down
- Provides user communication during the resize
- Prevents support tickets from confused users seeing 500 errors

**Do NOT proceed to Step 2 until user confirms maintenance mode is active.**

**Rationalization counter:** "Services are already down, skip this!" → Users still need to see maintenance page, not errors.
**Rationalization counter:** "This is wasting time!" → 30 seconds to ask prevents hours of user confusion.

### 2. Stop Services

```bash
# Stop all services that connect to the database
gcloud app versions stop [VERSION] --service=default --project=[PROJECT]
gcloud app versions stop [VERSION] --service=background-tasks --project=[PROJECT]
```

**Why MANDATORY:**
- Instance restart kills active connections
- In-flight transactions will be rolled back
- Connection pool exhaustion during reconnect can cascade
- Clean shutdown prevents data inconsistency

**Rationalization counter:** "But the system is already down!" → A controlled shutdown is different from crash-state connections. Stop services anyway.

### 3. Create Manual Backup

```bash
# CREATE NEW BACKUP - not "check existing", CREATE NEW
gcloud sql backups create \
  --instance=[INSTANCE] \
  --project=[PROJECT] \
  --description="before [operation description]"
```

**Why MANDATORY:**
- Automatic backups may not exist, may be old, may have failed
- You need a known-good state from THIS MOMENT before the change
- Rollback without CURRENT backup = potential data loss
- Verification that backup system is working RIGHT NOW

**Rationalization counter:** "Cloud SQL has automatic backups!" → CREATE NEW BACKUP. Not check, CREATE.
**Rationalization counter:** "I verified backups exist!" → CREATE NEW BACKUP FROM THIS MOMENT.

### 4. Wait for Backup Completion

```bash
# Do not proceed until backup completes
gcloud sql operations list --instance=[INSTANCE] --project=[PROJECT] --limit=5
```

**Why MANDATORY:**
- Backup must finish before you modify the instance
- Incomplete backup = no safety net
- 2-5 minutes is acceptable for production safety

**Rationalization counter:** "This adds downtime!" → Downtime with safety net is better than quick change with risk.

### 5. Execute Resize

```bash
# For custom machine types, specify both CPU and memory
gcloud sql instances patch [INSTANCE] \
  --cpu=[COUNT] \
  --memory=[SIZE]GB \
  --project=[PROJECT]
```

**Why this way:**
- Custom machine types require both --cpu and --memory flags
- Omitting either flag causes error
- Operation triggers automatic instance restart

### 6. Wait for Operation Completion

```bash
# Monitor until DONE
gcloud sql operations wait [OPERATION_ID] --project=[PROJECT]

# Or poll
gcloud sql operations list --instance=[INSTANCE] --project=[PROJECT] --limit=1
```

**Why MANDATORY:**
- Need to know when instance is available
- Typically 2-8 minutes for tier change
- Starting services too early = connection failures

### 7. Verify Configuration Applied

```bash
# Confirm new tier
gcloud sql instances describe [INSTANCE] \
  --project=[PROJECT] \
  --format="value(settings.tier)"
```

**Why MANDATORY:**
- Confirms the change actually applied
- Catches configuration mistakes before restarting services
- Documents new state for rollback if needed

### 8. Restart Services

```bash
# Start services in order
gcloud app versions start [VERSION] --service=default --project=[PROJECT]
gcloud app versions start [VERSION] --service=background-tasks --project=[PROJECT]
```

**Why this order:**
- Default service (API) first so frontend has backend
- Background tasks second to avoid queue buildup
- Verify each starts successfully before proceeding

### 9. Verify Services Running

```bash
# Confirm both services are SERVING
gcloud app versions list --service=default --project=[PROJECT]
gcloud app versions list --service=background-tasks --project=[PROJECT]
```

**Why MANDATORY:**
- Services may start but not serve traffic
- Confirms 100% traffic routing to started version
- Final verification before declaring success

## Quick Reference

| Step | Command | Wait? |
|------|---------|-------|
| 1. Enable maintenance mode | Ask user to confirm app.enquirytracker.net shows "Maintenance" | YES - user confirmation |
| 2. Stop services | `gcloud app versions stop` | No |
| 3. Create backup | `gcloud sql backups create` | YES - wait for DONE |
| 4. Wait for backup | `gcloud sql operations list` | YES - wait for DONE |
| 5. Patch instance | `gcloud sql instances patch` | YES - wait for DONE |
| 6. Wait for patch | `gcloud sql operations wait` | YES - wait for DONE |
| 7. Verify config | `gcloud sql instances describe` | No |
| 8. Start services | `gcloud app versions start` | YES - wait for SERVING |
| 9. Verify services | `gcloud app versions list` | No |

## Real-World Example

From production maintenance 2025-10-24:

```bash
# 1. Enable maintenance mode
# Asked user: "Please confirm https://app.enquirytracker.net/ shows 'Maintenance Mode' before proceeding"
# User confirmed: Maintenance page is live ✓

# 2. Stop services
gcloud app versions stop 20251020t064815 --service=background-tasks --project=astute-baton-177602
gcloud app versions stop 20251020t064803 --service=default --project=astute-baton-177602

# 3. Create backup
gcloud sql backups create \
  --instance=enquiry-db-prod \
  --location=australia-southeast2 \
  --description="before reduce RAM from 10 to 5gb" \
  --project=astute-baton-177602

# 4. Wait for backup (output showed "done")

# 5. Patch instance
gcloud sql instances patch enquiry-db-prod \
  --cpu=4 \
  --memory=5GB \
  --project=astute-baton-177602

# 6. Wait for operation
gcloud sql operations wait [operation-id] --project=astute-baton-177602

# 7. Verify
gcloud sql instances describe enquiry-db-prod \
  --project=astute-baton-177602 \
  --format="value(settings.tier)"
# Result: db-custom-4-5120 ✓

# 8. Start services
gcloud app versions start 20251020t064803 --service=default --project=astute-baton-177602
gcloud app versions start 20251020t064815 --service=background-tasks --project=astute-baton-177602

# 9. Verify
gcloud app versions list --service=default --project=astute-baton-177602
gcloud app versions list --service=background-tasks --project=astute-baton-177602
```

## Common Mistakes (Rationalizations)

| Excuse | Reality | Action |
|--------|---------|--------|
| "Services are down, skip maintenance page" | Users need to see status, not errors | Ask for maintenance mode confirmation |
| "Asking about maintenance mode wastes time" | 30 seconds prevents hours of confusion | Wait for user confirmation |
| "Production is already down" | Controlled shutdown ≠ crashed connections | Stop services anyway |
| "Users are ALREADY locked out" | More reason to do it safely, not fast | Follow checklist completely |
| "Automatic backups exist" | Assumptions = risk | Verify and create manual backup |
| "Cloud SQL already has automatic backups" | May not exist, may be old, may fail | Create manual backup anyway |
| "This adds X minutes downtime" | Safety > speed in production | Wait for backup completion |
| "Taking snapshot adds 5-15 minutes" | Data loss adds hours or days | Take the backup |
| "The resize operation itself is reversible" | Not without data loss risk | Create backup before |
| "Memory increase is low-risk" | Low ≠ zero risk | Follow full procedure |
| "Tier changes are safe" | Safe ≠ risk-free | Follow full procedure |
| "We can rollback if needed" | Without backup? With what? | Document and test rollback plan |
| "System is in emergency state" | Emergencies need MORE safety, not less | Follow checklist under pressure |
| "CEO is asking for updates every 5 minutes" | Pressure is not permission | Follow procedure anyway |
| "I've done this before" | Experience ≠ immunity to mistakes | Follow checklist every time |

## Rollback Procedure

If resize causes problems:

```bash
# 1. Stop services (same as Step 1)
# 2. Patch back to original tier
gcloud sql instances patch [INSTANCE] \
  --cpu=[ORIGINAL_CPU] \
  --memory=[ORIGINAL_MEMORY]GB \
  --project=[PROJECT]

# 3. Wait for completion
# 4. Verify tier
# 5. Start services
# 6. Verify services
```

If data corruption occurred:

```bash
# Restore from backup created in Step 2
gcloud sql backups restore [BACKUP_ID] \
  --backup-instance=[INSTANCE] \
  --backup-id=[BACKUP_ID] \
  --project=[PROJECT]
```

## Red Flags - STOP and Review

- Skipping maintenance mode verification
- "Services are already down, users see errors anyway"
- "Asking about maintenance mode wastes time"
- Not asking user to confirm maintenance mode
- Proceeding without user confirmation of maintenance page
- Skipping backup "to save time"
- Skipping backup "because users are already locked out"
- Skipping backup "because resize is reversible"
- Not waiting for backup completion
- Not stopping services first
- Assuming automatic backups are sufficient
- Assuming automatic backups exist without verifying
- Not verifying tier change applied
- Not testing service restart
- "I'll document this later"
- "This is different because..."
- "This is an emergency"
- "CEO/management is pressuring me"
- "Memory increase is low-risk"

**All of these mean: Stop. Follow the checklist. Every step. No exceptions.**

**Violating checklist under pressure is violating the checklist.** There is no "emergency exception" to safety procedures.

## Why This Matters

**Production database changes without proper procedure:**
- Risk data loss (no verified backup)
- Risk cascading failures (services crash-reconnecting)
- Risk extended outage (can't rollback safely)
- Risk data corruption (mid-transaction kills)

**The checklist adds 5-10 minutes. Data loss adds hours or days.**

Following procedure under pressure is the definition of professionalism. Skipping steps under pressure is the definition of risk.
