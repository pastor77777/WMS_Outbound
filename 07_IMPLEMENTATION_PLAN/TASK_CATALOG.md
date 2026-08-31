# WMS Outbound v1 — Task Catalog

**Status:** complete implementation plan; product implementation not started by ETAP 2.  
**Tasks:** 37  
**Architect requirements covered:** 109/109  
**Acceptance model:** component/integration → normal UI Playwright → Human Verified.

Every item below is an implementation work item, not a new business requirement. When wording differs, the immutable Architect Source wins.

## FND-001 — Establish target Outbound domain ownership and persistence boundaries

**Objective:** Introduce the target CustomerOrder/Line, OutboundOrder/Line and shared identifiers as explicit WMS concepts without collapsing them into legacy SO/outbound_delivery or public Sales semantics.

**Architect source/reference:** P1 R7–P1 R8; P1 R59; P1 R69; P2 R5–P2 R6; P2 R43; P3 KROK 1, P4 KROK 1

**Requirement/rule:** `FR-P1-04`, `FR-P1-33`, `FR-P1-43`, `FR-P2-03`, `FR-P2-25`, `INT-06`

**Dependencies:** none

**Target components:** Mercato WMS Outbound module, Sales/ordering adapter, domain entities/repositories

**DB impact:** Additive target tables/relations and status persistence; no destructive legacy cutover in this task.

**Backend impact:** Create owning module boundaries, repositories, transaction boundaries and adapter contracts.

**Mercato impact:** Admin/read views only as needed to inspect target orders/lines; no extra product UX.

**Scanner impact:** None directly.

**Integration impact:** Define inbound order/cancellation adapter boundary; do not invent external-system behavior beyond INT-06.

**Migration/data impact:** Coexist with legacy wms_sales_order.so_header, wms_outbound.outbound_delivery and empty public Sales candidates; record explicit mapping/cutover markers.

**Tests:** Entity/repository integration tests; state persistence tests; adapter contract tests

**Acceptance mapping:** `TC-001`, `TC-002`, `TC-003`, `TC-020`, `TC-029`, `TC-036`, `TC-092`, `TC-106`, `TC-119`, `TC-121`, `TC-134`

**Evidence:** Test names + commit SHA + migration/schema evidence + persisted-state assertions; no secrets in evidence.

**Definition of Done:** Target objects are independently addressable and traceable to source demand; legacy objects remain intact; no business state is silently shared by incompatible models.

## FND-002 — Build authoritative state/event transition and audit foundation

**Objective:** Make architect-defined state transitions server-authoritative and observable, with transition guards, domain events and audit correlation reused across P1–P4.

**Architect source/reference:** P1 Funkcja ciągła F1; P1 R70; P2 R23–P2 R24; P3 R1–P3 R2; P4 R3–P4 R4; P1 R43–P1 R46

**Requirement/rule:** `FR-P1-27`, `FR-P1-44`, `FR-P2-12`, `FR-P3-01`, `FR-P4-02`, `CON-05`

**Dependencies:** `FND-001`

**Target components:** Mercato WMS Outbound domain services, wms_orchestration

**DB impact:** Persist transition/audit facts and correlation/idempotency keys where needed.

**Backend impact:** Transition services reject illegal/non-regressive transitions and emit durable facts.

**Mercato impact:** Expose only architect-required status/action visibility to Supervisor/admin surfaces.

**Scanner impact:** Scanner consumes server state; it must not own business transition truth.

**Integration impact:** Reuse/extend orchestration outbox/retry patterns for durable effects.

**Migration/data impact:** No reinterpretation of existing Inbound state machines; additive event types only.

**Tests:** State-machine component tests; illegal transition tests; event/outbox integration tests; idempotency regression

**Acceptance mapping:** `TC-001`, `TC-009`, `TC-021`, `TC-027`, `TC-040`, `TC-041`, `TC-050`, `TC-051`, `TC-053`, `TC-128`, `TC-129`

**Evidence:** Test names + commit SHA + migration/schema evidence + persisted-state assertions; no secrets in evidence. Regression of accepted Inbound behavior for every touched shared Inventory/TU/warehouse/lock/orchestration primitive.

**Definition of Done:** All target state changes in later tasks can route through one guarded observable mechanism; illegal transitions fail safely and accepted shared behavior remains unchanged.

## FND-003 — Protect shared Inventory, TU, task-lock and warehouse compatibility

**Objective:** Prepare compatibility seams for Outbound to reuse Inventory, TU, active warehouse, task ownership/locks and orchestration without breaking accepted Inbound behavior.

**Architect source/reference:** P1 R53; P1 R71; P2 R40; P1 R4–P1 R6, P1 R52; P1 R10; P2 R6, P2 R29–P2 R30

**Requirement/rule:** `FR-P1-28`, `FR-P1-45`, `FR-P2-22`, `CON-01`, `CON-02`, `CON-03`

**Dependencies:** `FND-001`, `FND-002`

**Target components:** wms_inventory, wms_tu, wms_warehouse, record locks, Scanner session/warehouse context

**DB impact:** Prefer additive columns/relations/role tables over redefining existing Inbound process_status semantics.

**Backend impact:** Add compatibility services/guards and explicit ownership APIs.

**Mercato impact:** Mercato diagnostics only where needed.

**Scanner impact:** Reuse auth, warehouse context, scan abstraction and lock heartbeat/release.

**Integration impact:** No new external integration.

**Migration/data impact:** Additive reversible migration; shared TU rows and existing Inbound data remain valid.

**Tests:** Shared-schema migration tests; Inventory/TU regression; record-lock concurrency tests; warehouse-context tests

**Acceptance mapping:** `TC-010`, `TC-101`, `TC-130`, `TC-131`

**Evidence:** Test names + commit SHA + migration/schema evidence + persisted-state assertions; no secrets in evidence. Regression of accepted Inbound behavior for every touched shared Inventory/TU/warehouse/lock/orchestration primitive.

**Definition of Done:** Outbound can reference shared primitives without changing accepted Inbound semantics; regression suite proves compatibility.

## P1-001 — CustomerOrder intake, validation, hold, warning and continuous aggregation

**Objective:** Implement demand header/line intake and the architect-defined CustomerOrder/Line aggregation, ON_HOLD, WARNING and cancellation-visible states.

**Architect source/reference:** P1 R41–P1 R42; P1 R49–P1 R50; P1 Funkcja ciągła F1; P1 R61; P1 Wyjątki — `CustomerOrder.ON_HOLD`; P1 Wyjątki — żądanie anulowania `CustomerOrder` lub `CustomerOrderLine`; P3 KROK 1, P4 KROK 1

**Requirement/rule:** `FR-P1-21`, `FR-P1-25`, `FR-P1-27`, `FR-P1-35`, `FR-P5-08`, `FR-P5-09`, `INT-06`

**Dependencies:** `FND-001`, `FND-002`

**Target components:** CustomerOrder, CustomerOrderLine, ordering adapter

**DB impact:** Persist target header/line status, priority, slaDeadline, allowPartialShipment and warning fields.

**Backend impact:** Validation, aggregate-state recalculation and cancellation eligibility services.

**Mercato impact:** Supervisor/order visibility and permitted actions only.

**Scanner impact:** No Picker flow yet.

**Integration impact:** Receive cancellation/correction correlation from ordering system.

**Migration/data impact:** Map legacy/public Sales input non-destructively.

**Tests:** Validation/aggregation unit tests; status transition tests; INT-06 contract tests; Mercato component tests; Playwright order/hold/cancel visibility

**Acceptance mapping:** `TC-001`, `TC-003`, `TC-008`, `TC-009`, `TC-064`, `TC-065`, `TC-108`

**Evidence:** Playwright trace/screenshots/video where useful, visible identifiers, persisted server/DB assertion, and Human Verified walkthrough record for the affected user journey.

**Definition of Done:** Header/line state is derived exactly as architect rules require; ON_HOLD/WARNING/cancel paths are observable and UI traversable.

## P1-002 — ATP soft reservation queue and recalculation

**Objective:** Implement zero-ATP/backorder queue, soft ATPReservation lifecycle, priority/SLA recalculation and manual ATPReservation reduction/removal without status side effects.

**Architect source/reference:** P1 R1–P1 R2; P1 R3–P1 R4; P1 R5–P1 R6; P1 R11–P1 R12; P1 R51–P1 R52; P1 R4–P1 R6, P1 R52

**Requirement/rule:** `FR-P1-01`, `FR-P1-02`, `FR-P1-03`, `FR-P1-06`, `FR-P1-26`, `CON-01`

**Dependencies:** `P1-001`, `FND-003`

**Target components:** CustomerOrderLine ATP reservation, wms_inventory ATP service

**DB impact:** Persist ATPReservation/queue metadata separately from hard Allocation.

**Backend impact:** Atomic ATP availability evaluation/recalculation and Supervisor adjustment API.

**Mercato impact:** Supervisor inspection/manual adjustment where architect permits.

**Scanner impact:** No direct Scanner action.

**Integration impact:** None external.

**Migration/data impact:** Reuse Inventory ATP reads; do not convert current wms_inventory_reservations directly into soft reservation semantics.

**Tests:** ATP math tests; priority queue tests; parallel reservation tests; manual adjustment integration; Playwright zero ATP/replan path

**Acceptance mapping:** `TC-001`, `TC-002`, `TC-003`, `TC-135`

**Evidence:** Playwright trace/screenshots/video where useful, visible identifiers, persisted server/DB assertion, and Human Verified walkthrough record for the affected user journey. Regression of accepted Inbound behavior for every touched shared Inventory/TU/warehouse/lock/orchestration primitive.

**Definition of Done:** No double ATP promise under concurrency; BACKORDERED/replanning behavior is visible and matches source.

## P1-003 — Cyclic planning, grouping and OutboundOrder/Line creation

**Objective:** Create STANDARD OutboundOrders/Lines from uncovered demand with architect grouping, split, allowPartialShipment and requiredQty rules.

**Architect source/reference:** P1 R7–P1 R8; P1 R9–P1 R10; P1 R57; P1 R59; P1 R60; P1 R69; P1 Wyjątki — P5 E17

**Requirement/rule:** `FR-P1-04`, `FR-P1-05`, `FR-P1-31`, `FR-P1-33`, `FR-P1-34`, `FR-P1-43`, `FR-P5-17`

**Dependencies:** `P1-001`, `P1-002`

**Target components:** planning service, OutboundOrder, OutboundOrderLine

**DB impact:** Persist channel, requiredQty, source line quantity links and grouping keys.

**Backend impact:** Cyclic planner enforces customer/address/priority/SLA/allowPartial rules and avoids double coverage with CROSSDOCK.

**Mercato impact:** Planning/status observability for Supervisor.

**Scanner impact:** No direct Scanner UI.

**Integration impact:** None external.

**Migration/data impact:** Do not reuse legacy outbound_delivery grouping/status as target truth.

**Tests:** Grouping/split tests; allowPartial false coverage tests; requiredQty lifecycle tests; idempotent planning integration; Playwright planning observability

