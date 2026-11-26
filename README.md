# skills

Personal collection of agent skills.

## Install

```bash
npx skills add tshanli/skills
```

Install a specific skill:

```bash
npx skills add tshanli/skills --skill orch-round
```

## Skills

| Skill | Description |
| --- | --- |
| [`orch-round`](skills/orch-round/SKILL.md) | Multi-model orchestration loop on `herdr-core` (`hc`) with quota-aware routing and worktree isolation. |
| [`review-and-capture`](skills/review-and-capture/SKILL.md) | Runs `code-review` and records reusable findings into the repo's review ledger. |
| [`capture-review`](skills/capture-review/SKILL.md) | Appends recurring review findings to `.agents/review-memory.jsonl` and promotes repeated patterns to rules. |

## License

[MIT](LICENSE)
