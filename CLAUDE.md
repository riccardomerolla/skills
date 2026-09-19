Skills are organized into bucket folders under `skills/`:

- `engineering/`: daily code work
- `productivity/`: daily non-code workflow tools
- `misc/`: kept around but rarely used, not promoted
- `in-progress/`: beta: public on purpose, feedback wanted, not shipped in the plugin
- `personal/`: tied to my own setup, not promoted
- `deprecated/`: no longer used, kept for reference

Every skill in `engineering/` or `productivity/` (the **promoted** buckets) must have a reference in the top-level `README.md` and an entry in `.claude-plugin/plugin.json`'s `skills` array (the Claude Code plugin ships exactly the promoted set). Skills in `misc/`, `in-progress/`, `personal/`, and `deprecated/` must not appear in either.

This repo is a fork of [mattpocock/skills](https://github.com/mattpocock/skills) and tracks it. Upstream skills keep their upstream names and content; do not edit them here except to rename `setup-matt-pocock-skills` to `setup-ricky-skills`, which is the fork's personalised setup skill. Fork-added skills are the legacy-modernization pipeline (`legacy-inventory`, `legacy-extract-flow`, `lbp-to-target-map`, `target-map-to-prd`), the clean-room pair (`clean-room-extract`, `csp-to-prd`), the Effect pair (`zen-of-ricky`, `effect-ts-conventions`), `zoom-out`, `caveman`, and everything in `personal/` and `deprecated/`. To sync with upstream, reset a branch to `upstream/main` and re-add the fork-only work on top; the `pre-upstream-sync` tag marks the history before the last sync.

Install commands are copied verbatim from [.agents/install-block.md](./.agents/install-block.md). `.claude-plugin/marketplace.json` makes the repo its own single-plugin marketplace (a fallback the install block explains, not the documented route). Run `claude plugin validate . --strict` after touching either manifest. Why a Claude plugin but not (yet) a Codex one lives in [.agents/adr/0002-ship-as-a-claude-code-plugin.md](./.agents/adr/0002-ship-as-a-claude-code-plugin.md).

Each skill entry in the top-level `README.md` must link the skill name to its `SKILL.md`.

Each bucket folder has a `README.md` that lists every skill in the bucket with a one-line description, with the skill name linked to its `SKILL.md`. The promoted buckets' `README.md`s and the top-level `README.md` group entries into **User-invoked** and **Model-invoked**; non-promoted bucket `README.md`s (`misc/`, `in-progress/`) use a flat list.

The docs pages under `docs/engineering/` and `docs/productivity/` belong to upstream and are published on aihero.dev, not by this fork; keep them as upstream ships them (with the setup skill renamed) and do not add pages for fork-added skills. The rest of this paragraph describes upstream's rule for reference. Skills in `engineering/` and `productivity/` also have a human-facing docs page at `docs/<bucket>/<skill-name>.md` (the docs tree mirrors those two bucket folders under `skills/`). The published URL is `https://aihero.dev/skills-<skill-name>` regardless of bucket: the docs path is repo organisation only. When you add, rename, or change the behaviour of a skill in `engineering/` or `productivity/`, create or re-sync its docs page following [.agents/writing-docs.md](./.agents/writing-docs.md). A finished page carries four sections: **What it does**, **When to reach for it**, **Common questions**, and **It's working if**. `writing-docs.md` holds the template, the section order, and where to hunt for the questions. Skills in the non-promoted buckets (`misc/`, `in-progress/`, `deprecated/`) get **no** docs page.

Every `SKILL.md` is either user-invoked (`disable-model-invocation: true` plus `policy.allow_implicit_invocation: false` in `agents/openai.yaml`, reachable only by the human) or model-invoked (model- or user-reachable). See [.agents/invocation.md](./.agents/invocation.md).

[`ask-me`](./skills/engineering/ask-me/SKILL.md) is the router that maps every user-reachable skill and how they relate, including the fork-added ones under its "Fork additions" section. The same trigger that re-syncs a docs page applies to it: whenever you add, rename, remove, or change how a user-reachable skill fits the flows, re-read `ask-me`'s `SKILL.md` and update it so the map stays accurate: a new skill it never mentions, or a stale one it still routes to, is a router that lies.

The Effect skills (`zen-of-ricky`, `effect-ts-conventions`) declare the official `effect-ts` skill from [Effect-TS/skills](https://github.com/Effect-TS/skills) as a prerequisite and never copy its content; every Effect snippet in them targets `effect@4` and is checked against the installed source before it is written.

To (re)link every skill outside `deprecated/` and `misc/` into the local harness skill directories (`~/.claude/skills`, `~/.agents/skills`), run `scripts/link-skills.sh`. Each entry is a symlink into this repo, so a `git pull` keeps installed skills current; re-run the script after adding, removing, or renaming a skill.

No em-dashes anywhere in this repo's prose (`SKILL.md` files, docs, `README.md`, `CHANGELOG.md`, ADRs, changesets, code comments). Where a sentence reaches for one, rewrite it instead with a comma, colon, period, parentheses, or a conjunction, whichever the sentence actually wants; never do a blind character substitution.
