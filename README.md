# Fragbyte Agent Skills

A collection of reference skills for [OpenCode](https://opencode.ai) — reusable, opinionated instructions for specific domains and tasks.

## Included Skills

| Skill | Domain | Last Updated |
|---|---|---|
| [ddd-guideliness](skills/ddd-guideliness) | Domain-Driven Design (backend & frontend) | 2026-08-26 |

## How to Use

Each skill lives in its own folder under `skills/`. The main file is `SKILL.md` — read it to understand what the skill covers and when to use it. Each skill also has a `references/` folder with deeper dives and a `CHANGELOG.md` for version history.

### Adding a New Skill

1. Create a folder under `skills/` (e.g., `skills/my-new-skill/`).
2. Add a `SKILL.md` with the frontmatter (`name`, `description`, `license`, `metadata`).
3. Add reference files under `references/`.
4. Update this README's table.
5. Create a `CHANGELOG.md` in the skill folder.

See `AGENTS.md` for file conventions and linting rules.

## License

All skills in this repository are licensed under the MIT License — see [LICENSE](LICENSE).
