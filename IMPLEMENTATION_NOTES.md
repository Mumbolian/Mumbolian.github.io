# Implementation: Automatic Adjustments for "Must Be Asleep During" Constraints

## Bug Fixed
The `adjustScheduleForConstraints()` function previously only handled "must be awake by" constraints and would return immediately if that constraint wasn't set, never attempting adjustments for "must be asleep during" constraints.

## Changes Made

### 1. Updated Constraint Detection (lines 939-986)
**Before:**
- Only checked for `mustBeAwakeByInput.value`
- Early return if not present

**After:**
- Checks for BOTH constraint types
- Parses and normalizes both "must be awake by" AND "must be asleep during"
- Only returns early if NO constraints exist at all

### 2. Strategy 1 & 2: Already Working! ✅
**Wake Window Adjustments** and **Bedtime Adjustments** already work for sleep window constraints because they:
- Test different schedule variations
- Use `validateConstraints()` which checks BOTH constraint types
- No changes needed!

### 3. Strategy 3: Split into 3a and 3b

#### Strategy 3a: "Must Be Awake By" - Nap Reduction (lines 1075-1177)
- Existing logic preserved
- Finds conflicting nap and reduces it
- Tries redistribution to later naps

#### Strategy 3b: "Must Be Asleep During" - Nap Extension (lines 1179-1294) **NEW!**
Adds three new capabilities:

1. **Find Closest Nap** (lines 1184-1224)
   - Calculates distance from each nap to the required sleep window
   - Identifies which nap is closest/best positioned to cover the window

2. **Extend Nap** (lines 1239-1264) **NEW CAPABILITY!**
   - Extends the closest nap by up to 30 minutes
   - Calculates exact extension needed to cover sleep window
   - Tests if extended nap satisfies all constraints

3. **Shift Earlier by Reducing Prior Naps** (lines 1266-1292)
   - If target nap starts too late, reduces earlier naps to shift schedule
   - Tries reducing naps before the target by 5-30 minutes
   - Shifts entire schedule earlier to position target nap over sleep window

### 4. Strategy 4: Already Working! ✅
**Nap Count Switching** works for both constraint types (uses `validateConstraints()`).

## How It Works for Your Test Case

**Constraint:** Must be asleep 15:00-15:30
**Default Schedule:** Nap 3 at 16:30-17:00 (closest nap, but too late)

### Strategies Attempted (in order):

1. **Wake Window Compression:** Try reducing maxWake from 210→150 min
   - Shifts Nap 3 from 16:30 → ~15:00 ✅ **LIKELY TO SUCCEED**

2. **Bedtime Compression:** Try moving bedtime from 20:30 → 19:45
   - Compresses entire day, shifts all naps earlier
   - Nap 3 moves to ~15:00-15:30 ✅ **LIKELY TO SUCCEED**

3. **Nap Extension:** Extend Nap 3 from 30 → 60 min
   - If combined with earlier shift, could cover window

4. **Reduce Earlier Naps:** Reduce Nap 1 or 2 by 5-30 min
   - Shifts schedule earlier, moves Nap 3 toward 15:00

5. **Switch to 2 Naps:** Try 2-nap schedule with defaults (90, 30)
   - Complete schedule restructure

## Expected Result

With default settings and constraint "15:00-15:30":
- ✅ Should now find a solution (most likely Strategy 1 or 2)
- ✅ Will show success message like: "Reduced max wake window by 60 min to shift naps earlier"
- ✅ Schedule will adjust to satisfy the constraint automatically

## Testing Instructions

1. Open https://bingpotstudio.com in browser
2. Use default settings (or reset if changed)
3. Add constraint: "Must be asleep during 15:00 to 15:30"
4. Observe:
   - ✅ Green success message (not red warning)
   - ✅ Adjusted schedule shown
   - ✅ Message explaining which strategy was used
   - ✅ One nap now covers 15:00-15:30 fully

## Code Quality

- ✅ Preserves all existing functionality
- ✅ Reuses existing validation logic
- ✅ Follows existing code patterns and style
- ✅ No breaking changes
- ✅ Backward compatible with existing constraints
