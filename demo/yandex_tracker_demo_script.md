# Demo Script - Yandex Tracker Routing

## Goal

Show that `n8n` is the decision-making layer and `Yandex Tracker` is the operational destination where the routed ticket becomes visible to support teams.

## Flow to Show

```text
Telegram -> n8n -> DeepSeek V3 -> Yandex Tracker queue
                                 -> Google Sheets audit log
```

## Scenario 1 - Auto Route

Send in Telegram:

```text
Мне дважды списали деньги за подписку в этом месяце. Верните деньги!
```

What to show:
- Telegram confirmation with issue key
- Tracker issue appears in the billing queue
- Google Sheets row appears with:
  - `category = billing`
  - `priority = high`
  - `needs_review = false`
  - `tracker_issue_key = ...`

Talk track:

> n8n calls the model, validates the JSON, applies the confidence guardrail, and creates the final ticket in the destination queue. Tracker is the execution surface, not the orchestration layer.

## Scenario 2 - Human Review

Send in Telegram:

```text
Кто-то взломал мой аккаунт! Вижу входы из другой страны!
```

What to show:
- Telegram confirmation with issue key
- Tracker issue appears in the review queue, not in the final team queue
- Reviewer gets a Telegram notification with:
  - original message
  - suggested category / priority / confidence
  - reason for review
  - Tracker link
- Google Sheets row shows `needs_review = true`

Talk track:

> Critical tickets never auto-route. n8n deliberately parks them in a review queue so a human validates the AI suggestion before the issue is handed off.

## Scenario 3 - Why Not Native Tracker Automation

Key answer:

> Tracker stores and displays the final work item. n8n owns the cross-system orchestration: Telegram intake, LLM call, guardrails, retries, fallback logic, reviewer notifications, and audit logging.

## Demo Close

Final line:

> The routing is visible in a real ticket system, but the business logic stays outside the ticket system. That makes the workflow portable: the same n8n layer could route to another desk later without rewriting the AI logic.
