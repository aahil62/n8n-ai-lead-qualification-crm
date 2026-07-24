# Test Report — AI Lead Qualification, CRM, and Follow-Up System

**Status:** `representative_live_path_passed`. The qualified-lead path passed intake, Airtable persistence, AI qualification, HubSpot contact/company/deal creation, Airtable verification, and Asana notification.

| Test ID | Scenario | Fixture path | Expected result | Actual result | n8n execution | External evidence | Result | Notes |
|---|---|---|---|---|---|---|---|---|
| P1-E2E-001 | High-quality new lead | `fixtures/valid/P1-E2E-001.json` | One contact + one deal upserted; correct score, owner, alert, `synced` state, external IDs | Score 100, hot, confidence 0.90; Airtable lead marked synced; HubSpot contact/company/deal created and verified; Asana task created; final status `crm_handoff_complete` | parent 75; children 76/77 | Airtable `[sanitized Airtable record]`; HubSpot contact `[sanitized HubSpot record]`, company `[sanitized HubSpot record]`, deal `[sanitized HubSpot record]`; Asana `[sanitized Asana task]` | `passed` | Service-key credential `REPLACE-HUBSPOT-CREDENTIAL-ID`; deal stage `qualifiedtobuy` in default pipeline |
| P1-E2E-002 | Warm lead missing budget | `fixtures/valid/P1-E2E-002.json` | No deal; approved question/draft created; follow-up state stored | — | — | — | `not_run_live` | Route = nurture; follow_up_at set |
| P1-E2E-003 | Cold lead | `fixtures/valid/P1-E2E-003.json` | No deal and no sales alert; audited | — | — | — | `not_run_live` | Route = spam (score < 30) or cold nurture |
| P1-E2E-004 | Low AI confidence | `fixtures/edge-cases/P1-E2E-004.json` | Review record + human alert; no external side effect | — | — | — | `not_run_live` | intent_confidence < 0.70 → review_required |
| P1-E2E-005 | Existing customer request | `fixtures/valid/P1-E2E-005.json` | Support queue; no new sales deal | — | — | — | `not_run_live` | intent = existing_customer |
| P1-E2E-006 | Spam/newsletter | `fixtures/valid/P1-E2E-006.json` | Stopped and audited; no reply/deal/task | — | — | — | `not_run_live` | intent = spam_or_other |
| P1-E2E-007 | Duplicate webhook replay | `fixtures/edge-cases/P1-E2E-007.json` | Same lead state + CRM IDs; no duplicate contact/deal/alert | — | — | — | `not_run_live` | Send same input twice; idempotency_key dedup |
| P1-E2E-008 | Returning lead adds timeline | `fixtures/edge-cases/P1-E2E-008.json` | Existing facts preserved, new signal merged, score recalculated | — | — | — | `not_run_live` | Seed known signals then send new message |
| P1-E2E-009 | CRM 429/5xx | `fixtures/failure-cases/P1-E2E-009.json` | Capped retry; success or dead-letter w/ replay; no false `synced` | — | — | — | `not_run_live` | Induce 429; retryable=true in dead_letter |
| P1-E2E-010 | CRM 401 | `fixtures/failure-cases/P1-E2E-010.json` | Immediate config alert; no retry storm | — | — | — | `not_run_live` | Induce 401; retryable=false; single Asana alert |
| P1-E2E-011 | LLM timeout/malformed | `fixtures/failure-cases/P1-E2E-011.json` | Review path; no deal created | — | — | — | `not_run_live` | continueOnFail on AI chain → gate → review |
| P1-E2E-012 | Notification failure after CRM success | `fixtures/failure-cases/P1-E2E-012.json` | Deal stays synced; notification `failed` + replayable | — | — | — | `not_run_live` | Break Asana post-sync; deal_unchanged=true |
| P1-E2E-013 | Natural-language follow-up date | `fixtures/edge-cases/P1-E2E-013.json` | Timezone-aware date stored; reminded exactly once when due | — | — | — | `not_run_live` | Asia/Kolkata; reminder_sent_at idempotency |
| P1-E2E-014 | Malicious instructions in lead text | `fixtures/edge-cases/P1-E2E-014.json` | Text treated as data; schema/policy intact | — | — | — | `not_run_live` | Prompt forbids following in-message instructions |
| P1-E2E-015 | Weekly report | `fixtures/valid/P1-E2E-015.json` | Aggregates match seeded records; failed syncs visible | — | — | — | `not_run_live` | Deterministic aggregates; AI only narrates |

## Summary

- **Total scenarios:** 15
- **Passed:** 1
- **Failed:** 0
- **Blocked by external auth:** 0
- **Not run live:** 14
- **Runtime evidence captured:** Airtable, HubSpot, Asana, and n8n executions 75/76/77

The representative positive path is verified. The other 14 fixtures remain explicit pre-production matrix work, not implied passes.
