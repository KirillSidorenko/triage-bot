# Roadmap: AI Triage Bot

## Now (v1.2 — built and evaluated)

**Цель:** рабочий end-to-end triage workflow с visible routing, evaluation и human review guardrail.

- [x] Discovery: as-is процесс, категории, маршруты
- [x] PM artifacts: PRD, roadmap, PM notes
- [x] Telegram bot setup
- [x] n8n workflow: intake -> classify -> route -> audit log
- [x] DeepSeek V3 prompt engineering + structured output
- [x] Confidence threshold + human review trigger
- [x] Yandex Tracker integration как operational ticketing layer
- [x] Google Sheets integration как audit log
- [x] Evaluation set и canonical eval run
- [x] Demo script + case write-up
- [x] v1.1 critical detection fix
- [x] v1.2 priority calibration (`feature_request -> low`, few-shot examples)

**Текущее измеренное состояние:** последний canonical run = category 92%, routing 92%, priority 76%, review trigger 94%, latency ~3 сек. Рабочий benchmark-набор уже расширен до 55 строк; свежий full rerun pending.

## Next (v1.3 — pilot hardening)

**Цель:** довести workflow до controlled pilot на реальных сообщениях.

- [ ] Controlled pilot на реальных сообщениях
- [ ] Reviewer feedback loop: `reviewer_decision` / `final_category`
- [ ] Complaint priority calibration (`medium` vs `high`)
- [ ] Deterministic rule для very short / vague messages
- [ ] Shadow mode vs manual triage
- [ ] Full rerun after benchmark sync / next logic change
- [ ] Обновлённые demo screenshots из Tracker-версии workflow

## Later (v2+)

**Цель:** расширение каналов, аналитики и производственной зрелости.

- Multi-channel intake: email, web form, chat widgets
- Additional ticket systems beyond Yandex Tracker
- Knowledge base integration и suggested replies
- Agent assist для L2
- SLA monitoring и alerting
- Analytics dashboard с трендами и аномалиями
- Predictive routing с учётом нагрузки команд
- Custom model benchmark / possible fine-tuning after pilot
