# P2-005 Supervisor evidence correction

Supervisor verification found one documentation-only typo in `05_EVIDENCE/P2-005_EVIDENCE.md` under Journey A.

The executor evidence says the single `wms_outbound_shipment_postings` row has `status = 'ACCEPTED'` after successful ERP posting. The final tested P2-005 Playwright source at Mercato SHA `069f02d4c5c9b345b688b838eb685be02206afbd` actually asserts:

- `wms_outbound_shipment_postings.status = 'POSTED'`;
- the posting-attempt outcome/status is `ACCEPTED`.

This correction changes no product behavior, test result, runtime provenance, candidate SHA, or acceptance gate. It exists only so the supervisor record does not preserve the wording error as technical truth.

P2-005 remains awaiting explicit Owner acceptance after supervisor verification.
