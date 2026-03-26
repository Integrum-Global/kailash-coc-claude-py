---
name: wrapup
description: "Write session notes before ending. Captures context for next session."
---

Write session notes to preserve context for the next session.

1. Determine the active workspace:
   - If `$ARGUMENTS` specifies a project name, use `workspaces/$ARGUMENTS/`
   - Otherwise, use the most recently modified directory under `workspaces/` (excluding `instructions/`)

### Journal Check

Before writing session notes, review the session's work and create journal entries for any un-journaled decisions, discoveries, or risks:
- Were any significant decisions made without DECISION entries?
- Were any technical findings discovered without DISCOVERY entries?
- Were any risks identified without RISK entries?

Create entries for anything missing, then proceed to write session notes.

2. Write a `.session-notes` file in the workspace root with:
   - **Accomplished**: What was completed this session
   - **In progress**: What is partially done
   - **Blockers**: Any issues or decisions needed
   - **Next steps**: What to work on next session
   - **Active todo**: Which todo was being worked on

3. Keep it concise (under 30 lines). This file is read by the next session's startup to restore context.

4. Overwrite any existing `.session-notes` — only the latest matters.
