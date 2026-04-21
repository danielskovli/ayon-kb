# CLAUDE.md

**→ Read [AGENTS.md](AGENTS.md) first.** It contains the canonical
project brief, vocabulary traps, skills catalog, and verification policy.

## Claude-specific notes

- The `ayon-*` skills in `.claude/skills/` auto-load on relevance — each
  one's description keyword-matches against the user's message. Don't
  re-paste that material into chat; let the skill load.
- Task skills (`/ayon-verify`, `/ayon-new-addon`) are user-invoked only;
  treat them as commands, not background context.
- The user's auto-memory has pointers to this KB
  (`ayon_kb_reference.md`, `project_ayon_kb.md`).
- When extending the KB, touch both layers: the root `0N-*.md` file
  (human-readable, canonical source) and its matching skill
  (agent-facing distillation). Skill sources should stay aligned with
  the KB file's Sources block.
