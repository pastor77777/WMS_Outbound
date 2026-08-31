# Traceability

This directory provides agent- and machine-friendly indexes over the immutable architect source.

## Files

- `requirements_index.csv` — all 109 requirement IDs with architect source reference.
- `coverage_matrix.csv` — architect rule/exception → requirement → component criterion → scenario IDs, generated from the architect coverage matrix.
- `rule_catalog.csv` — normalized rule/exception catalog for machine use.
- `test_index.csv` — named TC scenario index and linked requirements.
- `state_event_transitions.csv` — state transition/event/actor/source-ref extraction from the state model.
- `IMPLEMENTATION_TRACEABILITY.md` — requirement → implementation task → acceptance task → architect scenario planning chain.
- `requirement_task_matrix.csv` — machine-readable equivalent; readiness requires zero empty implementation-task mappings.

The canonical source is still `01_ARCHITECT_SOURCE`; these generated indexes must be regenerated rather than hand-corrected if the source changes.

## Required chain

`ARCHITECT SOURCE`
→ `section/rule/decision`
→ `requirement`
→ `business behaviour`
→ `domain state/event`
→ `DB / Backend / Mercato / Scanner / Integration`
→ `implementation task`
→ `test`
→ `Playwright normal UI evidence where human-facing`
→ `Human Verified`.

An implementation task without a source/requirement reference is invalid. A requirement not mapped to at least one implementation task is an orphan and blocks plan readiness.
