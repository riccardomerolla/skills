# The canonical install block

One install story, one wording. `README.md` and `.changeset/*` must say **this** and nothing else. Change it here first, then propagate. (The pages under `docs/` are upstream's and are not a consumer of this block.)

This fork is **not** in Claude Code's official marketplace. The repo is its own single-plugin marketplace via `.claude-plugin/marketplace.json`, so the Claude Code route is the two-step marketplace install below.

## Claude Code: the plugin

<canonical-block name="claude-code">

```
/plugin marketplace add riccardomerolla/skills
```

```
/plugin install riccardomerolla-skills@riccardomerolla
```

This fork is not in Claude Code's official marketplace, so the first command registers the repo as a marketplace and the second installs the plugin from it.

</canonical-block>

## Codex, and other agents: skills.sh

The plugin is Claude Code only. Everywhere else, [skills.sh](https://skills.sh/riccardomerolla/skills) copies editable skill files into the project. Use the whole-set form on `README.md`:

<canonical-block name="skills-sh-whole-set">

```bash
npx skills@latest add riccardomerolla/skills
```

Pick the skills you want, and which coding agents to install them on. **The installer lets you choose which skills to take: make sure `setup-ricky-skills` is one of them.**

</canonical-block>

…and the single-skill form wherever one skill is named on its own.

<canonical-block name="skills-sh-one-skill">

```bash
npx skills@latest add riccardomerolla/skills --skill=<name>
```

```bash
npx skills@latest update <name>
```

</canonical-block>

## Effect-TS repos

`zen-of-ricky` and `effect-ts-conventions` assume the official Effect skill is installed and has been run once in the repo:

<canonical-block name="effect-ts-official">

```bash
npx skills add Effect-TS/skills
```

</canonical-block>

## The two routes are exclusive

The plugin is a managed bundle you subscribe to. skills.sh writes files you own and edit. Installing both leaves the user with every skill twice: always say "pick one".
