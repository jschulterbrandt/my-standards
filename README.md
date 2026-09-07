# my-standards

A Claude Code plugin packaging my personal coding standards and conventions as skills.

## Structure

- `.claude-plugin/plugin.json` — plugin manifest (name, description, version, author).
- `.claude-plugin/marketplace.json` — marketplace manifest for installing this plugin locally via `source: "."`.
- `skills/` — one directory per skill, each with a `SKILL.md`.

## Skills

- **api-endpoints** — guides adding new routes, handlers, or controllers: where route files live, naming conventions, the error handling pattern, the response shape, and what must be true before committing.
