# Bug Fix: Wake Window Parameters Must Be Hard Limits

## Problem Found

User observed a schedule with a 4-hour (240 minute) wake window, despite having `maxWake=210` set in their preferences:

```
Wake-up: 08:30
Nap 1: 11:00 – 12:30 (90 min)  ← 2h30 wake ✓
Nap 2: 16:30 – 17:00 (30 min)  ← 4h00 wake ❌ (exceeds 210 max)
Bedtime: 20:30                 ← 3h30 wake ✓
```

## Root Cause

**Strategy 1** in `adjustScheduleForConstraints()` was modifying wake window parameters to satisfy constraints:

```javascript
// WRONG: Increases maxWake beyond user's setting
for (let increase = 5; increase <= 60; increase += 5) {
  const adjustedMaxWake = maxWake + increase;  // 210 + 30 = 240
  // ... builds schedule with violated constraint
}
```

This violated the design principle that **wake windows are biological constraints**, not flexible preferences.

## The Fix

### Code Changes (index.html)

1. **Removed Strategy 1 entirely** (lines 988-1029)
   - No longer compresses maxWake (reducing it)
   - No longer expands maxWake (increasing it)

2. **Added clear documentation**
   ```javascript
   // Strategy 1: Adjust bedtime
   // NOTE: We do NOT adjust wake window parameters (minWake, maxWake, lastWake) as these
   // are biological constraints that must be respected as hard limits
   ```

3. **Renumbered remaining strategies**
   - Old Strategy 2 (Bedtime) → New Strategy 1
   - Old Strategy 3 (Naps) → New Strategy 2
   - Old Strategy 4 (Nap Count) → New Strategy 3

### README Changes

1. **Updated strategy list** to remove wake window adjustments
2. **Added prominent note**: "Wake window parameters (minWake, maxWake, lastWake) are treated as hard biological constraints and are never adjusted."
3. **Renumbered and documented remaining strategies**

## What Still Works

The system can still adjust schedules to meet constraints using:

1. **Adjust Bedtime** (±15-45 min)
   - Changes day length
   - Shifts all naps earlier or later

2. **Adjust Naps**
   - Reduce conflicting naps (for "awake by")
   - Extend naps to cover windows (for "asleep during")
   - Shift naps by reducing earlier ones

3. **Switch Nap Count** (2 ↔ 3)
   - Completely restructures schedule

## Impact

- ✅ Wake windows now truly respected as hard limits
- ✅ No more confusing violations of user's settings
- ⚠️ Some constraints may fail that previously succeeded
  - This is CORRECT behavior - the old "success" violated biological constraints
  - Users should adjust their constraint timing or wake window limits if needed

## Design Principle Established

**Wake window parameters represent biological capabilities of the baby:**
- minWake = shortest wake time baby can handle
- maxWake = longest wake time baby can handle
- lastWake = required wake window before bed

These are **not negotiable** for meeting scheduling constraints. The schedule must work within these limits.
