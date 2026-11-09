# binary-ninja-plugin-builder

An agent skill for building, debugging, and packaging [Binary Ninja](https://binary.ninja/) (Binja) plugins in Python.

## What it does

Loads whenever the user mentions Binary Ninja / Binja (BinaryView, BNDB, LLIL/MLIL/HLIL, PluginCommand, BackgroundTaskThread, sidebar widgets, binaryninjaui, plugin.json, ...) or asks to automate or script reverse-engineering workflows in Binary Ninja.

The skill covers:

- Command plugins, background tasks, and settings
- UI / sidebar panels (with headless-safe guards)
- Headless scripting and testing
- AI-assisted plugins (LLM / external service calls)
- Common pitfalls (threading, IL analysis, user-level APIs, pip deps)
- `plugin.json` packaging and Plugin Manager submission

## Golden rule

Binary Ninja's API changes between versions. **Verify the API before using it** — introspect the user's install, grep the shipped Python examples, or check docs.binary.ninja / api.binary.ninja. Never write an API call from memory without a check when uncertain.

## Usage

This repo follows the agent-skill format: a `SKILL.md` file with YAML frontmatter (`name`, `description`) followed by Markdown instructions. Point your agent (Claude Code, Cursor, Zed, custom agent platforms, etc.) at this repo, or copy `SKILL.md` into your skills directory.

## License

MIT — see [LICENSE](LICENSE).
