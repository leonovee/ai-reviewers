# AI Reviewers

A project-agnostic Claude Code plugin that bundles four **independent external
code-review subagents** plus one shared review skill. Install it once per
machine and use the same reviewers across every project, instead of
copy-pasting `.claude/agents/` into each repo.

The reviewers are **advisors, not authors.** Each one runs an independent model
over a diff, captures its raw output verbatim into an audit artifact, and
returns a severity-tagged summary. They never edit your source, never author
fix-up commits, and never invent output for a CLI run that did not actually
produce it.

## Always the strongest model

**Every reviewer ALWAYS runs its strongest available model** — review quality is
the whole point, and review is an infrequent, deliberate event, so cost/speed
never justify a weaker model here. The pins:

| Reviewer | Always-on model | Fallback (only on reject/overload, with a loud warning) |
|----------|-----------------|----------------------------------------------------------|
| **codex** | `gpt-5.5-codex` | CLI default |
| **agy / Gemini** | `Gemini 3.1 Pro (High)` (always a Gemini — never Claude/GPT-OSS via agy) | `Gemini 3.5 Flash (High)` |
| **kimi** | `kimi-k2.7-code` + `--thinking` | `kimi-k2.6` |
| **deepseek** | `deepseek-v4-pro` | `deepseek-chat` |

A reviewer steps down to its fallback only when the top model is rejected or
overloaded, and says so verbatim — a degraded run is never silent. The single
source of truth for these pins is the `external-review` skill (§0).

## What's inside

```
ai-reviewers/                  ← repo root = the marketplace AND the plugin
  .claude-plugin/
    marketplace.json           ← lists this plugin (source ".")
    plugin.json                ← the plugin manifest
  agents/
    codex-reviewer.md
    kimi-reviewer.md
    agy-reviewer.md
    deepseek-reviewer.md
  skills/
    external-review/SKILL.md   ← the shared contract (severity scale, trust
                                  model, defensive success detection, artifact
                                  naming, invocation contract)
  README.md
```

The shared `external-review` skill is the single source of truth for the
machinery every reviewer inherits. Each agent file adds only its tool-specific
invocation and points back to the skill for everything else — so you change a
rule once, in the skill, and all four reviewers follow.

## The four reviewers — when to use each

| Reviewer | Tool | Use it for |
|----------|------|-----------|
| **codex-reviewer** | `codex` CLI | **Primary per-commit / per-phase code review.** Focused code defects: security, type errors, logic. |
| **kimi-reviewer** | Kimi CLI → Moonshot API | **Second independent code reviewer** (different model family from Codex, so it catches what same-model review misses) and **adversarial test-idea brainstorm.** |
| **agy-reviewer** | Antigravity `agy` CLI (Gemini-family) | **Prompt-engineering audits** and **whole-repo architectural sweeps.** |
| **deepseek-reviewer** | DeepSeek API | **Optional fourth reviewer** — a deep reasoning pass that complements the CLI reviewers. |

Run several in parallel and cross-reference their findings — keep each
reviewer's voice (and artifact) separate; do not merge them.

## How a review runs

You invoke a reviewer with a phase id + a commit range
(e.g. `HEAD~1..HEAD` or explicit SHAs) and optional focus files. The reviewer
builds the prompt from `git diff <range>` into a temp file, runs the tool
headless, and writes the result to:

```
<reviews-dir>/<tool>_<phase>_<UTC>.md
```

`<reviews-dir>` defaults to `audit/external_reviews/`, but the reviewer uses
whatever reviews directory your project already has (it looks for an existing
one first). The artifact is a runtime audit record — **gitignored, never
committed.**

A headless CLI can exit `0` with empty output; every reviewer treats that as a
**loud failure**, not a silent clean pass — it requires real review content (a
`P0`–`P3` severity tag or a `REVIEW_COMPLETE` end-marker) before it will write
an artifact, and surfaces any CLI error verbatim.

## Auth model — in plain terms

**The plugin carries NO secrets.** Each tool authenticates per machine:

- **Codex** — subscription login (`codex login`).
- **agy / Gemini-family** — subscription OAuth. No `agy login` subcommand — auth
  fires on first run; on a headless box/container it is the **device-authorization**
  flow (agy prints an auth URL + one-time code, approved on another device).
- **Kimi** — CLI login (`kimi login`) for the CLI path; `MOONSHOT_API_KEY` env
  var for the API fallback.
- **DeepSeek** — `DEEPSEEK_API_KEY` env var.

So on a **new machine**: install the plugin, then redo the CLI logins / set the
API-key env vars **once.** The plugin itself never holds a key or a token. If a
tool prints an auth prompt or an OAuth URL instead of a review, that is a STOP —
the reviewer surfaces it verbatim and you re-auth (re-auth needs a browser /
your own terminal).

## Configuration (`.env.example`)

Copy **`.env.example`** and fill in only the reviewers you use. Two kinds of knobs:

- **API-key reviewers** — set the key to enable: `MOONSHOT_API_KEY` (Kimi),
  `DEEPSEEK_API_KEY` (DeepSeek).
- **Subscription reviewers** — no key; they auth via their own CLI login/OAuth.
  Set a flag so the plugin knows you have the subscription (and skips the ones you
  don't): `OPENAI_SUBSCRIPTION=true` (Codex, after `codex login`),
  `GOOGLE_SUBSCRIPTION=true` (agy/Gemini, after the agy device-auth).

A reviewer whose key/flag is absent reports **"not configured — set X"** and is
skipped cleanly, rather than failing mid-run. Put the values in your shell profile
(`~/.bashrc`) or a project `.env` and `source` it before launching Claude Code
(child processes inherit the environment at launch).

## Install

This repository is itself the marketplace (both `.claude-plugin/marketplace.json`
and `.claude-plugin/plugin.json` live at the repo root). Install straight from
GitHub:

```
/plugin marketplace add leonovee/ai-reviewers
/plugin install ai-reviewers@ai-reviewers-marketplace
```

(You can also add it from a local checkout: `/plugin marketplace add <path-to-this-repo>`.)

After installing, the four reviewer subagents and the `external-review` skill
are available in every project on that machine. Do the per-machine CLI
logins / API-key setup once (see the auth model above).
