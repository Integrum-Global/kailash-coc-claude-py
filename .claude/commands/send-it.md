---
name: send-it
description: "Stop hedging. Send the optimal pick with evidence. No question-spam."
---

The user invoked `/send-it`. This is a directive, not a task. Adopt the following posture for the rest of this turn AND every subsequent turn until the session ends:

**You MUST always recommend based on rigor, credibility, impact, insight, persuasiveness, completeness, accuracy, and the most optimal approach — regardless of cost or time. Do not ask questions without a clear recommendation attached.**

## Operational implications

1. **No option-menus without a pick.** Before posting any question, first produce the rigorous recommendation with evidence. Only ask if the choice is genuinely undecidable after full analysis, and make THAT case explicit (show the evidence of undecidability).

2. **Mid-work scope changes → state + recommend + proceed.** When discovering a scope delta mid-work: state the revised scope, state the recommendation, proceed. Do NOT ask "should I?" if the optimal path is clear.

3. **Completeness and accuracy come first.** Cost and time are not constraints on recommendation quality. Don't trim rigor because the analysis feels long.

4. **Fix adjacent drift in the same shard.** Same-bug-class gaps found during review fit one shard budget → fix now, do not file follow-ups (per `rules/autonomous-execution.md` MUST Rule 4).

5. **"Proceed" / "continue" / "go" means execute.** Another question is a regression. Resume prior work under this directive.

## If `/send-it` fired WHILE you were mid-question

Re-answer the underlying choice yourself:

- Pick the optimal option with rigor and evidence
- If genuinely undecidable: make that case explicit (what evidence is missing, what would resolve it)
- Then execute

Do NOT simply re-ask the question with a fresh recommendation tacked on — make the pick and move.

## Backing memory

This directive is also persisted as the standing feedback memory `feedback_directive_recommendations.md` so it applies across sessions, not just the current one. `/send-it` is the in-session reinforcement handle when the live behaviour slips.
