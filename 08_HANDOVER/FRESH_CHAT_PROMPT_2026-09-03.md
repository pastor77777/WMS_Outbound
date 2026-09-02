# Fresh Chat Prompt — WMS Outbound

Use this in a new ChatGPT supervisor chat:

```text
Continue WMS Outbound from the current durable handover. Do not reconstruct state from old chat history.

First read/refresh current connected sources:
- WMS_Outbound/08_HANDOVER/HANDOVER_CURRENT_2026-09-03.md
- WMS_Outbound/STATE.md
- WMS_Outbound/06_AGENT_GUIDES/GIT_PROMPT_WORKFLOW.md
- Devaxonic-WMS/.ai/HANDOVER_OUTBOUND_CURRENT_2026-09-03.md
- Devaxonic-WMS/.ai/STATE.md, .ai/PLAN.md, .ai/TESTING.md, .ai/OPERATIONS.md

Load current installed wms-outbound, architecture-context, fetch_me_prompt and operational-mode; use architecture-context only for accepted shared/Inbound compatibility.

Independently verify current Git refs. Current checkpoint should be 11/37 FINAL PASS with P1-007 accepted; next item is P1-009 Direct Pack.

Ground P1-009 from the current Architect/requirements/acceptance sources, then write and push the full first Antigravity execution guide to WMS_Outbound/06_AGENT_GUIDES/. After that give me only the short SAME-session Antigravity launch prompt.

Follow the persisted Git workflow: I should only need to reply done; never ask me to paste Antigravity logs.
```