**Acceptance mapping:** `TC-001`, `TC-002`, `TC-003`, `TC-104`, `TC-106`, `TC-107`, `TC-121`, `TC-125`, `TC-127`

**Evidence:** Playwright trace/screenshots/video where useful, visible identifiers, persisted server/DB assertion, and Human Verified walkthrough record for the affected user journey.

**Definition of Done:** Repeated planner execution is safe; every created OOL traces to uncovered CustomerOrderLine quantity and obeys grouping constraints.

## P1-004 — Allocation hard reservation lifecycle

**Objective:** Implement target Allocation states and atomic blocked quantity transfer from ATPReservation to hard reservation.

**Architect source/reference:** P1 R9–P1 R10; P1 R11–P1 R12; P1 R71; P1 R4–P1 R6, P1 R52; P1 R10

**Requirement/rule:** `FR-P1-05`, `FR-P1-06`, `FR-P1-45`, `CON-01`, `CON-02`

**Dependencies:** `P1-003`, `FND-003`

**Target components:** Allocation, wms_inventory reservations/ledger

**DB impact:** Add target Allocation state, OOL ownership, reservedQty and transaction constraints; reuse Inventory balance/ledger safely.

**Backend impact:** Reserve/release/confirm/consume services with row locking and PickTask immutability guard.

**Mercato impact:** Allocation visibility for operations/Supervisor.

**Scanner impact:** Scanner sees resulting task availability only.

**Integration impact:** None external.

**Migration/data impact:** Existing inventory reservation rows remain compatible; migration is additive.

**Tests:** Allocation state tests; parallel reserve tests; ledger exactly-once tests; PickTask immutability tests

**Acceptance mapping:** `TC-001`, `TC-002`, `TC-003`, `TC-130`, `TC-131`

**Evidence:** Test names + commit SHA + migration/schema evidence + persisted-state assertions; no secrets in evidence. Regression of accepted Inbound behavior for every touched shared Inventory/TU/warehouse/lock/orchestration primitive.

**Definition of Done:** reservedQty equals actual blocked stock; duplicate/parallel operations cannot over-reserve or regress state.

## P1-005 — PickTask creation, zone ordering and operator assignment

**Objective:** Generate zone-scoped PickTasks and implement module+zone initiated assignment with single-active-warehouse-task protection.

**Architect source/reference:** P1 R13–P1 R14; P1 R54, P1 R56; P1 R55; P1 R10

**Requirement/rule:** `FR-P1-07`, `FR-P1-29`, `FR-P1-30`, `CON-02`

**Dependencies:** `P1-004`, `FND-003`

**Target components:** PickTask backend, warehouse zone permissions, Scanner task assignment

**DB impact:** Persist task zone, order, operator, status and assignment timestamps.

**Backend impact:** Task generation/ordering and atomic assignment service.

**Mercato impact:** Supervisor task visibility if existing operational patterns support it.

**Scanner impact:** Picker selects picking module and one zone; receives next eligible task; no invented pool selector.

**Integration impact:** None external.

**Migration/data impact:** Reuse lock/task ownership primitives without borrowing Putaway business rules.

**Tests:** Task generation tests; parallel assignment tests; zone permission tests; Scanner component tests; Playwright select zone/receive task

**Acceptance mapping:** `TC-001`, `TC-004`, `TC-096`, `TC-097`, `TC-098`

**Evidence:** Playwright trace/screenshots/video where useful, visible identifiers, persisted server/DB assertion, and Human Verified walkthrough record for the affected user journey. Regression of accepted Inbound behavior for every touched shared Inventory/TU/warehouse/lock/orchestration primitive.

**Definition of Done:** Two operators cannot receive same task; operator with active warehouse task is blocked; next task respects configured order.

## P1-006 — RF picking and multi-zone Picking TU continuation

**Objective:** Implement normal Scanner picking into a TU, formal pickedQty confirmation, continuation across consecutive zones and task completion without auto-closing TU.

**Architect source/reference:** P1 R15–P1 R16; P1 R55; P1 R62; P1 R67

**Requirement/rule:** `FR-P1-08`, `FR-P1-30`, `FR-P1-36`, `FR-P1-41`

**Dependencies:** `P1-005`, `P1-008`

**Target components:** Scanner picking journey, PickTask service, TU contents

**DB impact:** Persist scans/confirmed quantities/TU-task-line links and idempotency correlation.

**Backend impact:** Validate source/task/SKU/quantity/TU ownership and formal pick confirmation.

**Mercato impact:** Operational read-only visibility.

**Scanner impact:** Normal scan/click journey for task, source, SKU/qty, TU and next-zone continuation.

**Integration impact:** None external.

**Migration/data impact:** Supersede current receiving-style Outbound scanner path; reuse shared UI/auth/scan/locks.

**Tests:** Pick confirmation integration; duplicate scan/idempotency tests; multi-zone continuation tests; Playwright full RF pick

**Acceptance mapping:** `TC-001`, `TC-004`, `TC-098`, `TC-109`, `TC-118`

**Evidence:** Playwright trace/screenshots/video where useful, visible identifiers, persisted server/DB assertion, and Human Verified walkthrough record for the affected user journey. Regression of accepted Inbound behavior for every touched shared Inventory/TU/warehouse/lock/orchestration primitive.

**Definition of Done:** Formal pickedQty is authoritative, task completion does not close TU, and user can continue the same eligible TU across zones.

## P1-007 — SHORT_ALLOCATED / SHORT_PICKED recovery and Supervisor outcomes

**Objective:** Implement shortages, automatic reallocation limits, wait/cancel decisions and persistent partial-shipment policy changes.

**Architect source/reference:** P1 R41–P1 R42; P1 R43–P1 R44; P1 R45–P1 R46; P1 R47–P1 R48; P1 Wyjątki — `SHORT_ALLOCATED`, `allowPartialShipment = false`; P1 Wyjątki — `SHORT_ALLOCATED`, `allowPartialShipment = true`; P1 Wyjątki — `SHORT_PICKED`; P1 Wyjątki — `SHORT_PICKED`, `allowPartialShipment = false`, wynik „czekamy”; P1 Wyjątki — `SHORT_PICKED`, `allowPartialShipment = false`, wynik „anulowanie”; P1 Wyjątki — trwała zmiana `allowPartialShipment`

**Requirement/rule:** `FR-P1-21`, `FR-P1-22`, `FR-P1-23`, `FR-P1-24`, `FR-P5-01`, `FR-P5-02`, `FR-P5-03`, `FR-P5-04`, `FR-P5-05`, `FR-P5-06`

**Dependencies:** `P1-004`, `P1-005`, `P1-006`

**Target components:** Allocation, PickTask, CustomerOrder policy, Supervisor UI

**DB impact:** Persist shortage/reallocation counters and decision outcomes.

**Backend impact:** Create replacement Allocation/PickTask only when allowed; terminate old SHORT_PICKED task.

**Mercato impact:** Supervisor wait/cancel/allow-partial decisions and visible shortage context.

**Scanner impact:** Picker records short pick and returns to valid next action.

**Integration impact:** None external.

**Migration/data impact:** No legacy shortcut status propagation.

**Tests:** short allocation tests; reallocation limit tests; decision tests; Playwright short pick auto-retry + Supervisor paths

**Acceptance mapping:** `TC-001`, `TC-008`, `TC-060`, `TC-061`, `TC-062`, `TC-121`

**Evidence:** Playwright trace/screenshots/video where useful, visible identifiers, persisted server/DB assertion, and Human Verified walkthrough record for the affected user journey.

**Definition of Done:** Automatic retries stop exactly at configured limit; wait/cancel/partial paths produce architect-defined states and next UI action.

## P1-008 — Outbound TU identity, TUSetup, numbering, capacity and issueability

**Objective:** Extend shared TU with Outbound role/lifecycle support and implement TUSetup, TU_NUMBER/SSCC rules, weight/volume computation and issue thresholds.

**Architect source/reference:** P1 R53; P1 R63; P1 R64; P1 R65; P1 R66; P1 R68

**Requirement/rule:** `FR-P1-28`, `FR-P1-37`, `FR-P1-38`, `FR-P1-39`, `FR-P1-40`, `FR-P1-42`

**Dependencies:** `FND-003`

**Target components:** wms_tu, TUSetup, Sequence

**DB impact:** Add compatible Outbound TU role/status relation, TUSetup config and atomic number sequence; do not redefine Inbound process_status.

**Backend impact:** Number generation, capacity calculation and issuability/override guards.

**Mercato impact:** TUSetup/admin configuration where existing configuration surface requires it.

**Scanner impact:** Display/select allowed TU types and override only architect-permitted thresholds.

**Integration impact:** None external.

**Migration/data impact:** Additive shared-TU migration with rollback path and Inbound regression.

**Tests:** unique numbering concurrency; TU calculation tests; issuability/override tests; shared TU regression; Playwright TU selection/threshold behavior

**Acceptance mapping:** `TC-010`, `TC-110`, `TC-111`, `TC-114`, `TC-115`, `TC-116`, `TC-117`, `TC-120`

**Evidence:** Playwright trace/screenshots/video where useful, visible identifiers, persisted server/DB assertion, and Human Verified walkthrough record for the affected user journey. Regression of accepted Inbound behavior for every touched shared Inventory/TU/warehouse/lock/orchestration primitive.

**Definition of Done:** TU numbers are unique; external type semantics and absolute/non-absolute issue blocks match architect source; Inbound TU behavior remains accepted.

## P1-009 — Direct Pack declaration and automatic sealing

**Objective:** Implement immutable directPackDeclared at first scan and automatic qualification/sealing/line PACKED behavior when issue criteria allow.

**Architect source/reference:** P1 R15–P1 R16; P1 R17–P1 R18

**Requirement/rule:** `FR-P1-08`, `FR-P1-09`

**Dependencies:** `P1-006`, `P1-008`

**Target components:** TU, OutboundOrderLine, Scanner picking

**DB impact:** Persist immutable directPackDeclared and resulting TU/line transition facts.

**Backend impact:** Guard first-scan declaration and automatic READY_TO_PACK→PACK_QUALIFIED→PACKING_SEALED path.

**Mercato impact:** Visibility of direct-pack result.

**Scanner impact:** Picker declares direct pack on first scan and sees outcome; no Packer action when auto path succeeds.

**Integration impact:** None external.

**Migration/data impact:** None beyond additive target fields.

**Tests:** immutability tests; issue criteria tests; transition integration; Playwright direct-pack journey

**Acceptance mapping:** `TC-001`, `TC-004`, `TC-005`

**Evidence:** Playwright trace/screenshots/video where useful, visible identifiers, persisted server/DB assertion, and Human Verified walkthrough record for the affected user journey.

**Definition of Done:** Declaration cannot change later; successful direct pack reaches sealed/PACKED states without hidden manual packing step.

## P1-010 — Packing, repack, consolidation and discrepancy handling

**Objective:** Implement READY_TO_PACK processing, qualification, repack-all/repack-by-SKU, consolidation and packing shortages/damage/unexpected SKU outcomes.

