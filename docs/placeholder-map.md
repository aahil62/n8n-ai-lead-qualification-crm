# Placeholder map

Every placeholder is resource-specific so replacements can be reviewed safely.
Do not perform a global replacement across unrelated resources.

| Placeholder | Resource | Used in | Required permission |
|---|---|---|---|
| `REPLACE-AIRTABLE-BASE-ID` | Airtable base | `p1-crm-handoff.json`, `p1-follow-up-sweeper.json`, `p1-lead-intake.json`, `p1-qualification-core.json`, `p1-weekly-pipeline-report.json`, `shared-error-handler.json` | Read/write for only the documented tables |
| `REPLACE-ASANA-PROJECT-ID` | Asana project | `p1-crm-handoff.json`, `p1-follow-up-sweeper.json`, `p1-weekly-pipeline-report.json`, `shared-error-handler.json` | Task creation |
| `REPLACE-ASANA-WORKSPACE-ID` | Asana workspace | `p1-crm-handoff.json`, `p1-follow-up-sweeper.json`, `p1-weekly-pipeline-report.json`, `shared-error-handler.json` | Workspace lookup |
| `REPLACE-DEAD-LETTER-TABLE-ID` | Airtable dead-letter table | `p1-crm-handoff.json`, `p1-lead-intake.json`, `shared-error-handler.json` | Read/write |
| `REPLACE-EXECUTION-AUDIT-TABLE-ID` | Airtable execution-audit table | `shared-error-handler.json` | Read/write |
| `REPLACE-LEADS-TABLE-ID` | Airtable leads table | `p1-crm-handoff.json`, `p1-follow-up-sweeper.json`, `p1-lead-intake.json`, `p1-qualification-core.json`, `p1-weekly-pipeline-report.json` | Read/write |
| `REPLACE-P1-CRM-HANDOFF-WORKFLOW-ID` | Imported P1-03 CRM Handoff workflow | `p1-lead-intake.json` | Execute |
| `REPLACE-P1-QUALIFICATION-WORKFLOW-ID` | Imported P1-02 Qualification Core workflow | `p1-lead-intake.json` | Execute |
| `REPLACE-REVIEW-QUEUE-TABLE-ID` | Airtable review-queue table | `p1-qualification-core.json` | Read/write |
| `REPLACE-ROUTING-CONFIG-TABLE-ID` | Airtable routing-config table | `p1-qualification-core.json` | Read |
| `REPLACE-SCORING-CONFIG-TABLE-ID` | Airtable scoring-config table | `p1-qualification-core.json` | Read |
| `REPLACE-SHARED-ERROR-WORKFLOW-ID` | Imported shared error-handler workflow | `p1-crm-handoff.json`, `p1-follow-up-sweeper.json`, `p1-lead-intake.json`, `p1-qualification-core.json`, `p1-weekly-pipeline-report.json` | Error-workflow assignment |

After replacement, open every affected node in n8n and confirm the selected
resource by name. A syntactically valid ID can still point to the wrong table,
workflow, mailbox, workspace, or project.
