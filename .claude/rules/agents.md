---
priority: 0
scope: baseline
---

# Agent Orchestration Rules

<!-- slot:neutral-body -->

## Specialist Delegation (MUST)

When working with Kailash frameworks, MUST consult the relevant specialist:

- **dataflow-specialist**: Database or DataFlow work
- **nexus-specialist**: API or deployment work
- **kaizen-specialist**: AI agent work
- **mcp-specialist**: MCP integration work
- **mcp-platform-specialist**: FastMCP platform server, contributor plugins, security tiers
- **pact-specialist**: Organizational governance work
- **ml-specialist**: ML lifecycle, feature stores, training, drift monitoring, AutoML
- **align-specialist**: LLM fine-tuning, LoRA adapters, alignment methods, model serving

**Applies when**: Creating workflows, modifying DB models, setting up endpoints, building agents, implementing governance, training ML models, fine-tuning LLMs, configuring MCP platform server.

**Why:** Framework specialists encode hard-won patterns and constraints that generalist agents miss, leading to subtle misuse of DataFlow, Nexus, or Kaizen APIs.

## Specs Context in Delegation (MUST)

Every specialist delegation prompt MUST include relevant spec file content from `specs/`. Read `specs/_index.md`, select relevant files, include them inline. See `rules/specs-authority.md` MUST Rule 7 for the full protocol and examples.

