---
name: ws
description: "Show workspace status dashboard. Read-only."
---

Display the current workspace status in plain language. Do not modify any files.

1. List all directories under `workspaces/` (excluding `instructions/`).

2. For the most recently modified workspace (or `$ARGUMENTS` if specified):
   - Show workspace name and path
   - Derive current phase and present using plain-language descriptions:
     - Has `01-analysis/` files → "Research and planning completed"
     - Has `todos/active/` files → "Project roadmap created — ready for your review"
     - Has `todos/completed/` files → "Building in progress" + progress summary
     - Has `04-validate/` files → "Testing completed — results ready for your review"
     - Agents/skills were updated in phase 05 → "Knowledge captured — the AI remembers this project"
   - Show progress: "X of Y tasks completed" (count files in `todos/active/` vs `todos/completed/`)
   - List the 5 most recently modified files in the workspace
   - If `.session-notes` exists, show its contents and age
   - Suggest the next action: "Next step: run `/implement` to continue building" or similar

3. Present as a compact, friendly summary. Use plain language throughout.
