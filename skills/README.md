# Skills

Agent skills for the Skyr plugin live here, one directory per skill:

```
skills/
  <skill-name>/
    SKILL.md
```

Each skill is a subdirectory containing a `SKILL.md` following the
[Agent Skills](https://code.claude.com/docs/en/skills) format, which both
Claude Code and Codex load.

| Skill | Teaches |
|-------|---------|
| [`scl`](scl/SKILL.md) | Writing SCL/SCLE (syntax, types, modules, `Package.scle`, the resource model), looking up stdlib documentation, and verifying with `skyr fmt`/`skyr check`. |
