# Constraint Analysis: "Must be asleep during 15:00-15:30"

## Default Settings Used
- Wake: 07:00
- Bed: 20:30
- 3 naps: 30, 60, 30 minutes
- lastWake: 210 min (3h30)
- minWake: 135 min (2h15)
- maxWake: 210 min (3h30)

## Default 3-Nap Schedule Calculation
- Total day: 20:30 - 07:00 = 810 minutes (13.5 hours)
- Total nap time: 30 + 60 + 30 = 120 minutes
- Total awake: 690 minutes
- Wake windows before naps: [135, 135, 210] minutes
- Last wake window: 210 minutes

### Resulting Schedule:
```
Wake:   07:00
↑ 2h15 awake
Nap 1:  09:15 - 09:45 (30 min)
↑ 2h15 awake
Nap 2:  12:00 - 13:00 (60 min)
↑ 3h30 awake
Nap 3:  16:30 - 17:00 (30 min)
↑ 3h30 awake
Bed:    20:30
```

## Constraint: Must be asleep from 15:00 to 15:30

**Gap in schedule:** Between Nap 2 ending (13:00) and Nap 3 starting (16:30), there's a 3.5-hour wake window. The required sleep time (15:00-15:30) falls right in the middle of this gap.

## Key Finding: NO AUTOMATIC ADJUSTMENTS FOR "MUST BE ASLEEP DURING"

Looking at `adjustScheduleForConstraints()` function (line 936-945):
```javascript
function adjustScheduleForConstraints(params) {
  // Get constraint values
  const mustBeAwakeByStr = mustBeAwakeByInput.value;

  // If no "must be awake by" constraint, return null (no adjustment needed)
  if (!mustBeAwakeByStr) {
    return null;
  }
  ...
}
```

**The function returns immediately if there's no "must be awake by" constraint!**

This means:
- ✅ "Must be awake by" constraints → automatic adjustments attempted
- ❌ "Must be asleep during" constraints → NO automatic adjustments, only validation

## Validation Logic

The validation at line 1253-1257 requires:
```javascript
// Check if nap covers the entire required sleep window
if (napStart <= sleepStart && napEnd >= sleepEnd) {
  isAsleepDuringWindow = true;
}
```

This requires a nap to **fully cover** the required window:
- Nap must start ≤ 15:00
- Nap must end ≥ 15:30
- For a 30-minute nap, it would need to be exactly 15:00-15:30 or wider

## Why Manual Adjustments Would Be Difficult

### To move Nap 3 from 16:30 to 15:00 (90 minutes earlier):

**Strategy 1: Compress wake windows**
- Current wake window 3: 210 minutes
- Needed: 120 minutes (13:00 + 2h = 15:00)
- Reduction: 90 minutes
- **Problem:** Limited to 60-minute compression max

**Strategy 2: Compress bedtime**
- Max compression: 45 minutes (bed → 19:45)
- This moves Nap 3 to ~14:45-15:15
- **Problem:** Doesn't fully cover 15:00-15:30 (ends at 15:15)

**Strategy 3: Extend Nap 3**
- If Nap 3 is at 14:45-15:15, extend by 15 min → 14:45-15:30
- **Problem:** System only reduces naps, doesn't extend them

**Strategy 4: Switch to 2 naps**
- With default 2-nap settings (90, 30 min)
- Nap 2 would be at 15:30-16:00
- **Problem:** Starts at 15:30, doesn't cover from 15:00

## Conclusion

**This is NOT a legitimate denial in the sense that:**

1. ❌ The system doesn't attempt ANY automatic adjustments for "must be asleep during" constraints
2. ❌ The README promises automatic adjustment for both constraint types, but only "must be awake by" is implemented
3. ⚠️ The validation is correct according to the spec (requires full coverage), but no adjustment strategies are attempted

**This IS a legitimate denial in the sense that:**

1. ✅ With default settings and no adjustments, no nap fully covers 15:00-15:30
2. ✅ The validation correctly identifies that the constraint is not met
3. ✅ The "full coverage" requirement is strict but intentional per the README

## Bug Identified

**The `adjustScheduleForConstraints()` function only handles "must be awake by" constraints, not "must be asleep during" constraints.** This is a gap between the documented behavior and actual implementation.
