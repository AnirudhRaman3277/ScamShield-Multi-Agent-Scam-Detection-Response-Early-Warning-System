# Phase 3 Main Orchestrator

Import `workflows/phase3-main-controller.json` only after importing all four agent workflows. Configure its five **Execute Sub-workflow** nodes after import:

| Controller node | Workflow to select |
| --- | --- |
| Invoke Analysis Agent | Phase 1 - Scam Analysis Agent |
| Invoke Verification Agent | Phase 2 - Verification Agent |
| Invoke Re-verification Agent | Phase 2 - Verification Agent |
| Invoke Search Agent | Phase 1 - Scam Search Agent |
| Invoke Alert Agent | Phase 2 - Scam Alert Agent |

The controller processes a safe demo report in this order: Analysis → Verification → one bounded Search feedback round when requested → Re-verification → Alert only when requested. It carries a versioned case snapshot and agent-history event records through the execution.

Telegram is called only when the deterministic **Publication Guard** verifies all of these:

- `verification_status` is `APPROVED`
- `verification.publication_eligible` is `true`
- Alert returns message type `ALERT_APPROVED`
- `final_alert` contains evidence references

For a Telegram test channel, add `TELEGRAM_BOT_TOKEN` and `TELEGRAM_CHANNEL_ID` to the untracked `.env`, restart n8n, and make the bot a channel administrator. Do not run with real public claims until the agent workflows and guard have been tested using safe fixtures.
# Frontend test form

The repository includes a local test form in `frontend/`. It submits a report
to the Phase 3 controller webhook, which starts the workflow automatically.

1. Import the current `workflows/phase3-main-controller.json` into n8n.
2. Re-select the five agent workflows in the Execute Workflow nodes, because
   n8n does not preserve those selections in portable workflow JSON.
3. Save and activate the Phase 3 controller. Activation exposes
   `POST /webhook/scam-report-test`.
4. Start the frontend with `docker compose up -d frontend` and open
   `http://localhost:8080`.

The form responds as soon as n8n accepts the report. Follow the full agent
execution in n8n's Executions view. For public access, tunnel port `8080`
through ngrok rather than port `5678`; this keeps the n8n editor private.

