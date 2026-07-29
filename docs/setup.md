# Setup guide

## Requirements

- a disposable n8n test instance
- test accounts for the integrations named in the README
- credentials created inside n8n's encrypted credential store
- fictional data for all test runs

## Import order

1. `workflows/shared-error-handler.json`
2. `workflows/p1-lead-intake.json`
3. `workflows/p1-qualification-core.json`
4. `workflows/p1-crm-handoff.json`
5. `workflows/p1-follow-up-sweeper.json`
6. `workflows/p1-weekly-pipeline-report.json`

The shared error handler is intentionally first. After import, replace
`REPLACE-SHARED-ERROR-WORKFLOW-ID` with its assigned n8n ID.

## Workflow-to-workflow mapping

| Placeholder | Imported workflow | Used by |
|---|---|---|
| `REPLACE-P1-QUALIFICATION-WORKFLOW-ID` | Imported P1-02 Qualification Core | P1-01 |
| `REPLACE-P1-CRM-HANDOFF-WORKFLOW-ID` | Imported P1-03 CRM Handoff | P1-01 |

Reconnect each Execute Workflow node through the n8n editor rather than editing
an exported ID blindly.

## Credentials and minimum permissions

| Credential | Minimum intended access |
|---|---|
| Airtable | Read/write access only to the portfolio base tables |
| HubSpot service key/private app | Contacts, companies, and deals read/write |
| Asana | Task creation in the selected project |
| OpenAI-compatible model | Structured inference for qualification only |

Credentials belong in n8n's encrypted credential store. Never paste a token
into a Set, Edit Fields, Code, or HTTP-header text field.

## Resource mapping

Create the Airtable leads, scoring-config, routing-config, review-queue, execution-audit, and dead-letter tables. Configure HubSpot contacts, companies, deals, and the target Asana project.

Use [the placeholder map](placeholder-map.md), [the Airtable schema](../config/airtable-schema.md),
and the CSV samples under `config/`. Confirm every field name and option
against the destination account instead of accepting defaults.

## Model configuration

Bind an OpenAI-compatible credential to the model nodes and request structured
output matching the data contract. Run the invalid-schema and timeout fixtures
before relying on the model path. Provider interchangeability is not proof that
every provider has been tested.

## Controlled test sequence

1. Keep all workflows inactive.
2. Run `node scripts/validate-repository.mjs`.
3. Import into a disposable n8n instance and validate every complete workflow.
4. Re-open each Execute Workflow node and verify its selected target.
5. Run one valid fixture through the full path.
6. Run the duplicate or unchanged case.
7. Run a malformed or low-confidence input.
8. Run an authentication or downstream-service failure.
9. Inspect destination, audit, review, and dead-letter state.
10. Deactivate the workflows after evidence capture.

## Activation checklist

- [ ] All placeholders are replaced with test or client-specific resources.
- [ ] Credentials use least-privilege scopes and contain no secrets in node text.
- [ ] Webhooks are authenticated or protected at the gateway.
- [ ] Workflow and connection validation pass.
- [ ] Duplicate and failure behavior pass with fictional data.
- [ ] Retention, owner notifications, and human-review responsibility are agreed.
- [ ] External triggers and schedules are enabled only after sign-off.

## Rollback and deactivation

Disable schedules, polling triggers, and webhooks first. Deactivate the project
workflows, preserve execution/audit evidence, and revert to the last validated
workflow version or exported JSON. Do not delete destination records during
rollback unless the client has explicitly approved the cleanup plan.

## Production boundary

The exports are a reusable reference implementation, not a drop-in production deployment. A real rollout needs client-specific credentials, field ownership, retention, failure policy, rate limits, and acceptance tests.
