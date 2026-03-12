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
| Human review trigger | **94%** | All critical cases caught |
| Priority accuracy | **76%** | +30pp through prompt iteration |
| Processing time | **~3 sec** | vs. 3–5 min manual |
| Cost per ticket | **~$0.0001** | vs. ~$0.50–2 labor cost |

*Results from v1.2 eval run. 50 synthetic test cases across 6 categories, RU/EN, including edge cases.*

**Prompt iteration:** Priority accuracy improved from 46% → 76% over two iterations. v1.1 tightened the `critical` definition (added explicit security triggers). v1.2 added a hard rule for `feature_request → always low` with supporting few-shot examples. Feature_request priority accuracy: 0% → 86%. Open issue: complaint category still biased toward higher severity (v1.3 candidate).

---

## What I Learned

**On AI behavior:** A model can be confidently wrong. 4 of 50 classification errors happened with confidence ≥ 0.9. This is why the human review layer exists — not just for uncertain cases, but as a backstop for high-stakes misclassifications.

**On scope:** The instinct is to build more. Auto-responses, email integration, a dashboard. I deliberately didn't. Getting one channel working correctly and evaluated is more valuable than three channels working unreliably.

**On PM process applied to AI:** Traditional PM artifacts (PRD, routing rules, success metrics) proved more valuable here than in typical software projects — because AI systems fail in non-obvious ways. Having defined categories upfront prevented the model from inventing its own taxonomy. Having an eval set caught the priority bias before it reached users.

---

## Next Steps

**Controlled pilot:**
- Run with real messages from a small team or friendly business
- Capture reviewer decisions to close the feedback loop
- Get one testimonial from a real reviewer

**v2 (multi-channel):**
- Email intake via Gmail/Outlook integration
- Auto-responses for `general_inquiry` (answer from knowledge base)
- Reviewer feedback loop → update few-shot examples monthly

**v3 (intelligence layer):**
- Detect duplicate/related tickets
- Predict resolution time based on historical data
- Route to specific agent, not just team
