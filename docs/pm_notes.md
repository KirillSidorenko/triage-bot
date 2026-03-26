# PM Notes: Trade-offs, Decisions, Failure Modes

## Почему Customer Support

1. Универсальная и понятная проблема
2. Влияние легко измерить через triage time, routing accuracy и manual touch rate
3. Можно показать полный цикл live
4. Это реальный операционный процесс, а не toy demo
5. Scope можно держать узким и доказуемым

## Почему Telegram

| Фактор | Telegram | Email |
|--------|----------|-------|
| Setup time | Минуты | Часы |
| Real-time | Да | Обычно polling |
| Наглядность | Высокая | Ниже |
| Структура данных | Простой текст | HTML, вложения, threads |

Решение: Telegram для v1, другие каналы позже.

## Почему n8n

- visual workflow и понятная архитектура
- быстрые итерации без отдельного backend
- удобно показывать на интервью и стейкхолдерам
- orchestration layer отделена от конечной ticketing system

## Почему Yandex Tracker

- routing виден прямо в queue UI
- для российского рынка это понятная и реалистичная support system
- не возникает путаницы между operational destination и audit log
- позволяет показать, что `n8n` принимает решение, а Tracker исполняет маршрут

## Почему Google Sheets

Не как ticketing system, а как audit log:
- нулевой setup
- удобно смотреть полный decision trail
- подходит для eval и ручной проверки качества
- легко заменить позже, не трогая triage logic

## Current State

- runtime: `Telegram -> n8n -> DeepSeek V3 -> Yandex Tracker`
- audit trail: `Google Sheets`
- measured quality: category 92%, routing 92%, priority 76%, review trigger 94%
- open gap: complaint severity bias
- next evidence needed: controlled pilot + reviewer feedback capture

## Key Design Decisions

### Confidence threshold = 0.8
- ниже -> выше риск unsafe auto-route
- выше -> слишком много кейсов уходит в ручной review
- 0.8 сейчас рабочая стартовая точка, но должна быть проверена на реальных данных

### Critical всегда уходит на review
- это safety rule, а не удобная эвристика
- цена ошибки для security/data-loss кейсов слишком высока

### 6 категорий
- меньше -> недостаточно полезно для routing
- больше -> падает stability и explainability
- 6 категорий дают баланс между точностью и пользой

### Summary на языке обращения
- снижает риск потерять нюансы
- делает handoff естественнее для команды

## Failure Modes

### 1. Multi-topic messages
Пример: логин не работает и деньги списались дважды.

Текущая стратегия:
- выбирать primary issue
- снижать confidence
- при необходимости уводить в review

### 2. Very short / vague messages
Примеры: `Помогите`, `Не работает`.

Текущий риск:
- модель иногда слишком уверена при слабом контексте

План:
- добавить deterministic review rule

### 3. Emotional complaints
Текущий принцип:
- эмоции не должны автоматически переводить кейс в `complaint`
- если есть операционная проблема, выбираем операционную категорию

### 4. API / parse failure
Текущий fallback:
- тикет создаётся в review queue
- клиент всё равно получает подтверждение
- запись попадает в audit log

### 5. Complaint severity bias
Текущее known issue:
- часть complaint-кейсов переоценивается как `high` вместо `medium`

## Что важно не сломать

- deterministic category -> route mapping
- human review для `critical`
- separation of concerns:
  - n8n -> decision layer
  - Yandex Tracker -> operational execution
  - Google Sheets -> audit trail

## Видение следующего этапа

После pilot проект должен усилиться не новыми интеграциями ради интеграций, а evidence:
- reviewer corrections
- actual review rate
- false positive / false negative patterns
- обновлённый canonical eval run после изменений логики