**Architect source/reference:** P1 R19–P1 R20; P1 R21–P1 R22; P1 R23–P1 R24; P1 R25–P1 R26; P1 Wyjątki — brak lub uszkodzenie przy „repack by SKU”

**Requirement/rule:** `FR-P1-10`, `FR-P1-11`, `FR-P1-12`, `FR-P1-13`, `FR-P5-07`

**Dependencies:** `P1-006`, `P1-008`

**Target components:** Packing backend, Mercato/Packer UI, TU contents, QC handoff

**DB impact:** Persist source/target TU content moves, packing confirmations and discrepancy facts.

**Backend impact:** Server validates repack rules, shortage confirmation and terminal TU/line effects.

**Mercato impact:** Packer flow for scan/select/repack/confirm discrepancy and resulting status.

**Scanner impact:** Scanner only if packing is delivered there by existing product boundary; otherwise no duplicate UI.

**Integration impact:** QC handoff/event only as architect requires; no invented QC workflow.

**Migration/data impact:** No destructive shared TU changes.

**Tests:** packing calculation tests; repack transaction tests; discrepancy tests; Playwright repack/consolidation/shortage path

**Acceptance mapping:** `TC-001`, `TC-005`, `TC-006`, `TC-063`

**Evidence:** Playwright trace/screenshots/video where useful, visible identifiers, persisted server/DB assertion, and Human Verified walkthrough record for the affected user journey. Regression of accepted Inbound behavior for every touched shared Inventory/TU/warehouse/lock/orchestration primitive.

**Definition of Done:** Packer can complete allowed modes through normal UI; shortage requires recheck+explicit confirmation; damage/unexpected behavior matches source.

## P1-011 — Shipment grouping, closure and partial-shipment gates

**Objective:** Create stable Shipment grouping from packed TUs and enforce SLA/allowPartialShipment/customer-order completeness guards, including late-TU follow-up shipments.

**Architect source/reference:** P1 R25–P1 R26; P1 R27–P1 R28; P1 R29–P1 R30; P1 R57; P1 R58; P1 R60; P1 Wyjątki — P5 E17; P1 R37–P1 R41

**Requirement/rule:** `FR-P1-13`, `FR-P1-14`, `FR-P1-15`, `FR-P1-31`, `FR-P1-32`, `FR-P1-34`, `FR-P5-17`, `CON-04`

**Dependencies:** `P1-003`, `P1-009`, `P1-010`

**Target components:** Shipment, Shipment-TU/line links, CustomerOrder guard

**DB impact:** Persist grouping keys, contributing TUs/orders/lines and closure boundary.

**Backend impact:** Idempotent grouping/closure and allowPartial false gate.

**Mercato impact:** Shipment operational view and blocked reason.

**Scanner impact:** No RF action beyond showing handoff state if needed.

**Integration impact:** Downstream carrier/ERP starts only after ready state.

**Migration/data impact:** Do not treat public sales_shipments as authoritative lifecycle without explicit adapter/mapping.

**Tests:** grouping key tests; parallel grouping tests; allowPartial guard tests; late TU tests; Playwright shipment readiness/blocking

**Acceptance mapping:** `TC-001`, `TC-006`, `TC-007`, `TC-104`, `TC-105`, `TC-107`, `TC-122`, `TC-123`, `TC-124`, `TC-125`, `TC-126`, `TC-127`, `TC-128`

**Evidence:** Playwright trace/screenshots/video where useful, visible identifiers, persisted server/DB assertion, and Human Verified walkthrough record for the affected user journey.

**Definition of Done:** Each TU belongs to at most one active Shipment grouping; repeated planner runs are non-regressive and partial-forbidden orders cannot dispatch early.

## P1-012 — Carrier, Region and CarrierSetup selection

**Objective:** Implement delivery Region and CarrierSetup applicability with weight/volume ranges, unique priority tie-break and Supervisor manual fallback/override.

**Architect source/reference:** P1 R31–P1 R32; P1 R33–P1 R34; P1 R51–P1 R52; P1 Wyjątki — brak wyniku Carrier Selection

**Requirement/rule:** `FR-P1-16`, `FR-P1-17`, `FR-P1-26`, `FR-P5-10`

**Dependencies:** `P1-011`, `P1-008`

**Target components:** Carrier master adapter, Region, CarrierSetup, Carrier selection service, Mercato Supervisor UI

**DB impact:** Add/extend carrier setup/region config and uniqueness for priority where architect requires.

**Backend impact:** Deterministic selection: region + maxWeight/current contents + maxVolume/TUSetup; narrowest ranges then priority.

**Mercato impact:** Show automatic result, no-result reason and manual choice/override.

**Scanner impact:** No Scanner-owned carrier rules.

**Integration impact:** Do not call external carrier API for v1 selection.

**Migration/data impact:** Map existing ref_carrier/provider data without importing provider-label lifecycle.

**Tests:** selection/tie-break tests; no-match/manual tests; config validation; Playwright automatic + manual fallback

**Acceptance mapping:** `TC-001`, `TC-003`, `TC-007`, `TC-066`

**Evidence:** Playwright trace/screenshots/video where useful, visible identifiers, persisted server/DB assertion, and Human Verified walkthrough record for the affected user journey.

**Definition of Done:** Same inputs always yield same carrier; external TU/no maxVolume follows manual path; Supervisor can override as source allows.

## P1-013 — WMS label generation and pre-manifest loading carrier correction

**Objective:** Generate/print label from WMS-owned data and support architect-permitted carrier correction before manifest close without introducing carrier Label API.

**Architect source/reference:** P1 R33–P1 R34; P1 R35–P1 R36; P1 Wyjątki — błąd etykiety lub odrzucenie przez przewoźnika

**Requirement/rule:** `FR-P1-17`, `FR-P1-18`, `FR-P5-11`

**Dependencies:** `P1-012`

**Target components:** Shipment label service, Mercato print UI

**DB impact:** Persist label generation/print result needed for state/evidence; no provider label ownership.

**Backend impact:** Generate internal label payload and guarded correction behavior.

**Mercato impact:** Print/reprint/visible error path exactly within source scope.

**Scanner impact:** No Scanner flow unless existing dispatch UI is Scanner-owned.

**Integration impact:** Explicitly no external carrier label API in v1.

**Migration/data impact:** Do not reuse carrier_shipments label_url/label_data as business authority.

**Tests:** label data tests; print/error tests; carrier correction state tests; Playwright label generation/print/error

**Acceptance mapping:** `TC-001`, `TC-007`, `TC-066`

**Evidence:** Playwright trace/screenshots/video where useful, visible identifiers, persisted server/DB assertion, and Human Verified walkthrough record for the affected user journey.

**Definition of Done:** A user can generate/print the WMS label through normal UI; errors remain recoverable without invented carrier rejection integration.

## P1-014 — ERP Shipment POST, error state and safe retry

**Objective:** Implement POSTING_PENDING/POSTED/POSTING_ERROR with unique Shipment identifier, durable attempt evidence and idempotent safe retry.

**Architect source/reference:** P1 R35–P1 R36; P1 R37–P1 R38; P1 KROK 13; P1 R43–P1 R46

**Requirement/rule:** `FR-P1-18`, `FR-P1-19`, `INT-04`, `INT-05`, `CON-05`

**Dependencies:** `P1-011`, `P1-013`, `FND-002`

**Target components:** Shipment posting service, wms_orchestration outbox/retry, ERP adapter

**DB impact:** Persist posting state, error details safe for operations, idempotency/correlation and attempt history.

**Backend impact:** Use durable outbox/retry; duplicate acceptance cannot double-apply business effects.

**Mercato impact:** Supervisor sees error and can trigger architect-allowed retry/cancel action.

**Scanner impact:** No RF action.

**Integration impact:** INT-04/05 request/response contract; downstream ERP rejection is observable.

**Migration/data impact:** Reuse orchestration infrastructure; do not repurpose carrier provider posting.

**Tests:** contract tests; retry/idempotency tests; failure recovery; Playwright POSTING_ERROR→retry→POSTED

**Acceptance mapping:** `TC-001`, `TC-007`, `TC-008`

**Evidence:** Playwright trace/screenshots/video where useful, visible identifiers, persisted server/DB assertion, and Human Verified walkthrough record for the affected user journey. Regression of accepted Inbound behavior for every touched shared Inventory/TU/warehouse/lock/orchestration primitive.

**Definition of Done:** ERP duplicate/retry behavior is non-regressive; operational UI exposes error and safe retry; state advances only on accepted response.

## P1-015 — CarrierManifest lifecycle and dispatch boundaries

**Objective:** Implement one-manifest assignment, OPEN→CLOSED→HANDED_OVER→CONFIRMED, irreversible CLOSED composition boundary and final warehouse confirmation.

**Architect source/reference:** P1 R39–P1 R40; P1 R70; P1 R72; P1 R37–P1 R41; P1 R43–P1 R46

**Requirement/rule:** `FR-P1-20`, `FR-P1-44`, `FR-P1-46`, `CON-04`, `CON-05`

**Dependencies:** `P1-014`

**Target components:** CarrierManifest, Shipment, dispatch UI

**DB impact:** Persist manifest, Shipment membership, status/audit timestamps and uniqueness constraints.

**Backend impact:** Guard close/handover/confirm and forbid membership changes after CLOSED.

**Mercato impact:** Dispatch/Supervisor UI for manifest lifecycle.

**Scanner impact:** No Scanner action unless loading handover is scanner-owned in current product.

**Integration impact:** No external carrier API required.

**Migration/data impact:** New target model; do not equate carrier_shipments with manifest.

**Tests:** manifest transition tests; composition lock concurrency; idempotent confirm tests; Playwright close/handover/confirm

**Acceptance mapping:** `TC-001`, `TC-008`, `TC-128`, `TC-129`, `TC-132`, `TC-133`

**Evidence:** Playwright trace/screenshots/video where useful, visible identifiers, persisted server/DB assertion, and Human Verified walkthrough record for the affected user journey.

**Definition of Done:** CLOSED is irreversible and distinct from physical handover; duplicate confirm cannot settle stock twice.

## P1-016 — Final line/order/inventory settlement and cancellation boundaries

**Objective:** Settle OutboundOrderLine/Allocation/Inventory quantities per confirmed contributing manifests, complete orders only when all contributing manifests are confirmed, and enforce cancellation boundaries.

**Architect source/reference:** P1 R37–P1 R38; P1 R41–P1 R42; P1 R49–P1 R50; P1 Funkcja ciągła F1; P1 R70; P1 R71; P1 R72; P1 Wyjątki — żądanie anulowania `CustomerOrder` lub `CustomerOrderLine`; P3 KROK 1, P4 KROK 1; P1 R43–P1 R46

**Requirement/rule:** `FR-P1-19`, `FR-P1-21`, `FR-P1-25`, `FR-P1-27`, `FR-P1-44`, `FR-P1-45`, `FR-P1-46`, `FR-P5-09`, `INT-06`, `CON-05`

**Dependencies:** `P1-015`, `P1-004`, `P1-001`

**Target components:** Inventory ledger, Allocation, OutboundOrder/Line, CustomerOrder/Line

