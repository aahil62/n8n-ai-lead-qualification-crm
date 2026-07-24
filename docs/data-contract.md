# Data Contract — AI Lead Qualification, CRM, and Follow-Up System

Source of truth: `PORTFOLIO_MASTER_PROJECTS.md` §5.5 (Canonical lead contract, required state fields, configuration tables) and §8.2 (Shared tables).

## 1. Canonical lead contract

The shape every intake adapter normalizes to **before any AI node**. This is the payload P1-01 sends into P1-02.

```json
{
  "correlation_id": "uuid",
  "idempotency_key": "source:stable-event-id",
  "source": "form|webhook|gmail|other",
  "source_event_id": "string",
  "received_at": "ISO-8601",
  "name": "string",
  "email": "string",
  "phone": "string|null",
  "company": "string|null",
  "message": "string",
  "consent_status": "known-opt-in|unknown|not-applicable",
  "campaign": "string|null",
  "owner_hint": "string|null"
}
```

### Derivation rules
- `correlation_id`: RFC-4122-style UUID generated in the intake Code node (pure JS, no `crypto`).
- `idempotency_key`: `source:stable-event-id` when a stable event id exists (form submission id, webhook event id, email Message-ID). **Fallback:** `source:content-<fnv1a-hash>` of `email|message|source` when no stable id is present. Content hash is a fallback only, never the primary key.
- `email`: lowercased, regex-validated (`/^[^\s@]+@[^\s@]+\.[^\s@]+$/`).
- `consent_status`: defaults to `unknown` when not supplied.

### Validation rules (P1-01)
Reject to `dead_letter` when any of:
- invalid or missing email
- missing name
- missing/too-short message
- payload > 20000 chars
- `source` not in the allowed set (`form`, `webhook`, `gmail`, `other`)

## 2. Required state fields (leads table)

```text
lead_id, correlation_id, idempotency_key, source, source_event_id,
intent, intent_confidence, extracted_signals, score, temperature,
qualification_status, review_status, crm_sync_status, crm_contact_id,
crm_company_id, crm_deal_id, owner_id, follow_up_at, last_action_at,
last_error_code, created_at, updated_at
```

Additional operational fields used by the workflows: `message`, `name`, `email`, `phone`, `company`, `consent_status`, `campaign`, `owner_hint`, `summary`, `reminder_sent_at`, `last_reminded_at`, `crm_synced_at`, `received_at`, `review_status`.

### State semantics

| Field | Values | Set by |
|---|---|---|
| `qualification_status` | `received` → `review_required` \| `qualified` \| `nurture` \| `existing_customer` \| `spam` | P1-01 sets `received`; P1-02 sets the route |
| `crm_sync_status` | `pending` → `synced` \| `failed` \| `not_applicable` | P1-01 `pending`; P1-03 `synced` only after ID verification, `failed` on failure; `not_applicable` for non-deal routes |
| `review_status` | `none` \| `pending` \| `reviewed` | P1-02 review path sets `pending` |
| `temperature` | `cold` \| `warm` \| `hot` | P1-02 deterministic score (hot ≥ 70, warm ≥ 40, else cold) |
| `extracted_signals` | JSON string of `{company,budget,timeline,project_size,decision_authority,readiness}` | P1-02 (stored via `JSON.stringify`) |

### External-write state rule (corrected prototype defect)
`crm_sync_status` is `pending` **before** the CRM call. `synced` (and `crm_contact_id`, `crm_company_id`, `crm_deal_id`, `crm_synced_at`) is written **only after** the returned HubSpot contact and deal IDs are verified present. On failure → `failed` + `last_error_code`. The old prototype set `crm_synced: true` before the CRM request succeeded; that is no longer the case.

## 3. Configuration tables

### `scoring_config` (Airtable table `REPLACE-AIRTABLE-TABLE-ID`)

```text
client_id, rule_id, field, operator, expected_value, points,
required, active, version, effective_from
```

