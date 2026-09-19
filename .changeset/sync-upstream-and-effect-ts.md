---
"riccardomerolla-skills": minor
---

Re-sync the fork with upstream `mattpocock/skills` and move the house stack from Scala 3 + ZIO to TypeScript + Effect 4.

- Reset to upstream `main` and re-add the fork-only work on top, adopting upstream's renamed skills (`diagnosing-bugs`, `to-tickets`, `to-spec`, `writing-for-agents`) and dropping the fork's old copies. `setup-matt-pocock-skills` becomes `setup-ricky-skills`; `caveman` and `zoom-out` return from the fork.
- Move `scala3-zio` and `zen-of-ricky` (now `zen-of-ricky-scala`) to `deprecated/`.
- Add `zen-of-ricky` (house principles for TypeScript + Effect 4) and `effect-ts-conventions` (repo mechanics for an Effect 4 codebase), both layered on the official `effect-ts` skill from `Effect-TS/skills`, with every snippet checked against `effect@4.0.0-rc`.
- Strip em-dashes from the re-added legacy-modernization and clean-room skills, add `agents/openai.yaml` to every re-added skill, and point their cross-references at the upstream names.
- Repoint the plugin, marketplace, package, changeset config, install block, README, bucket READMEs, and the `ask-matt` router at the fork.