**DB impact:** Exactly-once settlement records keyed by manifest/line/allocation; cancellation audit.

**Backend impact:** Aggregate shipped/fulfilled/completed states and choose P3/P4 cancellation path from formal pickedQty.

**Mercato impact:** Supervisor sees final status and blocked cancellation reason.

**Scanner impact:** Scanner receives recovery task only through P4 when applicable.

**Integration impact:** INT-06 cancellation correlation.

**Migration/data impact:** Supersede legacy trigger that marks SO SHIPPED merely from outbound_delivery.SHIPPED.

**Tests:** multi-shipment settlement tests; duplicate manifest tests; cancellation boundary tests; Playwright final close + cancellation rejection

**Acceptance mapping:** `TC-001`, `TC-003`, `TC-008`, `TC-009`, `TC-065`, `TC-128`, `TC-129`, `TC-130`, `TC-131`, `TC-132`, `TC-133`

**Evidence:** Playwright trace/screenshots/video where useful, visible identifiers, persisted server/DB assertion, and Human Verified walkthrough record for the affected user journey. Regression of accepted Inbound behavior for every touched shared Inventory/TU/warehouse/lock/orchestration primitive.

**Definition of Done:** No line/order closes early; quantities settle exactly once; cancellation routes correctly before/after formal pick and respects manifest boundary.

## P2-001 — Inbound crossdock boundary and demand/source eligibility

**Objective:** Consume eligible ELEMENTARY inbound TU at IN_CROSS_DOCK and calculate demandEligibleQty/sourceEligibleQty/crossDockEligibleQty without double coverage.

**Architect source/reference:** P2 R1–P2 R2; P2 R5–P2 R6; P2 R41; P2 R42; P2 R43; P2 KROK 1; P2 R6, P2 R29–P2 R30

**Requirement/rule:** `FR-P2-01`, `FR-P2-03`, `FR-P2-23`, `FR-P2-24`, `FR-P2-25`, `INT-01`, `CON-03`

**Dependencies:** `FND-001`, `FND-003`, `P1-002`, `P1-003`

**Target components:** Inbound crossdock adapter, CustomerOrderLine demand, TU source eligibility

**DB impact:** Persist sourceInboundTU correlation and eligibility/assignment facts.

**Backend impact:** Eligibility service excludes ATPReservation and non-cancelled OOL coverage; source capacity subtracts planned/confirmed/damaged use.

**Mercato impact:** Supervisor visibility only.

**Scanner impact:** Crossdock module receives only tasks created after binding demand.

**Integration impact:** INT-01 receives TU id, SKU, ASN-declared qty, receipt correlation.

**Migration/data impact:** Reuse existing wms_cross_dock_demands only if mapping is explicit; do not mutate Inbound qualification semantics.

**Tests:** eligibility math tests; ELEMENTARY guard; parallel assignment tests; INT-01 contract; Inbound regression

**Acceptance mapping:** `TC-020`, `TC-029`, `TC-036`, `TC-092`, `TC-102`, `TC-103`, `TC-119`, `TC-134`

**Evidence:** Test names + commit SHA + migration/schema evidence + persisted-state assertions; no secrets in evidence. Regression of accepted Inbound behavior for every touched shared Inventory/TU/warehouse/lock/orchestration primitive.

**Definition of Done:** No quantity is double planned; aggregate TU is rejected; zero eligible demand creates no task and leaves residual for Inbound settlement.

## P2-002 — Crossdock OutboundOrder/Line and CrossDockPickTask planning

**Objective:** Create CROSSDOCK OutboundOrder/Line and exactly-once CrossDockPickTask assignments without Allocation, preserving target continuity and inherited SLA/priority.

**Architect source/reference:** P2 R3–P2 R4; P2 R5–P2 R6; P2 R7–P2 R8; P2 R29–P2 R30; P2 R39; P2 R40; P2 R43; P2 R6, P2 R29–P2 R30

**Requirement/rule:** `FR-P2-02`, `FR-P2-03`, `FR-P2-04`, `FR-P2-16`, `FR-P2-21`, `FR-P2-22`, `FR-P2-25`, `CON-03`

**Dependencies:** `P2-001`, `FND-002`

**Target components:** CrossDock planner, CrossDockPickTask, OutboundOrder/Line

**DB impact:** Persist task plannedQty/source/target line/status and unique assignment constraints.

**Backend impact:** Lazy target outbound TU creation at first placement; active task always has target; no Allocation.

**Mercato impact:** Operational visibility.

**Scanner impact:** Operator enters Crossdock module and receives next eligible task; no zone selector.

**Integration impact:** None beyond Inbound source correlation.

**Migration/data impact:** New task model; existing crossdock demand table remains source primitive only.

**Tests:** task planning tests; unique assignment concurrency; channel/SLA inheritance tests; Playwright crossdock module assignment

**Acceptance mapping:** `TC-020`, `TC-021`, `TC-022`, `TC-029`, `TC-030`, `TC-032`, `TC-033`, `TC-034`, `TC-036`, `TC-092`, `TC-099`, `TC-101`, `TC-119`, `TC-134`

**Evidence:** Playwright trace/screenshots/video where useful, visible identifiers, persisted server/DB assertion, and Human Verified walkthrough record for the affected user journey.

**Definition of Done:** One source quantity is assigned at most once; active task has target continuity; no Allocation is created.

## P2-003 — RF crossdock sorting into outbound TUs

**Objective:** Implement Scanner sorting of one source TU across target outbound TU(s), including lazy target creation, physical-full/SLA sealing and task completion.

**Architect source/reference:** P2 R3–P2 R4; P2 R7–P2 R8; P2 R9–P2 R10; P2 R19–P2 R20; P2 R23–P2 R24; P2 R34; P2 R39; P2 R40

**Requirement/rule:** `FR-P2-02`, `FR-P2-04`, `FR-P2-05`, `FR-P2-10`, `FR-P2-12`, `FR-P2-19`, `FR-P2-21`, `FR-P2-22`

**Dependencies:** `P2-002`, `P1-008`

**Target components:** Scanner crossdock journey, CrossDockPickTask service, Outbound TU

**DB impact:** Persist confirmed placements, target TU links and task/TU transitions.

**Backend impact:** Validate source SKU/qty/task target and target TU close conditions.

**Mercato impact:** Read-only operational visibility.

**Scanner impact:** Normal scan flow from task→source TU→SKU/qty→target TU(s)→complete.

**Integration impact:** None external.

**Migration/data impact:** Supersede receiving-style Outbound mode while reusing scanner primitives.

**Tests:** placement transaction tests; multi-target TU tests; duplicate scan tests; Playwright 1:1 + n:n crossdock

**Acceptance mapping:** `TC-020`, `TC-021`, `TC-022`, `TC-023`, `TC-027`, `TC-030`, `TC-032`, `TC-033`, `TC-034`, `TC-035`, `TC-037`, `TC-099`, `TC-101`

**Evidence:** Playwright trace/screenshots/video where useful, visible identifiers, persisted server/DB assertion, and Human Verified walkthrough record for the affected user journey. Regression of accepted Inbound behavior for every touched shared Inventory/TU/warehouse/lock/orchestration primitive.

**Definition of Done:** User can physically sort through normal Scanner UI; task/TU states and confirmed quantities match persisted facts.

## P2-004 — Crossdock shortage, damage, empty-TU and cancellation recovery

**Objective:** Implement confirmed shortage, DAMAGED, unexpected SKU, empty source TU, in-progress cancellation and residual/finalization rules.

**Architect source/reference:** P2 R11–P2 R12; P2 R13–P2 R14; P2 R15–P2 R16; P2 R17, P2 R18, P2 R37; P2 R19–P2 R20; P2 R21–P2 R22; P2 R23–P2 R24; P2 R28; P2 R34; P2 Wyjątki — niedobór lub `DAMAGED`, `allowPartialShipment = false`; P2 Wyjątki — niedobór lub `DAMAGED`, `allowPartialShipment = true`; P2 Wyjątki — puste `TU`; P2 Wyjątki — ogólne anulowanie przy `CrossDockPickTask IN_PROGRESS`

**Requirement/rule:** `FR-P2-06`, `FR-P2-07`, `FR-P2-08`, `FR-P2-09`, `FR-P2-10`, `FR-P2-11`, `FR-P2-12`, `FR-P2-15`, `FR-P2-19`, `FR-P5-12`, `FR-P5-13`, `FR-P5-14`, `FR-P5-15`

**Dependencies:** `P2-003`

**Target components:** CrossDockPickTask, Scanner crossdock exceptions, Inbound settlement handoff

**DB impact:** Persist confirmedQty/damagedQty/residual and cancellation/finalization facts.

**Backend impact:** Apply allowPartial outcomes and source/target TU recovery non-regressively.

**Mercato impact:** Supervisor decisions where architect requires.

**Scanner impact:** Operator can record shortage/damage/empty source and see valid next action.

**Integration impact:** Produce crossdock result for Inbound settlement; no extra residual field beyond contract.

**Migration/data impact:** Preserve Inbound TU ownership outside Outbound-confirmed effects.

**Tests:** quantity conservation tests; damage/empty/cancel tests; partial-policy tests; Playwright shortage/damage/empty/cancel paths

**Acceptance mapping:** `TC-021`, `TC-023`, `TC-024`, `TC-025`, `TC-026`, `TC-027`, `TC-035`, `TC-039`, `TC-067`, `TC-081`, `TC-121`

**Evidence:** Playwright trace/screenshots/video where useful, visible identifiers, persisted server/DB assertion, and Human Verified walkthrough record for the affected user journey. Regression of accepted Inbound behavior for every touched shared Inventory/TU/warehouse/lock/orchestration primitive.

**Definition of Done:** confirmed+damaged+residual quantities reconcile to declared source quantity; cancellation never leaves orphan active target/task.

## P2-005 — Goods Receipt correlation gate and re-evaluation

**Objective:** Correlate GR by sourceInboundTU, store GR acceptance status per contributing source, gate Shipment posting, and re-evaluate on later messages.

**Architect source/reference:** P2 R25–P2 R26; P2 R35–P2 R36; P2 R31–P2 R32; P2 R33; P2 Wyjątki — `P2 R35`–`P2 R36`; P2 KROK 3; P2 KROK 4

**Requirement/rule:** `FR-P2-13`, `FR-P2-14`, `FR-P2-17`, `FR-P2-18`, `FR-P5-16`, `INT-02`, `INT-03`

**Dependencies:** `P2-004`, `P1-014`

**Target components:** Crossdock GR status, Shipment posting gate, wms_orchestration

**DB impact:** Persist GR_PENDING/GR_ACCEPTED/GR_REJECTED per task/source and correlation.

**Backend impact:** Re-evaluate every inbound GR message; GR_REJECTED keeps gate unsatisfied without forcing Shipment POSTING_ERROR.

**Mercato impact:** Supervisor can inspect contributing source GR statuses.

**Scanner impact:** No direct Scanner decision.

**Integration impact:** INT-02 sends confirmedQty/damagedQty; INT-03 correlates sourceInboundTU/GR_SETTLEMENT_SOURCE; Inbound owns GR retry.

