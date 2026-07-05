# Skyr agent plugin

A cross-tool agent plugin for [Skyr](https://skyr.cloud), the Git-native
infrastructure orchestrator. It bundles agent skills that help you author and
operate Skyr configuration written in the Skyr Configuration Language (SCL).

The plugin is compatible with both [Claude Code](https://code.claude.com) and
[Codex](https://developers.openai.com/codex) plugin formats, and its skills
follow the cross-tool [Agent Skills](https://code.claude.com/docs/en/skills)
standard.

## Layout

```
.claude-plugin/plugin.json   # Claude Code manifest
.codex-plugin/plugin.json    # Codex manifest (identical content)
skills/                      # one subdirectory per skill (SKILL.md)
```

No skills ship yet — the plugin currently provides structure only.

## Install

### Claude Code

Add the marketplace, then install the `skyr` plugin:

```sh
claude plugin marketplace add skyr-cloud/agent-plugin
claude plugin install skyr
```

### Codex

Install the plugin from this repository following the
[Codex plugin documentation](https://developers.openai.com/codex).

## Read-only mirror

This repository is a read-only mirror. The plugin is developed in the Skyr
monorepo and published here automatically; changes pushed directly to this
mirror will be overwritten. Please file issues and open pull requests against
the monorepo instead.

## License

MIT — see [LICENSE](LICENSE).
