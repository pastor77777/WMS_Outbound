# Scanner routing

For an Outbound Scanner implementation question, use this order:

1. **Outbound Architect Source/Golden Record** — decides business flow, role, state, guards, expected behavior and acceptance.
2. **`scanner-context`** — use only for generic scanner/RF/AIDC architecture, standards/vendor facts, scan parsing, device integration, retry/idempotency/offline UX and testing guidance.
3. **`Devaxonic-scanner` current source** — proves what the app currently implements and what primitives can be reused.

If generic scanner guidance conflicts with Outbound architect behavior, Outbound wins.

Current Scanner Outbound mode is not target behavior; it is a receiving-style UI-only legacy pass and must not be used as source truth.

Human-facing Scanner work must be proved through normal clickable/scannable UI and final Human Verified walkthrough.