**Migration/data impact:** Reuse orchestration; do not move Inbound retry responsibility into Outbound.

**Tests:** GR correlation tests; gate aggregation tests; re-evaluation tests; contract tests; Playwright Supervisor GR gate visibility

**Acceptance mapping:** `TC-028`, `TC-030`, `TC-031`, `TC-038`, `TC-067`

**Evidence:** Playwright trace/screenshots/video where useful, visible identifiers, persisted server/DB assertion, and Human Verified walkthrough record for the affected user journey. Regression of accepted Inbound behavior for every touched shared Inventory/TU/warehouse/lock/orchestration primitive.

**Definition of Done:** Shipment cannot post until all contributing source TUs are GR_ACCEPTED; later accepted message unblocks without manual data repair.

## P2-006 — Crossdock join into common Shipment/dispatch downstream

**Objective:** Ensure CROSSDOCK lines/TUs join the same Shipment, carrier, ERP, manifest and settlement pipeline as P1 while preserving priority/SLA and mixed-channel completeness.

**Architect source/reference:** P2 R25–P2 R26; P2 R33; P2 R43; P1 R57; P1 R58; P1 R37–P1 R41; P1 R43–P1 R46

**Requirement/rule:** `FR-P2-13`, `FR-P2-18`, `FR-P2-25`, `FR-P1-31`, `FR-P1-32`, `CON-04`, `CON-05`

**Dependencies:** `P2-005`, `P1-011`, `P1-012`, `P1-014`, `P1-015`, `P1-016`

**Target components:** Shipment pipeline, CustomerOrder mixed-channel guard

**DB impact:** Use same Shipment/TU/line relation and settlement records.

**Backend impact:** Mixed STANDARD+CROSSDOCK coverage uses one canonical completeness guard.

**Mercato impact:** Normal Shipment operational UI.

**Scanner impact:** No duplicate Scanner dispatch model.

**Integration impact:** Same ERP/manifest paths as P1, with GR gate inserted.

**Migration/data impact:** No separate crossdock shipment schema.

**Tests:** mixed-channel grouping tests; allowPartial false mixed coverage; end-to-end crossdock dispatch; Playwright crossdock→Shipment→manifest

**Acceptance mapping:** `TC-028`, `TC-030`, `TC-031`, `TC-104`, `TC-105`, `TC-119`, `TC-122`, `TC-123`, `TC-124`, `TC-125`, `TC-126`

**Evidence:** Playwright trace/screenshots/video where useful, visible identifiers, persisted server/DB assertion, and Human Verified walkthrough record for the affected user journey.

**Definition of Done:** Crossdock does not fork a second logistics lifecycle; mixed-channel order cannot dispatch partially when forbidden.

## P3-001 — Reservation Release before formal pick

**Objective:** Implement cancellation/shortage release of Allocation and Inventory before formal physical confirmation, including line/order state effects and exact source-location race instruction.

**Architect source/reference:** P3 R1–P3 R2; P3 R3–P3 R4; P3 R5–P3 R6; P3 KROK 1, P4 KROK 1

**Requirement/rule:** `FR-P3-01`, `FR-P3-02`, `FR-P3-03`, `INT-06`

**Dependencies:** `P1-004`, `P1-001`, `P1-016`

**Target components:** Reservation Release service, Allocation, Inventory, Mercato/Scanner cancellation feedback

**DB impact:** Persist release reason/correlation and exactly-once inventory effect.

**Backend impact:** Choose P3 only when formal pickedQty=0; cancel task/release reservation as required.

**Mercato impact:** Supervisor cancellation entry/feedback.

**Scanner impact:** If physical movement occurred before formal scan, instruct return to exact source without creating PutBackTask.

**Integration impact:** INT-06 cancellation correlation.

**Migration/data impact:** No destructive legacy cleanup.

**Tests:** release state tests; inventory exactly-once tests; cancel task tests; Playwright pre-pick cancellation

**Acceptance mapping:** `TC-040`, `TC-041`

**Evidence:** Playwright trace/screenshots/video where useful, visible identifiers, persisted server/DB assertion, and Human Verified walkthrough record for the affected user journey. Regression of accepted Inbound behavior for every touched shared Inventory/TU/warehouse/lock/orchestration primitive.

**Definition of Done:** Released stock is available once; no PutBackTask is created for true pre-pick release; line/order state is consistent.

## P3-002 — Reservation retention policy and automatic release timer

**Objective:** Implement warehouse policy variants retain / auto-release-after-time / Supervisor decision, with configured time independent of priority/SLA.

**Architect source/reference:** P3 R9; P3 R10

**Requirement/rule:** `FR-P3-05`, `FR-P3-06`

**Dependencies:** `P3-001`

**Target components:** warehouse Outbound policy, scheduler/decision service

**DB impact:** Persist policy/config and due timestamp/decision audit as needed.

**Backend impact:** Deterministic policy evaluation and idempotent scheduled release.

**Mercato impact:** Supervisor decision UI for policy variant.

**Scanner impact:** No Picker action unless task availability changes.

**Integration impact:** None external.

**Migration/data impact:** Use existing scheduler/config infrastructure where safe.

**Tests:** policy tests; timer/idempotency tests; priority/SLA independence; Playwright Supervisor retention decision

**Acceptance mapping:** `TC-112`, `TC-113`

**Evidence:** Playwright trace/screenshots/video where useful, visible identifiers, persisted server/DB assertion, and Human Verified walkthrough record for the affected user journey.

**Definition of Done:** All three policy variants behave exactly as configured and repeated scheduler runs do not double-release.

## P3-003 — Cancellation race: physical movement before formal confirmation

**Objective:** Handle the architect race window between source-location physical pick and scan into Picking TU, including exact-source return instruction and escalation to P4 once formal pickedQty exists.

**Architect source/reference:** P3 R7–P3 R8; P3 KROK 1, P4 KROK 1

**Requirement/rule:** `FR-P3-04`, `INT-06`

**Dependencies:** `P3-001`, `P4-001`

**Target components:** PickTask cancellation guard, Scanner picking state, cancellation router

**DB impact:** Persist enough correlation to distinguish formal pickedQty from pre-scan physical movement.

**Backend impact:** Atomic cancellation decision chooses P3 vs P4 from authoritative pickedQty.

**Mercato impact:** Supervisor sees selected recovery path.

**Scanner impact:** Picker receives exact source-location return instruction in pre-scan race.

**Integration impact:** INT-06 cancellation router.

**Migration/data impact:** No new business state beyond source-defined paths.

**Tests:** race concurrency tests; router tests; Playwright cancellation during pick window

**Acceptance mapping:** `TC-042`, `TC-043`

**Evidence:** Playwright trace/screenshots/video where useful, visible identifiers, persisted server/DB assertion, and Human Verified walkthrough record for the affected user journey.

**Definition of Done:** No ambiguous recovery: pickedQty=0 uses P3 race instruction; pickedQty>0 routes to P4.

## P4-001 — Post-pick/post-pack cancellation approval and logical effects

**Objective:** Implement approval/boundary rules for cancellation after formal pick, with immediate logical cancellation/release and block at/after architect dispatch boundary.

**Architect source/reference:** P4 R1–P4 R2; P4 R3–P4 R4; P3 KROK 1, P4 KROK 1

**Requirement/rule:** `FR-P4-01`, `FR-P4-02`, `INT-06`

**Dependencies:** `P1-014`, `P1-015`, `P1-016`

**Target components:** cancellation router, CustomerOrder/Line, OutboundOrder/Line, Shipment

**DB impact:** Persist approval/correlation, logical cancellation and recovery quantity.

**Backend impact:** Enforce packed Supervisor approval and POSTING_PENDING/manifest boundaries.

**Mercato impact:** Supervisor cancellation action with explicit blocked reason.

**Scanner impact:** Scanner receives PutBackTask later, not cancellation authority.

**Integration impact:** INT-06 cancellation correlation.

**Migration/data impact:** Supersede legacy coarse cancellation shortcuts.

**Tests:** boundary tests; approval tests; duplicate cancellation tests; Playwright packed cancellation allowed/blocked

**Acceptance mapping:** `TC-050`, `TC-051`, `TC-053`

**Evidence:** Playwright trace/screenshots/video where useful, visible identifiers, persisted server/DB assertion, and Human Verified walkthrough record for the affected user journey.

**Definition of Done:** Logical state changes immediately when allowed; forbidden late cancellation is rejected visibly; recovery quantity is preserved for P4 task.

## P4-002 — PutBackTask model, FIFO assignment and task lifecycle

**Objective:** Create PutBackTask for pickedQty>0 and assign through RF module with source-defined lifecycle and FIFO behavior.

**Architect source/reference:** P4 R5–P4 R6; P4 R9

**Requirement/rule:** `FR-P4-03`, `FR-P4-05`

**Dependencies:** `P4-001`, `FND-003`

**Target components:** PutBackTask backend, Scanner module assignment

**DB impact:** Persist task source material/TU, proposed location, operator, status and timestamps.

**Backend impact:** FIFO assignment, single-active-task guard and state transition service.

**Mercato impact:** Supervisor task visibility.

**Scanner impact:** Operator enters PutBack module and receives next task; no zone selection.

**Integration impact:** None external.

**Migration/data impact:** Reuse generic task/lock patterns, not Inbound Putaway business states.

**Tests:** task creation tests; FIFO/parallel assignment tests; state tests; Playwright PutBack task assignment

**Acceptance mapping:** `TC-050`, `TC-051`, `TC-052`, `TC-100`

**Evidence:** Playwright trace/screenshots/video where useful, visible identifiers, persisted server/DB assertion, and Human Verified walkthrough record for the affected user journey. Regression of accepted Inbound behavior for every touched shared Inventory/TU/warehouse/lock/orchestration primitive.

**Definition of Done:** Every eligible picked cancellation creates exactly one recoverable task and assignment cannot duplicate work.

## P4-003 — RF PutBack location validation loop and Inventory recovery

**Objective:** Implement proposed/operator destination scan, invalid-location rejection loop with no auto escalation, completion and Inventory PICKED→AVAILABLE recovery.

**Architect source/reference:** P4 R5–P4 R6; P4 R7–P4 R8; P4 R9

**Requirement/rule:** `FR-P4-03`, `FR-P4-04`, `FR-P4-05`

**Dependencies:** `P4-002`

**Target components:** Scanner PutBack journey, location validation, Inventory ledger

**DB impact:** Persist validation attempts/completion and exactly-once stock movement.

**Backend impact:** Validate destination server-side and settle inventory only on successful completion.

**Mercato impact:** Operational completion visibility.

**Scanner impact:** Normal scan flow supports repeated invalid location attempts until valid.

**Integration impact:** None external.

**Migration/data impact:** Reuse warehouse location primitives; keep Inbound Putaway semantics untouched.

**Tests:** location validation tests; retry loop tests; inventory ledger tests; Playwright invalid→invalid→valid completion

**Acceptance mapping:** `TC-050`, `TC-051`, `TC-052`, `TC-100`

