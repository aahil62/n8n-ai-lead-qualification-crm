# Verification and limitations

## Confirmed

A representative lead path passed through intake, AI signal extraction, deterministic scoring, Airtable state management, HubSpot contact/company/deal handling, and an Asana owner notification.

The workflow JSON is sanitized, parses successfully, has no credential blocks, contains no live account identifiers, and is exported inactive.

## Not yet claimed

The remaining replay, malformed-input, rate-limit, and notification-failure matrix is documented but has not been run live as a complete production gate.

Do not describe the package as fully production-ready until the documented matrix passes against the target client's real field configuration and rate limits.

## Test assets

15 lead scenarios covering valid, duplicate, low-confidence, prompt-injection, CRM failure, and follow-up cases.
