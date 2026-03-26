# Инструкция по настройке инфраструктуры

## 1. Telegram Bot

### Создание бота
1. Открыть Telegram и найти [@BotFather](https://t.me/BotFather)
2. Отправить `/newbot`
3. Ввести имя бота, например `Triage Support Bot`
4. Ввести username, заканчивающийся на `bot`
5. Сохранить bot token для n8n credential

### Настройка описания
Отправить BotFather:

```text
/setdescription — AI-бот для регистрации обращений в поддержку
/setabouttext — Отправьте ваш вопрос, и мы направим его в нужную команду
```

### Review-чат
1. Создать отдельную внутреннюю Telegram-группу для ревьюеров
2. Добавить туда того же бота, который используется в n8n
3. Отправить в чат любое сообщение, например `/review`
4. Взять `message.chat.id` из execution `Telegram Trigger` в n8n

Важно:
- для групп и супергрупп `chat_id` обычно отрицательный и имеет вид `-100...`
- не используйте `getUpdates`, если бот уже работает через webhook в n8n

---

## 2. n8n

### Импорт workflow
1. Открыть n8n
2. Выбрать `Workflows -> Import from File`
3. Импортировать [n8n/workflow_export.json](/Users/sidorenko/Documents/triage_bot/n8n/workflow_export.json)

Текущий основной export уже синхронизирован с Tracker-вариантом. Отдельный debug/staging export лежит в [n8n/workflow_export_yandex_tracker.json](/Users/sidorenko/Documents/triage_bot/n8n/workflow_export_yandex_tracker.json).

### Credentials
Нужно создать 4 credentials:

1. `Telegram Bot`
- тип: `Telegram API`
- значение: bot token от BotFather

2. `DeepSeek API`
- тип: тот, который использует ваш узел `DeepSeek Chat Model`
- модель: `deepseek-chat`

3. `Yandex Tracker`
- тип: `HTTP Request -> Custom Auth`
- имя credential: `Yandex Tracker`
- JSON:

```json
{
  "headers": {
    "Authorization": "OAuth <token>",
    "X-Cloud-Org-Id": "<org_id>"
  }
}
```

4. `Google Sheets`
- тип: `Google Sheets OAuth2`
- выбрать spreadsheet и листы через `Select From list`

Важно:
- текущий workflow не зависит от env vars для Tracker routing
- reviewer chat id и staging queue keys зашиты в export и должны быть заменены перед production

---

## 3. Yandex Tracker

### Обязательные очереди
Для production нужны очереди:

| Назначение | Queue key |
|------------|-----------|
| Billing | `BILLING` |
| Engineering Support | `ENGSUP` |
| Account Security | `ACCSEC` |
| Product Team | `PRODUCT` |
| Customer Success | `CSM` |
| General Support | `GENSUP` |
| Human Review | `SUPREV` |

В текущем staging export используются:

| Назначение | Queue key |
|------------|-----------|
| Billing | `TBILLING` |
| Engineering Support | `TENGSUP` |
| Account Security | `TACCSEC` |
| Product Team | `TPRODUCT` |
| Customer Success | `TCSM` |
| General Support | `TGENSUP` |
| Human Review | `TSUPREV` |

### Что делает workflow
- создаёт тикет в целевой очереди при `confidence >= 0.8` и `priority != critical`
- создаёт тикет в review-очереди при `confidence < 0.8` или `priority = critical`
- создаёт тикет в review-очереди при parse/API fallback

Подробный Tracker setup: [n8n/yandex_tracker_setup.md](/Users/sidorenko/Documents/triage_bot/n8n/yandex_tracker_setup.md)

---

## 4. Google Sheets

### Таблица для audit log
1. Создать spreadsheet `Triage Bot Results`
2. Создать лист `Triage Log - Staging`
3. Добавить колонки:

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

4. В n8n выбрать spreadsheet и sheet в каждом `Google Sheets` узле через `Select From list`

### Для eval
Eval workflow можно оставить на отдельной вкладке или отдельном spreadsheet. Он не управляет Tracker routing.

---

## 5. DeepSeek API

1. Зарегистрироваться на [platform.deepseek.com](https://platform.deepseek.com)
2. Создать API key
3. Убедиться, что ключ привязан к модели `deepseek-chat`

Текущий runtime использует:
- модель: `deepseek-chat`
- structured JSON output
- системный prompt из [config/prompts.md](/Users/sidorenko/Documents/triage_bot/config/prompts.md)

---

## 6. Проверка после импорта

Проверьте в n8n:
- `Needs Review?` использует `OR`, а не `AND`
- все 3 Tracker HTTP узла используют credential `Yandex Tracker`
- `Telegram - Notify Reviewer` отправляет во внутренний reviewer-чат
- все `Google Sheets` узлы смотрят на нужный spreadsheet и tab
- в Telegram узлах отключён footer `sent automatically with n8n`

Smoke test: [n8n/yandex_tracker_smoke_test.md](/Users/sidorenko/Documents/triage_bot/n8n/yandex_tracker_smoke_test.md)

---

## Чеклист готовности

- [ ] Telegram bot создан и подключён в n8n
- [ ] Внутренний reviewer-чат создан, bot добавлен, chat id проверен
- [ ] DeepSeek credential настроен
- [ ] Yandex Tracker custom auth credential настроен
- [ ] Очереди Tracker созданы
- [ ] Google Sheets spreadsheet и tab выбраны в workflow
- [ ] Workflow импортирован и протестирован на billing, login и critical кейсах
