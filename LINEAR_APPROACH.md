# Linear Programming Approach to Schedule Adjustment

## Problem Formulation

This is a **Linear Constraint Satisfaction Problem** with:

### Variables
- `w[1..n]` = wake windows before each nap
- `n[1..n]` = nap lengths
- All continuous (measured in minutes)

### Constraints (All Linear!)

1. **Total day budget:**
   ```
   Σw[i] + Σn[i] = totalDay - lastWake
   ```

2. **Wake window bounds:**
   ```
   minWake ≤ w[i] ≤ maxWake  (for all i)
   ```

3. **Nap length bounds:**
   ```
   minNap ≤ n[i] ≤ original[i] + maxAdjust  (for all i)
   ```

4. **Position of nap k:**
   ```
   start[k] = wakeTime + Σ(i=1 to k-1)[w[i] + n[i]] + w[k]
   end[k] = start[k] + n[k]
   ```

5. **Time constraints:**
   - **"Awake by T":** For conflicting nap k:
     ```
     end[k] ≤ T
     => wakeTime + Σ(i=1 to k)[w[i] + n[i]] ≤ T
     ```

   - **"Asleep during S to E":** For target nap k:
     ```
     start[k] ≤ S  AND  end[k] ≥ E
     => wakeTime + Σ(i=1 to k-1)[w[i] + n[i]] + w[k] ≤ S
     => wakeTime + Σ(i=1 to k)[w[i] + n[i]] ≥ E
     ```

## Key Insight: This is Feasibility, Not Optimization

We don't need to optimize anything - just find **any** valid solution.

## Efficient Solution Without LP Solver

### Approach 1: Backward Calculation

For "asleep during S to E" targeting nap k:

```javascript
function calculateFeasibleSchedule(targetNap, sleepStart, sleepEnd) {
  const n = napCount;
  const w = new Array(n);
  const napLengths = [...originalNapLengths];

  // Required nap length to cover window
  napLengths[targetNap] = Math.max(
    sleepEnd - sleepStart,
    minNap,
    originalNapLengths[targetNap] - maxAdjust
  );

  if (napLengths[targetNap] > originalNapLengths[targetNap] + maxAdjust) {
    return null; // Can't extend enough
  }

  // Calculate required position
  const targetNapStart = sleepStart; // Must start by this time

  // Work backwards from target nap to calculate wake windows before it
  let requiredTime = targetNapStart - wakeTime;
  for (let i = 0; i < targetNap; i++) {
    requiredTime -= napLengths[i];
  }

  // Distribute requiredTime across w[0..targetNap-1]
  // Start with minimum, distribute leftover to later ones
  let remaining = requiredTime;
  for (let i = 0; i < targetNap; i++) {
    w[i] = minWake;
    remaining -= minWake;
  }

  // Add leftover to later wake windows (target nap's wake window first)
  for (let i = targetNap - 1; i >= 0 && remaining > 0; i--) {
    const canAdd = Math.min(maxWake - w[i], remaining);
    w[i] += canAdd;
    remaining -= canAdd;
  }

  if (remaining > 0) {
    return null; // Can't fit - wake windows would exceed maxWake
  }
  if (remaining < 0) {
    return null; // Over-constrained - would need wake windows below minWake
  }

  // Now calculate wake windows AFTER target nap
  let totalUsed = sum(w[0..targetNap-1]) + sum(napLengths[0..targetNap]);
  let totalRemaining = totalDay - lastWake - totalUsed;

  // Distribute to remaining wake windows
  const remainingWindows = n - targetNap;
  for (let i = targetNap; i < n; i++) {
    totalRemaining -= napLengths[i];
  }

  // Distribute totalRemaining across w[targetNap..n-1]
  for (let i = targetNap; i < n; i++) {
    w[i] = minWake;
    totalRemaining -= minWake;
  }

  for (let i = n - 1; i >= targetNap && totalRemaining > 0; i--) {
    const canAdd = Math.min(maxWake - w[i], totalRemaining);
    w[i] += canAdd;
    totalRemaining -= canAdd;
  }

  if (totalRemaining !== 0) {
    return null; // Budget doesn't work out
  }

  // Build schedule with these wake windows and nap lengths
  return buildScheduleFromArrays(w, napLengths);
}
```

### Approach 2: Constraint Propagation

Even simpler - use the constraints to narrow down possibilities:

```javascript
function findFeasibleSchedule(constraint) {
  // For each nap that could satisfy the constraint
  for (let k = 0; k < napCount; k++) {
    // Calculate bounds on what's needed
    const bounds = calculateRequiredBounds(k, constraint);

    if (bounds.feasible) {
      // Try to build schedule with these requirements
      const schedule = tryBuildWithBounds(k, bounds);
      if (schedule.valid) {
        return schedule;
      }
    }
  }
  return null;
}
```

## Complexity

- **Variables:** 2n (n wake windows + n nap lengths)
- **Constraints:** O(n) linear inequalities
- **Without LP:** O(n²) to try each nap and calculate
- **With LP:** O(n³) worst case for simplex, but overkill

## Implementation Strategy

**Recommended:** Use targeted calculation (Approach 1)

1. For each candidate nap (closest to constraint)
2. Calculate exact requirements using linear equations
3. Check if requirements fit within bounds
4. Return first feasible solution

**Advantages:**
- No external LP library needed
- Fast: O(n²) worst case, often O(n) if we target closest nap first
- Exact solution, not heuristic
- Transparent - can explain what was adjusted and why

## Comparison with Current Approach

**Current:** Brute force trying ±5 min increments
- Time: O(12^n) for 60-min range with 5-min steps
- May miss solutions
- Can't explain why no solution exists

**Linear approach:**
- Time: O(n²) or O(n³) worst case
- Finds solution if one exists
- Can explain infeasibility (which constraint is too tight)
- Can report "need to adjust X by Y minutes" when no solution

## Next Steps

Implement Strategy 1 using this approach:
1. Identify which nap(s) could satisfy the constraint
2. Calculate required wake windows using backward/forward propagation
3. Validate all bounds are satisfied
4. Return feasible schedule or explain why none exists
