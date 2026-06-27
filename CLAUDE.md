
<!-- context-saver-trigger -->
## Context discipline (context-saver skill) — apply every session
- Pre-flight: `python3 ~/.claude/skills/context-saver/scripts/bloat.py .` — split any SPLIT-flagged file before adding to it.
- NEVER read a file >400 lines whole. Map then slice:
  `python3 ~/.claude/skills/context-saver/scripts/filemap.py <file>`
  `python3 ~/.claude/skills/context-saver/scripts/slice.py <file> <start:end|term>`
- Before adding CSS/JS, confirm it isn't already there:
  `python3 ~/.claude/skills/context-saver/scripts/dupes.py <file>`  (and `--dead <file.css> .`)
  When you replace a system, DELETE the old one in the SAME change. Files should shrink or stay flat.
- After /compact: read STATE.md, not source. Prefer /clear + rehydrate over repeated /compact.
- Handoffs to Codex/Antigravity: `python3 ~/.claude/skills/context-saver/scripts/repomap.py . --write`, then point the plan at REPOMAP.md.
