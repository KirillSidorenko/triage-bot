# Workflow Design: AI Triage Bot

## Архитектура

```text
Telegram Trigger
  -> Normalize Input
  -> Valid Message?
    -> DeepSeek - Classify
    -> Parse AI JSON
    -> Validate AI Output
    -> Prepare Tracker Payload
    -> Needs Review?
      -> false: Yandex Tracker - Auto Route
             -> Google Sheets - Auto Route
             -> Telegram - Confirm (Auto)
      -> true:  Yandex Tracker - Needs Review
             -> Google Sheets - Needs Review
             -> Telegram - Confirm (Review)
             -> Telegram - Notify Reviewer

Parse/API fallback
  -> Yandex Tracker - Error Fallback
  -> Google Sheets - Error Fallback
  -> Telegram - Confirm (Error)
```

## Что является destination

- `Yandex Tracker` — operational destination для support-команд
- `Google Sheets` — audit log, а не конечная ticketing system
- `n8n` — orchestration layer и место, где живут guardrails

## Узлы n8n

### Telegram Trigger
- тип: `Telegram Trigger`
- получает входящее сообщение и metadata чата

### Valid Message?
- проверяет, что сообщение текстовое и не длиннее 4000 символов
- false branch ведёт в `Telegram - Invalid Input`

### Normalize Input
Готовит поля:
- `text`
- `sender`
- `sender_id`
- `chat_id`
- `timestamp`
- `received_at_ms`

### DeepSeek - Classify
- модель: `deepseek-chat`
- промпт: [config/prompts.md](/Users/sidorenko/Documents/triage_bot/config/prompts.md)
- выход: JSON с полями
  - `summary`
  - `category`
  - `priority`
  - `confidence`
  - `route`

### Parse AI JSON
- парсит строковый ответ модели в JSON
- при parse error workflow уходит в fallback

### Validate AI Output
Нормализует:
- category в фиксированный список из 6 категорий
- priority в `critical/high/medium/low`
- confidence в диапазон `0..1`

### Prepare Tracker Payload
Добавляет:
- `review_reason`
- `tracker_queue_auto`
- `tracker_queue_review`
- `tracker_summary_auto`
- `tracker_summary_review`
- `tracker_description_auto`
- `tracker_description_review`

Текущий staging mapping:
- `billing -> TBILLING`
- `technical_issue -> TENGSUP`
- `account_access -> TACCSEC`
- `feature_request -> TPRODUCT`
- `complaint -> TCSM`
- `general_inquiry -> TGENSUP`
- `review -> TSUPREV`

### Needs Review?
Условие:

```text
priority == critical OR confidence < 0.8
```

Это обязательный guardrail. Critical никогда не идёт в full auto.

### Yandex Tracker - Auto Route
- создаёт тикет в целевой очереди команды
- summary: краткое описание проблемы
- description: сообщение клиента + summary + служебный контекст

### Yandex Tracker - Needs Review
- создаёт тикет в review-очереди
- в description добавляет причину ручной проверки

### Yandex Tracker - Error Fallback
- создаёт тикет в review-очереди при parse/API failure
- summary: `Требуется ручная обработка обращения`

### Google Sheets terminal nodes
Пишут audit trail в spreadsheet:
- `Google Sheets - Auto Route`
- `Google Sheets - Needs Review`
- `Google Sheets - Error Fallback`

Колонки:
- `timestamp`
- `sender`
- `original_message`
- `summary`
- `category`
- `priority`
- `confidence`
- `route`
- `needs_review`
- `reviewer_decision`
- `final_category`
- `processing_time_ms`
- `tracker_issue_key`
- `tracker_queue`
- `tracker_issue_url`

### Telegram клиентские ответы
Клиент получает только подтверждение, без внутренних полей triage:
- `Telegram - Confirm (Auto)`
- `Telegram - Confirm (Review)`
- `Telegram - Confirm (Error)`

### Telegram - Notify Reviewer
- отправляет уведомление только во внутренний reviewer-чат
- содержит ссылку на тикет, сообщение клиента, summary и причину review

## Credentials

Workflow требует:
- `Telegram Bot`
- `DeepSeek`
- `Yandex Tracker` (`HTTP Request -> Custom Auth`)
- `Google Sheets`

## Known Runtime Constraints

- staging export использует встроенные queue keys
- reviewer chat id в export тоже встроен
- для n8n Community Edition workflow не опирается на env vars для Tracker routing

## Error Handling

- invalid input -> клиент получает короткую ошибку
- parse/API failure -> тикет создаётся в `TSUPREV` и клиент всё равно получает подтверждение
- Google Sheets пишет audit trail на каждой terminal branch

## Performance

- end-to-end latency: около `3 секунд`
- category/routing determinism обеспечивается фиксированным mapping из категории в команду и очередь
