---
name: wrapup
description: "Write session notes before ending. Captures context for next session."
---

Write session notes to preserve context for the next session.

1. Determine the active workspace:
   - If `$ARGUMENTS` specifies a project name, use `workspaces/$ARGUMENTS/`
   - Otherwise, use the most recently modified directory under `workspaces/` (excluding `instructions/`)

2. Write a `.session-notes` file in the workspace root using plain language a non-technical user can understand:
   - **Accomplished**: Describe in terms of what users can now do or what's been decided, not technical tasks ("Users can now sign up and log in" not "Implemented auth endpoints")
   - **In progress**: Describe what's being worked on in terms of user-visible features
   - **Blockers**: Describe any decisions needed from the user in clear, specific terms with the implications of each option
   - **Next steps**: Describe what will happen next session in terms of outcomes
   - **Active todo**: Which todo was being worked on (for the AI's context restoration)

3. Keep it concise (under 30 lines). This file is read by the next session's startup to restore context.

4. Overwrite any existing `.session-notes` — only the latest matters.
