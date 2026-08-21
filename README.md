# agent-harness

This repository stores skills, prompts, notes, and files that are useful for work with AI agents.

## Structure

```
agent-harness/
├── skills/    # Skills (instructions + resources) for agents
├── prompts/   # Reusable prompts and templates
├── info/      # Notes, guides, and documentation you find or write
├── files/     # Other useful files (configs, examples, scripts)
└── ATTRIBUTIONS.md  # Register of external content and its authors
```

## Conventions

- Put one topic in each folder or file. Use descriptive names in kebab-case (`my-skill/`, `code-review-prompt.md`).
- Each skill has a `SKILL.md` file with a name, a description, and instructions.

## Attribution

Add one line per item to [ATTRIBUTIONS.md](ATTRIBUTIONS.md), so all external sources are easy to audit:

```markdown
- skills/foo/ — "Foo Skill" by Author Name — https://example.com/foo — MIT — imported 2026-08-21
```

If an item is only inspired by another work (not a copy), write this instead:

```markdown
- skills/foo/ — "Foo Skill" inspired by Author Name — https://example.com/foo — MIT — reworked 2026-08-21
```
