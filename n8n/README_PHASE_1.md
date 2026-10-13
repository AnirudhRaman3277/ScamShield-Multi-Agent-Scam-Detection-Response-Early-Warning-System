# Phase 1 n8n Implementation

This directory contains the actual Phase 1 agent implementations as n8n workflow exports:

- `workflows/phase1-scam-search-agent.json` — Search Agent
- `workflows/phase1-scam-analysis-agent.json` — Analysis Agent

There is no Python, JavaScript service, or external agent runtime. The reasoning, search/API calls, structured-output checks, and agent output construction occur inside n8n nodes. The only external services are the configured search provider and LLM API.

## 1. Configure n8n

1. Copy the repository `.env.example` to `.env`.
2. Keep the free defaults: `OPENAI_API_KEY=ollama`, `OPENAI_MODEL=llama3.2:3b`, and `OPENAI_API_URL=http://ollama:11434/v1/chat/completions`. The Compose stack starts Ollama and downloads the selected local model.
3. Set a stable random `N8N_ENCRYPTION_KEY` value.
4. Start the local runtime with `docker compose up -d`, then open `http://localhost:5678`.
5. Ensure the Docker n8n service loads `.env` with `env_file: .env`.
6. Restart n8n after changing environment values.
7. In the n8n UI, import both workflow JSON files from **Workflows → Import from File**.

Do not put API keys in workflow nodes or Git.

If `docker ps` reports that it cannot connect to `dockerDesktopLinuxEngine`, open Docker Desktop normally and wait until it says the engine is running. The workflow cannot be executed until that command succeeds.

For a public demo URL, install ngrok, create a free account, and put the assigned HTTPS dev domain into both `WEBHOOK_URL` and `N8N_EDITOR_BASE_URL` in `.env` before restarting the stack. Keep the ngrok process running for the duration of the demo.

## 2. How the agents work in n8n

### Search Agent workflow

`When Executed by Another Workflow` → validate input → public Google News RSS search → build LLM prompt → Ollama-compatible chat-completions request → validate JSON and build `case_patch`.

The LLM sees only canonical candidate evidence IDs derived from the RSS response. Its evidence selections are checked against those IDs, so it cannot add an invented source to the saved state. The Verification Agent invokes this workflow in Phase 2 to research a possible **known** scam pattern.

### Analysis Agent workflow

`When Executed by Another Workflow` → validate the user's narrative/evidence → build LLM prompt → Ollama-compatible chat-completions request → validate JSON, claim-to-evidence references, and next action → build `case_patch`.

The Analysis Agent can return `REQUEST_MORE_USER_EVIDENCE`, `SEND_TO_VERIFICATION`, or `ESCALATE`. It does not browse or decide whether the case is a known scam. Phase 3 must route these actions; this workflow only makes the decision and returns it.

## 3. Invocation Input

Phase 3 invokes each workflow through n8n’s **Execute Workflow** node. Pass one item shaped like this:

```json
{
  "case_state": {
    "case_id": "SCAM-001",
    "state_version": 0,
    "suspected_scam": "Suspicious investment website reported online",
    "sources": [],
    "evidence": [],
    "analysis": null,
    "indicators": [],
    "missing_information": [],
    "research_count": 0,
    "revision_count": 0,
    "agent_history": []
  },
  "incoming_message": {
    "objective": "Find independent public evidence about the reported website."
  }
}
```

The Search Agent requires a `case_id` and a research objective from Verification. The Analysis Agent requires the same state plus a `user_submission.narrative`; evidence is optional and may be supplied by the user.

## 4. Returned Output

Both workflows return:

```json
{
  "case_id": "SCAM-001",
  "message": {"sender": "...", "recommended_action": "..."},
  "case_patch": {"...": "fields to merge into shared state"}
}
```

Phase 3 must validate the message against `schemas/agent-message.schema.json`, merge the patch with optimistic state-version handling, append the message to `agent_history`, and then route using `message.recommended_action`.

## 5. Required Phase 3 Router Rules

| Search/Analysis output | Next n8n action |
|---|---|
| `REQUEST_MORE_USER_EVIDENCE` | Persist patch and request the named missing information from the user |
| `SEND_TO_VERIFICATION` | Persist patch, hand case to Phase 2 Verification workflow |
| `SEND_TO_VERIFICATION` from Search | Persist patch and return the research bundle to Verification |
| `REQUEST_REVISION` | Persist patch and invoke named recipient |
| `ESCALATE` | Persist patch, hold case; do not publish |

## 6. Test safely

Use a controlled test query, a Telegram-free test case, and fixture URLs where practical. Confirm that:

1. Search returns only selected canonical evidence IDs.
2. Analysis maps claims only to supplied/received evidence IDs and treats the report as unverified.
3. One weak source leads to a research request.
4. Search failures and malformed LLM responses stop safely and appear in n8n execution logs.
5. Retrieved prompt-injection text is treated as evidence, not a command.

## Important compatibility note

The workflows use standard n8n **Code**, **HTTP Request**, and **Execute Workflow Trigger** nodes. Import them into a current n8n version. If n8n requests a node-version upgrade during import, accept the upgrade and re-check the HTTP request headers, JSON body expressions, and Code-node output with a harmless test case.