**Evidence:** Playwright trace/screenshots/video where useful, visible identifiers, persisted server/DB assertion, and Human Verified walkthrough record for the affected user journey. Regression of accepted Inbound behavior for every touched shared Inventory/TU/warehouse/lock/orchestration primitive.

**Definition of Done:** Invalid location never completes/moves stock; valid completion returns exact quantity AVAILABLE once, with no automatic escalation invented.

## X-001 — Enforce CON-01..05 concurrency and exactly-once business effects

**Objective:** Apply explicit transactions, row/advisory locks, unique constraints and idempotency keys at every authoritative boundary named by the concurrency requirements.

**Architect source/reference:** P1 R4–P1 R6, P1 R52; P1 R10; P2 R6, P2 R29–P2 R30; P1 R37–P1 R41; P1 R43–P1 R46

**Requirement/rule:** `CON-01`, `CON-02`, `CON-03`, `CON-04`, `CON-05`

**Dependencies:** `P1-004`, `P1-011`, `P1-014`, `P1-015`, `P2-002`

**Target components:** DB transaction boundaries, Inventory, Shipment, Crossdock, ERP/manifest effects

**DB impact:** Add required unique indexes/locking strategy/idempotency records and failure-safe transactions.

**Backend impact:** Centralize guards in owning services; clients cannot bypass them.

**Mercato impact:** Only surface deterministic conflict/retry feedback.

**Scanner impact:** Scanner handles conflict response and recovers safely without local business override.

**Integration impact:** ERP/manifest duplicates are safe.

**Migration/data impact:** Additive migration with stress tests before any legacy cutover.

**Tests:** parallel ATP tests; PickTask immutability tests; crossdock double-assignment race; Shipment grouping race; ERP/manifest duplicate tests

**Acceptance mapping:** No standalone architect TC; acceptance is through linked requirements/downstream journey.

**Evidence:** Test names + commit SHA + migration/schema evidence + persisted-state assertions; no secrets in evidence. Regression of accepted Inbound behavior for every touched shared Inventory/TU/warehouse/lock/orchestration primitive.

**Definition of Done:** All five CON requirements have an executable race/idempotency test that fails without the guard and passes with it.

## X-002 — Integration correlation, observability and operational recovery

**Objective:** Standardize correlation IDs, durable integration facts, safe error visibility and operational audit across INT-01..06 without moving business responsibility across bounded contexts.

**Architect source/reference:** P2 KROK 1; P2 KROK 3; P2 KROK 4; P1 KROK 13; P3 KROK 1, P4 KROK 1

**Requirement/rule:** `INT-01`, `INT-02`, `INT-03`, `INT-04`, `INT-05`, `INT-06`

**Dependencies:** `FND-002`, `P1-014`, `P2-005`, `P3-001`, `P4-001`

**Target components:** wms_orchestration, integration adapters, Mercato diagnostics

**DB impact:** Persist correlation/idempotency/attempt status but not secrets or raw unnecessary payloads.

**Backend impact:** Adapters expose bounded-context contracts and deterministic recovery state.

**Mercato impact:** Supervisor can inspect only architect-required operational status/errors.

**Scanner impact:** Scanner sees actionable errors for user flows, not integration internals.

**Integration impact:** INT-01..06 exact contract ownership; Inbound keeps GR retry.

**Migration/data impact:** Reuse orchestration foundation and preserve accepted Inbound event delivery.

**Tests:** contract suite INT-01..06; correlation tests; retry/error observability; integration regression

**Acceptance mapping:** No standalone architect TC; acceptance is through linked requirements/downstream journey.

**Evidence:** Test names + commit SHA + migration/schema evidence + persisted-state assertions; no secrets in evidence. Regression of accepted Inbound behavior for every touched shared Inventory/TU/warehouse/lock/orchestration primitive.

**Definition of Done:** Every integration message can be correlated to business object/effect and safely retried/diagnosed without duplicate business outcome.

## ACC-001 — Complete automated component/integration requirement suite

**Objective:** Provide executable automated coverage for every one of the 109 requirements before final user acceptance, including state, quantity, transaction and integration behavior.

**Architect source/reference:** P1 R1–P1 R2; P1 R3–P1 R4; P1 R5–P1 R6; P1 R7–P1 R8; P1 R9–P1 R10; P1 R11–P1 R12; P1 R13–P1 R14; P1 R15–P1 R16; P1 R17–P1 R18; P1 R19–P1 R20; P1 R21–P1 R22; P1 R23–P1 R24; P1 R25–P1 R26; P1 R27–P1 R28; P1 R29–P1 R30; P1 R31–P1 R32; P1 R33–P1 R34; P1 R35–P1 R36; P1 R37–P1 R38; P1 R39–P1 R40; P1 R41–P1 R42; P1 R43–P1 R44; P1 R45–P1 R46; P1 R47–P1 R48; P1 R49–P1 R50; P1 R51–P1 R52; P1 Funkcja ciągła F1; P1 R53; P1 R54, P1 R56; P1 R55; P1 R57; P1 R58; P1 R59; P1 R60; P1 R61; P1 R62; P1 R63; P1 R64; P1 R65; P1 R66; P1 R67; P1 R68; P1 R69; P1 R70; P1 R71; P1 R72; P2 R1–P2 R2; P2 R3–P2 R4; P2 R5–P2 R6; P2 R7–P2 R8; P2 R9–P2 R10; P2 R11–P2 R12; P2 R13–P2 R14; P2 R15–P2 R16; P2 R17, P2 R18, P2 R37; P2 R19–P2 R20; P2 R21–P2 R22; P2 R23–P2 R24; P2 R25–P2 R26; P2 R35–P2 R36; P2 R28; P2 R29–P2 R30; P2 R31–P2 R32; P2 R33; P2 R34; P2 R39; P2 R40; P2 R41; P2 R42; P2 R43; P3 R1–P3 R2; P3 R3–P3 R4; P3 R5–P3 R6; P3 R7–P3 R8; P3 R9; P3 R10; P4 R1–P4 R2; P4 R3–P4 R4; P4 R5–P4 R6; P4 R7–P4 R8; P4 R9; P1 Wyjątki — `SHORT_ALLOCATED`, `allowPartialShipment = false`; P1 Wyjątki — `SHORT_ALLOCATED`, `allowPartialShipment = true`; P1 Wyjątki — `SHORT_PICKED`; P1 Wyjątki — `SHORT_PICKED`, `allowPartialShipment = false`, wynik „czekamy”; P1 Wyjątki — `SHORT_PICKED`, `allowPartialShipment = false`, wynik „anulowanie”; P1 Wyjątki — trwała zmiana `allowPartialShipment`; P1 Wyjątki — brak lub uszkodzenie przy „repack by SKU”; P1 Wyjątki — `CustomerOrder.ON_HOLD`; P1 Wyjątki — żądanie anulowania `CustomerOrder` lub `CustomerOrderLine`; P1 Wyjątki — brak wyniku Carrier Selection; P1 Wyjątki — błąd etykiety lub odrzucenie przez przewoźnika; P2 Wyjątki — niedobór lub `DAMAGED`, `allowPartialShipment = false`; P2 Wyjątki — niedobór lub `DAMAGED`, `allowPartialShipment = true`; P2 Wyjątki — puste `TU`; P2 Wyjątki — ogólne anulowanie przy `CrossDockPickTask IN_PROGRESS`; P2 Wyjątki — `P2 R35`–`P2 R36`; P1 Wyjątki — P5 E17; P2 KROK 1; P2 KROK 3; P2 KROK 4; P1 KROK 13; P3 KROK 1, P4 KROK 1; P1 R4–P1 R6, P1 R52; P1 R10; P2 R6, P2 R29–P2 R30; P1 R37–P1 R41; P1 R43–P1 R46

**Requirement/rule:** `FR-P1-01`, `FR-P1-02`, `FR-P1-03`, `FR-P1-04`, `FR-P1-05`, `FR-P1-06`, `FR-P1-07`, `FR-P1-08`, `FR-P1-09`, `FR-P1-10`, `FR-P1-11`, `FR-P1-12`, `FR-P1-13`, `FR-P1-14`, `FR-P1-15`, `FR-P1-16`, `FR-P1-17`, `FR-P1-18`, `FR-P1-19`, `FR-P1-20`, `FR-P1-21`, `FR-P1-22`, `FR-P1-23`, `FR-P1-24`, `FR-P1-25`, `FR-P1-26`, `FR-P1-27`, `FR-P1-28`, `FR-P1-29`, `FR-P1-30`, `FR-P1-31`, `FR-P1-32`, `FR-P1-33`, `FR-P1-34`, `FR-P1-35`, `FR-P1-36`, `FR-P1-37`, `FR-P1-38`, `FR-P1-39`, `FR-P1-40`, `FR-P1-41`, `FR-P1-42`, `FR-P1-43`, `FR-P1-44`, `FR-P1-45`, `FR-P1-46`, `FR-P2-01`, `FR-P2-02`, `FR-P2-03`, `FR-P2-04`, `FR-P2-05`, `FR-P2-06`, `FR-P2-07`, `FR-P2-08`, `FR-P2-09`, `FR-P2-10`, `FR-P2-11`, `FR-P2-12`, `FR-P2-13`, `FR-P2-14`, `FR-P2-15`, `FR-P2-16`, `FR-P2-17`, `FR-P2-18`, `FR-P2-19`, `FR-P2-21`, `FR-P2-22`, `FR-P2-23`, `FR-P2-24`, `FR-P2-25`, `FR-P3-01`, `FR-P3-02`, `FR-P3-03`, `FR-P3-04`, `FR-P3-05`, `FR-P3-06`, `FR-P4-01`, `FR-P4-02`, `FR-P4-03`, `FR-P4-04`, `FR-P4-05`, `FR-P5-01`, `FR-P5-02`, `FR-P5-03`, `FR-P5-04`, `FR-P5-05`, `FR-P5-06`, `FR-P5-07`, `FR-P5-08`, `FR-P5-09`, `FR-P5-10`, `FR-P5-11`, `FR-P5-12`, `FR-P5-13`, `FR-P5-14`, `FR-P5-15`, `FR-P5-16`, `FR-P5-17`, `INT-01`, `INT-02`, `INT-03`, `INT-04`, `INT-05`, `INT-06`, `CON-01`, `CON-02`, `CON-03`, `CON-04`, `CON-05`

**Dependencies:** `FND-001`, `FND-002`, `FND-003`, `P1-001`, `P1-002`, `P1-003`, `P1-004`, `P1-005`, `P1-006`, `P1-007`, `P1-008`, `P1-009`, `P1-010`, `P1-011`, `P1-012`, `P1-013`, `P1-014`, `P1-015`, `P1-016`, `P2-001`, `P2-002`, `P2-003`, `P2-004`, `P2-005`, `P2-006`, `P3-001`, `P3-002`, `P3-003`, `P4-001`, `P4-002`, `P4-003`, `X-001`, `X-002`

**Target components:** Mercato tests, Scanner tests, DB integration tests, contract tests

