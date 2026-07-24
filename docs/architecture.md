# Architecture — AI Lead Qualification, CRM, and Follow-Up System

Source of truth: `PORTFOLIO_MASTER_PROJECTS.md` §5.3 (Reusable architecture), §5.4 (Workflows), and §8 (Shared build foundation).

## 1. Reusable architecture (§5.3)

```text
Form / Webhook / Email Adapter
  → Validate + Normalize to Canonical Lead
  → Generate Correlation and Idempotency Keys
  → Deduplicate / Load Existing Lead State
  → AI Intent + Signal Extraction (structured output)
  → Confidence Gate
      ├─ low confidence → Human Review Queue
      └─ sufficient confidence
          → Configurable Deterministic Score
          → Route by Intent and Qualification
              ├─ qualified → CRM Upsert → Verify IDs → Alert → Confirmation/Booking
              ├─ nurture → Ask Approved Missing Questions → Set Follow-Up
              ├─ existing customer → Support Queue, not Sales Deal
              └─ spam/other → Audit and Stop
  → Weekly Pipeline Analytics

Any unrecovered failure
  → Shared Error Workflow (P0-99)
  → Dead-Letter Record + Alert + Replay Instructions
```

## 2. Workflow map (§5.4)

| ID | Workflow | File | Trigger | Responsibility |
|---|---|---|---|---|
| P1-01 | Lead Intake Adapters | `workflows/p1-lead-intake.json` | Form + Webhook | Normalize each source into the canonical lead contract; validate, dedup, write lead, call qualification core, orchestrate CRM handoff for qualified leads |
| P1-02 | Qualification Core | `workflows/p1-qualification-core.json` | Execute Sub-workflow | Load lead state, AI intent + signal extraction, confidence gate, deterministic score, route, return decision |
| P1-03 | CRM Handoff | `workflows/p1-crm-handoff.json` | Execute Sub-workflow | Set pending, search+upsert HubSpot contact/company/deal, verify IDs, update synced state, notify owner, differentiated failure handling |
| P1-04 | Follow-Up Sweeper | `workflows/p1-follow-up-sweeper.json` | Schedule (daily 09:00 IST) | Find due follow-ups, prevent repeat reminders (idempotent), alert owner via Asana |
| P1-05 | Weekly Pipeline Report | `workflows/p1-weekly-pipeline-report.json` | Schedule (weekly Mon 09:00 IST) | Deterministic aggregates from leads; AI narrates but cannot invent values; post report |
| P0-99 | Shared Error Handler | `workflows/shared-error-handler.json` | Error Trigger | Record execution_audit, dead-letter, alert with replay context |

## 3. Data flow between workflows

```text
P1-01 (Form/Webhook)
  │  canonical lead + lead_id
  ▼
P1-02 Qualification Core  ──(Execute Workflow)──  returns: route, score, temperature, deal_eligible, crm_sync_status
  │  if deal_eligible (qualified only)
  ▼
P1-03 CRM Handoff  ──(Execute Workflow)──  returns: crm_sync_status synced/failed, external IDs, notification_status

P1-04 (Schedule) reads leads table directly
P1-05 (Schedule) reads leads table directly
```

- P1-01 calls P1-02 via an **Execute Workflow** node (`workflowId` bound by controlled setup).
- P1-01 calls P1-03 via an **Execute Workflow** node, **only** when P1-02 returns `deal_eligible: true`. Existing-customer, spam, nurture, and review paths never call P1-03.
- P1-04 and P1-05 are independent scheduled workflows that read the `leads` Airtable table directly — they do not depend on the intake chain.

## 4. Per-workflow node structure

### P1-01 Lead Intake (17 nodes)
Two intake adapters (Form Trigger, Webhook Trigger) → per-source normalizer Code nodes → a single `Validate & Build Canonical Lead` Code node (email/required/size/source validation, correlation id, idempotency key, dedup formula) → `Lead Valid?` If → reject to `dead_letter` (Airtable) on invalid → `Dedup Check` (Airtable list by `idempotency_key`) → `Lead Already Exists?` If → return existing state (no duplicate) or `Write New Lead` (Airtable create, `qualification_status: received`, `crm_sync_status: pending`) → `Prepare Core Input` → `Call Qualification Core` (Execute Workflow) → `Build Intake Result` → `Qualified for CRM?` If → `Call CRM Handoff` (Execute Workflow) for qualified leads → `Final Intake Result`.

### P1-02 Qualification Core (17 nodes)
Execute Workflow Trigger → `Load Lead State` (Airtable) → `Prepare AI Input` (preserve known facts) → `AI Intent + Signal Extraction` (chainLlm + structured parser: intent, intent_confidence, extracted_signals, summary) with `continueOnFail` so LLM timeout/malformed does not crash the workflow → `Confidence & Schema Gate` (Code; threshold 0.70) → `Confidence Gate Passed?` If → false: `Write Review Queue` + `Mark Lead Review Required` + `Return Review Decision` (no deal) → true: `Load Scoring Config` → `Compute Deterministic Score` (Code; the LLM does not score) → `Load Routing Config` → `Route Lead` (Code; qualified/nurture/existing_customer/spam) → `Persist Qualification Result` (Airtable update) → `Return Qualification Decision`.