- `field`: one of the `extracted_signals` keys (`company`, `budget`, `timeline`, `project_size`, `decision_authority`, `readiness`).
- `operator`: `exists` / `is_not_empty` / `equals` / `eq` / `contains` / `regex`.
- `points`: integer added to the score when the rule matches.
- `active`: only rules with `active !== false` are applied.
- The **LLM does not read or write this table** and never assigns the score. P1-02 `Compute Deterministic Score` is the only scorer.

Example seed (illustrative):
| field | operator | expected_value | points | active |
|---|---|---|---:|---|
| budget | exists |  | 25 | true |
| timeline | exists |  | 20 | true |
| project_size | exists |  | 15 | true |
| decision_authority | regex | `owner\|founder\|director\|head\|vp\|c[ -]?level\|ceo\|cfo` | 25 | true |
| readiness | regex | `ready\|now\|immediate\|asap\|this (week\|month)\|approved\|purchas` | 15 | true |

### `routing_config` (Airtable table `REPLACE-AIRTABLE-TABLE-ID`)

```text
client_id, route_id, condition, owner_id, notification_channel,
calendar_link, active
```

- `condition`: one of `qualified`, `nurture`, `existing_customer`, `spam`.
- `owner_id`: sales owner assigned for that route.
- `notification_channel`: `asana` (default), `gmail`, `slack`, etc.
- `calendar_link`: booking link included in the sales alert for qualified leads.
- P1-02 `Route Lead` matches the computed route to the first active rule and falls back to `owner_hint` / `unassigned`.

## 4. Shared tables (§8.2)

### `execution_audit` (Airtable table `REPLACE-AIRTABLE-TABLE-ID`)

```text
correlation_id, project_id, workflow_id, execution_id, stage,
status, attempt, started_at, completed_at, external_ids,
error_code, sanitized_error, replay_of
```

Written by P0-99 Shared Error Handler on workflow-level failures.

### `dead_letter` (Airtable table `REPLACE-AIRTABLE-TABLE-ID`)

```text
dead_letter_id, correlation_id, project_id, workflow_id,
failed_stage, sanitized_input, error_code, retryable,
attempts, status, created_at, resolved_at, replay_execution_id
```

Written by P1-01 (rejected invalid leads), P1-03 (CRM failures and notification failures), and P0-99. `retryable` is `true` for transient 429/5xx and notification failures, `false` for auth and validation errors.

### `review_queue` (Airtable table `REPLACE-AIRTABLE-TABLE-ID`)

```text
review_id, message_id, reason, draft_text, status, assigned_to,
created_at, reviewed_at, final_action
```

Written by P1-02 when the confidence/schema gate fails (low confidence, malformed, or LLM timeout). No CRM deal is created for any review-routed lead.

## 5. Airtable base and table IDs (non-secret, for the target environment)

Base: `REPLACE-AIRTABLE-BASE-ID` ("Portfolio Automation OS").

| Table | ID |
|---|---|
| leads | `REPLACE-AIRTABLE-TABLE-ID` |
| scoring_config | `REPLACE-AIRTABLE-TABLE-ID` |
| routing_config | `REPLACE-AIRTABLE-TABLE-ID` |
| execution_audit | `REPLACE-AIRTABLE-TABLE-ID` |
| dead_letter | `REPLACE-AIRTABLE-TABLE-ID` |
| review_queue | `REPLACE-AIRTABLE-TABLE-ID` |

## 6. HubSpot CRM objects (P1-03)

- Contact: `POST /crm/v3/objects/contacts`, search `POST /crm/v3/objects/contacts/search` (filter `email EQ`), update `PATCH /crm/v3/objects/contacts/{id}`.
- Company: `POST /crm/v3/objects/companies`, search `POST /crm/v3/objects/companies/search` (filter `name EQ`). Skipped when no company name.
- Deal: `POST /crm/v3/objects/deals` with associations to the verified contact (and company when present).
- Authentication: `predefinedCredentialType` / `hubspotAppToken`; live-verified n8n credential `REPLACE-HUBSPOT-CREDENTIAL-ID`.

## 7. Notification contract (sales alert)

The qualified-lead alert (Asana task in the live demo) contains: lead name + email, company, temperature + score, intent, source, owner, summary, verified HubSpot contact/company/deal IDs, and the booking `calendar_link` from `routing_config`.
