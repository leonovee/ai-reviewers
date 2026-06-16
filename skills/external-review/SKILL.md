---
name: external-review
description: Shared contract for the external code-review subagents (Codex, Kimi, agy/Gemini-family, DeepSeek). Defines the severity scale, the trust model, defensive CLI-success detection, artifact naming, and the invocation contract. Every reviewer agent references this file — change it here once and all reviewers inherit it.
---

# External review — shared contract

This skill is the single source of truth for the machinery shared by every
external reviewer in this plugin. Each reviewer agent (`codex-reviewer`,
`kimi-reviewer`, `agy-reviewer`, `deepseek-reviewer`) embeds a short summary of
this contract and points back here for the full version. When something below
changes, change it HERE — the agents inherit it by reference.

The reviewers are **advisors, not authors**. They run an independent model over
a diff, capture its raw output verbatim into an audit artifact, and return a
structured summary. They never edit project source, never author fix-up
commits, and never invent output for a CLI run that did not actually produce it.

---

## 0. Max-model policy (ALWAYS use the strongest model)

**Every reviewer ALWAYS runs its strongest available model.** This is a hard
policy, not a per-call choice — review quality is the whole point, and review is
an infrequent, deliberate event, so cost/speed never justify a weaker model
here. The pinned model per reviewer:

| Reviewer | Always-on model | One-tier fallback (on reject/overload only) |
|----------|-----------------|---------------------------------------------|
| **codex** | `gpt-5.5-codex` (strongest codex) | CLI default (`-m` dropped) |
| **agy / Gemini** | `Gemini 3.1 Pro (High)` | `Gemini 3.5 Flash (High)` |
| **kimi** | `kimi-k2.7-code` + `--thinking` | `kimi-k2.6` (API path) |
| **deepseek** | `deepseek-v4-pro` (reasoning) | `deepseek-chat` |

Rules:

- **agy MUST stay in the Gemini family** — never pick Claude or GPT-OSS from
  `agy models`. The reviewer's value is model-family DIVERSITY (different family
  from the host session and from Codex); a Claude/GPT pick collapses it.
- A reviewer drops to its fallback **only** when the top model is rejected
  (unknown/unavailable) or overloaded (429/5xx) — and then emits a **verbatim
  fallback warning** in the artifact and the reply, so a degraded run is never
  silent.
- When a stronger model ships, bump the pin here (and in the reviewer's
  `## Model selection`) — this table is the single source of truth for "which
  model."

---

## 1. The reviews directory (`<reviews-dir>`)

Every artifact is written to a single per-project reviews directory, referred
to throughout as `<reviews-dir>`.

Resolution order (the agent picks the directory once, at the start of a run):

1. If the host project already has a reviews directory, use it. Look for, in
   order: `audit/external_reviews/`, `audit/reviews/`, `.reviews/`,
   `docs/reviews/`. Use the first that exists.
2. Otherwise default to `audit/external_reviews/` and create it
   (`mkdir -p`).

The reviews directory is a **runtime artifact directory** — it is gitignored
and its contents are NEVER committed. If the host project does not already
ignore it, the artifact still must not be added to a commit (trust invariant 1
below). On a fresh project, add a line such as `audit/external_reviews/*.md` to
`.gitignore` (do this only if asked — the reviewer's job is the review, not
repo housekeeping).

---

## 2. Severity scale (identical across all reviewers)

- **P0 / BLOCKING** — security regression, data-integrity violation, contract
  violation, prompt-injection vector.
- **P1 / HIGH** — incorrect logic, breaking change, missing test for a new code
  path.
- **P2 / MEDIUM** — spec drift (divergence from a documented contract / wording),
  naming inconsistency, anti-pattern.
- **P3 / LOW** — cosmetic, optional improvement, documentation gap.

Quote the model's own wording when summarizing a finding — do not paraphrase —
and attach a `file:line` reference where the model gave one.

The architect-facing verdict line is derived from the highest severity present:

- `NEEDS FIXES` — any P0 or P1 present.
- `NEEDS REVIEW` — P2 present (no P0/P1).
- `CLEAN` — P3 only, or no findings.

---

## 3. Trust model — four invariants

