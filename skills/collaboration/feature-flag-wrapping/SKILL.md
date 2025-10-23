---
name: Feature Flag Wrapping
description: Wrap new functionality with feature flags while preserving old behavior for safe rollout and instant rollback
when_to_use: when deploying risky changes, replacing core functionality, or need ability to instantly disable new code in production
version: 1.0.0
languages: all
---

# Feature Flag Wrapping

## Overview

**Wrap new functionality with feature flags while preserving old behavior.** This enables gradual rollout, A/B testing, and instant rollback without redeployment.

**Core principle:** Both code paths must coexist until new functionality is proven in production.

## When to Use

**Use feature flag wrapping when:**
- Replacing core functionality (authentication, payment processing, API clients)
- High-risk changes that could break production
- Need gradual rollout to subset of users
- Want to A/B test new implementation
- Refactoring critical paths
- Moving functionality (client-side → server-side, one service → another)

**Don't use when:**
- Simple bug fixes with obvious correct behavior
- New features with no existing equivalent
- Internal tools or non-production code
- Changes can be tested thoroughly in staging

## Core Pattern

### Before: Direct implementation (risky)
```typescript
// Old code removed, new code deployed
// If new code breaks, need emergency rollback/redeployment

async function processPayment(amount: number): Promise<PaymentResult> {
    // New implementation calling different API
    return newPaymentGateway.process(amount);
}
```

### After: Feature-flagged implementation (safe)
```typescript
async function processPayment(amount: number): Promise<PaymentResult> {
    const useNewGateway = isFeatureFlagEnabled('new-payment-gateway');

    if (useNewGateway) {
        // New implementation
        return newPaymentGateway.process(amount);
    } else {
        // Legacy implementation (preserved)
        return oldPaymentGateway.process(amount);
    }
}
```

## Implementation Checklist

Use TodoWrite to track these steps:

- [ ] **Identify the change boundary** - Find the function/class/module being replaced
- [ ] **Add feature flag constant** - Define flag name in central location
- [ ] **Preserve old implementation** - Don't delete legacy code yet
- [ ] **Implement new functionality** - Add new code path
- [ ] **Add flag check** - Use if/else or strategy pattern to switch implementations
- [ ] **Restore configuration** - If new code removed config/dependencies, add them back
- [ ] **Test both paths** - Verify flag ON (new) and OFF (old) work correctly
- [ ] **Document flag** - Add comments explaining what flag controls
- [ ] **Configure flag service** - Set up flag in feature flag system (PostHog, LaunchDarkly, etc)
- [ ] **Deploy with flag OFF** - Roll out code with old behavior active
- [ ] **Gradual rollout** - Enable for small percentage, monitor, increase
- [ ] **Remove old code** - After new code proven, delete legacy path and flag check

## Quick Reference

### Common Feature Flag Patterns

**Boolean flag (simple on/off)**
```typescript
if (isFeatureFlagEnabled('feature-name')) {
    // New code
} else {
    // Old code
}
```

**Strategy pattern (cleaner for complex logic)**
```typescript
class ServiceFactory {
    create(): Service {
        return isFeatureFlagEnabled('new-service')
            ? new NewService()
            : new LegacyService();
    }
}
```

**Percentage rollout (gradual)**
```typescript
// Flag service returns true for 10% of users
if (isFeatureFlagEnabled('new-algorithm')) {
    return newAlgorithm(data);
} else {
    return legacyAlgorithm(data);
}
```

**User-specific flags**
```typescript
if (isFeatureFlagEnabledForUser(userId, 'beta-ui')) {
    return <NewUI />;
} else {
    return <LegacyUI />;
}
```

## Real-World Example: Client-Side to Server-Side Migration

**Context:** Moving Google Translate API calls from client-side (exposing API key) to server-side (secure).

**Before:** Client calls Google directly
```typescript
async translate(text: string, lang: string): Promise<string> {
    // Calls Google Translate API directly from browser
    return this.http.post(
        'https://translation.googleapis.com/...',
        { text, lang, key: environment.apiKey }
    ).toPromise();
}
```

**After:** Feature-flagged dual implementation
```typescript
async translate(text: string, lang: string): Promise<string> {
    const useServerSideTranslation = isFeatureFlagEnabled('server-side-translation');

    if (useServerSideTranslation) {
        // New: Server-side translation (secure)
        return this.http.post(
            `${environment.apiUrl}/language/translate`,
            { texts: [text], lang }
        ).toPromise();
    } else {
        // Legacy: Client-side translation (fallback)
        return this.http.post(
            'https://translation.googleapis.com/...',
            { text, lang, key: environment.apiKey }
        ).toPromise();
    }
}
```

