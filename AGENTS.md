# Agent Guidelines — Fragbyte Agent Skills

## File Structure

```
fragbyte-agent-skills/
├── README.md                     # Repo overview, list of skills
├── AGENTS.md                     # This file
├── LICENSE                       # MIT (covers all skills)
├── CHANGELOG.md                  # Repo-wide changelog
└── skills/
    ├── <skill-name>/             # One folder per skill
    │   ├── README.md             # Skill description
    │   ├── CHANGELOG.md          # Skill version history
    │   ├── SKILL.md              # Main skill file (frontmatter + body)
    │   ├── AGENTS.md             # Skill-specific notes (optional)
    │   └── references/           # Deep-dive reference files
    │       └── *.md
```

## Naming Conventions

- **Skill folders**: lowercase, hyphenated, no version suffix (e.g., `ddd-guideliness`, not `ddd-guideliness-v1`).
- **Reference files**: descriptive, lowercase (e.g., `tactical-patterns.md`, `spring-boot.md`).
- **Example domain**: keep consistent within a skill. The DDD skill uses **CargoRoute** throughout.

## Content Rules

1. **SKILL.md** is the entry point. It must list all reference files in its "Map of this skill" table.
2. **Cross-references** between files must resolve. A reference to `references/xxx.md` from within a skill folder means `skills/<skill-name>/references/xxx.md`.
3. **Stack-specific files** (e.g., `spring-boot.md`, `angular.md`) contain code examples. **Conceptual files** (e.g., `tactical-patterns.md`, `strategic-design.md`) must NOT contain code — only conceptual guidance, checklists, and worked examples in prose.
4. **Version field** in SKILL.md frontmatter uses semver (e.g., `1.0.0`).

## Linting

Run markdownlint on all `.md` files before committing:

```bash
markdownlint "**/*.md"
```

Known acceptable deviations (add to `.markdownlint.json`):

- Lines can exceed 80 chars (technical content).
- Fenced code blocks may omit a language (pseudo-code and ASCII diagrams).
- Duplicate headings are allowed (reference sections reuse headings like "CargoRoute example" and "What breaks without it").

## Adding a Reference to an Existing Skill

1. Create the file under `references/`.
2. Add a row to the "Map of this skill" table in `SKILL.md`.
3. Add an entry to the relevant section reference in `SKILL.md`.
4. Update the skill's `CHANGELOG.md`.
