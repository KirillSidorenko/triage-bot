# AI Triage Bot — Case Write-up

**Role:** AI Product Manager / Automation Lead
**Timeline:** 1 week (solo)
**Stack:** Telegram · n8n · DeepSeek V3 · Google Sheets

---

## Problem

Customer support teams manually read, categorize, prioritize, and route every incoming ticket. At 200+ tickets/day this means:

- **3–5 min per ticket** spent on triage before any real work begins
- **15–25% misrouting rate** — tickets sent to the wrong team, bounced back, delayed
- **Inconsistency** — priority and category depend on who's on shift
- **No visibility** — no structured data on what's coming in and why

The cost isn't just time. Misrouted critical issues (data loss, security breaches) can sit unnoticed while the team is busy with routine requests.

---

## Solution

An AI triage pipeline that processes every incoming message in ~3 seconds:

1. **Intake** — customer sends a message to a Telegram bot
2. **Classify** — DeepSeek V3 returns structured JSON: summary, category, priority (critical/high/medium/low), confidence score, routing decision
3. **Route** — high-confidence results auto-route; low-confidence and all critical issues go to a human reviewer with AI-generated context
4. **Log** — every decision recorded in Google Sheets with full audit trail

**Key design principle:** Human augmentation, not replacement. The bot never makes the final call on critical issues — it prepares context so a human can decide faster.

---

## Key Decisions

**Why a confidence threshold?**
The system flags cases for human review when confidence < 0.8. This keeps the automation rate high (~85% of tickets) while ensuring a human catches ambiguous cases. Critical priority always triggers review regardless of confidence.

**Why DeepSeek V3?**
Direct API access from Russia, OpenAI-compatible interface, ~10x cheaper than GPT-4o ($0.028/1M tokens). Classification of one ticket costs ~$0.0001.

**Why Google Sheets as storage?**
Zero infrastructure overhead. The ops team already uses it. A Postgres database would add complexity without adding value at this stage. Sheets can be replaced in v2 without touching the classification logic.

**Why n8n over custom code?**
Faster to build, easier to hand off. A non-technical ops manager can read the workflow, understand it, and modify routing rules without a developer. Reduces dependency on engineering.

---

## Results

Evaluation on 50 synthetic test cases (balanced across 6 categories, RU/EN, including edge cases):

| Metric | Result | vs. Human Baseline |
|--------|--------|--------------------|
| Category accuracy | **92%** | +12pp vs ~80% manual |
| Routing accuracy | **92%** | Tied to category |
| Human review trigger | **90%** | All critical cases caught |
| Processing time | **~3 sec** | vs. 3–5 min manual |
| Cost per ticket | **~$0.0001** | vs. ~$0.50–2 labor cost |

**Known issue:** Priority accuracy at 46% — the model is biased toward higher severity. Feature requests and general inquiries get rated "medium" instead of "low". Root cause identified (prompt definition of "low" is too vague), fix ready for v1.1.

---

## What I Learned

**On AI behavior:** A model can be confidently wrong. 4 of 50 classification errors happened with confidence ≥ 0.9. This is why the human review layer exists — not just for uncertain cases, but as a backstop for high-stakes misclassifications.

**On scope:** The instinct is to build more. Auto-responses, email integration, a dashboard. I deliberately didn't. Getting one channel working correctly and evaluated is more valuable than three channels working unreliably.

**On PM process applied to AI:** Traditional PM artifacts (PRD, routing rules, success metrics) proved more valuable here than in typical software projects — because AI systems fail in non-obvious ways. Having defined categories upfront prevented the model from inventing its own taxonomy. Having an eval set caught the priority bias before it reached users.

---

## Next Steps (v1.1 → v2)

**v1.1 (prompt tuning):**
- Fix priority calibration — explicit `low` examples in system prompt
- Add rule: message length < 20 chars → auto-flag for review

**v2 (multi-channel):**
- Email intake via Gmail/Outlook integration
- Auto-responses for `general_inquiry` (answer from knowledge base)
- Reviewer feedback loop → update few-shot examples monthly

**v3 (intelligence layer):**
- Detect duplicate/related tickets
- Predict resolution time based on historical data
- Route to specific agent, not just team
