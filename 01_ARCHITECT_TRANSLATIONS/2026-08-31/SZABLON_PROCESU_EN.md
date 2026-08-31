<!--
SZABLON_PROCESU.md — process description template (WMS Outbound / WMSAI)
Adaptation of the WMSAI_INBOUND/SZABLON_PROCESU.md template for the Outbound project (BACKLOG.md B5) —
substantive content identical, status source reference changed to model_stanow_outbound.md.

GENERAL RULES (read before use):
- Copy this file as proces_N_nazwa.md and fill in the sections.
- Sections 1–11 are MANDATORY and must appear in this order.
  If a section does not apply to the process — leave the heading and enter "Not applicable" (this is a decision, not a gap).
- OPTIONAL sections (Continuous functions, Design notes) are included only when the process requires them,
  always under the fixed heading shown below.
- Statuses: only canonical EN UPPER_SNAKE; prose in Polish.
  Every used status MUST exist in model_stanow_outbound.md (source of truth). The process DOES NOT introduce new states.
- Business rules: Rn numbering is LOCAL per file (context = this process).
- Traceability (rule Rn -> FR-Px-nn) is kept EXCLUSIVELY in macierz_pokrycia.md — NOT in this file.
- With every change: bump the version in the header + add an entry in "Change history";
  update references in intro.md, scenariusze_testowe.md, macierz_pokrycia.md.
Remove <!-- ... --> comments in the finished document.
-->

# PROCESS N: NAZWA_EN

<!-- N = position in the Outbound chain (P1–P5; P1–P4 have their own proces_N files, P5 lives in the exception sections of proces_1/proces_2). NAZWA_EN canonically in English, consistent with model_stanow_outbound.md. -->

**Version:** 1.0
<!-- x.y format; bump with every process change -->

## Process objective

<!-- 2–4 sentences: why this process exists and what value it delivers. No step description. -->

## Participants

<!-- Roles participating in the process. One line = one role + scope of responsibility.
     Technical role "System WMS" described by the scope of what it does automatically. -->

- **Role A** — scope of responsibility.
- **System WMS** — scope of automation.

## Start event

<!-- Condition/event that STARTS the process (trigger). One, unambiguous. -->

## Process flow

<!-- Sequence of steps. Numbering starts at 1 and resets per process (substeps such as 1a/6a retain the parent step number).
     Mark loops with 🔁 and name the loop scope. Branches as Path A / Path B gates.
     One STEP = one responsibility. Statuses in EN. -->

### STEP 1 — <description>

<!-- Who does what, which object, and which status it transitions to. -->

### STEP 2 — <description>

<!-- ...subsequent steps. -->

## End event

<!-- State/event indicating COMPLETION of the process (and optionally triggering the adjacent process). -->

## Data objects

<!-- Table: object -> key fields -> status(es) in EN. Consistent with model_stanow_outbound.md. -->

| Object | Key fields | Status(es) |
|---|---|---|
|  |  |  |

## Exceptions and alternative paths

<!-- Deviations from the main path: condition -> system/role behavior -> status effect.
     Table or list. -->

| Condition | Behavior | Effect |
|---|---|---|
|  |  |  |

## Business rules

<!-- Rn identifier (LOCAL, per file). Each rule = one unambiguous statement.
     DO NOT add FR references here — Rn -> FR-Px-nn mapping lives in macierz_pokrycia.md.
     Step prose, object/state descriptions and Rn rules DO NOT cite decision codes (DEC-Lxx) —
     they must be self-contained (AGENTS.md §65). -->

- **R1** — <rule>.
- **R2** — <rule>.

## Relationship to adjacent processes

<!-- Preceding process (what it passes and in which state) and following process (what we trigger and with what). -->

- **Preceding:** <process> — passes <object> in state <STATUS>.
- **Following:** <process> — triggered by <event/state>.

<!-- ===================== OPTIONAL SECTIONS ===================== -->

## Continuous functions (cross-cutting)

<!-- OPTIONAL. Include only when the process has functions operating in parallel with the steps
     (not in sequence). Fixed heading as above. Fn identifier per file. -->

### CONTINUOUS FUNCTION F1 — <description>

## Design notes (for verification)

<!-- OPTIONAL. Open questions/assumptions requiring a decision. Fixed heading as above.
     When an issue is resolved — move the decision to the appropriate section and remove it from here. -->

<!-- =============================================================== -->

## Change history

<!-- Entry for EVERY process change. Format: - **x.y (YYYY-MM-DD)** — change description. -->

- **1.0 (YYYY-MM-DD)** — Baseline version.
