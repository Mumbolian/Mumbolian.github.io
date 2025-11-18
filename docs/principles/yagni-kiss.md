---
title: YAGNI/KISS Principles for Development
status: current
updated: 2025-11-18
---

# YAGNI/KISS Principles for Development

## Purpose

This document defines the anti-scope-creep and simplicity principles that guide all development, including:
- Manual code development
- AI-assisted development (Claude Code)
- Product specifications
- Code reviews
- Feature planning

**Core Philosophy**: Build the simplest thing that solves the actual problem. Nothing more.

---

## The Six Core Principles

### 1. YAGNI (You Aren't Gonna Need It)

**Rule**: Every feature must be **necessary now**, not "might be useful later"

**Red Flags**:
- "This will make it easier to add X later"
- "We might need this for future requirements"
- "Let's add this for flexibility"
- Building features for hypothetical problems

**Test**: If you can't point to a current, concrete requirement that needs this feature **today**, you don't need it.

**Example**:
```python
# ❌ YAGNI Violation
def calculate_nap_time(self, child, mode='simple'):
    if mode == 'simple':
        return self._simple_calc(child)
    elif mode == 'advanced':  # Nobody asked for this
        return self._advanced_calc(child)
    elif mode == 'ml':  # Hypothetical future feature
        return self._ml_calc(child)

# ✓ YAGNI Compliant
def calculate_nap_time(self, child):
    return self._simple_calc(child)
```

### 2. KISS (Keep It Simple, Stupid)

**Rule**: Always prefer the simplest solution that solves the stated problem

**Red Flags**:
- "This is more robust/flexible/scalable"
- "This is a more elegant solution"
- "This uses a better design pattern"
- Choosing complexity over simplicity

**Test**: If there's a boring, simple solution that works, use it. Reject "clever" solutions.

**Example**:
```javascript
// ❌ KISS Violation - Over-engineered
class ScheduleStrategyFactory {
    constructor() {
        this._strategies = {}
    }

    register(name, strategy) {
        this._strategies[name] = strategy
    }

    create(name) {
        return this._strategies[name]()
    }
}

// ✓ KISS Compliant
function shouldAdjustSchedule(napTotal, targetTotal) {
    return napTotal < targetTotal
}
```

### 3. Zero-Based Justification

**Rule**: Every feature must directly trace back to a specific requirement

**Red Flags**:
- "Nice-to-have" features
- "While we're at it..." additions
- Features without clear requirement linkage
- "Industry best practice" without specific need

**Test**: Can you point to the exact requirement this feature solves? If not, it doesn't belong.

**Requirement Traceability Matrix**:
```
| Requirement (from objectives) | Feature | Justification |
|------------------------------|---------|---------------|
| "Adjust nap times when constraints conflict" | Constraint satisfaction algorithm | Direct implementation of requirement |
| [None] | AI-powered nap prediction dashboard | ❌ SCOPE CREEP - not in requirements |
```

### 4. Minimum Viable Solution (MVS)

**Rule**: What's the **bare minimum** to satisfy the requirement? Design that first.

**Red Flags**:
- "Let's build a robust framework"
- "We should generalize this"
- "Let's add proper error handling" (beyond basic needs)
- Building for scale before proving concept

**Test**: Can you remove this component/feature/abstraction and still meet the requirement? If yes, remove it.

**Example**:
```javascript
// ❌ MVS Violation - Building too much upfront
class ConfigurationManager {
    // Loads config from localStorage, IndexedDB, or server
}

class ConfigValidator {
    // Validates config against schema
}

class ConfigWatcher {
    // Hot-reloads config on changes
}

// ✓ MVS Compliant
// Just use localStorage with hard-coded defaults first
const DEFAULT_CONFIG = {
    wake_time: '07:00',
    bedtime: '20:30',
    nap_count: 3
}
```

### 5. Rule of Three

**Rule**: Don't add abstractions/frameworks until you have **3+ concrete use cases**

**Red Flags**:
- Creating base classes with single implementation
- Building frameworks for one use case
- "Future-proof" abstractions
- Premature generalization

**Test**: Do you have 3 or more actual, current uses for this abstraction? If not, use concrete code.

**Example**:
```javascript
// ❌ Rule of Three Violation
class BaseScheduleProcessor {  // Only 1 concrete implementation exists
    process(data) { throw new Error('Not implemented') }
}

class StandardScheduleProcessor extends BaseScheduleProcessor {
    process(data) { /* implementation */ }
}

// ✓ Rule of Three Compliant
// Just write the concrete function, abstract later if needed
function processSchedule(data) {
    /* implementation */
}
```

### 6. Boring Technology

**Rule**: Prefer proven, simple solutions over new/shiny approaches

**Red Flags**:
- "Let's use the latest framework"
- "This new library is better"
- Novel approaches when standard ones exist
- Technology choices driven by interest, not need

**Test**: Is there a boring, battle-tested solution? Use that. Don't innovate unless necessary.

