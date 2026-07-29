# Airtable schema guide

Use fictional data while configuring the package. Exact field types and state
semantics are documented in [the data contract](../docs/data-contract.md).

| Table | Purpose | Key fields |
|---|---|---|
| leads | Canonical lead and CRM state | idempotency_key, email, intent, score, route, crm_sync_status, follow_up_at |
| scoring_config | Business-editable score rules | signal, operator, value, weight, active |
| routing_config | Thresholds and owner routes | route, min_score, owner, follow_up_days, active |
| review_queue | Low-confidence or unsafe cases | correlation_id, reason, status, reviewer, reviewed_at |
| execution_audit | Sanitized run history | correlation_id, workflow, stage, status, timestamps |
| dead_letter | Recoverable failure context | correlation_id, failed_stage, retryable, sanitized_input, status |
