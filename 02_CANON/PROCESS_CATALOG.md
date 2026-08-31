# Process Catalog

| ID | Process | Active version | Purpose | Primary user/system surface |
|---|---|---:|---|---|
| P1 | STANDARD_FULFILLMENT | 1.20 | Standard demand → allocation → picking → packing → shipment → ERP → manifest → settlement | Mercato + Scanner + system automation |
| P2 | OUTBOUND_CROSSDOCK | 1.13 | Bind eligible Inbound ELEMENTARY TU quantity to Outbound demand without Allocation, sort via RF, then join P1 downstream | Scanner + Mercato + Inbound boundary |
| P3 | RESERVATION_RELEASE | 1.2 | Logical release/cancellation before formal picked quantity or by reservation-retention policy | Mercato + Scanner feedback + system automation |
| P4 | PHYSICAL_PUTBACK | 1.2 | Physical recovery of already picked/packed quantity after approved cancellation | Scanner + Mercato |

There is no standalone Process 5. Cross-cutting exceptions live in P1/P2 and are indexed as `FR-P5-*`.

Source behavior remains in the immutable P1–P4 process files; this catalog is navigation only.
