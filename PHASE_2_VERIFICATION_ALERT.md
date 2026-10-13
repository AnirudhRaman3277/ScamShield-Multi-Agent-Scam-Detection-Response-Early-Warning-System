# Phase 2 — Verification and Scam Alert Agents

## Goal

Build the two reliability and communication agents:

1. Verification Agent
2. Scam Alert Agent

Phase 2 consumes the shared message/state contract from Phase 1. It must not publish unverified accusations and must not change the shared JSON field names without team agreement.

## Shared Contract

Use the common `case_id`, `state_version`, `message_type`, `evidence_refs`, `confidence`, `rationale`, `requested_action`, `recommended_action`, `status`, and `timestamp` fields. Every material claim must reference one or more evidence IDs.

## Agent 3: Verification Agent

### Responsibility

Check whether the user's analysed report matches a known scam pattern. It invokes Scam Search to find public warnings, regulator notices, and corroborating reports, then returns a bounded match result and practical reporting guidance. This is the primary reliability gate.

### Inputs

- Intake Analysis summary, the user's supplied evidence, and source provenance.
- Scam Search results for known patterns and corroborating sources.
- Existing conflicts and missing information.
- Research/revision limits.

### Required reasoning

Verification is not a simple `if confidence > X` rule. It must distinguish a known-scam match, insufficient information, and a genuinely emerging pattern. A failed match is **not** proof of a new scam. The agent must reason about:

- What each source directly establishes.
- Source credibility, independence, and recency.
- Whether multiple sources merely repeat the same claim.
- Material conflicts and whether they are resolved.
- Whether proposed alert wording is narrower than the evidence.
- Whether further targeted research is justified.

### Outputs

| Condition | Message/action |
|---|---|
| Known scam matches the report | `KNOWN_SCAM_CONFIRMED` → response plan for the user |
| Specific evidence gap | `RESEARCH_REQUEST` → Search |
| Intake is incomplete | `REQUEST_MORE_USER_EVIDENCE` → user/intake |
| No known match but corroborated pattern | `EMERGING_PATTERN_CANDIDATE` → Alert review |
| One unverified/noisy report or unresolved conflict | `HELD_FOR_REVIEW` → no public alert |
| Evidence does not support a case | `REJECTION` → close without publication |
| Limits reached or sensitive issue | `ESCALATE` → hold for review |

### Example conflict report

```json
{
  "case_id": "SCAM-001",
  "sender": "verification_agent",
  "recipient": "scam_search_agent",
  "message_type": "CONFLICT_REPORT",
  "evidence_refs": ["E-01", "E-04"],
  "confidence": 0.55,
  "rationale": "Sources disagree on a material ownership claim.",
  "conflicts": [{"claim": "domain ownership", "material": true}],
  "requested_action": "SEARCH_ADDITIONAL_SOURCES",
  "recommended_action": "REQUEST_CLARIFICATION",
  "status": "IN_PROGRESS",
  "timestamp": "2026-10-09T10:10:00Z"
}
```

### Required approval fields

A known-match result must name the matching pattern, say why the evidence matches it, identify uncertainty, and give safe next steps. For an India-facing implementation, the response plan should tell victims of immediate cyber-financial fraud to call **1930** and report at `cybercrime.gov.in`, while preserving evidence and contacting their bank/payment provider promptly. An emerging-pattern result is not publication approval: it must include corroboration, uncertainty, and a human/review threshold. The agent cannot call Telegram.

## Agent 4: Scam Alert Agent

### Responsibility

Create plain-language public safety alerts only for an emerging/suspected pattern that Verification has marked alert-eligible. Include what is observed, uncertainty, warning signs, precautions, and evidence references. Do not call an unverified allegation a confirmed scam and do not name private people or expose personal data.

### Inputs

- Verification's `EMERGING_PATTERN_CANDIDATE` decision, corroboration, and rationale.
- Approved claims and evidence references.
- Alert template and state snapshot.

### Required behaviour

- Confirm `verification_status` is `APPROVED`.
- Include citations/evidence references for material claims.
- Use clear, proportional wording; do not overstate confidence.
- Exclude personal/sensitive data and unsupported allegations.
- Request revision if required support, status, or details are absent.
- Return a draft for validation, then `ALERT_APPROVED` only when publication-ready.

### Example draft

```json
{
  "case_id": "SCAM-001",
  "sender": "scam_alert_agent",
  "recipient": "n8n_orchestrator",
  "message_type": "ALERT_DRAFT",
  "evidence_refs": ["E-02", "E-03"],
  "confidence": 0.86,
  "rationale": "All public claims map to verification-approved evidence.",
  "recommended_action": "APPROVE_ALERT",
  "final_alert": {
    "title": "Warning: suspected investment-site impersonation",
    "warning_signs": ["guaranteed-return claims"],
    "precautions": ["Independently verify registration before paying."]
  },
  "status": "READY_FOR_PUBLICATION",
  "timestamp": "2026-10-09T10:15:00Z"
}
```

## State Fields Owned or Updated

| Field | Primary updater |
|---|---|
| `verification_status` | Verification |
| `verification_rationale` | Verification |
| `conflicts` | Verification |
| `confidence.verification` | Verification |
| `final_alert` | Alert |
| `confidence.alert` | Alert |
| `agent_history` | n8n appends validated messages |

## n8n Integration Requirements

Supply schema-valid agent subworkflows. The Verification route must be able to return Search/Analysis for feedback, not only forward to Alert. The Alert route must never invoke Telegram itself. Phase 3 adds a hard guard requiring Verification `APPROVAL`, `publication_eligible: true`, and Alert `ALERT_APPROVED` before the Telegram node is called.

## Tests and Deliverables

Deliver:

- Prompts and agent configurations.
- Schema-compatible structured outputs.
- Tests for insufficient evidence, duplicated sources, conflicting sources, reasoned rejection, escalation, unsupported alert claims, missing verification approval, unsafe wording, and malformed JSON.
- Safe simulated fixtures and documented n8n subworkflows.
- README explaining expected input and output contracts.

## Definition of Done

Phase 2 is complete when Verification can identify known scams and return reporting guidance, request research, hold weak reports, and route only corroborated emerging patterns to Alert. Alert must draft only uncertainty-bounded warnings and request revisions. No direct Telegram publishing is permitted from either agent.