**Example**:
```javascript
// ❌ Boring Technology Violation
// Using heavyweight charting library for simple time display
import fancyChartLibrary from 'heavy-charts'
const chart = fancyChartLibrary.create(scheduleData)

// ✓ Boring Technology Compliant
// Simple HTML/CSS does the job
scheduleContainer.innerHTML = formatScheduleHTML(scheduleData)
```

---

## Scope Creep Red Flags - Automatic Rejections

If you encounter these phrases, **immediately reject the feature**:

| Phrase | Why It's Wrong | What to Do |
|--------|----------------|------------|
| "This will make it easier to add X later" | YAGNI violation - solving future problems | Remove feature, address when X is actually needed |
| "This is more robust/flexible/scalable" | KISS violation - unjustified complexity | Use simplest solution that works today |
| "Industry best practice" | Not needed for this specific problem | Question if it solves current requirement |
| "While we're at it..." | Classic scope creep | Stop. Only do what was requested. |
| "Nice to have" | Not necessary = not included | Move to "deferred features" list |
| "This makes it more maintainable" | Premature abstraction | Wait until you have 3+ cases |
| "We should generalize this" | Rule of Three violation | Keep it specific until proven need |
| "Let's add proper X" (logging/monitoring/error handling/etc.) | Over-engineering | Add only what's necessary for MVP |

---

## Decision Framework

When evaluating any feature, code, or design decision, ask these questions **in order**:

### 1. Necessity Check
- ❓ Is this solving a problem that exists **right now**?
- ❓ Can I point to the specific requirement that needs this?
- ❓ What happens if we don't build this?

**If you can't answer these clearly → REJECT**

### 2. Simplicity Check
- ❓ What's the simplest possible solution?
- ❓ Am I choosing complexity over simplicity?
- ❓ Is there a boring, proven approach?
- ❓ Can this be done with existing components?

**If the answer isn't "this IS the simplest solution" → SIMPLIFY**

### 3. Minimalism Check
- ❓ Can I remove this and still meet requirements?
- ❓ Am I building abstractions prematurely?
- ❓ Do I have 3+ concrete use cases for this generalization?
- ❓ Can this be a single file instead of multiple?
- ❓ Can this use hard-coded values instead of configuration?

**If you can remove it without breaking requirements → REMOVE**

### 4. Scope Check
- ❓ Is this in the original objectives/requirements?
- ❓ Am I solving hypothetical future problems?
- ❓ Did I add this because it's "nice to have"?
- ❓ Is this scope creep?

**If it's not in the original requirements → DEFER or REMOVE**

---

## Practical Application

### For Product Specifications

1. **Create Requirement Traceability Matrix** - Map every feature to an objective
2. **Aggressive "What's NOT in Scope"** - List everything you're NOT building
3. **Choose simplest approach** - Not most robust/flexible/elegant
4. **Self-audit for scope creep** - Review every feature against principles

### For Code Development

1. **Start with hard-coded values** - Config systems come later if needed
2. **Write concrete code first** - Abstract only when you have 3+ cases
3. **Single file when possible** - Don't split prematurely
4. **Boring solutions preferred** - Battle-tested > innovative

### For Code Reviews

1. **Challenge every feature** - "Why do we need this?"
2. **Question complexity** - "What's the simpler way?"
3. **Hunt for scope creep** - "Is this in the requirements?"
4. **Demand justification** - Burden of proof on feature advocates

---

## Common Pitfalls

### "But it's only 10 more lines..."

**Problem**: Death by a thousand cuts - small additions add up to massive scope creep

**Solution**: Judge each feature independently against principles. Small != acceptable if it violates YAGNI/KISS.

### "We'll need this eventually..."

**Problem**: Building for imagined future instead of known present

**Solution**: Wait until "eventually" becomes "now". You might never need it, or requirements might change.

### "This is the right way to do it"

**Problem**: Confusing "right" with "complex" or "sophisticated"

**Solution**: The right way is the simplest way that works. Period.

### "But <famous company> does it this way"

**Problem**: Copying solutions for problems you don't have at scale you haven't reached

**Solution**: Famous companies have different problems. Solve YOUR problems, not theirs.

---

## Exceptions (Rare)

These principles can be violated **only** when:

1. **Security Risk**: Simplest solution has genuine security vulnerability
2. **Data Integrity**: Simplest solution risks data corruption/loss
3. **Explicit Requirement**: User explicitly requests the complexity
4. **Known Immediate Need**: You have concrete, scheduled requirement within current iteration

**Note**: "Future-proofing", "maintainability", "scalability" are **NOT** valid exceptions unless backed by concrete, immediate requirements.

---

## Summary

**Default stance**: Every feature is scope creep until proven necessary.

**Guiding question**: What's the simplest thing that could possibly work?

**When in doubt**: Simpler. Smaller. Less.

**Remember**: You can always add complexity later. You can rarely remove it once added.

**Mantra**: Build what's needed. Nothing more.
