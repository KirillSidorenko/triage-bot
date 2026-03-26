# Customer Support: целевой процесс (To-Be)

## Обзор

AI-assisted triage: автоматическая классификация, приоритизация и маршрутизация обращений через `n8n`, с созданием тикета в `Yandex Tracker` и human-in-the-loop для low-confidence и critical кейсов.

## Целевой процесс

### 1. Получение обращения
- Клиент пишет в Telegram-бот
- Сообщение попадает в n8n workflow

### 2. AI-триаж (~3 секунды)
- n8n нормализует текст, sender и timestamp
- DeepSeek V3 возвращает:
  - `summary`
  - `category`
  - `priority`
  - `confidence`
  - `route`

### 3. Guardrails
- `confidence >= 0.8` и `priority != critical` -> авто-маршрут
- `confidence < 0.8` или `priority = critical` -> ручная проверка

### 4. Operational routing
- На auto-route ветке создаётся тикет в целевой очереди `Yandex Tracker`
- На review ветке создаётся тикет в review-очереди
- Ревьюер получает отдельное внутреннее Telegram-уведомление

### 5. Audit trail
- Полный след решения записывается в Google Sheets
- Там же видны `tracker_issue_key`, `tracker_queue`, `tracker_issue_url`

### 6. Ответ клиенту
- Клиент получает короткое подтверждение
- Внутренние поля triage не показываются клиенту

## Что меняется для команды

- L1 больше не тратит время на механический triage
- Команда получает тикет уже в рабочей очереди `Yandex Tracker`
- Ревьюер видит отдельный поток только по ambiguous/critical обращениям
- Audit trail остаётся доступным в Google Sheets для анализа и eval

## Целевые метрики

| Метрика | As-Is | To-Be | Улучшение |
|---------|-------|-------|-----------|
| Время triage | 5 мин | ~3 сек | ~100x |
| Routing accuracy | 75-85% | >=90% | +10-15pp |
| Manual touch rate | 100% | <25% | -75pp |
| Consistency | ~70% | ~95% | +25pp |
| First response time | 2-4 часа | <1 мин | ~100x |

## Ограничения текущей версии

- `complaint` priority всё ещё требует калибровки
- reviewer feedback loop ещё не замкнут
- production rollout без pilot пока не обоснован
