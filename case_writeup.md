# AI Triage Bot — Case Write-up

**Role:** AI Product Manager / Automation Lead
**Timeline:** 1 week (solo)
**Stack:** Telegram · n8n · DeepSeek V3 · Yandex Tracker · Google Sheets

---

## Problem

Customer support teams manually read, categorize, prioritize, and route every incoming ticket. At 200+ tickets/day this creates:

- 3-5 minutes of triage before any real work starts
- 15-25% misrouting rate
- inconsistent priority decisions across shifts
- no structured audit trail for improvement

The risk is operational, not just clerical. A security or data-loss issue can sit in the wrong queue while the team handles routine requests.

---

## Solution

An AI triage workflow that processes an incoming Telegram message in about 3 seconds:

1. **Intake** — the customer sends a message to a Telegram bot
2. **Classify** — DeepSeek V3 returns structured JSON: `summary`, `category`, `priority`, `confidence`, `route`
3. **Decide** — n8n applies guardrails: `critical` or `confidence < 0.8` always go to review
4. **Route** — the final ticket is created in `Yandex Tracker`
5. **Log** — the decision trail is written to Google Sheets

**Design principle:** human augmentation, not replacement. The system automates mechanical triage, but keeps a human in the loop for high-risk cases.

---

## Why This Architecture

**Why n8n?**
It is the orchestration and decision layer. The triage logic stays visible, editable, and separate from the ticketing system.

**Why Yandex Tracker?**
It makes routing tangible. You can show the final queue, the review queue, and the operational handoff in a way that looks like a real support setup for the Russian market.

**Why Google Sheets?**
Not as storage of record, but as an audit log. It gives immediate visibility into each decision without adding database infrastructure.

**Why DeepSeek V3?**
Direct access from Russia, low cost, OpenAI-compatible API, and enough quality for a pilot-stage triage workflow.

---

## Results

Canonical v1.2 evaluation on 50 synthetic test cases:

| Metric | Result | vs. Human Baseline |
|--------|--------|--------------------|
| Category accuracy | **92%** | +12pp vs ~80% manual |
| Routing accuracy | **92%** | +12pp |
| Human review trigger | **94%** | reliable guardrail |
| Priority accuracy | **76%** | +30pp through prompt iteration |
| Processing time | **~3 sec** | vs. 3-5 min manual |
| Cost per ticket | **~$0.0001** | vs. ~$0.50-2 labor cost |

After the canonical eval run, five additional regression cases were added to the repository for billing/login boundary checks. The repository benchmark now contains 55 rows aligned to the current policy rules, so a fresh full rerun is still needed before treating the table above as a new live score.

---

## What Changed in v1.2

- tightened `critical` definition for security/data-loss signals
- enforced `feature_request -> low`
- added targeted few-shot examples
- moved the operational destination from a spreadsheet-centric demo to visible queue routing in `Yandex Tracker`

Open issue:
- `complaint` severity still sometimes drifts from `medium` to `high`

---

## What I Learned

**A model can be confidently wrong.**
That is why the review layer exists. Confidence is useful, but not enough for risky automation.

**Operational destination matters.**
Using `Yandex Tracker` instead of only `Google Sheets` made the system much more realistic. It became obvious where the ticket actually lands and why `n8n` exists.

**Eval before rollout matters more than another prompt tweak.**
The next hard evidence now has to come from a controlled pilot and reviewer corrections, not from more speculative prompt changes.

---

## Next Steps

- run a controlled pilot on real messages
- capture reviewer corrections
- recalibrate `complaint` priority
- refresh screenshots and stakeholder materials to match the Tracker-based runtime
