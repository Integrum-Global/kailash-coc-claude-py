---
priority: 0
scope: baseline
---

# Worktree Isolation Rules

<!-- slot:neutral-body -->

This rule targets **orchestrator behavior** — the parent session that spawns agents with `isolation: "worktree"`. It governs session-spawn-time decisions (what to pass in the delegation prompt, how to verify deliverables), NOT file-edit-time behavior. It loads universally because orchestration happens in sessions that rarely edit the `agents/`, `commands/`, or rule files it used to be path-scoped to — exactly the sessions where its absence caused the failure mode to recur.

Agents launched with `isolation: "worktree"` run in their own git worktree so parallel jobs do not fight over the same resources. The original motivation was compile-heavy contention (Rust `target/`, Python `.venv/`, JS `node_modules/`) but **the rule applies to every worktree agent regardless of whether it compiles** — prose-drafting, config edits, markdown briefs, one-line pytest.ini fixes all exhibit the same failure mode. When an agent drifts back to the main checkout — because the system prompt didn't pin cwd, because absolute paths were copied from the orchestrator, because the tool defaulted to `process.cwd()` — the isolation silently breaks: two workers overwrite each other's changes, one commits the other's half-done code, and the "parallel" session produces garbage that only surfaces at `/redteam` or (more often) post-hoc when the lost work is noticed missing.

This rule mandates a self-verification step at agent start AND a pre-flight check in the orchestrator's delegation prompt. The verification is cheap (one `git status`) and the failure mode is expensive (a whole session's worth of parallel work corrupted).

## MUST Rules

### 1. Orchestrator Prompts MUST Pin The Worktree Path

Any delegation that uses worktree isolation MUST include the absolute worktree path in the prompt AND MUST instruct the agent to verify `git -C <worktree> status` at the start of its run. Passing the isolation flag without the explicit path is BLOCKED.

See **Examples § Rule 1 — Orchestrator Prompts Pin The Worktree Path** below for the DO / DO NOT delegation syntax.

**BLOCKED rationalizations:**

- "The isolation flag handles the cwd for me"
- "The tool sets up the worktree automatically"
- "I'll just use relative paths, they're shorter"
- "The agent will figure out the right directory"
- "I tested it once, it worked — should keep working"

**Why:** The `isolation: "worktree"` flag creates the worktree but does not pin every tool call inside it — file-writing tools that accept absolute paths will happily write to the main checkout if the orchestrator's prompt uses a main-checkout path. Multiple specialist agents (ml, dataflow, kaizen) drifted back to the main tree during parallel sessions; the corruption was only caught by `git status` after the fact. One-line verification at agent start converts a silent corruption into a loud refusal.

### 2. Specialist Agents MUST Self-Verify Cwd At Start

Every specialist agent file (`.claude/agents/**/*.md`) that may be launched with `isolation: "worktree"` MUST include a "Working Directory Self-Check" step at the top of its process section. The check prints the resolved cwd and the git branch, and refuses to proceed if either is unexpected.

```markdown
# DO — self-check baked into the agent file

## Step 0: Working Directory Self-Check

Before any file edit, run:

    git rev-parse --show-toplevel
    git rev-parse --abbrev-ref HEAD

If the top-level path does NOT match the worktree path passed in the
prompt, STOP and emit "worktree drift detected — refusing to edit
main checkout". Do NOT fall back to process.cwd().

# DO NOT — assume orchestrator pinned cwd

## Step 1: Read the task

Read the prompt, start editing files…
```

**Why:** The orchestrator's pinned-path instruction can be lost to context compression across long delegation chains; a self-check inside the specialist file is a belt-and-suspenders guarantee that survives prompt truncation. Verified cost: one git call (~30 ms). Verified benefit: prevents the parallel-specialist drift seen in long sessions across compile-heavy languages (Rust cargo locks, Python `.venv` install races, JS `node_modules` writes).

### 3. Parent MUST Verify Deliverables Exist After Agent Exit

When an agent reports completion of a file-writing task, the parent orchestrator MUST verify the claimed files exist at the worktree path via the CLI's filesystem-read primitive before trusting the completion claim. Agent completion messages are NOT evidence of file creation.

