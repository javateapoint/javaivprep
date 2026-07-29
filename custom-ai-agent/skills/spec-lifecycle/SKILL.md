---
name: spec-lifecycle
description: >
  Internal Specification Lifecycle skill for Coding Mentor in GitHub Copilot IntelliJ.
  Manages specification discovery, creation, updates, and index synchronization before coding.
---

# Specification Management Lifecycle Skill

## Overview & Scope
Specification management is an **internal Coding Mentor lifecycle responsibility**, applied during clarification and verification phases whenever behavior or requirements need to be captured, refined, or verified.

It ensures developers understand system specifications before modifying or adding source code.

---

## Preferred Directory Conventions
- Workspace root folders: `.agents/specs/` (default), `specs/`, or repository convention.
- Central index file: `<chosen-spec-folder>/00-index.md` (e.g., `.agents/specs/00-index.md`).
- Spec file naming convention: `<N>-<short-feature-name>.spec.md` (e.g., `001-order-retry-policy.spec.md`).

---

## Standard Spec Template & Structure

```markdown
# Spec <ID>: <Title>

## Overview / Goal
Brief description of the feature, behavior, or architectural contract.

## Requirements
- R1: Requirement 1
- R2: Requirement 2

## Acceptance Criteria
- [ ] AC1: Scenario 1 behavior
- [ ] AC2: Scenario 2 behavior

## Error Handling
- EH1: Expected exception types and error response contracts.
- EH2: Fallback or retry behavior.

## Technical Design & Implementation Details (Optional & Isolated)
- Key interfaces, database schemas, or Spring bean definitions (kept strictly separate from functional requirements).
```

---

## Operational Contract (Internal Commands)

### 1. `ensure_spec_index(location)`
- Check if `<location>/00-index.md` exists.
- If missing, create `00-index.md` with header, metadata, and index table:
  ```markdown
  # Specification Index

  | ID | Spec File | Title | Status | Short Description |
  |---|---|---|---|---|
  ```

### 2. `create_spec(title, requirements, acceptance_criteria)`
- Determine next sequential ID `<N>` (e.g., `001`, `002`).
- Format filename: `<N>-<short-feature-name>.spec.md`.
- Populate file using the Standard Spec Template.
- Register entry in `00-index.md`.

### 3. `update_spec(spec_id, deltas)`
- Locate spec file matching `spec_id`.
- Update user-visible behavior, requirements, acceptance criteria, or error handling.
- Preserve section structure and keep references valid.

### 4. `reindex_specs()`
- Rescan `<location>/` for all `*.spec.md` files.
- Rebuild `00-index.md` table to ensure ID alignment, links, and status entries are synchronized.

---

## Spec Content Rules
1. **Focus on Behavior & Contracts**: Focus on what the code *must do*, inputs/outputs, edge cases, and business constraints.
2. **Isolate Implementation Details**: Never mix framework setup or deep code details into functional requirements. Keep them strictly in `Technical Design & Implementation Details`.
3. **Keep Tokens Low**: Write concise, bulleted specifications without conversational fluff.
