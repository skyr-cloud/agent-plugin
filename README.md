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
.claude-plugin/plugin.json        # Claude Code plugin manifest
.claude-plugin/marketplace.json   # Claude Code marketplace catalog (self-cataloging)
.codex-plugin/plugin.json         # Codex manifest (same content as the Claude Code one)
skills/                           # one subdirectory per skill (SKILL.md)
```

No skills ship yet — the plugin currently provides structure only.

## Install

### Claude Code

This repository is its own plugin marketplace (named `skyr-cloud`). Add it,
then install the `skyr` plugin from it:

```sh
claude plugin marketplace add skyr-cloud/agent-plugin
claude plugin install skyr@skyr-cloud
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