See **Examples § Rule 3 — Parent Verifies Deliverables After Agent Exit** below for the DO / DO NOT delegation syntax.

**BLOCKED rationalizations:**

- "The agent said 'done', that's good enough"
- "Verifying every file slows the orchestrator"
- "The agent would have errored if the write failed"
- "Now let me write the file..." followed by no actual write

**Why:** Agents hit their budget mid-message and emit "Now let me write X..." without having written X. Multi-agent sessions have logged repeated occurrences (kaizen-specialist round 6, ml-specialist round 7) where an agent reported success with zero files on disk. An `ls` check is O(1) and converts "silent no-op" into "loud retry".

### 4. Worktree Prompts MUST NOT Reference The Parent-Checkout Path

Any absolute path in a worktree-isolation delegation prompt MUST be anchored to the pinned worktree path (see Rule 1). Absolute paths pointing to the parent checkout (`/Users/<you>/repos/<project>/<subpath>` with no worktree prefix) are BLOCKED — agents resolve them against the filesystem root, silently bypassing the worktree and writing into the parent. Relative paths are the safer default because they always resolve to the agent's cwd (the worktree).

See **Examples § Rule 4 — Worktree Prompts Do Not Reference Parent-Checkout Path** below for the DO / DO NOT delegation syntax.

**BLOCKED rationalizations:**

- "Absolute paths are unambiguous"
- "The agent should figure out its own cwd"
- "This worked the one time I tested it"
- "I included 'Repo root: /Users/...' at the top; the agent will use that"

