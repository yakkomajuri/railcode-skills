# Agent Notes

- On every update to any Railcode skill, check the latest published Railcode CLI version on npm: https://www.npmjs.com/package/railcode
- Use the latest CLI behavior to update the skill accordingly, especially commands, SDK globals, local dev, deploy flow, and package-manager assumptions.
- Always bump the updated skill's `version` field in `SKILL.md`.
- After committing a skill update, create a matching git tag named `<skill-name>-v<version>`.
- If npm cannot be reached, say that clearly in the handoff and avoid claiming the skill reflects the latest CLI.
