# AGENTS.md - Codex Operating Instructions for `triage_bot`

Purpose: project-specific guidance for Codex when working in this repository.

## Project Snapshot (as of March 16, 2026)
- Current triage prompt/runtime: `deepseek-chat` via n8n HTTP node.
- Status: workflow is pilot-ready with human review gate.
- Key known gap: `complaint` priority can still overfire `high` vs `medium`.
- Human-in-the-loop rule: `priority=critical` always requires human review.

## Codex Role
Codex acts as a pragmatic implementation partner for this support triage system.

Primary goals:
1. Keep classification/routing behavior stable and explainable.
2. Improve precision without increasing critical-risk automation.
3. Preserve reproducibility of demo/eval artifacts.

## Source of Truth
Use these files first before proposing or changing behavior:
- `README.md`
- `config/prompts.md`
- `config/routing_rules.json`
- `eval/test_cases.csv`
- `eval/eval_report.md`
- `n8n/workflow_export.json`

If these conflict, trust runtime behavior in `n8n/workflow_export.json`, then align docs.

## Repository Map
- `config/` - model prompt, categories, routing/policy rules.
- `n8n/` - exported production/eval workflows.
- `eval/` - benchmark set and quality report.
- `docs/` - PRD, process, design, roadmap.
- `demo/` - screenshots and demo materials.

## Change Policy
When editing triage logic:
1. Update prompt/policy in `config/` first.
2. Mirror the same logic in `n8n/workflow_export.json`.
3. Update eval artifacts/docs if expected behavior changed.

Avoid partial updates where prompt/routing/eval disagree.

## Quality Gates (minimum)
For any logic change, verify:
1. JSON schema output still includes: `summary`, `category`, `priority`, `confidence`, `route`.
2. Category-to-route mapping remains one-to-one and deterministic.
3. `feature_request` and `general_inquiry` remain `low` unless policy intentionally changes.
4. `critical` remains forced to human review regardless of confidence.
5. Fallback route remains valid (`General Support` / `#support-general`).

## Evaluation Expectations
- Prefer fast local checks first (targeted rows in `eval/test_cases.csv`).
- If behavior changes materially, rerun full eval workflow and summarize deltas by:
  - category accuracy
  - priority accuracy
  - routing accuracy
  - human-review trigger precision
- Explicitly call out regressions, especially for `account_access` and `critical` signals.

## Safety / Risk Rules
- Do not remove human review for low-confidence or critical tickets.
- Do not silently weaken security incident detection patterns.
- Do not claim quality improvement without referencing measurable eval deltas.
- Keep routing deterministic; avoid probabilistic/LLM-only route post-processing.

## Prompt Editing Guidelines
- Keep instructions concrete and conflict-free.
- Use explicit boundary rules for ambiguous classes:
  - broken existing behavior -> `technical_issue`
  - missing capability request -> `feature_request`
- Add few-shot examples only when they fix a known error mode from eval.
- Prefer minimal prompt diffs over full rewrites.

## n8n Editing Guidelines
- Preserve node names and structure unless refactor is required.
- If node contracts change, update dependent expressions in the same change.
- Keep exported JSON valid and re-importable.
- Document new env vars or credentials in setup docs.

## Demo/Stakeholder Readability
For stakeholder-facing outputs:
1. Show before/after behavior on concrete ticket examples.
2. Include confidence and human-review reason.
3. Include final destination (team + channel + escalation owner).
4. Keep Russian and English examples both represented.

## Preferred Working Style
- Make the smallest safe change that solves the issue.
- Surface assumptions explicitly when data is missing.
- Prioritize reproducible improvements over clever but brittle heuristics.

