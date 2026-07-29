# Test Plan — AI Lead Qualification, CRM, and Follow-Up System

15 end-to-end scenarios (P1-E2E-001 .. P1-E2E-015) from `PORTFOLIO_MASTER_PROJECTS.md` §5.8. Each scenario has one sanitized fixture in `fixtures/`.

## 1. Fixture classification

| Folder | Scenarios | Why |
|---|---|---|
| `fixtures/valid/` | 001, 002, 003, 005, 006, 015 | Normal/routing paths that produce a clean observable result |
| `fixtures/edge-cases/` | 004, 007, 008, 013, 014 | Idempotency, returning-lead merge, NL date parsing, prompt injection, low confidence |
| `fixtures/failure-cases/` | 009, 010, 011, 012 | CRM rate-limit/auth, LLM timeout, notification failure |

All fixtures use `@northstar.example` emails and Northstar Growth Studio fictional data.

## 2. Execution method

1. Bind credentials and workflow IDs per `docs/setup.md`.
2. For each scenario, send the fixture `input` through the appropriate entry point:
   - **Form/webhook scenarios**: POST the `input` body to the P1-01 webhook, or submit via the form test URL.
   - **Duplicate/replay (007)**: send the same `input` twice with the same `source_event_id`.
   - **Returning lead (008)**: seed an existing `leads` row with the known signals, then send the new message.
   - **Failure scenarios (009, 010, 012)**: induce the failure condition as in `docs/setup.md` §6 before sending.
   - **LLM timeout (011)**: force OpenAI timeout/malformed output.
   - **Weekly report (015)**: seed the `leads` table with a known set of records, then manually trigger P1-05.
   - **Follow-up date (013)**: send the nurture reply; trigger P1-04 and confirm the timezone-aware date.
3. Record the actual result, n8n execution ID, external evidence, and pass/fail in `test-report.md`.

## 3. Scenario matrix

| Test ID | Scenario | Fixture | Entry | Required observable result |
|---|---|---|---|---|
| P1-E2E-001 | High-quality new lead | `valid/P1-E2E-001.json` | P1-01 webhook | One contact + one deal upserted; correct score, owner, alert, `synced` state, and external IDs |
| P1-E2E-002 | Warm lead missing budget | `valid/P1-E2E-002.json` | P1-01 webhook | Missing information recorded, nurture state stored, and owner follow-up scheduled; no deal |
| P1-E2E-003 | Cold lead | `valid/P1-E2E-003.json` | P1-01 webhook | No deal and no sales alert; audited correctly |
| P1-E2E-004 | Low AI confidence | `edge-cases/P1-E2E-004.json` | P1-01 webhook | Review record + human alert; no automated external side effect |
| P1-E2E-005 | Existing customer request | `edge-cases/P1-E2E-005.json` | P1-01 webhook | Classified as an existing-customer request; no new sales deal |
| P1-E2E-006 | Spam/newsletter | `edge-cases/P1-E2E-006.json` | P1-01 webhook | Stopped and audited; no reply, deal, or task |
| P1-E2E-007 | Duplicate webhook replay | `edge-cases/P1-E2E-007.json` | P1-01 webhook ×2 | Same lead state and CRM IDs; no duplicate contact/deal/alert |
| P1-E2E-008 | Returning lead adds timeline | `edge-cases/P1-E2E-008.json` | seeded + P1-01 | Existing facts preserved, new signal merged, score recalculated |
| P1-E2E-009 | CRM 429/5xx | `failure-cases/P1-E2E-009.json` | P1-01 (induce 429) | Capped retry; success or dead-letter with replay data; no false `synced` |
| P1-E2E-010 | CRM 401 | `failure-cases/P1-E2E-010.json` | P1-01 (induce 401) | Immediate config alert; no repeated retry storm |
| P1-E2E-011 | LLM timeout/malformed | `failure-cases/P1-E2E-011.json` | P1-01 (force timeout) | Review path; no deal created |
| P1-E2E-012 | Notification failure after CRM success | `failure-cases/P1-E2E-012.json` | P1-01 (break Asana) | Deal remains recorded; notification marked `failed` and independently replayable |
| P1-E2E-013 | Natural-language follow-up date | `edge-cases/P1-E2E-013.json` | P1-01 + trigger P1-04 | Correct timezone-aware date stored and included exactly once when due |
| P1-E2E-014 | Malicious instructions in lead text | `edge-cases/P1-E2E-014.json` | P1-01 webhook | Text treated as data; policy and output schema intact; no policy change |
| P1-E2E-015 | Weekly report | `valid/P1-E2E-015.json` | trigger P1-05 | Aggregates match seeded records exactly and failed syncs are visible |

## 4. Publication gate (§5.8)

- All 15 scenarios pass with sanitized evidence.
- Two qualified submissions with the same idempotency key create only one CRM deal (007).
- Every successful CRM state contains real test-account object IDs.
- Zero low-confidence, existing-customer, or spam fixtures create a sales deal (004, 005, 006, 011).
- A forced CRM failure appears in the shared error workflow and can be replayed safely (009, 010).
- Workflow validation returns no unresolved errors.

## 5. Negative assertions to verify explicitly

- No `REPLACE-` placeholders remain after setup.
- No secrets in any exported workflow or screenshot.
- No `.example.com` endpoints in runnable workflows (fixtures use `.example` emails only).
- `crm_sync_status` is never `synced` without populated `crm_contact_id` + `crm_deal_id`.