**Configuration restored:**
```typescript
// environment.apiKey was removed in feature branch
// Must restore it from develop for legacy fallback
export const environment = {
    apiUrl: 'https://api.example.com/',
    apiKey: 'AIza...', // ← Restored for legacy path
    // ...
};
```

**Rollout plan:**
1. Deploy with flag OFF (everyone uses legacy)
2. Enable for 5% of users, monitor errors
3. Increase to 25%, 50%, 100%
4. After 1 week at 100%, remove legacy code

## Common Mistakes

### ❌ Deleting old code too early
```typescript
// BAD: Old implementation already deleted
async function processData(data: Data) {
    if (isFeatureFlagEnabled('new-processor')) {
        return newProcessor.process(data);
    } else {
        // Nothing here! Flag OFF = broken
        throw new Error('Old code was deleted');
    }
}
```

**Fix:** Keep both implementations until new code is proven.

### ❌ Forgetting to restore dependencies
```typescript
// BAD: New code doesn't need oldLibrary, so it was removed from package.json
// But flag OFF path still needs it!

if (useNewImplementation) {
    return newLibrary.doThing();
} else {
    return oldLibrary.doThing(); // ← Crashes, oldLibrary not installed
}
```

**Fix:** Restore dependencies needed by legacy path.

### ❌ Incomplete flag coverage
```typescript
// BAD: Some code paths not wrapped
function initializeService() {
    if (isFeatureFlagEnabled('new-service')) {
        return new NewService();
    } else {
        return new OldService();
    }
}

// This code path not wrapped!
function cleanupService() {
    newServiceOnlyMethod(); // ← Breaks when flag OFF
}
```

**Fix:** Wrap ALL code that differs between implementations.

### ❌ Not testing both paths
```typescript
// Deployed with flag ON, tested successfully
// Turned flag OFF to rollback emergency
// App breaks because legacy path was never tested!
```

**Fix:** Test both flag ON and flag OFF before deploying.

## Feature Flag Systems

### PostHog (behavioral flags)
```typescript
import { isFeatureEnabled } from 'app/common/utils';
import { FeatureFlags } from 'app/common/feature-flags';

if (isFeatureEnabled(FeatureFlags.NewFeature)) {
    // New code
}
```

Configuration: PostHog dashboard, no code changes needed

### LaunchDarkly
```typescript
const useNewFeature = await ldClient.variation('new-feature', false);
```

### Custom/Environment-based
```typescript
if (environment.features.newFeature) {
    // Requires redeployment to change
}
```

**Recommendation:** Use behavioral flag systems (PostHog, LaunchDarkly) for instant toggle without redeployment.

## Testing Strategy

### Test both code paths
```typescript
describe('PaymentService', () => {
    describe('with new gateway (flag ON)', () => {
        beforeEach(() => {
            mockFeatureFlag('new-payment-gateway', true);
        });

        it('processes payment via new gateway', async () => {
            // Test new implementation
        });
    });

    describe('with legacy gateway (flag OFF)', () => {
        beforeEach(() => {
            mockFeatureFlag('new-payment-gateway', false);
        });

        it('processes payment via legacy gateway', async () => {
            // Test old implementation
        });
    });
});
```

### Integration testing
- Deploy to staging with flag OFF (verify old path works)
- Enable flag in staging (verify new path works)
- Toggle flag multiple times (verify switching works)

## Rollout Strategy

### Phase 1: Dark launch (0%)
- Deploy code with flag OFF
- All users use legacy implementation
- Verify no issues from deployment itself

### Phase 2: Internal testing (1-5%)
- Enable for internal users/team accounts
- Monitor errors, performance, behavior
- Iterate on fixes with flag still OFF for users

### Phase 3: Gradual rollout (10% → 50% → 100%)
- Increase percentage incrementally
- Monitor metrics at each step
- Pause or rollback if issues arise

### Phase 4: Cleanup
- After new code proven (1+ weeks at 100%)
- Delete legacy implementation
- Remove flag checks
- Remove flag from flag service

## When to Remove Feature Flags

**Remove when:**
- New code at 100% for 1+ week
- No rollback needed in that time
- Metrics show new code is stable
- Team confirms cleanup is safe

**Keep longer if:**
- High-risk changes (payment, auth, data)
- Complex rollout (multiple services)
- Uncertain about new implementation

**Warning:** Accumulating dead flags creates technical debt. Schedule cleanup.

## Summary

1. **Preserve old implementation** - Don't delete working code
2. **Wrap with flag check** - if/else or strategy pattern
3. **Restore dependencies** - Old code needs its libraries/config
4. **Test both paths** - Flag ON and OFF must both work
5. **Deploy dark** - Flag OFF initially
6. **Gradual rollout** - Increase percentage, monitor, rollback if needed
7. **Clean up** - Remove old code after proven

**Remember:** Feature flags add complexity. Use for risky changes where instant rollback justifies the cost.
