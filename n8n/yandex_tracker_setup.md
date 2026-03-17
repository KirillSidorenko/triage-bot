# Yandex Tracker Setup

This guide adds `Yandex Tracker` as the operational ticket system while keeping `Google Sheets` as the audit log.

## What Changes

New runtime path:

```text
Telegram -> n8n -> DeepSeek V3 -> Yandex Tracker
                                 -> Google Sheets
```

`n8n` remains the orchestration layer:
- calls the LLM
- applies `confidence` / `critical` guardrails
- decides whether the ticket is auto-routed or sent to review
- creates the final ticket in `Yandex Tracker`
- logs the full decision trail in `Google Sheets`

## Required Tracker Queues

Create one queue per destination team and one queue for manual review.

Suggested example keys:

| Purpose | Example queue key |
|---------|-------------------|
| Billing | `BILLING` |
| Technical support | `ENGSUP` |
| Account / security | `ACCSEC` |
| Product requests | `PRODUCT` |
| Complaints / CS | `CSM` |
| General support | `GENSUP` |
| Human review | `SUPREV` |

You can use different keys. In the exported staging workflow the queue mapping is embedded directly in the node config, so Community Edition does not need env access.

Notes:
- This workflow uses the official Tracker API endpoint `https://api.tracker.yandex.net/v3/issues/`.
- Tracker authentication is stored in an n8n credential of type `HTTP Request -> Custom Auth`.
- The exported workflow expects the credential name `Yandex Tracker`.
- That credential should contain both headers:
  - `Authorization: OAuth <token>`
  - `X-Cloud-Org-Id: <org_id>`
- The OAuth app must include Tracker scopes. In practice you need at least `tracker:read` and `tracker:write`.
- If your Tracker org uses `X-Org-ID` instead of `X-Cloud-Org-Id`, update the HTTP Request headers in the imported workflow.

## Embedded Staging Defaults

`n8n/workflow_export_yandex_tracker.json` is currently a self-contained staging workflow with these built-in queue keys:

```text
billing=TBILLING
technical_issue=TENGSUP
account_access=TACCSEC
feature_request=TPRODUCT
complaint=TCSM
general_inquiry=TGENSUP
review=TSUPREV
```

The exported staging workflow sends reviewer notifications to the internal Telegram chat `-1003807392857`.

Before moving this workflow to production, change:
- queue keys in `Prepare Tracker Payload`
- review queue key in `Yandex Tracker - Error Fallback`
- reviewer target chat in `Telegram - Notify Reviewer`

Official docs used for this integration:
- [Create issue](https://yandex.ru/support/tracker/en/concepts/issues/create-issue)
- [Support process](https://yandex.ru/support/tracker/ru/support-process)
- [Support queue](https://yandex.ru/support/tracker/ru/support-process-create-queue)
- [SLA](https://yandex.ru/support/tracker/ru/sla-head)

## Google Sheets Columns

The workflow uses `Select From list` for both spreadsheet and sheet.

Current default selection in the exported workflow:
- Spreadsheet: `Triage Bot Results`
- Sheet: `Triage Log - Staging`

You can change both directly in the n8n node UI after import.

Add these extra columns to the target log sheet before running the Tracker workflow:

```text
tracker_issue_key
tracker_queue
tracker_issue_url
```

Recommended column order:

```text
timestamp
sender
original_message
summary
category
priority
confidence
route
needs_review
reviewer_decision
final_category
processing_time_ms
tracker_issue_key
tracker_queue
tracker_issue_url
```

## Workflow File

Import:

`n8n/workflow_export_yandex_tracker.json`

This workflow keeps the existing Telegram intake, DeepSeek classification, and reviewer notification flow, but adds Tracker ticket creation on all terminal branches:
- auto-route
- needs review
- parse / API fallback

## How Routing Looks in Tracker

- `confidence >= 0.8` and not `critical`:
  ticket is created directly in the routed queue.
- `confidence < 0.8` or `priority = critical`:
  ticket is created in the review queue.
- parse / API failure:
  ticket is also created in the review queue with an error summary.

This makes routing visible in the UI without moving the decision logic into Tracker automation.

## Separate Staging Test

To test the workflow in isolation before merging:

- use the pre-created sheet tab `Triage Log - Staging`
- import this workflow into a separate disabled n8n workflow first
- keep the embedded staging queues as-is
- attach a separate Telegram bot credential if you want a clean end-to-end test channel