1. **Raw output goes to `<reviews-dir>`, never directly into commits.** The
   artifact is an audit record, not source. Do not `git add` it.
2. **Findings need human triage — reviewers do NOT propose or author fix-up
   commits.** Real-bug candidates become tasks for the human to schedule; the
   reviewer reports, it does not fix.
3. **CLI / API errors are reported VERBATIM. Never fabricate output, never
   substitute manual review for a failed CLI run.** The entire value of an
   external reviewer is a real independent invocation; faking it removes the
   value. If the tool did not run, say so with the exact error text.
4. **Recursion stop-loss: max 2 levels of fix-of-fix.** A fix of a fix of a fix
   is a redesign signal — surface "Level 3 fix-of-fix — redesign signal" to the
   human and STOP rather than continuing to patch.

---

## 4. Defensive success detection (MANDATORY for every CLI reviewer)

A headless CLI can exit `0` with EMPTY output (sandbox refusal, silent auth
drop, an internal error swallowed by the wrapper). Treat that as a **LOUD
FAILURE**, never a silent clean pass — a "no findings" artifact that was never
actually written by the model is worse than no artifact, because it looks like
a green review.

After every run, REQUIRE all of:

- the process exited **zero**, AND
- the captured output is **non-zero length**, AND
- the output contains **review content** — at least one severity tag matching
  `P[0-3]`, OR a required end-marker line `REVIEW_COMPLETE` (instruct the model
  to end its reply with that exact line so the check is reliable).

If any of these fail — empty output, missing marker, non-zero exit, or an
auth-prompt / OAuth URL appeared in stdout/stderr — do **NOT** write a "clean"
artifact. Emit a failure block with the verbatim stderr and return a non-zero
result to the human.

Reference check (bash):

```bash
if [ "$EXIT" -ne 0 ] || [ ! -s "$RAW_OUT" ] || ! grep -qE 'P[0-3]|REVIEW_COMPLETE' "$RAW_OUT"; then
  echo "LOUD FAILURE: <tool> returned no usable review"
  # surface RAW_ERR verbatim; do NOT fabricate findings; do NOT write a clean artifact
fi
```

