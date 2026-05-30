# faster-gh-cli-skill

This repository is an Agent Skill package. Keep the root layout compatible with agentskills.io: `SKILL.md` at the repository root with YAML frontmatter.

## Project rules

- Keep `SKILL.md` concise and action-oriented. Put only patterns an agent is likely to get wrong without the skill.
- Preserve the skill name `faster-gh-cli-skill`; it must match the package/repository name.
- When adding examples, prefer commands that work with current `gh` behavior. Verify uncertain flags or JSON fields with `gh <command> --help` or a harmless command before documenting them.
- Do not include raw OpenCode session logs. Use aggregate findings or sanitized examples only.
- If meaningful project conventions change, update this file in the same change.

## Validation

- Check frontmatter against the Agent Skills spec: lowercase hyphenated `name`, non-empty `description`, and valid YAML.
- If `skills-ref` is available, run `skills-ref validate .`.
- Run `gh --version` before updating instructions that depend on `gh` behavior.
