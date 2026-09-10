# Synthesia Skills

[Agent skills](https://agentskills.io) for building with [Synthesia](https://www.synthesia.io). Each skill lives in `skills/<name>/` with a `SKILL.md` entry point, so they are discoverable by [`skills`](https://skills.sh) and other skill finders.

## Skills

| Skill | Description |
| --- | --- |
| [synthesia-interactive-avatar](skills/synthesia-interactive-avatar/) | Integrate the Synthesia Interactive Avatar into a LiveKit Agent |

## Install

With the [skills CLI](https://skills.sh) (works with Claude Code, Cursor, Codex, and others):

```sh
npx skills add synthesia-ai/skills
```

Or install a single skill:

```sh
npx skills add synthesia-ai/skills --skill synthesia-interactive-avatar
```

### Manual install (Claude Code)

Clone once, then symlink into your skills directory — `git pull` then updates the installed skill with no re-copy step:

```sh
git clone https://github.com/synthesia-ai/skills ~/src/synthesia-skills
ln -s ~/src/synthesia-skills/skills/synthesia-interactive-avatar \
      ~/.claude/skills/synthesia-interactive-avatar
```

## License

[MIT](LICENSE)