Failure block template (fill the tool's name and the verbatim error):

```
<tool> review failed.

Command (exact, NO secret):
<command text>

Error (verbatim):
<paste exact stderr / response body>

Recommended action: <install the CLI | re-auth | correct the model id | check the API key env var | defer review | switch to another reviewer>
```

---

## 5. Artifact naming

```
<reviews-dir>/<tool>_<phase>_<UTC>.md
```

- `<tool>` — `codex` | `kimi` | `agy` | `deepseek`.
- `<phase>` — the phase / task id the human passed (free-form, e.g. `auth-refactor`,
  `1.4.2`, `pr-318`).
- `<UTC>` — compact ISO-8601 UTC, e.g. `20260615T2200Z`
  (`date -u +%Y%m%dT%H%MZ`).

Full example: `audit/external_reviews/codex_auth-refactor_20260615T2200Z.md`.

Artifact skeleton (every reviewer writes this shape):

```markdown
# <Tool> review — Phase <phase> — <UTC>

## Invocation

Tool / CLI version: <...>
Model used: <... or "CLI default (no -m)">
Audit type: <code-review | prompt-engineering | architectural | adversarial-test-brainstorm>
Command (exact, including flags; NO secret):
<command text>

## Commit range / files in scope

<range> (resolves to <SHA1>..<SHA2>) or <file list>

## Raw output

<verbatim model output — no editing>

## Summary (<tool>-reviewer subagent reading)

<severity-grouped findings — quote the model's wording, attach file:line>
```

---

## 6. Invocation contract

The human invokes a reviewer with:

- a **phase id** (free-form label for the artifact), and
- a **commit range** — `HEAD~1..HEAD`, or explicit `SHA1..SHA2`, or a single
  `SHA`, and
- optional **focus files** and **areas of concern**, and
- optional **audit type** (`code-review` | `prompt-engineering` |
  `architectural` | `adversarial-test-brainstorm`).

If the phase id or commit range is missing, ask once, then proceed. Do not
invent identifiers.

Build the review prompt deterministically and pass it to the CLI in a way that
survives shell quoting:

1. Compute the diff with `git diff <range>` (and, for new files in the range,
   `git show` / `git diff --stat` to enumerate them). For a whole-repo
   architectural audit, give the tool the tree directly (e.g. agy's
   `--add-dir <repo-root>`) and describe the scope in the prompt instead of
   pasting a diff.
2. Write the full prompt (task instructions + the diff text + "end your reply
   with the line `REVIEW_COMPLETE`") into a **temp file** (`mktemp`).
3. Feed the temp file to the CLI via stdin or `"$(cat "$PROMPT_FILE")"` — never
   as a fragile inline positional string. Embedding the diff in a temp file is
   what lets a large diff survive shell quoting and lets the marker-based
   success check work.

Prompt-template skeleton (tailor the task line to the audit type):

```
You are an independent <audit type> reviewer. Review the changes in commit
range <range> below. Focus: <correctness / security / contract adherence /
missing tests / prompt wording / architecture per audit type>.

Report findings grouped by severity P0/P1/P2/P3 with file:line references.
Do NOT propose or apply fixes — review only.

End your reply with the line REVIEW_COMPLETE.

Diff:
<diff text>
```

---

## 7. Auth model (per machine, no secrets in the plugin)

This plugin carries **no secrets**. Each tool authenticates per machine:

- **Codex** — subscription login (`codex login`), machine-local CLI state.
- **agy / Gemini-family** — subscription OAuth. No `agy login` subcommand — auth
  fires on first run; on a headless box/container use the **device-authorization**
  flow (agy prints an auth URL + one-time code; approve on another device). Token
  storage is version/OS-dependent (a file under `~/.gemini/antigravity-cli/`,
  and/or an encrypted `credentials.enc` / system keyring) — do not hard-assert one.
- **Kimi** — CLI login (`kimi login`) for the CLI path; `MOONSHOT_API_KEY` env
  var for the API fallback path.
- **DeepSeek** — `DEEPSEEK_API_KEY` env var.

### Configuration knobs (`.env.example`)

The plugin ships an `.env.example` listing every auth knob; the user copies the
ones they need into their shell profile (`~/.bashrc`) or a project `.env`. Two kinds:

- **API-key reviewers** — set the key to enable: `MOONSHOT_API_KEY` (Kimi),
  `DEEPSEEK_API_KEY` (DeepSeek).
- **Subscription reviewers** — no key; they auth via their own CLI login/OAuth. A
  flag tells the plugin the user HAS that subscription, so it runs that reviewer and
  skips the ones they don't have: `OPENAI_SUBSCRIPTION=true` (Codex, after
  `codex login`), `GOOGLE_SUBSCRIPTION=true` (agy/Gemini, after the agy device-auth).

**Skip-if-unconfigured (mandatory):** before invoking, a reviewer checks its own
knob — its API-key env var, or its subscription flag (`OPENAI_SUBSCRIPTION` for
codex, `GOOGLE_SUBSCRIPTION` for agy). If the knob is absent/false, the reviewer
reports **"not configured — set `<KNOB>` (see `.env.example`)"** and exits cleanly,
rather than running the CLI and failing cryptically. A configured-but-rejected auth
(the CLI prints a login/OAuth URL at run time) is still the STOP below.

NEVER echo a key, NEVER print a token file. On a new machine: install the
plugin, set the knobs once (copy `.env.example`), redo each CLI's login. If a tool
prints an auth prompt / OAuth URL instead of a review, that is a STOP — surface
it verbatim (re-auth needs a human / a browser); do not attempt to auth on the
human's behalf.

---

## 8. Picking a reviewer

- **Codex** — primary per-commit / per-phase code reviewer. Focused on code
  defects: security, type errors, logic.
- **Kimi** — second independent code reviewer (different model family from
  Codex, so it catches what same-model review misses) and adversarial
  test-idea brainstorm.
- **agy / Gemini-family** — prompt-engineering audits and whole-repo
  architectural sweeps.
- **DeepSeek** — optional fourth reviewer; a deep reasoning pass that
  complements the CLI reviewers.

Run several in parallel and cross-reference findings — do not merge their
voices into one. Each reviewer writes its own artifact.
