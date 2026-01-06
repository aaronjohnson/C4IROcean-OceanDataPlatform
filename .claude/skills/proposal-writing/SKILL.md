---
name: proposal-writing
description: When and how to write proposals for ODP discussion topics
allowed-tools: Read, Write, Grep
---

## When to Write a Proposal

Create a proposal in `proposals/` when you encounter:

### 1. Unexpected API Behavior
- SDK returns different data than expected (e.g., indices instead of strings)
- Methods return empty results for data that exists elsewhere
- Error messages are unclear or misleading

**Example triggers:**
```python
# H3 aggregation returns integers, not hex strings
# files.list() returns [] but STAC shows the dataset
# aggregate() rejects "*": "count" syntax
```

### 2. Missing Cloud-Native Patterns
- No streaming access for FILE datasets
- Download-only workflows in cloud environment
- Missing integration with standard data libraries (xarray, geopandas)

### 3. Feature Gaps
- Functionality available in STAC but not SDK
- Common operations requiring complex workarounds
- Missing programmatic APIs (e.g., dataset creation)

### 4. Documentation Gaps
- Behavior that isn't documented
- Unclear distinctions (TABULAR vs FILE)
- Missing examples for common use cases

## Proposal Structure

```markdown
# Proposal: [Descriptive Title]

**Status:** Discussion
**Author:** [Name]
**Date:** [YYYY-MM-DD]
**Related:** [notebooks, skills, or code affected]

## Summary
One paragraph describing the issue and desired outcome.

## Current Behavior
Code examples showing what happens now.

## Expected/Desired Behavior
Code examples showing what we'd want.

## Questions for Discussion
Numbered list of specific questions for the ODP team.

## Proposed Solutions
Options A, B, C with code examples.

## Impact
How resolving this helps users.

## References
Links to docs, specs, related tools.
```

## Naming Convention

Use descriptive snake_case names:
- `cloud_native_file_access.md` - not `sdk_improvements.md`
- `server_side_h3_aggregation.md` - not `h3_issues.md`
- `my_data_api.md` - not `dataset_creation.md`

## Cross-References

When creating a proposal:

1. **Update troubleshooting skill** with workaround and link:
   ```markdown
   **Discussion:** See `proposals/your_proposal.md` for feature request.
   ```

2. **Add NOTE in affected notebooks**:
   ```python
   # NOTE: [Brief description of limitation]
   # See: proposals/your_proposal.md for discussion.
   ```

3. **Reference related proposals**:
   ```markdown
   ## Related Proposals
   - `other_proposal.md` - Brief description
   ```

## Decision Flow

```
Encounter unexpected behavior
         ↓
Can you work around it? ──Yes──→ Document in troubleshooting skill
         ↓ No
Is it a bug or design question?
         ↓
    Design question ──────────→ Write proposal
         ↓
       Bug ───────────────────→ File issue (if repo accepts issues)
```

## Current Proposals

| File | Topic | Status |
|------|-------|--------|
| `cloud_native_file_access.md` | FILE dataset streaming | Discussion |
| `server_side_h3_aggregation.md` | H3 hex strings, COUNT(*) | Discussion |
| `my_data_api.md` | Programmatic dataset creation | Discussion |

## Spec References

When proposals involve standards, clone/reference specs:
- STAC spec: `../stac-spec/` (adjacent to repo)
- ODP docs: `https://docs.hubocean.earth/`

Quote relevant spec sections in proposals to ground the discussion.
