# AI Triage Bot

Автоматический triage обращений в customer support: классификация, приоритизация и маршрутизация за ~3 секунды с созданием тикета в `Yandex Tracker`.

**[→ Кейс на GitHub Pages](https://KirillSidorenko.github.io/triage-bot)**

---

## Результаты

| Метрика | Результат | vs. ручной триаж |
|---------|-----------|-----------------|
| Точность категоризации | **92%** | +12pp от ~80% |
| Точность маршрутизации | **92%** | +12pp |
| Триггер human review | **94%** | — |
| Точность приоритета | **76%** | +30pp за 2 итерации промпт-инжиниринга |
| Время обработки | **~3 сек** | vs. 3–5 мин |
| Стоимость тикета | **$0.0001** | vs. ~$0.50–2 |

*Последний зафиксированный canonical v1.2 eval run от 2026-03-12 · 50 синтетических кейсов · 6 категорий · RU/EN · включая edge cases*

Рабочий benchmark-набор в `eval/test_cases.csv` сейчас содержит 55 строк: 50 кейсов из канонического eval run и 5 дополнительных regression-кейсов для billing/login boundary checks. Expectations в benchmark-наборе синхронизированы с текущими prompt/routing rules; для нового live-score нужен отдельный полный rerun.

**Текущий статус:** workflow готов к `controlled pilot` с human review. Главный открытый issue — `complaint` priority всё ещё местами смещается из `medium` в `high`, поэтому это пока не кейс для fully autonomous rollout.

---

## Как работает

```
Telegram → n8n → DeepSeek V3 → Yandex Tracker
              │
              └→ Google Sheets (audit log)

confidence ≥ 0.8 и not critical → авто-маршрут в целевую очередь
confidence < 0.8 или critical → очередь review + уведомление ревьюеру
```

1. Пользователь отправляет сообщение в Telegram-бот
2. n8n передаёт текст в DeepSeek V3 со структурированным промптом
3. Модель возвращает JSON: `summary`, `category`, `priority`, `confidence`, `route`
4. n8n применяет guardrails: `priority = critical` и `confidence < 0.8` всегда отправляют обращение на ручную проверку
5. При confidence ≥ 0.8 и not critical — тикет создаётся в целевой очереди `Yandex Tracker`
6. При confidence < 0.8 или `priority = critical` — тикет создаётся в review-очереди, а ревьюер получает отдельное уведомление в Telegram
7. Решение и маршрут логируются в Google Sheets как audit trail

---

## Стек

| Компонент | Технология |
|-----------|------------|
| Intake | Telegram Bot API |
| Оркестрация | n8n (self-hosted) |
| AI | DeepSeek V3 (`deepseek-chat`) |
| Ticketing | Yandex Tracker |
| Audit log | Google Sheets |

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

Для текущего staging workflow в `Yandex Tracker` используются очереди:

| Категория | Очередь |
|-----------|---------|
| `billing` | `TBILLING` |
| `technical_issue` | `TENGSUP` |
| `account_access` | `TACCSEC` |
| `feature_request` | `TPRODUCT` |
| `complaint` | `TCSM` |
| `general_inquiry` | `TGENSUP` |
| review | `TSUPREV` |

---

## Структура проекта

```
triage_bot/
├── AGENTS.md                 # Инструкции для Codex
├── docs/
│   ├── prd.md                 # Product Requirements Document
│   ├── roadmap.md             # Now / Next / Later
│   ├── pm_notes.md            # Trade-offs и решения
│   ├── process_as_is.md       # Текущий процесс (as-is)
│   ├── process_to_be.md       # Целевой процесс (to-be)
│   └── workflow_design.md     # Схема n8n workflow
├── eval/
│   ├── test_cases.csv         # 55 кейсов: 50 canonical + 5 regression additions
│   └── eval_report.md         # Анализ качества
├── config/
│   ├── categories.json        # Категории и приоритеты
│   ├── routing_rules.json     # Правила маршрутизации
│   ├── prompts.md             # Промпты для LLM
│   └── yandex_tracker.json    # Mapping категорий на очереди Tracker
├── n8n/
│   ├── workflow_export.json   # Основной workflow (Yandex Tracker)
│   ├── workflow_export_yandex_tracker.json  # Отдельный export для staging/debug
│   ├── yandex_tracker_setup.md # Setup Tracker + creds + queues
│   ├── yandex_tracker_smoke_test.md # Smoke-test checklist
│   └── eval_workflow.json     # Eval runner workflow
├── demo/
│   ├── demo_script.md         # Базовый сценарий демонстрации
│   └── yandex_tracker_demo_script.md # Demo routing через Tracker
├── index.html                 # GitHub Pages кейс
└── case_writeup.md            # 1-страничный write-up
```

---

## Запуск

### Требования
- n8n (self-hosted или cloud)
- Telegram Bot Token ([@BotFather](https://t.me/BotFather))
- DeepSeek API Key ([platform.deepseek.com](https://platform.deepseek.com))
- Yandex Tracker OAuth token + `X-Cloud-Org-Id`
- Google Sheets с OAuth2

### Установка

1. Клонировать репозиторий
2. Импортировать `n8n/workflow_export.json` в n8n
3. Создать credentials в n8n:
   - **Telegram API** — bot token от BotFather
   - **DeepSeek API** — ключ с platform.deepseek.com
   - **Yandex Tracker** — `HTTP Request -> Custom Auth` с заголовками:
     - `Authorization: OAuth <token>`
     - `X-Cloud-Org-Id: <org_id>`
   - **Google Sheets OAuth2** — авторизация через Google
4. Подготовить очереди в `Yandex Tracker` и review-чат для уведомлений
5. Создать или выбрать Google Sheets таблицу для лога triage
6. Привязать credentials к нодам и активировать workflow

Важно:
- текущий `workflow_export.json` синхронизирован с протестированным Tracker-вариантом
- в export по умолчанию зашиты staging-очереди и внутренний reviewer chat
- перед production запуском замените staging queue keys и reviewer chat id

Подробные инструкции:
- [`docs/setup_guide.md`](docs/setup_guide.md)
- [`n8n/yandex_tracker_setup.md`](n8n/yandex_tracker_setup.md)
- [`n8n/yandex_tracker_smoke_test.md`](n8n/yandex_tracker_smoke_test.md)

---

## Статус

- [x] Discovery и проектирование процесса
- [x] PM артефакты (PRD, roadmap, process docs)
- [x] n8n workflow (build + деплой)
- [x] Evaluation (canonical run: 50 кейсов, eval report, dashboard)
- [x] Demo script + case write-up
- [x] v1.1 — critical detection fix (security incidents)
- [x] v1.2 — priority calibration (46% → 76% за 2 итерации)
- [x] Yandex Tracker integration + visible queue routing
- [ ] Controlled pilot с реальными сообщениями
- [ ] Reviewer feedback loop (логирование решений ревьюера)
