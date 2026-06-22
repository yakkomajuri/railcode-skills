# railcode-skills

Agent skills for [Railcode](https://github.com/yakkomajuri/railcode), extracted from
the main `railcode` repo so they can be installed and versioned on their own.

These skills ship through the open agent-skills ecosystem (the `skills` CLI), so they
work across Claude Code, Codex, Cursor, and other agents.

## Skills

| Skill | Description |
| --- | --- |
| [`create-railcode-app`](create-railcode-app/SKILL.md) | Build, modify, debug, and deploy Railcode static apps end-to-end — scaffolding with the CLI, wiring the zero-config SDK globals, configuring access policies, testing with `railcode dev`, and deploying. |

## Install

Install a single skill by name:

```bash
npx skills add yakkomajuri/railcode-skills --skill create-railcode-app
```

Update it later:

```bash
npx skills update create-railcode-app
```
