# Magic Art — agent skill

Turn an idea into a finished visual with [Magic Art](https://magic-design.art): AI posters,
social card decks (Xiaohongshu-style multi-page notes), portraits and image series — then
download the results to your machine. Your agent drives the whole thing through the `magic`
CLI, including the human checkpoints (clarify questions, template picks, regional edits).

用 Magic Art 把一个想法做成成品视觉作品：AI 海报、小红书图文卡片、写真、系列图，
并把成图下载到本地。全程由 agent 通过 `magic` CLI 完成。

## Install

### The canonical way — npm (recommended)

```bash
npm i -g @magic-art/cli && magic skill install
```

`magic skill install` writes **one** copy to `~/.agents/skills/magic-art/` and symlinks it
into every agent runtime it finds on the machine (`~/.claude/skills/`, `~/.codex/skills/`, …).
Upgrading the CLI upgrades the skill for every agent at once:

```bash
npm i -g @magic-art/cli@latest && magic skill install
```

### Claude Code plugin

```
/plugin marketplace add FutrixDev/magic-art-skill
/plugin install magic-art@magic-art
```

### ClawHub / OpenClaw

```bash
clawhub install magic-art
```

### skills.sh (Cursor, Codex, Copilot, Gemini, Zed, …)

```bash
npx skills add FutrixDev/magic-art-skill
```

## Source of truth

**This repository is a mirror, not the source.** The skill files ship inside the
[`@magic-art/cli`](https://www.npmjs.com/package/@magic-art/cli) npm package so that the prose
telling an agent "run `magic generate --wait`" is always the same version as the binary that
implements it. `.github/workflows/sync-from-npm.yml` re-publishes `skill/` from the published
tarball, so `skills/magic-art/` here is byte-identical to some released CLI version — never
hand-edited.

The practical consequence: if you installed through a hub and the CLI has since moved on, the
skill's first step re-syncs the canonical copy from npm. Whichever channel you came in
through, `npm i -g @magic-art/cli@latest && magic skill install` is what makes every agent on
the machine current.

## Requirements

- Node.js 20+, macOS / Windows / Linux
- A [magic-design.art](https://magic-design.art) account — `magic login` uses the device
  authorization flow (RFC 8628), so the CLI never sees your password
- Magic points (魔法点数). Generation costs credits; the skill is written to quote
  (`magic billing quote`) before spending and to stop and ask you when the balance is short.
  It will never buy, upgrade, publish or delete on your behalf.

## What the skill covers

| File | What is in it |
| --- | --- |
| [`SKILL.md`](skills/magic-art/SKILL.md) | Auth/quote preflight, mode selection, the create → clarify → generate → download loop, regional annotated edits (`--mark` / `--mark-ui`), who-answers-which-checkpoint policy, hard rules |
| [`references/modes.md`](skills/magic-art/references/modes.md) | Every creation mode, what "count" means in each, bounds, input requirements |
| [`references/recipes.md`](skills/magic-art/references/recipes.md) | End-to-end command sequences with expected output |
| [`references/troubleshooting.md`](skills/magic-art/references/troubleshooting.md) | Exit-code contract, rate limits, expired tokens, `awaiting_human` reply templates, `job resume` decision tree |

The skill is prose only — no scripts, no binaries, no network calls of its own. Everything it
does goes through the `magic` CLI, which owns auth headers, idempotent `operation_id`s, poll
cadence and the exit-code contract.

## License

[MIT-0](LICENSE) — use, modify and redistribute freely, no attribution required. The `magic`
CLI itself and the Magic Art service are separately licensed.
