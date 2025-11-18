# Spec Director Skill - Usage Guide

## Overview

The **Spec Director** is a skill that orchestrates the creation of comprehensive technical specifications through a 5-step process involving multiple specialized sub-agents.

**What it does:**
- Gathers context about existing systems
- Designs product architecture with features
- Plans features into independent implementable chunks
- Reviews specification quality (with Codex CLI second opinion)
- Gates approval through human review

**When to use it:**
- Starting a new project or major feature
- Need a structured specification before implementation
- Want to ensure completeness and feasibility before coding

## Quick Start

### Basic Invocation

```
Use the spec-director skill to create a specification for project "schedule_sharing_v1" with these objectives:
- Add ability to share nap schedules with caregivers via link
- Generate shareable URL with read-only schedule view
- Support both planned and actual nap times
- Work on mobile devices (primary use case)
```

### With Explicit Context

```
Start the spec director for project "constraint_templates".

Objectives:
- Add reusable constraint templates (e.g., "Daycare pickup at 3pm")
- Allow saving/loading constraint sets
- Support multiple templates per user
- Persist templates in localStorage

Context:
- Related to existing constraint system (validateConstraints, line ~2176)
- Must work with current "Must be awake by" / "Must be asleep during" logic
- Should integrate with openModal() constraint UI
- Single HTML file architecture (no backend)
```

## The 5-Step Process

### STEP 1: Gather Context

**What happens:**
- You're asked if there are existing related systems
- Script-doc-tracker agent (if applicable) finds relevant scripts
- Grep/Glob searches for related code patterns
- Output: `1_context_gathered.md`

**Your involvement:**
- Answer questions about related systems
- Provide any known constraints or integration points

### STEP 2: Product Architecture Design

**What happens:**
- Product-architect agent designs the specification
- Takes context + your objectives as input
- Creates problem statement, solution approach, features, technical design
- Output: `2_product_spec.md`

**Your involvement:**
- Review will come later at approval gate

### STEP 3: Feature Planning

**What happens:**
- Feature-planner agent breaks spec into implementable features
- Creates acceptance criteria for each feature
- Maps dependencies and parallel execution opportunities
- Output: `3_features.md`

**Your involvement:**
- Review will come later at approval gate

### STEP 4: Specification Review

**What happens:**
- Spec-reviewer agent critically assesses the complete specification
- Codex CLI provides independent second opinion
- Issues categorized as Critical/Major/Minor
- Quality score (0-10) assigned
- Output: `4_review.md`

**Your involvement:**
- Review will come later at approval gate

### STEP 5: Human Approval Gate

**What happens:**
- You're presented with:
  - Spec summary
  - Key features
  - Review results (including Codex findings)
  - Quality score

**Your options:**
1. **APPROVE** → Proceeds to finalization (STEP 6)
2. **REQUEST CHANGES** → Iteration loop (STEP 5B)
3. **REJECT** → Document why and exit

**Iteration (STEP 5B)**:
- If you request changes, the system identifies the earliest impacted step
- Re-executes from that step through all downstream steps
- Incorporates your feedback at each stage
- Returns to approval gate with updated spec
- Max 3 automatic iterations

### STEP 6: Finalize Specification

**What happens:**
- All outputs consolidated into `FINAL_SPEC.md`
- Status marked as APPROVED
- Ready for handoff to implementation (Phase 2+, not yet implemented)

## File Structure

All outputs are written to:

```
code_director_docs/
└── {project_name}/
    └── spec_director/
        ├── 1_context_gathered.md
        ├── 2_product_spec.md
        ├── 3_features.md
        ├── 4_review.md
        ├── ITERATION_LOG.md (if iterations occurred)
        └── FINAL_SPEC.md (when approved)
```

### Iteration Versioning

When you request changes, previous versions are preserved:

```
code_director_docs/
└── {project_name}/
    └── spec_director/
        ├── 1_context_gathered.md (current)
        ├── 2_product_spec.md (current)
        ├── 2_product_spec_v1.md (previous iteration)
        ├── 2_product_spec_v2.md (if 2nd iteration)
        ├── 3_features.md (current)
        ├── 3_features_v1.md (previous iteration)
        ├── 4_review.md (current)
        ├── 4_review_v1.md (previous iteration)
        ├── ITERATION_LOG.md
        └── FINAL_SPEC.md (when approved)
```

## Example Usage Scenarios

### Scenario 1: New Feature - Multi-Day Planning

```
Use spec-director for project "multi_day_planner".

Objectives:
- Add ability to plan nap schedules across multiple days
- Allow parents to preview how schedule changes affect upcoming days
- Support copying/templating schedules across days
- Integrate with existing localStorage persistence (schema v6)

Context:
- Related to current single-day schedule building (tryBuildSchedule)
- Must work with existing constraint system (adjustScheduleForConstraints)
- Should maintain mobile-first design
```

**Expected flow:**
1. Context gathered about existing schedule building and storage
2. Product spec designed following single-file architecture patterns
3. Features broken into: date navigation UI, multi-day storage, schedule templating
4. Review catches localStorage schema migration needs
5. You approve or request changes

### Scenario 2: Algorithm Enhancement