The OpenAI Chat Model and the structured output parser connect to the chain via `ai_languageModel` and `ai_outputParser`.

### P1-03 CRM Handoff (30 nodes)
Execute Workflow Trigger → `Prepare CRM Input` → `Set CRM Sync Pending` (Airtable; `crm_sync_status: pending` **before** any CRM call) → `Search HubSpot Contact` (HTTP, `neverError`+`fullResponse`) → `Resolve Contact` → `Contact Action` Switch (update/create/error) → `Update` or `Create HubSpot Contact` (PATCH/POST, capped retry) → `Capture Contact ID` → `Search HubSpot Company` → `Resolve Company` → `Company Needed?` If → `Create HubSpot Company` (conditional) → `Capture Company ID` → `Prepare Deal Request` (associates verified contact + company) → `Create HubSpot Deal` → `Verify CRM Result` (verifies contact **and** deal IDs; classifies 401/403 vs 429/5xx) → `CRM Verified?` If → true: `Update Lead Synced` (`crm_sync_status: synced` + `crm_contact_id` + `crm_company_id` + `crm_deal_id` + `crm_synced_at` — only after verification) → `Notify Sales Owner` (Asana, `continueOnFail`) → `Mark Notification Result` → `Notify OK?` If → false: `Write Notification Dead Letter` (deal stays synced, notification independently replayable) → `Return CRM Handoff Result`. False branch: `CRM Failure Handler` → `CRM Failure Type` Switch → auth: `Alert Config Error` (single immediate alert) → `Mark Lead Failed`; transient/other: `Mark Lead Failed` → `Write Dead Letter` → `Return Failure Result`.

### P1-04 Follow-Up Sweeper (6 nodes)
Schedule (daily 09:00 IST) → `Get Due Follow-ups` (Airtable list, `follow_up_at <= TODAY()`) → `Filter Unreminded` (Code; skips leads where `reminder_sent_at`/`last_reminded_at` >= `follow_up_at` — exactly-once reminder) → `Create Reminder Task` (Asana, per item, `continueOnFail`) → `Mark Reminder Sent` (Airtable update `reminder_sent_at`/`last_reminded_at`) → `Build Sweep Summary`.

### P1-05 Weekly Pipeline Report (8 nodes)
Schedule (weekly Mon 09:00 IST) → `Get All Leads` (Airtable) → `Aggregate Pipeline` (Code; deterministic counts by temperature/qualification_status/crm_sync_status, failures, due follow-ups) → `AI Summary of Aggregates` (chainLlm; prompt forbids inventing values) → `Build Report` (Code; numeric section is deterministic, AI only narrates) → `Post Report Task` (Asana) → `Confirm Report Sent`.

## 5. Shared error handler integration

`P0-99 Shared Error Handler` (`workflows/shared-error-handler.json`) starts with a native **Error Trigger**, normalizes/redacts the error payload, derives project/severity/error-class, writes an `execution_audit` record, writes a `dead_letter` record when replay context exists, and creates an Asana alert task.

**Integration (assigned after import):** every P1 workflow will have `errorWorkflow` set in its `settings` to the P0-99 workflow ID. During offline construction this binding is intentionally omitted (no fake workflow ID is hard-coded, per the shared conventions). After import:
1. imports P0-99 and notes its workflow ID;
2. sets `settings.errorWorkflow` on P1-01..P1-05 to that ID;
3. re-validates and activates test copies.

In-workflow failure handling (P1-03's auth/transient classification and notification dead-letter) operates **before** the error trigger and handles recoverable item-level failures. The shared error handler catches workflow/execution-level failures that escape in-workflow handling. The two layers are complementary, not redundant.

## 6. Adapters (§5.9)

| Layer | Standard demo | Swappable options |
|---|---|---|
| Intake | n8n Form + Webhook | Gmail/IMAP, Facebook Lead Ads, Typeform, Tally, Jotform, chat, CSV |
| CRM | HubSpot (HTTP Request, v3 API) | GoHighLevel, Pipedrive, Zoho, Salesforce, Close |
| Notifications | Asana task | Gmail, Slack, Teams, WhatsApp/Twilio, SMS |
| Booking | Calendar link in alert | Calendly, Cal.com, Google Calendar |
| State | Airtable `leads` + config tables | n8n Data Tables, Supabase, Postgres, MySQL |
| Reporting | Asana task | Email, Google Sheets, Airtable Interface, Looker Studio |

Each proposal changes adapters and business rules, not the verified qualification core.

## 7. State vocabulary (§8.3)

```text
received → validated → processing → review_required → succeeded
                                  └→ retry_pending → failed → replayed
```

External side effects carry their own state (`crm_sync_status`: `pending` → `synced` | `failed`). The overall execution is not treated as successful merely because one external action succeeded.