**DB impact:** Test fixtures may use controlled test data only; verify migrations and constraints.

**Backend impact:** All domain/integration requirements have named executable tests.

**Mercato impact:** Mercato component tests for user-visible states/actions.

**Scanner impact:** Scanner component/integration tests for RF flows.

**Integration impact:** INT/CON automated contract/race coverage.

**Migration/data impact:** Migration/cutover and Inbound regression included.

**Tests:** 109/109 requirement-to-test automated coverage check; all affected repo test suites; migration verification; Inbound regression

**Acceptance mapping:** `TC-001`, `TC-002`, `TC-003`, `TC-004`, `TC-005`, `TC-006`, `TC-007`, `TC-008`, `TC-009`, `TC-010`, `TC-020`, `TC-021`, `TC-022`, `TC-023`, `TC-024`, `TC-025`, `TC-026`, `TC-027`, `TC-028`, `TC-029`, `TC-030`, `TC-031`, `TC-032`, `TC-033`, `TC-034`, `TC-035`, `TC-036`, `TC-037`, `TC-038`, `TC-039`, `TC-040`, `TC-041`, `TC-042`, `TC-043`, `TC-050`, `TC-051`, `TC-052`, `TC-053`, `TC-060`, `TC-061`, `TC-062`, `TC-063`, `TC-064`, `TC-065`, `TC-066`, `TC-067`, `TC-081`, `TC-092`, `TC-096`, `TC-097`, `TC-098`, `TC-099`, `TC-100`, `TC-101`, `TC-102`, `TC-103`, `TC-104`, `TC-105`, `TC-106`, `TC-107`, `TC-108`, `TC-109`, `TC-110`, `TC-111`, `TC-112`, `TC-113`, `TC-114`, `TC-115`, `TC-116`, `TC-117`, `TC-118`, `TC-119`, `TC-120`, `TC-121`, `TC-122`, `TC-123`, `TC-124`, `TC-125`, `TC-126`, `TC-127`, `TC-128`, `TC-129`, `TC-130`, `TC-131`, `TC-132`, `TC-133`, `TC-134`, `TC-135`

**Evidence:** Test names + commit SHA + migration/schema evidence + persisted-state assertions; no secrets in evidence.

**Definition of Done:** Machine-readable matrix has zero requirement orphans; all automated suites pass on the target commits.

## ACC-002 — Playwright Standard Fulfillment + P1 exception journeys

**Objective:** Prove through the normal Mercato/Scanner UI that Standard Fulfillment and P1 exception/recovery paths are usable by real roles.

**Architect source/reference:** P1 R1–P1 R2; P1 R3–P1 R4; P1 R5–P1 R6; P1 R7–P1 R8; P1 R9–P1 R10; P1 R11–P1 R12; P1 R13–P1 R14; P1 R15–P1 R16; P1 R17–P1 R18; P1 R19–P1 R20; P1 R21–P1 R22; P1 R23–P1 R24; P1 R25–P1 R26; P1 R27–P1 R28; P1 R29–P1 R30; P1 R31–P1 R32; P1 R33–P1 R34; P1 R35–P1 R36; P1 R37–P1 R38; P1 R39–P1 R40; P1 R41–P1 R42; P1 R43–P1 R44; P1 R45–P1 R46; P1 R47–P1 R48; P1 R49–P1 R50; P1 R51–P1 R52; P1 Funkcja ciągła F1; P1 R53; P1 R54, P1 R56; P1 R55; P1 R57; P1 R58; P1 R59; P1 R60; P1 R61; P1 R62; P1 R63; P1 R64; P1 R65; P1 R66; P1 R67; P1 R68; P1 R69; P1 R70; P1 R71; P1 R72; P1 Wyjątki — `SHORT_ALLOCATED`, `allowPartialShipment = false`; P1 Wyjątki — `SHORT_ALLOCATED`, `allowPartialShipment = true`; P1 Wyjątki — `SHORT_PICKED`; P1 Wyjątki — `SHORT_PICKED`, `allowPartialShipment = false`, wynik „czekamy”; P1 Wyjątki — `SHORT_PICKED`, `allowPartialShipment = false`, wynik „anulowanie”; P1 Wyjątki — trwała zmiana `allowPartialShipment`; P1 Wyjątki — brak lub uszkodzenie przy „repack by SKU”; P1 Wyjątki — `CustomerOrder.ON_HOLD`; P1 Wyjątki — żądanie anulowania `CustomerOrder` lub `CustomerOrderLine`; P1 Wyjątki — brak wyniku Carrier Selection; P1 Wyjątki — błąd etykiety lub odrzucenie przez przewoźnika; P1 Wyjątki — P5 E17; P1 KROK 13; P3 KROK 1, P4 KROK 1; P1 R4–P1 R6, P1 R52; P1 R10; P1 R37–P1 R41; P1 R43–P1 R46

**Requirement/rule:** `FR-P1-01`, `FR-P1-02`, `FR-P1-03`, `FR-P1-04`, `FR-P1-05`, `FR-P1-06`, `FR-P1-07`, `FR-P1-08`, `FR-P1-09`, `FR-P1-10`, `FR-P1-11`, `FR-P1-12`, `FR-P1-13`, `FR-P1-14`, `FR-P1-15`, `FR-P1-16`, `FR-P1-17`, `FR-P1-18`, `FR-P1-19`, `FR-P1-20`, `FR-P1-21`, `FR-P1-22`, `FR-P1-23`, `FR-P1-24`, `FR-P1-25`, `FR-P1-26`, `FR-P1-27`, `FR-P1-28`, `FR-P1-29`, `FR-P1-30`, `FR-P1-31`, `FR-P1-32`, `FR-P1-33`, `FR-P1-34`, `FR-P1-35`, `FR-P1-36`, `FR-P1-37`, `FR-P1-38`, `FR-P1-39`, `FR-P1-40`, `FR-P1-41`, `FR-P1-42`, `FR-P1-43`, `FR-P1-44`, `FR-P1-45`, `FR-P1-46`, `FR-P5-01`, `FR-P5-02`, `FR-P5-03`, `FR-P5-04`, `FR-P5-05`, `FR-P5-06`, `FR-P5-07`, `FR-P5-08`, `FR-P5-09`, `FR-P5-10`, `FR-P5-11`, `FR-P5-17`, `INT-04`, `INT-05`, `INT-06`, `CON-01`, `CON-02`, `CON-04`, `CON-05`

**Dependencies:** `ACC-001`

**Target components:** Playwright Mercato, Playwright Scanner, test fixtures/evidence

**DB impact:** DB/API only for deterministic setup or persisted assertions; not to replace accepted UI actions.

**Backend impact:** Expose deterministic fixture/evidence hooks only when they do not bypass business behavior.

**Mercato impact:** Click full order/planning/packing/shipment/carrier/ERP/manifest and Supervisor exception surfaces.

**Scanner impact:** Scan/click picking, TU, short-pick and relevant recovery surfaces.

**Integration impact:** Exercise posting retry and cancellation boundary through UI-visible state.

**Migration/data impact:** No production data migration; isolated test fixtures.

**Tests:** Playwright journeys 1–14 from Evidence Standard plus partial/multi-shipment exceptions; cross-browser smoke where supported

**Acceptance mapping:** `TC-001`, `TC-002`, `TC-003`, `TC-004`, `TC-005`, `TC-006`, `TC-007`, `TC-008`, `TC-009`, `TC-010`, `TC-060`, `TC-061`, `TC-062`, `TC-063`, `TC-064`, `TC-065`, `TC-066`, `TC-096`, `TC-097`, `TC-098`, `TC-104`, `TC-105`, `TC-106`, `TC-107`, `TC-108`, `TC-109`, `TC-110`, `TC-111`, `TC-114`, `TC-115`, `TC-116`, `TC-117`, `TC-118`, `TC-120`, `TC-121`, `TC-122`, `TC-123`, `TC-124`, `TC-125`, `TC-126`, `TC-127`, `TC-128`, `TC-129`, `TC-130`, `TC-131`, `TC-132`, `TC-133`, `TC-135`

**Evidence:** Playwright trace/screenshots/video where useful, visible identifiers, persisted server/DB assertion, and Human Verified walkthrough record for the affected user journey.

**Definition of Done:** Tests traverse normal application surfaces, record visible+persistent outcomes and leave a reproducible next human action for every journey.

## ACC-003 — Playwright Crossdock, Reservation Release and Physical Putback journeys

**Objective:** Prove P2/P3/P4 through normal Scanner/Mercato UI, including n:n sorting, GR gate, cancellation race and invalid-location loop.

**Architect source/reference:** P2 R1–P2 R2; P2 R3–P2 R4; P2 R5–P2 R6; P2 R7–P2 R8; P2 R9–P2 R10; P2 R11–P2 R12; P2 R13–P2 R14; P2 R15–P2 R16; P2 R17, P2 R18, P2 R37; P2 R19–P2 R20; P2 R21–P2 R22; P2 R23–P2 R24; P2 R25–P2 R26; P2 R35–P2 R36; P2 R28; P2 R29–P2 R30; P2 R31–P2 R32; P2 R33; P2 R34; P2 R39; P2 R40; P2 R41; P2 R42; P2 R43; P2 Wyjątki — niedobór lub `DAMAGED`, `allowPartialShipment = false`; P2 Wyjątki — niedobór lub `DAMAGED`, `allowPartialShipment = true`; P2 Wyjątki — puste `TU`; P2 Wyjątki — ogólne anulowanie przy `CrossDockPickTask IN_PROGRESS`; P2 Wyjątki — `P2 R35`–`P2 R36`; P3 R1–P3 R2; P3 R3–P3 R4; P3 R5–P3 R6; P3 R7–P3 R8; P3 R9; P3 R10; P4 R1–P4 R2; P4 R3–P4 R4; P4 R5–P4 R6; P4 R7–P4 R8; P4 R9; P2 KROK 1; P2 KROK 3; P2 KROK 4; P3 KROK 1, P4 KROK 1; P2 R6, P2 R29–P2 R30; P1 R43–P1 R46

**Requirement/rule:** `FR-P2-01`, `FR-P2-02`, `FR-P2-03`, `FR-P2-04`, `FR-P2-05`, `FR-P2-06`, `FR-P2-07`, `FR-P2-08`, `FR-P2-09`, `FR-P2-10`, `FR-P2-11`, `FR-P2-12`, `FR-P2-13`, `FR-P2-14`, `FR-P2-15`, `FR-P2-16`, `FR-P2-17`, `FR-P2-18`, `FR-P2-19`, `FR-P2-21`, `FR-P2-22`, `FR-P2-23`, `FR-P2-24`, `FR-P2-25`, `FR-P5-12`, `FR-P5-13`, `FR-P5-14`, `FR-P5-15`, `FR-P5-16`, `FR-P3-01`, `FR-P3-02`, `FR-P3-03`, `FR-P3-04`, `FR-P3-05`, `FR-P3-06`, `FR-P4-01`, `FR-P4-02`, `FR-P4-03`, `FR-P4-04`, `FR-P4-05`, `INT-01`, `INT-02`, `INT-03`, `INT-06`, `CON-03`, `CON-05`

