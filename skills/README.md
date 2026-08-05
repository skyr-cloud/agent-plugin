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
| [`scl`](scl/SKILL.md) | Writing SCL/SCLE (syntax, types, modules, `Package.scle`, the resource model, a module's own `@test` cases), looking up module documentation, and verifying with `skyr fmt`/`skyr check`/`skyr test`. |
| [`deploy`](deploy/SKILL.md) | How deployment to Skyr works: pushing to the `skyr` git remote, environments and the deployment lifecycle, exposing pods to the internet (ports, `InternetAddress`, DNS zones), private networking, sharing networks, volumes and addresses across repositories, first-party plugin capabilities, and checking rollout status and incidents. |