**Why:** Specialists without domain context produce technically correct but intent-misaligned output (e.g., schemas without tenant_id because multi-tenancy wasn't communicated).

## Analysis Chain (Complex Features)

1. **analyst** → Identify failure points
2. **analyst** → Break down requirements
3. **`decide-framework` skill** → Choose approach
4. Then appropriate specialist

**Applies when**: Feature spans multiple files, unclear requirements, multiple valid approaches.

## Parallel Execution

When multiple independent operations are needed, launch agents in parallel using Task tool, wait for all, aggregate results. MUST NOT run sequentially when parallel is possible.

**Why:** Sequential execution of independent operations wastes the autonomous execution multiplier, turning a 1-session task into a multi-session bottleneck.

## Quality Gates (MUST — Gate-Level Review)

Reviews happen at COC phase boundaries, not per-edit. Skip only when explicitly told to.

**Why:** Skipping lets gaps propagate to downstream repos where they are far more expensive to fix. Evidence: 0052-DISCOVERY §3.3 — six commits shipped without review because gates were "recommended." Background agents make MUST gates nearly free.

| Gate                             | After Phase          | Enforcement | Review                                                                         |
| -------------------------------- | -------------------- | ----------- | ------------------------------------------------------------------------------ |
| Analysis complete                | `/analyze`           | RECOMMENDED | **reviewer**: Are findings complete? Gaps?                                     |
| Plan approved                    | `/todos`             | RECOMMENDED | **reviewer**: Does plan cover requirements?                                    |
| Implementation done              | `/implement`         | **MUST**    | **reviewer** + **security-reviewer**: Run as parallel background agents.       |
| Validation passed                | `/redteam`           | RECOMMENDED | **reviewer**: Are red team findings addressed?                                 |
| Knowledge captured               | `/codify`            | RECOMMENDED | **gold-standards-validator**: Naming, licensing compliance.                    |
| Before release                   | `/release`           | **MUST**    | **reviewer** + **security-reviewer** + **gold-standards-validator**: Blocking. |
| After release (post-merge audit) | `/release` follow-up | RECOMMENDED | **reviewer** run against the MERGED release commit on main.                    |

**BLOCKED responses when skipping MUST gates:**

- "Skipping review to save time"
- "Reviews will happen in a follow-up session"
- "The changes are straightforward, no review needed"
- "Already reviewed informally during implementation"

**Background agent pattern for MUST gates** — the review costs nearly zero parent context. See **Examples § Quality Gates — Background Agent Pattern** below for the CLI-specific syntax.

### MUST: Reviewer Prompts Include Mechanical AST/Grep Sweep, Not Only Diff Review

Every gate-level reviewer prompt MUST include explicit mechanical sweeps that verify ABSOLUTE state (not only the diff). LLM-judgment review of the diff catches what's wrong with the new code; mechanical sweeps catch what's missing from the OLD code that the spec also touched.

See **Examples § Reviewer Mechanical Sweeps** below for the DO / DO NOT delegation block — the surrounding syntax differs per CLI runtime, so it lives in the overlay-replaceable `examples` slot.

**BLOCKED rationalizations:**

- "The reviewer is smart enough to spot orphans"
- "Mechanical sweeps are /redteam's job, not the reviewer's"
- "The diff IS the reviewer's scope"
- "Adding sweeps to every reviewer prompt is repetitive"

**Why:** Gate reviewers are constrained by the diff they're shown. The orphan failure mode of `rules/orphan-detection.md` §1 is invisible at diff-level. A 4-second `grep -c` sweep catches what 5 minutes of LLM judgment misses. See `skills/30-claude-code-patterns/worktree-orchestration.md § Reviewer Prompts` for the kailash-ml 0.12.0 post-mortem.

Origin: kailash-py session 2026-04-19 ML GPU-first Phase 1 codify cycle.

## Zero-Tolerance

Pre-existing failures MUST be fixed (see `rules/zero-tolerance.md` Rule 1). No workarounds for SDK bugs — deep dive and fix directly (Rule 4).

**Why:** Workarounds create parallel implementations that diverge from the SDK, doubling maintenance cost and masking the root bug from being fixed.

## MUST: Worktree Isolation for Compiling Agents

When launching agents that compile (Rust `cargo`, Python `.venv` installs, JS `node_modules`), MUST use the CLI's worktree-isolation primitive to avoid build directory lock contention. See **Examples § Worktree Isolation for Compiling Agents** below for the CLI-specific delegation syntax.

**Why:** Cargo holds an exclusive filesystem lock on `target/`. Two cargo processes in the same directory serialize completely. See `rules/worktree-isolation.md` + `skills/30-claude-code-patterns/worktree-orchestration.md § Rule 1`.

## MUST: Worktree Prompts Use Relative Paths Only

Any absolute path in a worktree-isolation delegation prompt MUST be anchored to the pinned worktree path — absolute paths pointing to the parent checkout are BLOCKED.

See **Examples § Worktree Prompts Use Relative Paths Only** below for the DO / DO NOT delegation syntax.

**Why:** Worktree-isolation primitives set cwd inside the worktree but file-write tools accept any absolute path — parent-rooted paths resolve there regardless of cwd. See `skills/30-claude-code-patterns/worktree-orchestration.md § Rule 2` for the 2026-04-19 three-shard ml-specialist post-mortem (Shard A lost 300+ LOC).

## MUST: Recover Orphan Writes From Zero-Commit Worktree Agents

When an `isolation: "worktree"` agent reports completion but the branch has zero commits AND the worktree has been auto-cleaned, the parent orchestrator MUST inspect the MAIN checkout for orphaned untracked files via `git status --short` BEFORE concluding the work was lost. Absolute-path writes from the agent resolve to the MAIN checkout cwd — the files are NOT lost; they are orphaned, uncommitted, and reachable. Abandoning orphans and re-launching the agent is BLOCKED.

```bash
# DO — detect, recover, PR with recovery/ prefix
git worktree list | grep <branch>; git log <branch> --oneline   # empty + zero = orphan
git checkout -b recovery/<branch> && git add <orphans> && git commit -m "feat(...): recovered"
# → fill missing deliverables (tests, specs, version bump, CHANGELOG) → gh pr create

# DO NOT — re-launch the agent on the same task
# Re-launch produces a second orphan set; next session must untangle two partial adapters.
```

**BLOCKED rationalizations:** "The agent said done, work must be committed somewhere" / "Re-launching is cleaner than recovery" / "Zero-commit branch means the work is gone" / "Main is clean, nothing to recover" / "`recovery/` is a workaround; `feat/` is more correct".

**Why:** `git status` surfaces orphans the moment you look. The first four rationalizations discard 1000+ LOC per occurrence; the fifth conflates branch-name aesthetics with provenance — `recovery/` grep surfaces this rescue class across history; `feat/` does not. See `skills/30-claude-code-patterns/worktree-orchestration.md § Rule 2a` for the 4-step protocol + 2026-04-20 PR #574 post-mortem (1129 LOC of `alignment.py` recovered from MAIN).

Origin: kailash-py Session 2026-04-20 Session 3b (issue #567) — PR #574 `recovery/pr3-alignment-diagnostics`.

## MUST: Worktree Agents Commit Incremental Progress

Every agent launched with `isolation: "worktree"` MUST receive an explicit instruction to `git commit` after each major milestone, not only at completion. The orchestrator MUST verify the branch has ≥1 commit before declaring the agent's work landed. **This applies to every worktree agent regardless of task type** — compile work, prose drafting, one-line config edits, markdown briefs, and one-off spikes all exhibit the same failure mode.

```python
# DO — prompt: "after each file, git add <f> && git commit -m 'wip: <what>'"
# DO NOT — "Implement feature X. Report when done." (truncation loses everything)
# DO NOT — "Draft 11 briefs" / "Fix one line in pytest.ini" without commit step
```

**BLOCKED rationalizations:** "My agent is just writing markdown, commit discipline is overkill" / "One-line config edit doesn't need a commit cadence" / "The agent's summary in the conversation is enough evidence".

**Why:** Worktree auto-cleanup deletes worktrees with zero commits. Truncation mid-message before commit loses 100% of output. See `skills/30-claude-code-patterns/worktree-orchestration.md § Rule 3` for the 2026-04-19 three-shard compile post-mortem (3 of 3 truncated, 2 lost work) AND `rules/worktree-isolation.md` § Rule 5 for the 2026-04-21 non-compile post-mortem (11 drafted briefs + pytest.ini diagnosis lost).

## MUST: Verify Agent Deliverables Exist After Exit

When an agent reports completion of a file-writing task, the parent orchestrator MUST `ls` or read the claimed file before trusting the completion claim. Agent "done" messages are NOT evidence of file creation.

See **Examples § Verify Agent Deliverables Exist After Exit** below for the CLI-specific file-read verification syntax.

**BLOCKED rationalizations:**

- "The agent said 'done', that's good enough"
- "Verifying every file slows the orchestrator"
- "Now let me write the file…" (with no subsequent tool call)

**Why:** Budget exhaustion truncates the final write; agent emits "Now let me write X..." with no tool call. `ls` is O(1) — silent no-op into loud retry. See `rules/worktree-isolation.md` Rule 3 + `skills/30-claude-code-patterns/worktree-orchestration.md § Rule 4` (kaizen round 6, ml-specialist round 7).

## MUST: Parallel-Worktree Package Ownership Coordination

When two or more parallel agents' worktrees touch the SAME sub-package, the orchestrator MUST designate ONE agent as the **version owner** AND tell every other agent: "do NOT edit `packages/<pkg>/pyproject.toml` (or `Cargo.toml`), the package's `__version__` / crate version, or `packages/<pkg>/CHANGELOG.md`".

```python
# DO — A bumps version+CHANGELOG; B's prompt includes "MUST NOT edit pyproject.toml, __version__, CHANGELOG.md"
# DO NOT — both agents race on pyproject.toml; git merge discards one's CHANGELOG prose silently
```

**BLOCKED rationalizations:**

- "Both agents are smart enough to see the existing version"
- "We'll resolve the conflict at merge time"
- "The CHANGELOG entries are for different issues, they'll concat cleanly"
- "Git's three-way merge handles this"
- "Each agent owns a section of the CHANGELOG"

**Why:** Parallel agents see the same base SHA and independently bump version + write top-level CHANGELOG entries. Git picks one; the other's prose is discarded. Pre-declared ownership is O(one-line-in-prompt). See `skills/30-claude-code-patterns/worktree-orchestration.md § Rule 5`.

Origin: Session 2026-04-20 three-agent parallel-release cycle (kailash-ml 0.13.0).

## MUST NOT

- Framework work without specialist — **Why:** Misuse produces code that looks correct but violates invariants (pool sharing, session lifecycle, trust boundaries).
- Sequential when parallel is possible — **Why:** See Parallel Execution above.
- Raw SQL when DataFlow exists — **Why:** Bypasses DataFlow's access controls, audit logging, and dialect portability.
- Custom API when Nexus exists — **Why:** Misses Nexus's session management, rate limiting, multi-channel deployment.
- Custom agents when Kaizen exists — **Why:** Bypasses Kaizen's signature validation, tool safety, structured reasoning.
- Custom governance when PACT exists — **Why:** Lacks PACT's D/T/R accountability grammar and verification gradient.

<!-- /slot:neutral-body -->

<!-- slot:examples -->

## Examples

### Quality Gates — Background Agent Pattern

```
Agent({subagent_type: "reviewer", run_in_background: true, prompt: "Review all changes since last gate..."})
Agent({subagent_type: "security-reviewer", run_in_background: true, prompt: "Security audit all changes..."})
```

### Reviewer Mechanical Sweeps

```
# DO — reviewer prompt enumerates mechanical sweeps
Agent(subagent_type="reviewer", prompt="""
... diff context ...
Mechanical sweeps (run BEFORE LLM judgment):
1. Parity grep — every `return TrainingResult(...)` call site must pass `device=...`
2. `pytest --collect-only -q` exit 0 across all test dirs
3. `pip check` — no new conflicts vs main
4. For every public symbol in __all__ added by this PR — verify eager import
""")

# DO NOT — reviewer prompt only includes diff context
Agent(subagent_type="reviewer", prompt="Review the diff between main and feat/X.")
```

### Worktree Isolation for Compiling Agents

```
# DO: Agent(isolation="worktree", prompt="implement feature X...")
# DO NOT: two agents sharing target/ serialize on cargo's exclusive lock
```

### Worktree Prompts Use Relative Paths Only

```python
# DO — relative paths, resolve to worktree cwd
Agent(isolation="worktree", prompt="Edit packages/kailash-ml/src/kailash_ml/trainable.py...")
# DO NOT — absolute path rooted in parent checkout
# (writes land in MAIN; worktree empty; auto-cleanup loses the work)
```

### Verify Agent Deliverables Exist After Exit

```python
# DO — verify after Agent() returns
Read("/abs/path/src/feature.py")  # raises if missing → retry

# DO NOT — trust completion message
```

<!-- /slot:examples -->