```
Start spec director for "smart_nap_timing_suggestions".

Objectives:
- Add ML-based nap timing suggestions based on historical actual naps
- Learn from patterns in actual naps vs planned schedule
- Suggest optimal nap durations and wake windows per child
- Display confidence scores for suggestions

Context:
- Related to actual naps tracking (openActualNapsModal)
- Must work with existing tryBuildSchedule algorithm
- Should stay client-side (no backend)
- Need to consider localStorage size limits
```

**Expected flow:**
1. Context gathered about actual naps tracking and schedule calculation
2. Product spec designs client-side learning approach
3. Features: data collection, pattern analysis, suggestion UI, confidence scoring
4. Review verifies client-side feasibility and storage constraints
5. You approve or iterate based on localStorage concerns

### Scenario 3: UI/UX Refactoring

```
Use spec-director for "settings_modal_redesign".

Objectives:
- Redesign Settings modal for better mobile UX
- Group related settings into collapsible sections
- Add inline help text and validation feedback
- Maintain backward compatibility with schema v6

Related systems:
- openNapConfigModal() and closeNapConfigModal() functions
- loadSettings() / saveSettings() persistence layer
- recomputeSchedule() reactive UI coordinator
```

**Expected flow:**
1. Context gathered about current modal management and settings
2. Product spec designs improved modal structure
3. Features: collapsible sections, inline help, validation, responsive layout
4. Review catches potential breaking changes with existing event handlers
5. You iterate based on backward compatibility concerns

## Tips for Success

### Writing Good Objectives

**Good:**
```
Objectives:
- Add weekly schedule export feature to share with caregivers
- Export schedules as PDF or shareable link
- Include nap times, wake windows, and constraint notes
- Support both planned and actual nap times
- Maintain mobile-first design (export from phone)
```

**Bad:**
```
Objectives:
- Make the app better
- Improve usability
- Add new features
```

### Providing Useful Context

**Good:**
```
Context:
- Related to existing schedule building (tryBuildSchedule, line ~1833)
- Must work with localStorage schema v6 (napPlannerSettings_v6)
- Should integrate with recomputeSchedule() reactive coordinator
- Mobile-first: needs to work on iOS Safari
- Single HTML file constraint (no build system)
```

**Bad:**
```
Context:
- It's in the index.html file
- Make it fast
```

### Giving Feedback During Iterations

**Good:**
```
Changes needed:
- The product spec doesn't explain how PDF generation works client-side
- Feature 3's acceptance criteria are too vague - "should be easy" isn't testable
- Missing risk analysis about localStorage size limits for multi-day storage
- Need to address iOS Safari compatibility for file downloads
```

**Bad:**
```
Changes needed:
- Needs more detail
- Fix the features
- Better integration
```

## Integration with Code Director (Future)

The Spec Director is **Phase 1** of a larger Code Director workflow:

**Current state:**
- Phase 1: Specification ✅ (implemented)
- Phase 2-9: NOT IMPLEMENTED

**Future phases** (planned):
- Phase 2: Data & Schema Design
- Phase 3: Dependency Checker
- Phase 4: Feature Implementation
- Phase 5: Code Review
- Phase 6: Product Review
- Phase 7: Testing
- Phase 8: Technical Debt Audit
- Phase 9: Documentation

Currently, after `FINAL_SPEC.md` is approved, you proceed with manual implementation. Future Code Director versions will orchestrate implementation phases.

## Troubleshooting

### "Context gathering found nothing"

**Problem:** No related systems found during STEP 1

**Solutions:**
- Provide explicit context in your invocation
- Specify file paths or module names
- If it's genuinely new, acknowledge "no related systems" and proceed

### "Review score is very low (<5/10)"

**Problem:** Spec has critical issues

**Solutions:**
- Don't override - the review is catching real problems
- Read the critical issues carefully
- Request changes addressing each critical issue
- Iteration will automatically re-run from the impacted step

### "Codex didn't find anything useful"

**Problem:** Codex CLI review returned limited results

**Solutions:**
- This is OK - spec-reviewer's assessment is still thorough
- Codex is a second opinion, not mandatory for success
- If spec-reviewer found issues, address those
- Codex works better on some specs than others

### "Iteration limit reached (3 iterations)"

**Problem:** Spec still not approved after 3 iterations

**Solutions:**
- Consider manual edits to the spec files yourself
- Have a direct conversation about requirements
- Start fresh with clearer objectives
- Sometimes iteration doesn't converge - manual intervention needed

## Sub-Agents Used

The Spec Director coordinates these agents:

1. **script-doc-tracker** - Gathers context about existing scripts (optional, based on user input)
2. **product-architect** - Designs product specification
3. **feature-planner** - Breaks spec into features
4. **spec-reviewer** - Reviews specification quality
5. **codex-spec-reviewer** - Provides Codex CLI second opinion

All agents use Claude Sonnet model for quality analysis and synthesis.

## Limitations

**Current limitations:**
- Phase 1 only - doesn't implement the spec
- Max 3 automatic iterations
- Codex CLI review effectiveness varies by spec type
- No code generation (that's Phase 4+)
- No automated testing (that's Phase 7)

**Not suitable for:**
- Quick bug fixes (too heavyweight)
- Trivial features (1-2 hour work, just implement directly)
- Exploratory coding (you're not sure what you want yet)
- Emergency hotfixes (too slow)

**Best for:**
- New projects or major features
- Complex integrations with existing systems
- When spec rigor prevents future rework
- Multi-week implementations
- Team coordination (spec as shared artifact)
