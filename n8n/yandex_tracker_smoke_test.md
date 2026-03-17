# Yandex Tracker Smoke Test

Use this checklist to test the Tracker-based workflow in isolation before merging it into the main branch.

## Goal

Verify the full path:

```text
Telegram -> n8n -> DeepSeek V3 -> Yandex Tracker -> Google Sheets
```

without sending tickets into the main queues or writing to the main log tab.

## Isolation Setup

Recommended:
- import the workflow as a separate n8n workflow, do not overwrite the main one
- keep the Google Sheets nodes pointed at `Triage Bot Results -> Triage Log - Staging` using `Select From list`
- make sure all three Tracker HTTP Request nodes use the `Yandex Tracker` custom auth credential
- use a separate Telegram bot credential if you want a clean signal path
- keep the workflow disabled until all credentials are set
- leave the built-in staging queues unchanged:
  - `TBILLING`
  - `TENGSUP`
  - `TACCSEC`
  - `TPRODUCT`
  - `TCSM`
  - `TGENSUP`
  - `TSUPREV`
- reviewer notifications are sent to the internal Telegram chat `-1003807392857`

## Test Cases

### 1. Auto Route

Send:

```text
Мне дважды списали деньги за подписку в этом месяце. Верните деньги!
```

Expected:
- issue created in `TBILLING`
- `needs_review = false`
- Google Sheets row written to `Triage Log - Staging`
- Telegram confirmation contains an internal ticket number

### 2. Human Review

Send:

```text
Кто-то взломал мой аккаунт! Вижу входы из другой страны!
```

Expected:
- issue created in `TSUPREV`
- `needs_review = true`
- reviewer notification sent
- Google Sheets row includes `tracker_issue_key`, `tracker_queue`, `tracker_issue_url`

### 3. Feature Request

Send:

```text
Please add dark mode to the interface.
```

Expected:
- issue created in `TPRODUCT`
- `priority = low`
- `needs_review = false`

### 4. Invalid Input

Send a non-text payload or a message longer than 4000 characters.

Expected:
- customer receives invalid-input response
- no Tracker issue is created

## Exit Criteria

The staging workflow is safe to merge only if:
- all three routed branches create issues in the expected staging queues
- the review path always lands in `TSUPREV`
- the staging Google Sheets tab contains the full audit trail
- there are no writes to the production queues or the main log tab