**Why:** `isolation: "worktree"` runs the agent with cwd set to the worktree, but file-write tools accept any absolute path — an absolute path that points to the parent resolves there regardless of cwd. Session 2026-04-19 logged 2 of 3 parallel ml-specialist shards writing to main before self-correcting (Shard B) or losing work entirely (Shard A's 300+ LOC of sklearn array-API impl was lost when its empty worktree auto-cleaned). Only one self-corrected; the failure mode is not agent-detectable by default.

### 5. Worktree Agents MUST Commit Incremental Progress

Every agent launched with worktree isolation MUST receive an explicit instruction in its prompt to `git commit` after each major milestone (each file written, each test batch passed, each draft brief completed, each config edit made), NOT only at completion. The orchestrator MUST then verify the branch has ≥1 commit before declaring the agent's work landed. Worktrees with zero commits auto-clean on agent exit and the work is unrecoverable — **this applies equally to compile work, prose/markdown drafting, one-line config edits, and every other agent task; "my agent is just writing markdown, commit discipline is overkill" is BLOCKED.**

See **Examples § Rule 5 — Worktree Agents Commit Incremental Progress** below for the DO / DO NOT delegation syntax.

**BLOCKED rationalizations:**

- "The agent will commit at the end"
- "Splitting into commits adds overhead"
- "The parent can recover from the worktree after the agent exits"
- "Budget exhaustion is rare"
- "My agent is just writing markdown / just editing one config line — commit discipline is overkill for that"
- "The agent's summary in the conversation is enough evidence"

**Why:** Worktree auto-cleanup silently deletes worktrees with zero commits on their branch. An agent that writes perfect code / prose / config but truncates mid-message before committing loses 100% of its output. Post-hoc file-existence verification (Rule 3) catches orphan files in main but CANNOT recover files that were only in a cleaned-up worktree. Sessions 2026-04-19 (compile-heavy: 3 of 3 parallel ml-specialist shards truncated at 250-370k tokens; 2 lost work) AND 2026-04-21 (non-compile: 11 drafted markdown briefs + a verified pytest.ini diagnosis lost when two worktree agents reported "done" without committing) demonstrate the same failure across both task types. The only reliable defense is instructing the agent to commit as it goes regardless of whether the task compiles or not.

## MUST NOT

- Launch an agent with `isolation: "worktree"` without passing the absolute worktree path in the prompt

**Why:** The isolation flag alone does not guarantee every tool call stays inside the worktree — the prompt is the only place the agent learns where it belongs.

- Trust an agent's "completion" message when it says "Now let me write…" followed by no tool call

**Why:** Budget exhaustion truncates the write. The completion message is misleading; the filesystem is the source of truth.

- Use `process.cwd()` or relative paths inside specialist agent files that may run in a worktree

**Why:** `process.cwd()` resolves to whatever the Claude Code process was launched with (the main checkout), not the worktree; relative paths inherit the same problem.

## Relationship To Other Rules

- `rules/agents.md` § "MUST: Worktree Isolation for Compiling Agents" — companion rule; this file is the verification layer for the isolation directive there.
- `rules/zero-tolerance.md` Rule 2 — a completed-looking file that doesn't exist is a stub under a different name.
- `rules/testing.md` § "Verified Numerical Claims In Session Notes" — same principle, applied to file deliverables.

Origin: kailash-py session 2026-04-19 — ml-specialist, dataflow-specialist, and kaizen-specialist each drifted back to the main tree during PRs #502-#508; kaizen round 6 and ml-specialist round 7 reported "Now let me write X..." completions with no actual file writes. The self-verify + parent-verify protocol closed both failure modes. Generalises to any compile-heavy language with parallel-agent contention (Rust `target/`, Python `.venv/`, JS `node_modules/`).

<!-- /slot:neutral-body -->

<!-- slot:examples -->

## Examples

### Rule 1 — Orchestrator Prompts Pin The Worktree Path

```python
# DO — explicit path + verification instruction
worktree = "/Users/me/repos/myproject/.claude/worktrees/agent-feature-abc123"
Agent(
    isolation="worktree",
    prompt=f"""
Working directory: {worktree}

STEP 0 — verify isolation before touching any file:
  git -C {worktree} status
If the output shows "not a git repository" OR the branch does not
match the worktree's expected branch, STOP and report "worktree
isolation broken" — do NOT fall back to the main checkout.

All file paths you write MUST be absolute and begin with {worktree}/.
""",
)

# DO NOT — isolation flag without pinned path
Agent(
    isolation="worktree",
    prompt="Implement feature X — use the framework-specialist patterns.",
)
# Agent starts in process.cwd() (main checkout), edits main's tree,
# reports success. Worktree is empty; main has half-done code.
```

### Rule 3 — Parent Verifies Deliverables After Agent Exit

```python
# DO — verify after agent returns
result = Agent(isolation="worktree", prompt=f"Write {worktree}/src/feature.py...")
assert_file_exists(f"{worktree}/src/feature.py")  # parent checks

# DO NOT — trust "done" and proceed
result = Agent(isolation="worktree", prompt="...")
# Parent commits based on result.completion_message without ls
```

### Rule 4 — Worktree Prompts Do Not Reference Parent-Checkout Path

```python
# DO — relative paths, resolve to worktree cwd
Agent(
    isolation="worktree",
    prompt="Edit packages/kailash-ml/src/kailash_ml/trainable.py at line 370..."
)

# DO — absolute paths anchored to the PINNED worktree path (Rule 1 style)
worktree = "/Users/me/repos/myproject/.claude/worktrees/agent-feature-abc123"
Agent(
    isolation="worktree",
    prompt=f"Edit {worktree}/packages/kailash-ml/src/kailash_ml/trainable.py at line 370..."
)

# DO NOT — absolute paths pointing to the PARENT checkout
Agent(
    isolation="worktree",
    prompt="Edit /Users/me/repos/myproject/packages/kailash-ml/src/kailash_ml/trainable.py at line 370..."
)
# ↑ writes land in the MAIN checkout; the worktree stays empty; auto-cleanup
#   deletes the empty worktree; agent's work is either silently on main OR lost.
```

### Rule 5 — Worktree Agents Commit Incremental Progress

```python
# DO — prompt includes incremental commit discipline
Agent(
    isolation="worktree",
    prompt="""...
**Commit discipline (MUST):**
- After each file is complete, run `git add <file> && git commit -m "wip(shard-X): <what>"`.
- Do NOT hold all work in the worktree's index until the final report.
- If you exit without committing (budget exhaustion / crash / interruption),
  the worktree is auto-cleaned and ALL work is lost.
""",
)

# DO NOT — trust that the agent commits at completion
Agent(isolation="worktree", prompt="Implement feature X. Report when done.")
# ↑ agent writes 4 files, hits budget on file 5, emits truncated message,
#   never reaches `git commit`, worktree auto-cleaned — all 5 files lost.
```

<!-- /slot:examples -->
