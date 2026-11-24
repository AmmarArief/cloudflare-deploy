# agent-skills

Personal Agent Skills following the [Agent Skills open standard](https://agentskills.io): each skill is a directory with a `SKILL.md` that carries YAML front matter (`name`, `description`) and the runbook itself.

Compatible with coding agents that read `SKILL.md` files — omp, Claude Code, Codex, Cursor, Gemini CLI.

## Skills

| Skill | Use it when |
|---|---|
| [`cloudflare-deploy`](./skills/cloudflare-deploy/SKILL.md) | Wiring a repo to auto-deploy on Cloudflare from GitHub pushes, or fixing a broken Cloudflare deploy pipeline |

## Install

Copy a skill into your agent's skills directory, then register it if your agent keeps an explicit list.

```bash
# omp
cp -R skills/cloudflare-deploy ~/.omp/agent/skills/
```

For omp, the skill only loads once its name is present under `skills.includeSkills` in `~/.omp/agent/config.yml`:

```yaml
skills:
  includeSkills:
    - cloudflare-deploy
```

## Layout

```
skills/
  <skill-name>/
    SKILL.md      # front matter + runbook
```

## License

MIT
