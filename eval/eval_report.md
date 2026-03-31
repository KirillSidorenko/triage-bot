# Eval Report — Triage Bot v1.2

**Дата:** 2026-03-12
**Модель:** DeepSeek V3 (`deepseek-chat`)
**Eval set:** 50 synthetic test cases
**Языки:** RU / EN
**Покрытие:** 6 категорий, 4 приоритета, edge cases: critical, multi-topic, very short, vague, ambiguous

Важно: ниже — исторический отчёт по каноническому прогону от 2026-03-12. По состоянию на 2026-03-31 рабочий benchmark-набор в репозитории синхронизирован с текущими policy rules и расширен до 55 строк; свежий полный rerun после этой синхронизации ещё не выполнен.

Дополнение: в [eval/test_cases.csv](/Users/sidorenko/Documents/triage_bot/eval/test_cases.csv) сейчас 55 строк. Кейсы `51-55` были добавлены позже как regression checks и не входят в проценты этого канонического v1.2 run.

---

## История итераций

| Версия | Что менялось | Category | Priority | Route | Needs Review |
|--------|--------------|----------|----------|-------|--------------|
| v1.0 | Базовый structured prompt | 92% | 46% | 92% | 90% |
| v1.1 | Уточнено определение `critical` | 92% | 48% | 92% | 92% |
| v1.2 | `feature_request → always low` + targeted few-shot examples | **92%** | **76%** | **92%** | **94%** |

**Главный вывод по итерациям:** category и routing были сильными уже в v1.0. Основная работа шла по priority calibration. За две итерации prompt/rules priority accuracy вырос с 46% до 76%.

---

## Текущие результаты v1.2

| Метрика | Результат | Интерпретация |
|---------|-----------|---------------|
| Category accuracy | **92%** | Достаточно для pilot gate |
| Routing accuracy | **92%** | Сильный результат для single-workflow triage |
| Priority accuracy | **76%** | Приемлемо для pilot с human review, но не для full auto |
| Needs review accuracy | **94%** | Review gate работает надежно |
| Processing time | **~3 сек** | Существенно быстрее ручного triage |

---

## Что улучшили в v1.2

### 1. Priority перестал завышаться для feature requests

В v1.0 модель почти не использовала `low` и системно завышала priority для `feature_request`.

Что изменили:
- в definition закрепили правило `feature_request → always low`
- добавили few-shot examples для urgent / blocking feature request
- усилили distinction между broken existing feature и missing feature

Результат:
- `feature_request` priority accuracy вырос с `0%` до `86%`
- overall priority accuracy вырос на `28pp`

### 2. Critical detection стала устойчивее

В v1.1 уточнили definition `critical`:
- hacking
- passwords leaked
- account compromised
- database lost / empty
- security incident

Это не изменило category accuracy, но улучшило consistency для critical handling и `needs_review`.

---

## Что работает хорошо

### Category / route

- Модель стабильно различает основные operational classes:
  - billing
  - technical_issue
  - account_access
  - feature_request
  - complaint
  - general_inquiry
- Routing accuracy совпадает с category accuracy, потому что route определяется категорией.
- Для четко сформулированных кейсов качество уже выглядит pilot-ready.

### Human-in-the-loop gate

- `priority = critical` всегда отправляет кейс на review
- `confidence < 0.8` отрабатывает как safety layer
- `needs_review` accuracy = 94% достаточно, чтобы использовать workflow как assisted triage, а не full automation

### Time-to-value

- End-to-end processing time около `3 секунд`
- Это делает кейс убедительным как AI workflow automation proof asset, а не только как LLM demo

---

## Открытая проблема

### Complaint priority bias

Оставшийся главный quality gap: модель все еще склонна завышать severity для части `complaint` кейсов.

Типичный паттерн:
- expected `medium`
- actual `high`

Что это значит practically:
- кейс почти не ломает routing
- но завышает urgency
- значит workflow безопаснее запускать в pilot-режиме с human review, а не как fully autonomous triage

---

## Рекомендации на v1.3

### P0

1. **Complaint priority calibration**
   - добавить explicit definition для `complaint: medium by default`
   - оставить `high` только для жалоб с прямым operational impact
   - добавить targeted few-shot examples

2. **Canonical export of eval artifacts**
   - сохранять row-level `eval_results.csv` после каждого run
   - это упростит audit trail и сравнение версий

### P1

3. **Deterministic rule для very short / vague messages**
   - если сообщение слишком короткое или явно ambiguous, принудительно ставить `needs_review = true`

4. **Controlled pilot**
   - прогнать workflow на реальных сообщениях
   - собрать reviewer corrections
   - сравнить AI output vs manual triage

### P2

5. **Reviewer feedback loop**
   - логировать `reviewer_decision` и `final_category`
   - использовать corrections как источник для следующих prompt iterations

---

## Go / No-Go

**Решение:** `Go for controlled pilot`

Почему:
- category и route уже на уровне, достаточном для pilot
- review gate надежный
- known issue по priority локализован и не скрыт
- workflow не пытается отвечать клиенту от лица компании, а решает более безопасную задачу triage

**Не go for full autonomous rollout**

Почему:
- priority calibration еще не достаточно стабильна
- reviewer feedback loop пока не замкнут
- нет evidence на реальных production-like сообщениях

---

## Итог

`v1.2` уже доказывает, что workflow работает как AI-assisted triage system:
- быстро
- понятно
- с измеримым качеством
- с безопасным human-in-the-loop path

Следующий шаг для усиления проекта уже не в “еще одном prompt tweak”, а в `controlled pilot + reviewer feedback capture`.
