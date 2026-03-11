# AI Triage Bot

Автоматический триаж обращений в customer support — классификация, приоритизация и маршрутизация за ~3 секунды.

**[→ Кейс на GitHub Pages](https://KirillSidorenko.github.io/triage-bot)**

---

## Результаты

| Метрика | Результат | vs. ручной триаж |
|---------|-----------|-----------------|
| Точность категоризации | **92%** | +12pp от ~80% |
| Точность маршрутизации | **92%** | +12pp |
| Триггер human review | **90%** | — |
| Время обработки | **~3 сек** | vs. 3–5 мин |
| Стоимость тикета | **$0.0001** | vs. ~$0.50–2 |

*Оценка на 50 синтетических кейсах: 6 категорий, RU/EN, включая edge cases*

---

## Как работает

```
Telegram → n8n → DeepSeek V3 → Google Sheets
              ↓
   confidence ≥ 0.8 → авто-маршрут
   confidence < 0.8 или critical → human review
```

1. Пользователь отправляет сообщение в Telegram-бот
2. n8n передаёт текст в DeepSeek V3 со структурированным промптом
3. Модель возвращает JSON: `summary`, `category`, `priority`, `confidence`, `route`
4. При confidence ≥ 0.8 — тикет уходит автоматически
5. При confidence < 0.8 или priority = critical — уведомление ревьюеру с AI-контекстом
6. Всё логируется в Google Sheets

---

## Стек

| Компонент | Технология |
|-----------|------------|
| Intake | Telegram Bot API |
| Оркестрация | n8n (self-hosted) |
| AI | DeepSeek V3 (`deepseek-chat`) |
| Хранилище | Google Sheets |

---

## Категории

| Категория | Описание | Маршрут |
|-----------|----------|---------|
| `billing` | Оплата, счета, подписки, возвраты | Billing Team |
| `technical_issue` | Баги, ошибки, сломанный функционал | Engineering Support |
| `account_access` | Вход, пароль, 2FA, блокировка | Account Security |
| `feature_request` | Запросы на новый функционал | Product Team |
| `complaint` | Жалобы на качество сервиса | Customer Success |
| `general_inquiry` | Общие вопросы, документация | General Support |

---

## Структура проекта

```
triage_bot/
├── docs/
│   ├── prd.md                 # Product Requirements Document
│   ├── roadmap.md             # Now / Next / Later
│   ├── pm_notes.md            # Trade-offs и решения
│   ├── process_as_is.md       # Текущий процесс (as-is)
│   ├── process_to_be.md       # Целевой процесс (to-be)
│   └── workflow_design.md     # Схема n8n workflow
├── eval/
│   ├── test_cases.csv         # 50 тест-кейсов
│   ├── eval_results.csv       # Результаты прогона
│   └── eval_report.md         # Анализ качества
├── config/
│   ├── categories.json        # Категории и приоритеты
│   ├── routing_rules.json     # Правила маршрутизации
│   └── prompts.md             # Промпты для LLM
├── n8n/
│   ├── workflow_export.json   # Основной workflow
│   └── eval_workflow.json     # Eval runner workflow
├── demo/
│   └── demo_script.md         # Сценарий демонстрации
├── index.html                 # GitHub Pages кейс
└── case_writeup.md            # 1-страничный write-up
```

---

## Запуск

### Требования
- n8n (self-hosted или cloud)
- Telegram Bot Token ([@BotFather](https://t.me/BotFather))
- DeepSeek API Key ([platform.deepseek.com](https://platform.deepseek.com))
- Google Sheets с OAuth2

### Установка

1. Клонировать репозиторий
2. Импортировать `n8n/workflow_export.json` в n8n
3. Создать credentials в n8n:
   - **Telegram API** — bot token от BotFather
   - **DeepSeek API** — ключ с platform.deepseek.com
   - **Google Sheets OAuth2** — авторизация через Google
4. Создать Google Sheets таблицу с листами `Triage Log` и `Eval Results`
5. Привязать credentials к нодам, активировать workflow

Подробная инструкция: [`docs/setup_guide.md`](docs/setup_guide.md)

---

## Статус

- [x] Discovery и проектирование процесса
- [x] PM артефакты (PRD, roadmap, process docs)
- [x] n8n workflow (build + деплой)
- [x] Evaluation (50 кейсов, eval report, dashboard)
- [x] Demo script + case write-up
- [ ] v1.1 — фикс калибровки приоритетов
