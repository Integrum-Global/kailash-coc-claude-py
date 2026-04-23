---
name: cli-audit
description: "Multi-CLI parity + drift audit. Dispatches 3 architects in parallel; runs emitter validators + cross-CLI drift check."
---

# Multi-CLI Audit

Dispatch entry point for `cli-orchestrator.audits` per spec v6 §6.2. Reviews the full multi-CLI emission pipeline (source rules → slot overlays → abridgement → per-CLI target files) for parity, drift, and cap compliance across CC, Codex, and Gemini.

## Your Role

Specify scope: `all` (default), `emission`, `parity`, `drift`, or `validators`.

## Dispatch Contract (v6 §6.2)

You MUST dispatch `cc-architect`, `codex-architect`, and `gemini-architect` via the Task tool in the SAME TURN with `run_in_background: true` (parallel launch per `rules/agents.md` § Parallel Execution). Sequential dispatch is BLOCKED — it bypasses the parallel-execution multiplier and re-runs the same audit three times in series.

```
# DO — single turn, three parallel architects
Agent(subagent_type="cc-architect", run_in_background=true, prompt="...CC audit brief with emission report...")
Agent(subagent_type="codex-architect", run_in_background=true, prompt="...Codex audit brief with AGENTS.md...")
Agent(subagent_type="gemini-architect", run_in_background=true, prompt="...Gemini audit brief with GEMINI.md...")

# DO NOT — sequential
Agent(subagent_type="cc-architect", ...)     # wait for return
Agent(subagent_type="codex-architect", ...)  # then this
```

## Phase 1: Produce the emission (dry-run)

Run the E4 emitter in dry-run mode to produce per-CLI baseline emissions the architects will audit:

```bash
node .claude/bin/emit.mjs --all --out /tmp/cli-audit-$(date +%s) -v
```

This writes:

- `/tmp/cli-audit-<ts>/codex/AGENTS.md` + `emit-report-codex.json`
- `/tmp/cli-audit-<ts>/gemini/GEMINI.md` + `emit-report-gemini.json`
- `/tmp/cli-audit-<ts>/codex-mcp-guard/policies.json` (V13 POLICIES table)

Exit code ≠ 0 means V12 slot-round-trip failed, V13 MCP bijection failed, or the emission exceeded `block_cap_bytes`. A non-zero exit is a HARD BLOCK on this audit — fix before dispatching architects.

## Phase 2: Parallel architect dispatch

For each architect, the brief includes:

- the emission target file it owns (`AGENTS.md`, `GEMINI.md`, or `.claude/**` source)
- the `emit-report-<cli>.json` for its CLI
- the `cli_variants` + `parity_enforcement` sub-sections of `.claude/sync-manifest.yaml`
- the expected parity contract from `.claude/rules/cross-cli-parity.md`

Each architect returns a structured JSON report enumerating findings in its ownership tree.

## Phase 3: cli-orchestrator.sees — cross-CLI drift

Independent of the architects (which each see only their own CLI), run the `sees` verb to check for drift ACROSS CLIs per `parity_enforcement.cross_cli_drift_audit`:

1. Load `.claude/sync-manifest.yaml → parity_enforcement.cross_cli_drift_audit`.
2. For each CRIT rule, compose the neutral-body slot under each CLI (CC, codex, gemini); verify byte-identity after scrub_tokens normalization.
3. For each CRIT rule, compose the examples slot under each CLI; soft-WARN on drift (expected divergence) per `warn_on_drift_in_slots: ["examples"]`.
4. For `frontmatter.priority` and `frontmatter.scope`, verify byte-identity (hard block on mismatch).

Drift in `neutral-body`, `frontmatter.priority`, or `frontmatter.scope` HARD BLOCKS sync. Drift in `examples` is expected per-CLI divergence (scrubbed via `scrub_tokens: ["Agent(", "codex_agent(", "@specialist", "subagent_type", "run_in_background"]`).

## Phase 4: Aggregate + report

Combine architect findings + drift-audit result into a single report with severity taxonomy:

- **CRITICAL** — V12 slot round-trip failure, V13 MCP bijection failure, `block_cap_bytes` exceeded, `neutral-body` drift, `frontmatter.priority|scope` drift, overlay introduces a slot not in global.
- **HIGH** — V13 POLICIES bijection spurious/missing entry, per-rule budget exceeds `+30%` tolerance, `warn_cap_bytes` exceeded.
- **NOTE** — expected `examples` slot drift, per-rule budget within tolerance but trending up, orchestrator filter applied (e.g. `main` in `validate-prod-deploy.js`).

Run iteratively until zero CRITICAL and zero HIGH remain. Each iteration MUST re-derive the emission + re-dispatch the three architects (parallel) + re-run `sees`. Do NOT trust a prior turn's verdict — the audit's strength is its repeatability.

## References

- `.claude/agents/cli-orchestrator.md` — the 5 verbs; `/cli-audit` dispatches `audits` + `sees`
- `.claude/agents/{cc,codex,gemini}-architect.md` — parallel audit targets
- `.claude/bin/emit.mjs` — Phase E4 emitter (V12 + V13 built-in)
- `.claude/sync-manifest.yaml` → `cli_variants` + `parity_enforcement` — emission + audit config
- `.claude/rules/cross-cli-parity.md` — parity contract source of truth
- `workspaces/multi-cli-coc/02-plans/07-loom-multi-cli-spec-v6.md` §4.4 + §6.2 — authoritative dispatch contract
