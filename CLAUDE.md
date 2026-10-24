# Working rules

## Context and sessions
- Keep each project CLAUDE.md under 150 lines: rules, paths, commands only. Long references go in separate files with a pointer.
- /clear between unrelated tasks. /compact with a focus note on long ones.
- Plan mode for any multi-file change. Sonnet for building, Opus for planning only.

## Reading and agents
- Never cat full logs or large files. Use tail, grep, or line ranges.
- No subagents for single-file edits or known-target lookups. Max 3 concurrent.

## Claude API code
- Static content first, cache_control on the last static block.
- No timestamps or IDs in system prompts.
- max_tokens sized to output.
