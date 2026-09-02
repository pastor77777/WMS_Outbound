# P1-007 — Owner Acceptance Record

**Date:** 2026-09-03 Europe/Warsaw  
**Item:** `P1-007 — SHORT_ALLOCATED / SHORT_PICKED recovery and Supervisor outcomes`  
**Decision:** **FINAL PASS / OWNER ACCEPTED**

## Acceptance basis

The owner authorized final closure if the independent final verification found all required gates green. The supervisor independently reconciled the final Git refs, closeout diff scope and durable evidence and found no remaining material blocker.

The owner accepted P1-007 based on the reviewed automated + real PostgreSQL evidence. This record does **not** relabel automated browser evidence as a manual human walkthrough: the underlying browser evidence remains truthfully labelled `PLAYWRIGHT VERIFIED`.

## Final tested refs

- Mercato `outbound/p1-007`: `134db31381b4db726cd550abe6ecd4079ac21d8c`
- Scanner `main`: `b23325aae1c4f83b79d01b3650dbead3486a1041`
- Durable Shot 2 evidence: WMS_Outbound `10be7a6e2c10a05d1fe5ce6dc5aacdd93dc400a8`
- Evidence file: `05_EVIDENCE/P1-007_EVIDENCE.md`

## Final verified gates

- P1-007 genuine PostgreSQL: `20/20`
- Mercato Supervisor Playwright: `6/6`
- Scanner P1-007 Playwright: `1/1`
- P1-004: `11/11`
- P1-005: `10/10`
- P1-006 backend: `12/12`
- P1-006 Scanner: `2/2`
- P1-001: `21/21`
- P1-003: `15/15`
- P1-008: `22/22`
- Inbound/shared compatibility: `17/17`
- full `wms_outbound`: `245/245`

## Decisive technical closeout

- real SAME-key PostgreSQL concurrency with distinct PIDs / database-side lock evidence;
- real rollback proof;
- canonical Supervisor replay/conflict idempotency;
- genuine MikroORM `Migrator` lifecycle with real migration history: UP -> DOWN -> re-UP;
- incompatible zero-quantity DOWN fails before constraint mutation, preserves zero rows and leaves remediation recorded as applied;
- no product changes were introduced by the final closeout shots; Shot 1C changed only the P1-007 migration-proof test harness.

## Preserved scope boundary

P1-007 persists a durable physical-return handoff where already picked goods require later physical recovery. It does not implement the future P4 PutBackTask/RF lifecycle. P1-010 packing/repack UI and later Shipment/Carrier/ERP scope also remain outside P1-007.

## Next item

Project progress advances to **11/37 FINAL PASS**.

Next planned item: **P1-009 — Direct Pack declaration and automatic sealing — item 12/37**.