**Dependencies:** `ACC-001`, `P2-006`, `P3-003`, `P4-003`

**Target components:** Playwright Scanner, Playwright Mercato, integration test fixtures

**DB impact:** Controlled Inbound/source TU and stock fixtures; persisted quantity assertions.

**Backend impact:** Fixture endpoints if required remain test-only and cannot replace acceptance action.

**Mercato impact:** Supervisor crossdock GR/cancellation visibility.

**Scanner impact:** Scan/click crossdock task, shortage/damage/empty, release feedback and PutBack loops.

**Integration impact:** Exercise INT-01..03/06 boundary with deterministic stubs/test endpoints while UI remains real.

**Migration/data impact:** No destructive Inbound data changes; regression environment isolated.

**Tests:** Playwright journeys 15–20 from Evidence Standard; crossdock 1:1 and n:n; GR pending/rejected/accepted re-evaluation; P3/P4 cancellation recovery

**Acceptance mapping:** `TC-020`, `TC-021`, `TC-022`, `TC-023`, `TC-024`, `TC-025`, `TC-026`, `TC-027`, `TC-028`, `TC-029`, `TC-030`, `TC-031`, `TC-032`, `TC-033`, `TC-034`, `TC-035`, `TC-036`, `TC-037`, `TC-038`, `TC-039`, `TC-040`, `TC-041`, `TC-042`, `TC-043`, `TC-050`, `TC-051`, `TC-052`, `TC-053`, `TC-067`, `TC-081`, `TC-092`, `TC-099`, `TC-100`, `TC-101`, `TC-102`, `TC-103`, `TC-112`, `TC-113`, `TC-119`, `TC-121`, `TC-134`

**Evidence:** Playwright trace/screenshots/video where useful, visible identifiers, persisted server/DB assertion, and Human Verified walkthrough record for the affected user journey.

**Definition of Done:** Each architect journey is traversable end-to-end by intended role; quantities and states reconcile after the UI flow.

## ACC-004 — Final Human Verified Outbound v1 walkthrough and acceptance record

**Objective:** Run the final human walkthrough over the same normal UI journeys after automated evidence is green, and record acceptance against exact application commits/environment.

**Architect source/reference:** P1 R1–P1 R2; P1 R3–P1 R4; P1 R5–P1 R6; P1 R7–P1 R8; P1 R9–P1 R10; P1 R11–P1 R12; P1 R13–P1 R14; P1 R15–P1 R16; P1 R17–P1 R18; P1 R19–P1 R20; P1 R21–P1 R22; P1 R23–P1 R24; P1 R25–P1 R26; P1 R27–P1 R28; P1 R29–P1 R30; P1 R31–P1 R32; P1 R33–P1 R34; P1 R35–P1 R36; P1 R37–P1 R38; P1 R39–P1 R40; P1 R41–P1 R42; P1 R43–P1 R44; P1 R45–P1 R46; P1 R47–P1 R48; P1 R49–P1 R50; P1 R51–P1 R52; P1 Funkcja ciągła F1; P1 R53; P1 R54, P1 R56; P1 R55; P1 R57; P1 R58; P1 R59; P1 R60; P1 R61; P1 R62; P1 R63; P1 R64; P1 R65; P1 R66; P1 R67; P1 R68; P1 R69; P1 R70; P1 R71; P1 R72; P2 R1–P2 R2; P2 R3–P2 R4; P2 R5–P2 R6; P2 R7–P2 R8; P2 R9–P2 R10; P2 R11–P2 R12; P2 R13–P2 R14; P2 R15–P2 R16; P2 R17, P2 R18, P2 R37; P2 R19–P2 R20; P2 R21–P2 R22; P2 R23–P2 R24; P2 R25–P2 R26; P2 R35–P2 R36; P2 R28; P2 R29–P2 R30; P2 R31–P2 R32; P2 R33; P2 R34; P2 R39; P2 R40; P2 R41; P2 R42; P2 R43; P3 R1–P3 R2; P3 R3–P3 R4; P3 R5–P3 R6; P3 R7–P3 R8; P3 R9; P3 R10; P4 R1–P4 R2; P4 R3–P4 R4; P4 R5–P4 R6; P4 R7–P4 R8; P4 R9; P1 Wyjątki — `SHORT_ALLOCATED`, `allowPartialShipment = false`; P1 Wyjątki — `SHORT_ALLOCATED`, `allowPartialShipment = true`; P1 Wyjątki — `SHORT_PICKED`; P1 Wyjątki — `SHORT_PICKED`, `allowPartialShipment = false`, wynik „czekamy”; P1 Wyjątki — `SHORT_PICKED`, `allowPartialShipment = false`, wynik „anulowanie”; P1 Wyjątki — trwała zmiana `allowPartialShipment`; P1 Wyjątki — brak lub uszkodzenie przy „repack by SKU”; P1 Wyjątki — `CustomerOrder.ON_HOLD`; P1 Wyjątki — żądanie anulowania `CustomerOrder` lub `CustomerOrderLine`; P1 Wyjątki — brak wyniku Carrier Selection; P1 Wyjątki — błąd etykiety lub odrzucenie przez przewoźnika; P2 Wyjątki — niedobór lub `DAMAGED`, `allowPartialShipment = false`; P2 Wyjątki — niedobór lub `DAMAGED`, `allowPartialShipment = true`; P2 Wyjątki — puste `TU`; P2 Wyjątki — ogólne anulowanie przy `CrossDockPickTask IN_PROGRESS`; P2 Wyjątki — `P2 R35`–`P2 R36`; P1 Wyjątki — P5 E17; P2 KROK 1; P2 KROK 3; P2 KROK 4; P1 KROK 13; P3 KROK 1, P4 KROK 1; P1 R4–P1 R6, P1 R52; P1 R10; P2 R6, P2 R29–P2 R30; P1 R37–P1 R41; P1 R43–P1 R46

**Requirement/rule:** `FR-P1-01`, `FR-P1-02`, `FR-P1-03`, `FR-P1-04`, `FR-P1-05`, `FR-P1-06`, `FR-P1-07`, `FR-P1-08`, `FR-P1-09`, `FR-P1-10`, `FR-P1-11`, `FR-P1-12`, `FR-P1-13`, `FR-P1-14`, `FR-P1-15`, `FR-P1-16`, `FR-P1-17`, `FR-P1-18`, `FR-P1-19`, `FR-P1-20`, `FR-P1-21`, `FR-P1-22`, `FR-P1-23`, `FR-P1-24`, `FR-P1-25`, `FR-P1-26`, `FR-P1-27`, `FR-P1-28`, `FR-P1-29`, `FR-P1-30`, `FR-P1-31`, `FR-P1-32`, `FR-P1-33`, `FR-P1-34`, `FR-P1-35`, `FR-P1-36`, `FR-P1-37`, `FR-P1-38`, `FR-P1-39`, `FR-P1-40`, `FR-P1-41`, `FR-P1-42`, `FR-P1-43`, `FR-P1-44`, `FR-P1-45`, `FR-P1-46`, `FR-P2-01`, `FR-P2-02`, `FR-P2-03`, `FR-P2-04`, `FR-P2-05`, `FR-P2-06`, `FR-P2-07`, `FR-P2-08`, `FR-P2-09`, `FR-P2-10`, `FR-P2-11`, `FR-P2-12`, `FR-P2-13`, `FR-P2-14`, `FR-P2-15`, `FR-P2-16`, `FR-P2-17`, `FR-P2-18`, `FR-P2-19`, `FR-P2-21`, `FR-P2-22`, `FR-P2-23`, `FR-P2-24`, `FR-P2-25`, `FR-P3-01`, `FR-P3-02`, `FR-P3-03`, `FR-P3-04`, `FR-P3-05`, `FR-P3-06`, `FR-P4-01`, `FR-P4-02`, `FR-P4-03`, `FR-P4-04`, `FR-P4-05`, `FR-P5-01`, `FR-P5-02`, `FR-P5-03`, `FR-P5-04`, `FR-P5-05`, `FR-P5-06`, `FR-P5-07`, `FR-P5-08`, `FR-P5-09`, `FR-P5-10`, `FR-P5-11`, `FR-P5-12`, `FR-P5-13`, `FR-P5-14`, `FR-P5-15`, `FR-P5-16`, `FR-P5-17`, `INT-01`, `INT-02`, `INT-03`, `INT-04`, `INT-05`, `INT-06`, `CON-01`, `CON-02`, `CON-03`, `CON-04`, `CON-05`

**Dependencies:** `ACC-002`, `ACC-003`

**Target components:** Mercato, Scanner, evidence records

**DB impact:** Read-only verification of persisted results during acceptance; no manual DB repair to make tests pass.

**Backend impact:** Freeze/record commit set and evidence references.

**Mercato impact:** Human traverses all relevant Mercato steps as real roles.

**Scanner impact:** Human traverses all relevant Scanner flows by click/type/scan with real warehouse context.

**Integration impact:** Observe integration/retry states but do not fabricate external success outside test harness contract.

**Migration/data impact:** Acceptance uses test data and non-destructive environment; no cutover unless separately authorized in implementation phase.

**Tests:** Human walkthrough checklist backed by successful Playwright/component/integration evidence

**Acceptance mapping:** `TC-001`, `TC-002`, `TC-003`, `TC-004`, `TC-005`, `TC-006`, `TC-007`, `TC-008`, `TC-009`, `TC-010`, `TC-020`, `TC-021`, `TC-022`, `TC-023`, `TC-024`, `TC-025`, `TC-026`, `TC-027`, `TC-028`, `TC-029`, `TC-030`, `TC-031`, `TC-032`, `TC-033`, `TC-034`, `TC-035`, `TC-036`, `TC-037`, `TC-038`, `TC-039`, `TC-040`, `TC-041`, `TC-042`, `TC-043`, `TC-050`, `TC-051`, `TC-052`, `TC-053`, `TC-060`, `TC-061`, `TC-062`, `TC-063`, `TC-064`, `TC-065`, `TC-066`, `TC-067`, `TC-081`, `TC-092`, `TC-096`, `TC-097`, `TC-098`, `TC-099`, `TC-100`, `TC-101`, `TC-102`, `TC-103`, `TC-104`, `TC-105`, `TC-106`, `TC-107`, `TC-108`, `TC-109`, `TC-110`, `TC-111`, `TC-112`, `TC-113`, `TC-114`, `TC-115`, `TC-116`, `TC-117`, `TC-118`, `TC-119`, `TC-120`, `TC-121`, `TC-122`, `TC-123`, `TC-124`, `TC-125`, `TC-126`, `TC-127`, `TC-128`, `TC-129`, `TC-130`, `TC-131`, `TC-132`, `TC-133`, `TC-134`, `TC-135`

**Evidence:** Record date, environment, Mercato SHA, Scanner SHA, migration version, actor, scenario path, result and defects; retain Playwright artifacts as supporting proof.

**Definition of Done:** Human Verified status is PASS for required user-facing journeys or defects are logged and task remains not done; no document-only acceptance.

